# Kubernetes Security Checklist: Authentication and Authorization 

## Check `system:masters` group is not used for user or authentication after bootstrapping

This step covers two Kubernetes security checklist items under Authentication & Authorization:
- Check `system:masters` group is not used for user or authentication after bootstrapping - The Role Based Access Control Good Practices are followed for guidance related to authentication and authorization.

In the [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/), the practice of granting least privilege or minimal RBAC rights and only explicitly granting required permissions. 

The system:masters group is a hardcoded group that bypasses all authorization checks and gives unrestricted rights (cluster admin) to the Kubernetes API server. With the concept of least privilege in mind, the system:masters group should never be used outside the bootstrapping mechanism/process.

### Roles, ClusterRoles, RoleBindings, ClusterRoleBindings, Groups, Users

By default, Kubernetees uses Role Based Access Control (RBAC) is used to drive authorization decisions and also allows policy configuration. 
The RBAC API uses four kinds of Kubernetes objects: Role, ClusterRole, RoleBinding, and ClusterRoleBinding.

Role and ClusterRole defines the set of permissions and permissions are additive -- there are no "deny" rules.

ClusterRoles are not namespaced and can be used cluster wide or in a single Namespace.
ClusterRoles are also used to define permisions on cluster-scoped resources such as Nodes, Namespaces, and CSIDriver.

An example of a Role or ClusterRole is to grant `get`, `watch`, `list` permissions for Pods and we can call this a "pod-reader" Role.

RoleBindings and ClusterRoleBindings grant persmission defined in a Role or ClusterRole to users, groups, or Service Accounts.
A RoleBinding grants permissions in a specific Namespace. 
A ClusterRoleBinding grants permissions cluster-wide.

A RoleBinding is used to grant the "pod-reader" Role to a user, group or Service Account for a specific Namespace.

Back to the Kubernetes Security Checklist item to not use the `system:masters` group, we will verify if any ClusterRoleBindings or RoleBindings grant permissions to the `system:masters` group.

#### Inspecting ClusterRoleBindings and RoleBindings

1. ClusterRoleBindings and RoleBindings bind sets of permissions defined in ClusterRoles and Roles to subjects like users, groups, and Service Accounts. 
Before we look to see which ClusterRoleBindings and RoleBindings may give rights to the `system:masters` group, let's take a look at ClusterRoleBindings and RoleBindings to see where we can find the subjects like users, groups, and Service Accounts.

List all ClusterRoleBindings:

```shell
kubectl get clusterolebindings
```

Expected output (truncate):
```shell
NAME                                                            ROLE                                                                               AGE
cilium                                                          ClusterRole/cilium                                                                 2d4h
cilium-operator                                                 ClusterRole/cilium-operator                                                        2d4h
cluster-admin                                                   ClusterRole/cluster-admin                                                          2d4h
kubeadm:cluster-admins                                          ClusterRole/cluster-admin                                                          2d4h
...
system:monitoring                                               ClusterRole/system:monitoring                                                      2d4h
system:node                                                     ClusterRole/system:node                                                            2d4h
system:node-proxier                                             ClusterRole/system:node-proxier                                                    2d4h
system:public-info-viewer                                       ClusterRole/system:public-info-viewer                                              2d4h
system:service-account-issuer-discovery                         ClusterRole/system:service-account-issuer-discovery                                2d4h
system:volume-scheduler                                         ClusterRole/system:volume-scheduler                                                2d4h
```

