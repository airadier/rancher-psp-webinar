# PSP Demo

## Prepare permissive-binding

```
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts \
  --group=system:authenticated
```

## Demo: No PSP enabled - danger

Folder: 01_no_psp_enabled

```
$ bat pod-privileged.yaml
$ kubectl create -f pod-privileged.yaml
$ kubectl attach -ti pod sudo-test
```

One-liner:

```
$ kubectl run no-psp-test --restart=Never -it \
    --image overriden --overrides '
  "spec": {
    "hostPID": true,
    "hostNetwork": true,
    "containers": [
      {
        "name": "busybox",
        "image": "alpine:3.7",
        "command": ["/bin/sh"],
        "stdin": true,
        "tty": true,
        "securityContext": {
          "privileged": true
        }
      }
    ]
  }
}' --rm --attach
```

We show we are in a chroot, `docker ps`does not work, but `ps -ef` or `ifconfig` show host PIDs and network interfaces, so we can do:


```
# nsenter --mount=/proc/1/ns/mnt -- /bin/bash
```

Now we are in the host as root. We can check /etc/kubernetes/manifests, docker ps, whatever

## Demo: Enable PSP + naive example

Folder: 02_psp_enabled

Let's enable PSP

```
$ minikube ssh
$ sudo su
$ cd /etc/kubernetes/manifests
$ vi kube-apiserver.yaml
...  Add PodSecurityPolicy at the end of --enable-admission-plugins option
```

Now let's try to create a standard Pod:

```
$ bat pod-not-privileged.yaml
$ kubectl create -f pod-not-privileged.yaml
Error from server (Forbidden): error when creating "pod-not-privileged.yaml": pods "not-privileged" is forbidden: no providers available to validate pod request
```

We have not created any PSP yet, let's create one:

```
$ bat psp-restricted.yaml
$ kubectl create -f psp-restricted.yaml
```

Now creating the non-privileged pod will work:
```
$ kubectl create -f pod-not-privileged.yaml
pod/not-privileged created
```


However, trying to create the privileged pod fails:

```
$ bat pod-privileged.yaml
$ kubectl create -f pod-privileged.yaml
Error from server (Forbidden): error when creating "pod-privileged.yaml": pods "sudo-test" is forbidden: unable to validate against any pod security policy: [spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.securityContext.hostPID: Invalid value: true: Host PID is not allowed to be used spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
```

No existing PSPs allow for "hostPID", "hostNetwork" or "privileged"

Now, let's create a "privileged" PSP:

```
$ bat psp-privileged.yaml
$ kubectl create -f psp-privileged.yaml
```

And now we can create the privileged pod:

```
$ bat pod-privileged.yaml
$ kubectl create -f pod-privileged.yaml
$ kubectl attach -ti pod sudo-test
```

that now succeeds, as there is a PSP allowing hostPID, hostNetwork and privileged.

If we describe the pod, we know which PSP was applied from the annotations:

```
$ kubectl describe po sudo-test
...
Annotations:  kubernetes.io/psp: privileged
...
$ kubectl describe po not-privileged
...
Annotations:  kubernetes.io/psp: restricted
...
```

However, what if we destroy and create the not-privileged pod again?

```
$ kubectl delete po not-privileged
$ kubectl create  -f pod-not-privileged.yaml
$ kubectl describe po not-privileged
...
Annotations:  kubernetes.io/psp: privileged
...
```
Wow! What happened? In short: when there are multiple PSPs allowing a Pod, the first one (under certain criteria) will be applied.

But what are the rules? And how do we map which pods can use which PSPs? Let's move on

Let's remove the pods before moving on:

```
$ kubectl delete po not-privileged sudo-test
```

