# Pod security

##  Appropriate Pod Security Standards policy is applied for all namespaces and enforced.

Privileges and access control can be granted or denied to Pods with setting the security context.

Security context settings include adding/removing Linux capabilities, setting UID, setting GID, setting the AppArmor profile used by the container, setting the seccomp profile to filter system calls, running container in privileged mode (container's process runs equivalent to root on the host), and more.

In order to restrict elevated privileges or access for Pods, Kubernetes has security policies.

There are three security policies defined by Pod Security Standards::
- `privileged`: unrestricted and open
- `baseline`: minimally restrictive policy to prevent known privilege escalations
- `restricted`: follows Pod hardening best practices 

Pod Security Admission is a validating admission controller that enforces Pod Security Standards at the Namespace level.

Pod Security Admission enforces Pod Security Standards in three modes: 
- `enforce`: violations are rejected
- `audit`: violations are allowed but noted in teh audit log
- `warn`: violations are allowed but user receives a warning

Policies are applied to a Namespace with labels.
e.g. The Namespace label `pod-security.kubernetes.io/warn: baseline` applies the `baseline` policy in the `warn` mode.

Pod Security Standard versions can also be specificed with a label and the default is latest.
e.g. `pod-security.kubernetes.io/enforce-version=v1.25` and `pod-security.kubernetes.io/enforce: baseline` pins the `baseline` policy to v1.25 of the policy.

In this lab, we will first use the `baseline` policy in `warn` mode then in `enforce` mode for Pods in the `secure-namespace` Namespace.
The baseline policy enforces the following:
- host namespaces are disallowed
- privileged containers are disallowed
- only specific Linux capabilities are allowed
- host ports are disallowed or restricted
- if hosts support AppArmor, the RuntimeDefault AppArmor profile is applied 
- setting the SELinux type is restricted
- the default /proc masks are setup to reduce attack surface
- Seccomp profile must not be set to Unconfined and restricted
- Use of sysctls security context is restricted

More than one Pod Security Policy can be applied to the same Namespace. Later in the lab, we'll use the `restricted` policy in `warn` mode.

For more details on what is allowed or required with the `baseline` and `restricted` Pod Security Standards, checkout the [Pod Security Standards page on kubernetes.io](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

1. Create the `secure-namespace` Namespace:

```shell
kubectl create namespace secure-namespace
```

Expected output:
```shell
namespace/secure-namespace created
```

2. Label the `secure-namespace` Namespace so that the `baseline` Pod Security Standard is enforced in `warn` mode:

```shell
kubectl label ns secure-namespace pod-security.kubernetes.io/warn=baseline
```
Expected output:
```shell
namespace/secure-namespace  labeled
```

3. Create a Pod that violates the `baseline` policy.
Create a Pod with a container that runs in privilege mode (the container's process is equivalant to root on the host):

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: secure-namespace 
spec:
  containers:
  - name: nginx
    image: nginx:1.27.0
    ports:
    - containerPort: 80
    securityContext:
      privileged: true
EOF
```

Expected output:
```shell
Warning: would violate PodSecurity "baseline:latest": privileged (container "nginx" must not set securityContext.privileged=true)
pod/nginx created
```

There is a warning of the `baseline` policy violation.
Since the `baseline` policy is in `warn` mode, the Pod is created.

4. Verify the creation of the Pod:

```shell
kubectl get pods -n secure-namespace
```

Expected output:
```shell
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          86s
```

5. Delete the `nginx` Pod in the `secure-namespace` Namespace:

```shell
kubectl delete pod nginx -n secure-namespace
```

Expected output:
```shell
pod "nginx" deleted
```

6. Add the enforcement mode of the `baseline` policy to the `enforce` level of the `secure-namespace` Namespace:

```shell
kubectl label --overwrite namespace secure-namespace pod-security.kubernetes.io/enforce=baseline
```

Expected output:
```shell
namespace/secure-namespace labeled
```

7. Create the same Pod which violates the `baseline` policy to test the `enforce` mode:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: secure-namespace 
spec:
  containers:
  - name: nginx
    image: nginx:1.27.0
    ports:
    - containerPort: 80
    securityContext:
      privileged: true
EOF
```

Expected output:
```shell
Error from server (Forbidden): error when creating "STDIN": pods "nginx" is forbidden: violates PodSecurity "baseline:latest": privileged (container "nginx" must not set securityContext.privileged=true)
```

8. Check the Pods in the `secure-namespace` Namespace:

```shell
kubectl get pods -n secure-namespace
```

Expected output:
```shell
No resources found in secure-namespace namespace.
```

Great, the Pod that violates the `baseline` policy was not created.

9. Create a Pod that satisfies the `baseline` policy by setting privileged mode in the security context to false:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: secure-namespace
spec:
  containers:
  - name: nginx
    image: nginx:1.27.0
    ports:
    - containerPort: 80
    securityContext:
      privileged: false
EOF
```

Expected output:
```shell
pod/nginx created
```
Great, no warning

```shell
kubectl get pods -n secure-namespace
```

Expected output:
```shell
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          74s
```

10. Delete the Pod:

```shell
kubectl delete pod nginx -n secure-namespace
```

Expected output:
```shell
pod "nginx" deleted
```

11. Label the `secure-namespace` Namespace so that the `restricted` policy is enforced in `warn` mode.

```shell
kubectl label namespace secure-namespace pod-security.kubernetes.io/warn=restricted --overwrite
```

Expected output:
```shell
namespace/secure-namespace labeled
```

12. Check the labels on the `secure-namespace` Namespace:

```shell
kubectl get namespace secure-namespace --show-labels
```

Expected output:
```shell
NAME               STATUS   AGE   LABELS
secure-namespace   Active   27m   kubernetes.io/metadata.name=secure-namespace,pod-security.kubernetes.io/enforce=baseline,pod-security.kubernetes.io/warn=restricted
```

The `secure-namespace` Namespace has two Pod Security Standards policies applied:
- `baseline` policy in `enforce` mode
- `restricted` policy in `warn` mode

13. Create a Pod that satisfies the `baseline` policy but not the `restricted` policy:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: secure-namespace
spec:
  containers:
  - name: nginx
    image: nginx:1.27.0
    ports:
    - containerPort: 80
    securityContext:
      privileged: false
EOF
```

Expected output:
```shell
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/nginx created
```

Looks like the Pod was created but we received a warning that the Pod violates the `restricted` policy.

14. Check the Pods in the `secure-namespace` Namespace:

```shell
kubectl get pods -n secure-namespace
```

Expected output:
```shell
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          70s
```


15. Delete the Pod:

```shell
kubectl delete pod nginx -n secure-namespace
```

Expected output:

```shell
pod "nginx" deleted
```

16. Before enforcing a policy to Namespaces, we want to check if the policy may disrupt running workloads. We can use the `--dry-run` and `--all` options to see if we were to enforce the `restricted` policy.

```shell
kubectl label --dry-run=server --overwrite ns --all pod-security.kubernetes.io/enforce=restricted
```

Expected output (your output will vary depending on Namespaces):
```shell
namespace/default labeled (server dry run)
namespace/kube-node-lease labeled (server dry run)
namespace/kube-public labeled (server dry run)
Warning: existing pods in namespace "kube-system" violate the new PodSecurity enforce level "restricted:latest"
Warning: cilium-c7twv (and 1 other pod): forbidden AppArmor profiles, host namespaces, privileged, seLinuxOptions, allowPrivilegeEscalation != false, unrestricted capabilities, restricted volume types, runAsNonRoot != true, seccompProfile
Warning: cilium-operator-65496b9554-wtrrq: host namespaces, hostPort, allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
Warning: coredns-7db6d8ff4d-dwff9 (and 1 other pod): runAsNonRoot != true, seccompProfile
Warning: etcd-kind-control-plane (and 3 other pods): host namespaces, allowPrivilegeEscalation != false, unrestricted capabilities, restricted volume types, runAsNonRoot != true
Warning: kube-proxy-7t26w (and 1 other pod): host namespaces, privileged, allowPrivilegeEscalation != false, unrestricted capabilities, restricted volume types, runAsNonRoot != true, seccompProfile
namespace/kube-system labeled (server dry run)
Warning: existing pods in namespace "local-path-storage" violate the new PodSecurity enforce level "restricted:latest"
Warning: local-path-provisioner-988d74bc-gsjg5: allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
namespace/local-path-storage labeled (server dry run)
namespace/secure-namespace labeled (server dry run)
```

Workloads will be affected. Existing Pods that violate the `restricted` policy will keep running but new Pods that violate the `restricted` policy will not be created.


17. Cleanup

Delete the `secure-namespace` Namespace:

```shell
kubectl delete namespace secure-namespace
```

Expected output:
```shell
namespace "secure-namespace" deleted
```