There are a large number of default ClusterRoleBindings, they are used to manage various components, controllers, and resources.
You can find more information on the default ClusterRoles and ClusterRoleBindings on the [Using RBAC Authorization page](https://kubernetes.io/docs/reference/access-authn-authz/rbac).

2. Let's look into a description of ClusterRoleBinding `system:public-info-viewer`.

```shell
kubectl describe clusterrolebinding system:public-info-viewer
```

Expected output:
```shell 
Name:         system:public-info-viewer
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  system:public-info-viewer
Subjects:
  Kind   Name                    Namespace
  ----   ----                    ---------
  Group  system:authenticated    
  Group  system:unauthenticated  
```

The detailed description of the `system:volume-scheduler` ClusterRoleBinding shows that this ClusterRoleBinding binds the permissions set in the `system:public-info-viewer` ClusterRole to two groups, `system:authenticated` and `system:unauthenticated`.

Both `system:authenticated` and `system:unauthenticated` are build-in groups.
`system:authenticated` is used for all authenticated users and `system:unauthenticated` is used to provide anonymous rights to the Kubernetes API.

3. Let's take the manifest of the same ClusterRoleBinding `system:public-info-viewer`.

```shell
kubectl get ClusterRoleBinding system:public-info-viewer -o yaml
```

Expected output:
```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2024-07-29T18:59:19Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:public-info-viewer
  resourceVersion: "175"
  uid: 686c849d-98eb-4fba-9804-6dd32d200ad3
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:public-info-viewer
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:unauthenticated
```

4. Get the same manifest in json.

```shell
kubectl get ClusterRoleBinding system:public-info-viewer -o json
```

Expected output:
```shell
{
    "apiVersion": "rbac.authorization.k8s.io/v1",
    "kind": "ClusterRoleBinding",
    "metadata": {
        "annotations": {
            "rbac.authorization.kubernetes.io/autoupdate": "true"
        },
        "creationTimestamp": "2024-07-29T18:59:19Z",
        "labels": {
            "kubernetes.io/bootstrapping": "rbac-defaults"
        },
        "name": "system:public-info-viewer",
        "resourceVersion": "175",
        "uid": "686c849d-98eb-4fba-9804-6dd32d200ad3"
    },
    "roleRef": {
        "apiGroup": "rbac.authorization.k8s.io",
        "kind": "ClusterRole",
        "name": "system:public-info-viewer"
    },
    "subjects": [
        {
            "apiGroup": "rbac.authorization.k8s.io",
            "kind": "Group",
            "name": "system:authenticated"
        },
        {
            "apiGroup": "rbac.authorization.k8s.io",
            "kind": "Group",
            "name": "system:unauthenticated"
        }
    ]
}
```

With the ability to get the manifest in json, we can use a command-line json processor called `jq` to search for any ClusterRoleBindings and RoleBindings that bind rights to the `system:masters` group.

5. Retrieve all ClusterRoleBindings in json.

```shell
kubectl get clusterrolebindings -o json
```

Expected output (truncated):
```shell
{
    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "rbac.authorization.k8s.io/v1",
            "kind": "ClusterRoleBinding",
            "metadata": {
                "annotations": {
                    "meta.helm.sh/release-name": "cilium",
                    "meta.helm.sh/release-namespace": "kube-system"
                },
                "creationTimestamp": "2024-07-29T18:59:45Z",
                "labels": {
                    "app.kubernetes.io/managed-by": "Helm",
                    "app.kubernetes.io/part-of": "cilium"
                },
                "name": "cilium",
                "resourceVersion": "484",
                "uid": "088b5465-d200-48b6-893a-478f2606f7ab"
            },
            "roleRef": {
                "apiGroup": "rbac.authorization.k8s.io",
                "kind": "ClusterRole",
                "name": "cilium"
            },
            "subjects": [
                {
                    "kind": "ServiceAccount",
                    "name": "cilium",
                    "namespace": "kube-system"
                }
            ]
        },
...
        {
            "apiVersion": "rbac.authorization.k8s.io/v1",
            "kind": "ClusterRoleBinding",
            "metadata": {
                "annotations": {
                    "rbac.authorization.kubernetes.io/autoupdate": "true"
                },
                "creationTimestamp": "2024-07-29T18:59:19Z",
                "labels": {
                    "kubernetes.io/bootstrapping": "rbac-defaults"
                },
                "name": "system:service-account-issuer-discovery",
                "resourceVersion": "182",
                "uid": "4916452b-46a4-431f-b004-4a87227811df"
            },
            "roleRef": {
                "apiGroup": "rbac.authorization.k8s.io",
                "kind": "ClusterRole",
                "name": "system:service-account-issuer-discovery"
            },
            "subjects": [
                {
                    "apiGroup": "rbac.authorization.k8s.io",
                    "kind": "Group",
                    "name": "system:serviceaccounts"
                }
            ]
        },
        {
            "apiVersion": "rbac.authorization.k8s.io/v1",
            "kind": "ClusterRoleBinding",
            "metadata": {
                "annotations": {
                    "rbac.authorization.kubernetes.io/autoupdate": "true"
                },
                "creationTimestamp": "2024-07-29T18:59:19Z",
                "labels": {
                    "kubernetes.io/bootstrapping": "rbac-defaults"
                },
                "name": "system:volume-scheduler",
                "resourceVersion": "180",
                "uid": "70e05f0d-8c2b-452f-8588-55f5fe0fd278"
            },
            "roleRef": {
                "apiGroup": "rbac.authorization.k8s.io",
                "kind": "ClusterRole",
                "name": "system:volume-scheduler"
            },
            "subjects": [
                {
                    "apiGroup": "rbac.authorization.k8s.io",
                    "kind": "User",
                    "name": "system:kube-scheduler"
                }
            ]
        }
    ],
    "kind": "List",
    "metadata": {
        "resourceVersion": ""
    }
}
```

The output in json is a list of all ClusterRoleBinding manifests.

6. One simple way to see if there is a ClusterRoleBinding that binds rights to the `system:masters` group is with grep.

```shell
kubectl get clusterrolebindings -o json | grep "system:masters"
```

Expected output:
```shell
                    "name": "system:masters"
```
There is one output with `system:masters`.

7. To verify if the output is the subject and to determine which ClusterRole or Role and ClusterRoleBinding, let's use the `--before-context or -B` option with grep

`kubectl get clusterrolebindings -o json | grep -B24 "system:masters"`

Expected output:
```shell
        {
            "apiVersion": "rbac.authorization.k8s.io/v1",
            "kind": "ClusterRoleBinding",
            "metadata": {
                "annotations": {
                    "rbac.authorization.kubernetes.io/autoupdate": "true"
                },
                "creationTimestamp": "2024-07-29T18:59:19Z",
                "labels": {
                    "kubernetes.io/bootstrapping": "rbac-defaults"
                },
                "name": "cluster-admin",
                "resourceVersion": "171",
                "uid": "16df3870-c40d-48ec-a042-41b22de2d870"
            },
            "roleRef": {
                "apiGroup": "rbac.authorization.k8s.io",
                "kind": "ClusterRole",
                "name": "cluster-admin"
            },
            "subjects": [
                {
                    "apiGroup": "rbac.authorization.k8s.io",
                    "kind": "Group",
                    "name": "system:masters"
```
The output shows the `cluster-admin` ClusterRoleBinding binds the `cluster-admin` ClusterRole to the `system:masters` Group.

According to the [Using RBAC Authorization page](https://kubernetes.io/docs/reference/access-authn-authz/rbac), the `cluster-admin` ClusterRole is a default ClusterRole that binds to the `system:masters` Group.
Additional details on the `cluster-admin` ClusterRole:
> Allows super-user access to perform any action on any resource. When used in a ClusterRoleBinding, it gives full control over every resource in the cluster and in all namespaces. When used in a RoleBinding, it gives full control over every resource in the role binding's namespace, including the namespace itself.

This is simple enough for a single case but what if there are many ClusterRoleBindings or RoleBindings that have `system:masters` group as the subject.

8. `jq` is a command-line json processor. We will use `jq` and `jq`'s ability to select specific objects if they have a specific value.
You can find more on the [`jq` manual](https://jqlang.github.io/jq/manual/).

Using `jq` we can iterate over lists and select objects based on if there is a subject that starts with `system:masters`.

Based on the output from `kubectl get clusterrolebindings -o json` that outputs all the manifests of all ClusterRoleBindings we can iterate over the `.items` and use `jq`'s select(boolean_expression) function to select any object that has the Group `system:masters` as a subject.

We can do so with the following command:

```shell
kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[].kind=="Group" and .subjects[].name=="system:masters")'
```

Expected output:
```shell
{
  "apiVersion": "rbac.authorization.k8s.io/v1",
  "kind": "ClusterRoleBinding",
  "metadata": {
    "annotations": {
      "rbac.authorization.kubernetes.io/autoupdate": "true"
    },
    "creationTimestamp": "2024-07-29T18:59:19Z",
    "labels": {
      "kubernetes.io/bootstrapping": "rbac-defaults"
    },
    "name": "cluster-admin",
    "resourceVersion": "171",
    "uid": "16df3870-c40d-48ec-a042-41b22de2d870"
  },
  "roleRef": {
    "apiGroup": "rbac.authorization.k8s.io",
    "kind": "ClusterRole",
    "name": "cluster-admin"
  },
  "subjects": [
    {
      "apiGroup": "rbac.authorization.k8s.io",
      "kind": "Group",
      "name": "system:masters"
    }
  ]
}
jq: error (at <stdin>:1506): Cannot iterate over null (null)
```

The query returns the expected result. Note the error, Cannot iterate over null. We get the error because there are ClusterRoleBindings that do not have subjects.
According to the [`jq` manual](https://jqlang.github.io/jq/manual/), we can add `?` to any `[]` in the query for the `.subjects[]? `to not output errors if there is no subject.

9. Update the query with `.subjects[]?` to not output errors if a ClusterRoleBinding has no subject.

```shell
kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[]?.kind=="Group" and .subjects[].name=="system:masters")'
```

Expected output:
```shell
{
  "apiVersion": "rbac.authorization.k8s.io/v1",
  "kind": "ClusterRoleBinding",
  "metadata": {
    "annotations": {
      "rbac.authorization.kubernetes.io/autoupdate": "true"
    },
    "creationTimestamp": "2024-07-29T18:59:19Z",
    "labels": {
      "kubernetes.io/bootstrapping": "rbac-defaults"
    },
    "name": "cluster-admin",
    "resourceVersion": "171",
    "uid": "16df3870-c40d-48ec-a042-41b22de2d870"
  },
  "roleRef": {
    "apiGroup": "rbac.authorization.k8s.io",
    "kind": "ClusterRole",
    "name": "cluster-admin"
  },
  "subjects": [
    {
      "apiGroup": "rbac.authorization.k8s.io",
      "kind": "Group",
      "name": "system:masters"
    }
  ]
}
```

Great! Now there is no error at the end.

10. Let's take the query to the next level with using `kubectl`'s ability to use custom columns, `jq`, and subshell.

We can use custom columns to output the important columns we would like to see from Kubernetes objects. Use the following command to output the ClusterRole and Subjects from the ClusterRoleBinding `cluster-admin`.

```shell
kubectl get clusterrolebindings cluster-admin -o custom-columns='ClusterRolBinding:.metadata.name,ClusterRole:.roleRef.name,Subject:.subjects[].name'
```

Expected output:
```shell
ClusterRolBinding   ClusterRole     Subject
cluster-admin       cluster-admin   system:masters
```

Great! We can output in easy-to-ready columns!

Let's use `jq`'s any() function as a filter to produce the name(s) of ClusterRoleBindings that have `system:masters` as a Group in the Subject.

```shell
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(any(.subjects[]?;.kind=="Group" and .name=="system:masters")).metadata.name'
```

Expected output:
```shell
cluster-admin
```

Now we have a query to obtain only the name of ClusterRoleBinding(s).

Let's use a subshell with the query to output names of ClusterRoleBindings that have `system:masters` Group in the Subject to the `kubectl` command that outputs custom columms to have a command that outputs ClusterRoleBindings with `system:masters` Group in the Subject and outputs the name of the ClusterRoleBinding, the name of the ClusterRole, the Subjects and any Labels -- labels can be helpful to determine what the ClusterRoleBinding is used for.

```shell
kubectl get clusterrolebindings $(kubectl get clusterrolebindings -o json | jq -r '.items[] | select(any(.subjects[]?;.kind=="Group" and .name=="system:masters")).metadata.name') -o custom-columns='ClusterRoleBinding:.metadata.name,ClusterRole:.roleRef.name,Subject:.subjects[].name,Labels:.metadata.labels'
```

Expected output:
```shell
ClusterRoleBinding   ClusterRole     Subject          Labels
cluster-admin        cluster-admin   system:masters   map[kubernetes.io/bootstrapping:rbac-defaults]
```

Now we have a command to check if Kubernetes clusters have a ClusterRoleBinding with `system:masters` Group in the Subject. In this case we see from the Label that it is a default ClusterRoleBinding from bootstrapping but Labels can be applied manually so it's best to check the [Using RBAC Authorization page](https://kubernetes.io/docs/reference/access-authn-authz/rbac).


To check if RoleBindings use `system:masters` Group in the subject, we can convert our command to query RoleBindings:

```shell
kubectl get rolebindings $(kubectl get rolebindings -o json | jq -r '.items[] | select(any(.subjects[]?;.kind=="Group" and .name=="system:masters")).metadata.name') -o custom-columns='RoleBinding:.metadata.name,Role:.roleRef.name,Subject:.subjects[].name,Labels:.metadata.labels'
```

Expected output:
```shell
RoleBinding   Role   Subject   Labels
```
Great there are no RoleBindings that use `system:masters` Group in the Subject.

To summarize, the `system:masters` Group should never be used by a ClusterRoleBinding or RoleBinding outside the defaults.
To check for this we have to check each ClusterRoleBinding and RoleBinding but with the help of `jq` to iterate over lists and `jq`'s ability to select objects with specific conditions we can create a single command to check which ClusterRoleBinding uses `system:masters` Group in the subject.