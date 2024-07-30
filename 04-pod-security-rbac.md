# Pod security
## RBAC rights to create, update, patch, delete workloads 

Create a namespace

`kubectl create namespace secure-namespace`

```shell
namespace/secure-namespace created
```


Roles are namespaced and ClusterRoles can be used cluster wide. In this case we will create a ClusterRole to view Pods.
The ClusterRole is not applied to all namespaces unless a ClusterRoleBinding is created.
In this case, we will be specific and use a RoleBinding to bind the ClusterRole to a specific Namespace.

Create a Pod reader role:

```shell
cat <<EOF > cr-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
EOF
```

`kubectl apply -f cr-pod-reader.yaml`

Output:
```shell
clusterrole.rbac.authorization.k8s.io/pod-viewer created
```

RoleBindings and ClusterRoleBindings bind a role to a subject (user, group, service account)

Create a RoleBinding to bind the pod-viewer ClusterRole to a user `devconf` in the pod-view group.

```shell
cat <<EOF > rb-view-pods.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-pods
  namespace: secure-namespace
subjects:
- kind: User
  name: devconf
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: pod-view
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
EOF
```

`kubectl apply -f rb-view-pods.yaml`

Output:
```shell
rolebinding.rbac.authorization.k8s.io/view-pods created
```

Test the devconf user and pod-view grou pif it can get pods in the secure-namespace Namespace:

`kubectl --as devconf --as-group pod-view get pods --namespace secure-namespace`

Output:
```shell
No resources found in secure-namespace namespace.
```
Success!

Now test the user and group to get Pods in the kube-system namespace
`kubectl --as devconf --as-group pod-view get pods --namespace kube-system`

Output:
```shell
Error from server (Forbidden): pods is forbidden: User "devconf" cannot list resource "pods" in API group "" in the namespace "kube-system"
```
The user and group can not get pods in the kube-system namespace.

Now check if the user and group can get Pods in the default namespace:

`kubectl --as devconf --as-group pod-view get pods --namespace default`

Output:
```shell
Error from server (Forbidden): pods is forbidden: User "devconf" cannot list resource "pods" in API group "" in the namespace "default"
```

The user and group can not get Pods in the default Namespace.

Let's make sure our user can not create a Pod in the secure-namespace:
`kubectl --as devconf --as-group pod-view run nginx-source --namespace secure-namespace --image nginx:1.27.0`

Output:
```shell
Error from server (Forbidden): pods is forbidden: User "devconf" cannot create resource "pods" in API group "" in the namespace "secure-namespace"
```
Great, the user and group can not create Pods in the secure-namespace Namespace.

With a Pod viewer user and group, let's create the necessary Roles and RoleBindings for a user to create a workload Pod.

Create a Role to create, update, patch and delete Pods and Deployments in the secure-namespace Namespace.

```shell
cat <<EOF > cr-pod-creator.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-creator
  namespace: secure-namespace
rules:
- apiGroups: [""]
  resources: ["pods","deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
EOF
```

Create the Role:
`kubectl apply -f cr-pod-creator.yaml`

Output:
```shell
role.rbac.authorization.k8s.io/pod-creator created
```

Create the RoleBinding that binds the pod-creator Role to a user devconf and pod-creator group in the secure-namespace Namespace.

```shell
cat <<EOF > rb-create-pods.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: create-pods
  namespace: secure-namespace
subjects:
- kind: User
  name: devconf
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: pod-creator
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-creator
  apiGroup: rbac.authorization.k8s.io
EOF
```

Create the RoleBinding
`kubectl apply -f rb-create-pods.yaml`

Output:
```shell
rolebinding.rbac.authorization.k8s.io/create-pods created
```

Using the devconf user, let's create a Pod in the secure-namespace Namespace:

`kubectl --as devconf --as-group pod-creator run nginx-source --namespace secure-namespace --image nginx:1.27.0`

Output:
```shell
pod/nginx-source created
```
Now we have a user than can create Pods in the secure-namespace Namespace.

