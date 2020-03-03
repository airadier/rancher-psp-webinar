# PSP Demo

## 00 - Prepare permissive-binding

```
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts \
  --group=system:authenticated
```

## Demo 01: No PSP enabled - danger

Folder: 01_no_psp_enabled

```
$ bat deploy-privileged.yaml
$ kubectl apply -f deploy-privileged.yaml
$ kubectl attach -ti deploy privileged-deploy
```

We are in a container, but with our *own* mount namespace. `docker ps`does not work, but `ps -ef` or `ifconfig` show host PIDs and network interfaces, so we can do:

```
# nsenter --mount=/proc/1/ns/mnt -- /bin/bash
```

Now we are in the host as root. We can check /etc/kubernetes/manifests, docker ps, whatever

Let's delete the pod before going on:

```
$ kubectl delete  -f deploy-privileged.yaml --grace-period=0 --force=true
```

## Demo 02: Enable PSP + naive example

Folder: 02_psp_enabled

### Enable PSP

Let's enable PSP

```
$ minikube ssh
$ sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
...  Add PodSecurityPolicy at the end of --enable-admission-plugins option
```

Now let's try to create a standard Pod:

### Non-privileged pod

```
$ bat deploy-not-privileged.yaml
$ kubectl apply -f deploy-not-privileged.yaml
```

The pod cannot be created, check the deploy and the replicaset events:

```
$ kubectl describe deploy not-privileged-deploy
...
  Normal  ScalingReplicaSet  53s   deployment-controller  Scaled up replica set not-privileged-deploy-684696d5b5 to 1

$ kubectl describe rs not-privileged-deploy-684696d5b5
...
  Warning  FailedCreate  16s (x15 over 98s)  replicaset-controller  Error creating: pods "not-privileged-deploy-684696d5b5-" is forbidden: no providers available to validate pod request

```

The reason is we have not created any PSP yet, let's create one:

```
$ bat psp-restricted.yaml
$ kubectl apply -f psp-restricted.yaml
```

Now if we wait a bit, the pod will be created (we can delete/create the deploy)

### Privileged pod

However, trying to create the privileged pod fails:

```
$ bat deploy-privileged.yaml
$ kubectl apply -f deploy-privileged.yaml
```

and again, if we check the status of the replicaset:

```
$ kubectl describe rs privileged-deploy-7569b9969d
...
  Warning  FailedCreate  2s (x12 over 13s)  replicaset-controller  Error creating: pods "privileged-deploy-7569b9969d-" is forbidden: unable to validate against any pod security policy: [spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.securityContext.hostPID: Invalid value: true: Host PID is not allowed to be used spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
```

No existing PSPs allow for "hostPID", "hostNetwork" or "privileged"

Now, let's create a "privileged" PSP:

```
$ bat psp-privileged.yaml
$ kubectl apply -f psp-privileged.yaml
```

Now if we wait a bit, the pod will be created (we can delete/create the deploy)

If we describe the pod, we know which PSP was applied from the annotations:

```
$ kubectl describe po privileged-deploy-7569b9969d-78j75 | grep psp
Annotations:  kubernetes.io/psp: privileged
$ kubectl describe po not-privileged-deploy-684696d5b5-hbmn7 | grep psp
Annotations:  kubernetes.io/psp: restricted
```

However, what if we destroy and create the not-privileged pod again?

```
$ kubectl delete po not-privileged-deploy-684696d5b5-hbmn7 --grace-period=0 --force=true
...
$ kubectl describe po not-privileged-deploy-684696d5b5-sn9tf | grep psp
Annotations:  kubernetes.io/psp: privileged
```
Wow! What happened? In short: when there are multiple PSPs allowing a Pod, the first one (under certain criteria) will be applied.

But what are the rules? And how do we map which pods can use which PSPs? Let's move on

Let's remove the pods before moving on:

```
$ kubectl delete -f deploy-not-privileged.yaml
$ kubectl delete -f deploy-privileged.yaml
```

## Demo 03: PSPs with RBAC

Folder: 03_rbac

The previous example was not real, because we have this:

```
$ kubectl describe clusterrolebinding permissive-binding
Name:         permissive-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name                    Namespace
  ----   ----                    ---------
  User   admin
  User   kubelet
  Group  system:authenticated
  Group  system:serviceaccounts
```

Let's remove it and see what happens:

```
$ kubectl delete clusterrolebinding permissive-binding
```

### Non-privileged pod

And now let's try to create a non-privileged pod:

```
$ bat deploy-not-privileged.yaml
$ kubectl apply -f deploy-not-privileged.yaml
```

Now the pod is not being created... let's describe the deployment, and the replicaset that the deployment creates:

```
$ k describe rs not-privileged-deploy-684696d5b5
Name:           not-privileged-deploy-684696d5b5
Namespace:      default
...
Events:
  Type     Reason        Age                  From                   Message
  ----     ------        ----                 ----                   -------
  Warning  FailedCreate  34s (x15 over 116s)  replicaset-controller  Error creating: pods "not-privileged-deploy-684696d5b5-" is forbidden: unable to validate against any pod security policy: []
```

So let's create a ClusterRole to allow USE verb on the the "restricted" PSP:

```
$ bat clusterrole-use-restricted.yaml
$ kubectl apply -f clusterrole-use-restricted.yaml
```

and another ClusterRole to allow USE verb on the "privileged" PSP:

```
$ bat clusterrole-use-privileged.yaml
$ kubectl apply -f clusterrole-use-privileged.yaml
```

For the moment, let's create a ClusterRoleBinding to allow restricted (not privileged) pods by default using any service account:

```
$ bat clusterrolebinding-restricted-role-bind.yaml
$ kubectl apply -f clusterrolebinding-restricted-role-bind.yaml
```

Now our deployment should work

### Privileged pod

Trying to create a privileged deployment shouldn't work:

```
$ bat deploy-privileged.yaml
$ kubectl apply -f deploy-privileged.yaml
```

the deploy is unable to spawn the pod, and the replica-set that it created complains:


```
$ kubectl describe rs privileged-deploy-7569b9969d
Name:           privileged-deploy-7569b9969d
Namespace:      default
Selector:       app=privileged-deploy,pod-template-hash=7569b9969d
Labels:         app=privileged-deploy
...
Events:
  Type     Reason        Age                From                   Message
  ----     ------        ----               ----                   -------
  Warning  FailedCreate  4s (x14 over 45s)  replicaset-controller  Error creating: pods "privileged-deploy-7569b9969d-" is forbidden: unable to validate against any pod security policy: [spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.securityContext.hostPID: Invalid value: true: Host PID is not allowed to be used spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
```

The existing PSP (restricted) does not allow hostPID, hostNetwork or privileged.

We will create a special service account "privileged-sa" and create a rolebinding for that account

```
$ bat sa-privileged.yaml
$ kubectl apply -f sa-privileged.yaml
$ bat deploy-privileged-with-sa.yaml
$ kubectl apply -f deploy-privileged-with-sa.yaml
$ bat rolebinding-privileged.yaml
$ kubectl apply -f rolebinding-privileged.yaml
```

And now our deployment should work

## Demo 04: Sysdig Secure
