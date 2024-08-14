# Authentication and Authorization 

## kube-controller-manager is running with `--use-service-account-credentials` enabled

Service Accounts are used to provide identity. 
Service Accounts are also used to grant priviledges and access control via Role Base Access Control (RBAC) like grant read-only access to Secrets.

Pods may need access to the Kubernetes API retrieve information.

The Kubernetes controller manager is a daemon that embeds core Kubernetes control loops.

The setting to enable Service Accounts is with the Kubernetes controller manager which runs as the `kube-controller-manager` Pod.

In this lab, we will check if Service Accounts are enabled.

In upstream Kubernetes, the kube-controller-manager settings are found in `/etc/kubernetes/kube-controller-manager.yaml` in the control plane nodes.

In a kind cluster, `/etc/kubernetes/kube-controller-manager.yaml` is not used. We will explore the `kube-controller-manager` Pod.

1. Let's check the `kube-controller-manager` Pod.

```shell
kubectl -n kube-system get pod kube-controller-manager-kind-control-plane 
```

Expected output:
```shell
NAME                                         READY   STATUS    RESTARTS   AGE
kube-controller-manager-kind-control-plane   1/1     Running   0          41m
```

Notice how kind is in the name of the `kube-controller-manager` Pod.

2. With another distribution, we can list the Pods in the `kube-system` namespace to get the name of the `kube-controller-manager` Pod. A simple shortcut is to use a JSON processor tool called `jq` to output the name of the `kube-controller-manager` Pod.

```shell
kubectl -n kube-system get pods -o json | jq -r '.items[] | select(.metadata.name | contains("kube-controller-manager")).metadata.name'
```

Expected output:
```shell
kube-controller-manager-kind-control-plane
```

3. Let's check what options are set with the `kube-controller-manager` Pod. For more information on `kube-controller-manager` options, see the [kube-controller-manager page on kubernetes.io](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).

```shell
kubectl get pod kube-controller-manager-kind-control-plane -n kube-system -o yaml
```

Expected output:
```shell
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/config.hash: b15a39def8c127158fba5be8bbcdbeaf
    kubernetes.io/config.mirror: b15a39def8c127158fba5be8bbcdbeaf
    kubernetes.io/config.seen: "2024-08-14T11:58:12.098498084Z"
    kubernetes.io/config.source: file
  creationTimestamp: "2024-08-14T11:58:23Z"
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager-kind-control-plane
  namespace: kube-system
  ownerReferences:
  - apiVersion: v1
    controller: true
    kind: Node
    name: kind-control-plane
    uid: b3c2ea9c-f0bd-4f67-adab-11f60083d096
  resourceVersion: "428"
  uid: 079f2dd4-6270-45d3-a1a3-a1e2871c865f
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kind
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --enable-hostpath-provisioner=true
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/16
    - --use-service-account-credentials=true
    image: registry.k8s.io/kube-controller-manager:v1.30.0
 ...
```

The .spec.containers[0].command field shows the command that runs when the container starts and the options used.
Notice the command is `kube-controller-manager`.
Do you see an option related to Service Accounts?

4. Verify `--use-service-account-credentials` option is used.
Refer back to the .spec.containers[0].command field to see if `--use-service-account-credentials` option is present.
We see `--use-service-account-credentials=true` in the .spec.containers[0].command field.

5. To make it easier and using `jq` we can output the .spec.containers[0].command field in a custom column with:

```shell
kubectl -n kube-system get pod $(kubectl -n kube-system get pods -o json | jq -r '.items[] | select(.metadata.name | contains("kube-controller-manager")).metadata.name') -o custom-columns='Pod:.metadata.name,Container:.spec.containers[].name,Command:.spec.containers[].command'
```

Expected output:
```shell
Pod                                          Container                 Command
kube-controller-manager-kind-control-plane   kube-controller-manager   [kube-controller-manager --allocate-node-cidrs=true --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=127.0.0.1 --client-ca-file=/etc/kubernetes/pki/ca.crt --cluster-cidr=10.244.0.0/16 --cluster-name=kind --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt --cluster-signing-key-file=/etc/kubernetes/pki/ca.key --controllers=*,bootstrapsigner,tokencleaner --enable-hostpath-provisioner=true --kubeconfig=/etc/kubernetes/controller-manager.conf --leader-elect=true --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --root-ca-file=/etc/kubernetes/pki/ca.crt --service-account-private-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/16 --use-service-account-credentials=true]
```

6. Kubernetes will create a default Service Account for each namespace. Let's create a new Namespace to test this.

```shell
kubectl create namespace service-account
```

Expected output:
```shell
namespace/service-account created
```

7. A Service Account is an object we can retrieve. Get the Service Accounts in the `service-account` Namespace

```shell
kubectl get serviceaccount -n service-account
```

Expected output:
```shell
NAME      SECRETS   AGE
default   0         60s
```

Without explicitly creating a Service Account, we see a `default` Service Account in the `service-account` Namespacek

8. Let's create a Pod

```shell
kubectl run nginx --image nginx:1.27.0 -n service-account
```

Expected out:
```shell
pod/nginx created
```

9. 
```shell
kubectl get pod nginx -n service-account -o yaml
```

Expected output:
```shell
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2024-08-14T15:19:23Z"
  labels:
    run: nginx
  name: nginx
  namespace: service-account
  resourceVersion: "23599"
  uid: 920a3b3a-e62d-4d53-8a50-d6acd19d53c5
spec:
  containers:
  - image: nginx:1.27.0
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-ltkkg
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: kind-worker
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  ...
```

The output is long but notice the .spec.containers[0].serviceAccount field and it's set to `default`.
Without setting the Service Account, Pods are assigned the default Service Account to use.

The default Service Account grants API discovery permissions to Pods to access the Kubernetes API.
Any additional permissions require a new Service Account and for the Pod to be specified the Service Account.