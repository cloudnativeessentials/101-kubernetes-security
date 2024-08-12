# Pod security
## RBAC rights to create, update, patch, delete workloads 

This step covers the following Kubernetes Security Checklist item under Pod Security:
-  RBAC rights to `create`, `update`, `patch`, `delete` workloads is only granted if necessary under the Pod Security category

We will learn about Roles, ClusterRoles, RoleBindings, and ClusterRole bindings.

1. Create a namespace

`kubectl create namespace rbac-namespace`

```shell
namespace/rbac-namespace created
```

Roles are namespaced and ClusterRoles can be used cluster wide. In this case we will create a ClusterRole to view Pods.
The ClusterRole is not applied to all namespaces unless a ClusterRoleBinding is created that binds the ClusterRole to a subject like a user, group, or Service Account.
In this case, we will be specific and use a RoleBinding to bind the ClusterRole to a specific Namespace.

Roles and ClusterRoles define a set of permissions to Kubernetes resource(s) e.g. create, list, get, and update Pods.

2. Create a Pod reader ClusterRole:

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

RoleBindings and ClusterRoleBindings bind a set of permisiones defined in a Role or ClusterRole to a subject (user, group, service account).

3. Create a RoleBinding to bind the pod-viewer ClusterRole to a user `devconf` in the `pod-view` group.

```shell
cat <<EOF > rb-view-pods.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-pods
  namespace: rbac-namespace
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

4. Test the devconf user and pod-view group if it can get pods in the rbac-namespace Namespace:

`kubectl --as devconf --as-group pod-view get pods --namespace rbac-namespace`

Output:
```shell
No resources found in rbac-namespace namespace.
```
Success!

5. Now test the user and group to get Pods in the kube-system namespace
`kubectl --as devconf --as-group pod-view get pods --namespace kube-system`

Output:
```shell
Error from server (Forbidden): pods is forbidden: User "devconf" cannot list resource "pods" in API group "" in the namespace "kube-system"
```
The `devconf` user in the `pod-view` group can not get Pods in the `kube-system` Namespace.

6. Now check if the user and group can get Pods in the default namespace:

`kubectl --as devconf --as-group pod-view get pods --namespace default`

Output:
```shell
Error from server (Forbidden): pods is forbidden: User "devconf" cannot list resource "pods" in API group "" in the namespace "default"
```

The user and group can not get Pods in the default Namespace.

7. Let's make sure our user can not create a Pod in the rbac-namespace:

`kubectl --as devconf --as-group pod-view run nginx-source --namespace rbac-namespace --image nginx:1.27.0`

Output:
```shell
Error from server (Forbidden): pods is forbidden: User "devconf" cannot create resource "pods" in API group "" in the namespace "rbac-namespace"
```
Great, the user and group can not create Pods in the rbac-namespace Namespace.

With a `pod-viewer` user and group, let's create the necessary Roles and RoleBindings for a user to create a workload Pod.

8. Create a Role to create, update, patch and delete Pods and Deployments in the rbac-namespace Namespace.

```shell
cat <<EOF > cr-pod-creator.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-creator
  namespace: rbac-namespace
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

9. Create the RoleBinding that binds the `pod-creator` Role to the `user` devconf and `pod-creator` group in the `rbac-namespace` Namespace.

```shell
cat <<EOF > rb-create-pods.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: create-pods
  namespace: rbac-namespace
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

`kubectl apply -f rb-create-pods.yaml`

Output:
```shell
rolebinding.rbac.authorization.k8s.io/create-pods created
```

10. Use the `devconf` user to create a Pod in the `rbac-namespace` Namespace:

`kubectl --as devconf --as-group pod-creator run nginx --namespace rbac-namespace --image nginx:1.27.0`

Output:
```shell
pod/nginx created
```
Now we have a user than can create Pods in the rbac-namespace Namespace.

To summarize we defined permissions in several roles, one to view Pods and one to create Pods. 
We bound the permissions to a user and group for a specific Namespace using a RoleBinding and tested the user and group to get Pods in various Namespaces and to create a Pod before and after permissions to create Pods were given.