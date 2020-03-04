# PSP Demo

## 00 - Prepare permissive-binding

```
kubectl apply -f sa-privileged.yaml
```

## Demo 01: No PSP enabled - danger

Folder: 01_no_psp_enabled

```
$ bat deploy-dangerous-pod.yaml
$ kubectl apply -f deploy-dangerous-pod.yaml
$ kubectl attach -ti deploy/dangerous-deploy
```

We are in a container, but with our *own* mount namespace. `docker ps` does not work, but `ps -ef` or `ifconfig` show host PIDs and network interfaces, so we can do:

```
# nsenter --mount=/proc/1/ns/mnt -- /bin/bash
```

Now we are in the host as root. We can check /etc/kubernetes/manifests, docker ps, whatever

Let's delete the pod before going on:

```
$ kubectl delete  -f deploy-dangerous-pod.yaml --grace-period=0 --force=true
```

## Demo 02: Enable PSP + naive example

Folder: 02_psp_enabled

### Enable PSP

Let's enable PSP

In Rancher, we can use the UI, edit the cluster, and mark PSP enabled, default PSP *restricted*.

For minikube we can do:

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

Will work out of the box, as we told rancher to enable PSP with restricted security policy, which allows non-privileged pods like this to run with no issues.

You can check what is exactly in the default PSP with:

```
$ kubectl get psp restricted-psp -o yaml
```

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


Let's remove the pods before moving on:

```
$ kubectl delete -f deploy-not-privileged.yaml
$ kubectl delete -f deploy-privileged.yaml
```

## Demo 03: PSPs with RBAC

Folder: 02_psp_enabled

Let's create a new namespace now, instead of using default, and deploy the same pod in there:

```
$ kubectl create ns psp-test
$ kubectl -n psp-test apply -f deploy-not-privileged.yaml
```

But it is failing:

```
$ kubectl -n psp-test describe rs
...
  Warning  FailedCreate  4s (x12 over 15s)  replicaset-controller  Error creating: pods "not-privileged-deploy-684696d5b5-" is forbidden: unable to validate against any pod security policy: []
```

As we will learn later, namespaces that are managed under a Rancher project get some RoleBindings automatically created to make using PSPs easier. But as we created this namespace manually (as in any kubernetes cluster not managed with Rancher), we need to manually create it.

Let's create a ClusterRole and a binding to allow USE verb on the the "restricted" PSP:

```
$ bat clusterrole-use-restricted.yaml
$ kubectl apply -f clusterrole-use-restricted.yaml
```

Now our deployment should work.

### Privileged pod

Trying to create a privileged deployment shouldn't work:

```
$ bat deploy-privileged.yaml
$ kubectl -n psp-test apply -f deploy-privileged.yaml
```

the deploy is unable to spawn the pod, and the replica-set that it created complains:

```
$ kubectl -n psp-test describe rs privileged-deploy-7569b9969d
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

Rancher already created a "default-psp" allowing privileged pods:
```
$ kubectl get psp default-psp -o yaml
```

so we just need to create the corresponding ClusterRole and RoleBinding to provide access to the pod service account (privileged-sa)

```
$ bat clusterrole-use-privileged.yaml
$ kubectl -n psp-test apply -f clusterrole-use-privileged.yaml
```
And now our deployment should work.

## Demo 04: Sysdig Secure

* Login to Sysdig Secure (audit log needs to be enabled for the cluster)
* Go to Policies -> Pod Security Policies
* New simulation
* Import 03_rbac/deploy-not-privileged.yaml
* It should generate a minimal PSP
* Change the PSP name to something meaningful
* Choose "default" as namespace
* Start Simulation
* Now run `kubectl apply -f deploy-privileged.yaml`
* After a few seconds, a new event should be triggered detecting the violation (the pod would violate the PSP)


# Demo 05: Rancher

* When namespaces are created inside projects in Rancher, RoleBindings for the default PSP are created automatically

---------
TMP notes

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

