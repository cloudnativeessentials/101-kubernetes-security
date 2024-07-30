# Network Security

## Ingress and egress network policies are applied to all workloads in the cluster
## Default network policies within each namespace, selecting all pods, denying everything, are in place

Create 2 Namespaces
`kubectl create namespace source`
`kubectl create namespace target`

Create 1 workload in each Namespace
`kubectl run nginx-source --namespace source --image nginx:1.27.0`
`kubectl run nginx-target --namespace target --image nginx:1.27.0`

Create a service
`kubectl expose pod nginx-target --port 80 --namespace target`
`kubectl expose pod nginx-source --port 80 --namespace source`

From the nginx-source Pod in the source Namespace, reach out to the nginx-target Pod in the target Namespace
(note: using the Service DNS record)

`kubectl exec nginx-source --namespace source -it -- /bin/bash`

`curl http://nginx-target.target.svc.cluster.local| head -n 4`
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0  55112      0 --:--:-- --:--:-- --:--:-- 55909
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

or 

`kubectl exec nginx-source --namespace source -it -- curl nginx-target.target.svc.cluster.local | head -n 4`
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

Create Network Policies

Create a default deny all Network Policy for the target Namespace:
```shell
cat <<EOF > default-deny-all-target-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

`kubectl apply -f default-deny-all-target-namespace.yaml`

Run the same test, from nginx-source Pod try to reach nginx-target Pod:
`kubectl exec nginx-source --namespace source -it -- curl nginx-target.target.svc.cluster.local | head -n 4`

Times out
```shell
curl: (28) Failed to connect to 10.244.1.63 port 80 after 128918 ms: Couldn't connect to server
command terminated with exit code 28
```

Let's creat a Network Policy to allow ingress from source namespace to target Namespace

```shell
cat <<EOF > ingress-source-to-target-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-source-to-target
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: source
EOF
```
`kubectl apply -f ingress-source-to-target-namespace.yaml`

Run the same test, from nginx-source Pod try to reach nginx-target Pod:
`kubectl exec nginx-source --namespace source -it -- curl --max-time 60 nginx-target.target.svc.cluster.local | head -n 4`
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Let's try from a new namespace
Create a new namspace
`kubectl create namespace new-source`

Create 1 workload in the new-source Namespace
`kubectl run nginx-new-source --namespace new-source --image nginx:1.27.0`

Run the same test, from nginx-new-source Pod try to reach nginx-target Pod:
`kubectl exec nginx-new-source --namespace new-source -it -- curl --max-time 60 nginx-target.target.svc.cluster.local | head -n 4`

```shell
curl: (28) Connection timed out after 60001 milliseconds
command terminated with exit code 28
```
Times out 

Run the same test, from nginx-source Pod try to reach nginx-target Pod:
`kubectl exec nginx-source --namespace source -it -- curl --max-time 60 nginx-target.target.svc.cluster.local | head -n 4`
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Since we have a default deny policy for egress, egress from the nginx-target Pod in the target namespace is blocked.
Let's test it
`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 nginx-source.source.svc.cluster.local | head -n 4`

Create an egress policy for the Target Namespace to allow egress to the source Namespace

```shell
cat <<EOF > egress-target-to-source-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-target-to-source
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: source
EOF
```
`kubectl apply -f egress-target-to-source-namespace.yaml`

```shell
networkpolicy.networking.k8s.io/egress-target-to-source created
```

Repeat egress test:
`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 nginx-source.source.svc.cluster.local | head -n 4`

Times out

but if you repeat the egress test with the nginx-source pod's vip
`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 10.244.1.156 | head -n 4`
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Pods need access to the kube-system namespace to get Kubernetes cluster DNS records.
Let's add ingress and egress to the kube-system namespace for the target Namespace.

```shell
cat <<EOF > all-all-target-to-kube-system-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-target-to-kube-system
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
EOF
```
`kubectl apply -f all-all-target-to-kube-system-namespace.yaml`

Repeat egress test:
`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 nginx-source.source.svc.cluster.local | head -n 4`
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```