## Demo: PSPs with RBAC

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
$ kubectl -n kube-system delete po kube-apiserver-minikube
$ kubectl -n kube-system get po
```

Now even kube-apiserver-minikube is unable to start. If we check kubelet logs, we would see something like:

```
$ sudo systemctl status kubelet
...
Mar 03 11:35:04 sudo--minikube kubelet[2373]: E0303 11:35:04.087071    2373 kubelet.go:1664] Failed creating a mirror pod for "kube-apiserver-minikube_kube-system(8c8ab4bb9206ea9479beced12bca99dc)": pods "kube-apiserver-minikube" is forbidden: unable to validate against
any pod security policy: []
...
```

Logs show that kube-apiserver-minikube pod in kube-system is not able to start after we enabled PSPs. 

Here is we you need to start being careful.

To fix this, we need to allow the user system:node:minikube (the user which kubelet authenticates as) to use privileged PSP

```
$ bat role-use-privileged-psp.yaml
$ kubectl create -f role-use-privileged-psp.yaml
$ bat rolebinding-privileged-role-bind.yaml
$ kubectl create -f rolebinding-privileged-role-bind.yaml
```

Ok, and now apiserver works, but creating our non-privileged pod should fail, right? As it does not have access to any PSP:

```
$ kubectl create -f pod-not-privileged.yaml
```

But it works! why?

In our kubeconfig, we are using Certificate Authentication for minikube:

```
...
users:
- name: minikube
  user:
    client-certificate: /Users/airadier/.minikube/client.crt
...
```

and if we check the certificate Subject:

```
$ cat /Users/airadier/.minikube/client.crt | openssl x509 -text                                                                                               (⎈ minikube|default)
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2 (0x2)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=minikubeCA
        Validity
            Not Before: Feb 29 20:43:38 2020 GMT
            Not After : Mar  1 20:43:38 2021 GMT
        Subject: O=system:masters, CN=minikube-user
        Subject Public Key Info:
...
```

so we are using user *minikube-user* in group *system:masters* which is binded to cluster-admin ClusterRole via cluster-admin ClusterRoleBinding, allowing us to USE any PSP on creation.

Let's do it more real by creating a deploy for the Pod:

```
$ kubectl delete po not-privileged
$ bat deploy-not-privileged.yaml
$ kubectl create -f deploy-not-privileged.yaml
```

Now the pod is not being created... let's describe the deployment, and the replicaset that the deployment creates:

```
$ k describe rs not-privileged-deploy-684696d5b5                                                                                                              (⎈ minikube|default)
Name:           not-privileged-deploy-684696d5b5
Namespace:      default
...
Events:
  Type     Reason        Age                  From                   Message
  ----     ------        ----                 ----                   -------
  Warning  FailedCreate  34s (x15 over 116s)  replicaset-controller  Error creating: pods "not-privileged-deploy-684696d5b5-" is forbidden: unable to validate against any pod security policy: []
```

So let's create a clusterrolebinding to allow any service account to use the "not-privileged" PSP:

```
$ bat sa-privileged.yaml
$ kubectl create -f sa-privileged.yaml
$ bat deploy-privileged-with-sa.yaml
$ kubectl create -f deploy-privileged-with-sa.yaml
```


```
$ bat clusterrole-use-restricted.yaml
$ kubectl create -f clusterrole-use-restricted.yaml
```

and then create a clusterrolebinding to allow restricted (not privileged) pods by default using any service account:

```
$ bat clusterrolebinding-restricted-role-bind.yaml
$ kubectl create -f clusterrolebinding-restricted-role-bind.yaml
```

Now our deployment should work, but trying to create a privileged deployment shouldn't:

```
$ bat deploy-privileged.yaml
$ kubectl create -f deploy-privileged.yaml
```

the replicaset complains:

```
$ kubectl describe rs privileged-deploy-7569b9969d                                                                                                                  (⎈ minikube|default)
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

unless we explicitely allow. We will create a special serviceaccount "privileged-sa" and create a rolebinding for that account

```
$ bat clusterrole-user-privileged.yaml
$ kubectl create -f clusterrole-user-privileged.yaml
$ bat rolebinding-privileged-role-bind-default-ns.yaml
$ kubectl create -f rolebinding-privileged-role-bind-default-ns.yaml
```

And now our deployment should work
