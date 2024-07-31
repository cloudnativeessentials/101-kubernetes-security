# Network Security

## Ingress and egress network policies are applied to all workloads in the cluster
## Default network policies within each namespace, selecting all pods, denying everything, are in place

This step we will cover Kubernetes Security Checklist items under Network Security: Ingress and egress network policies are applied to all workloads in the cluster and Default network policies within each namespace, selecting all pods, denying everything, are in place.

### Namespace and workload setup
Create two namespaces and a workload in each namespace to test Network Policies.

1. Create 2 Namespaces
`kubectl create namespace source`
`kubectl create namespace target`

2. Create 1 workload in each Namespace
`kubectl run nginx-source --namespace source --image nginx:1.27.0`
`kubectl run nginx-target --namespace target --image nginx:1.27.0`

3. Create Services to expose the Pod which will give use a DNS entry to use to access the Pods
`kubectl expose pod nginx-target --port 80 --namespace target`
`kubectl expose pod nginx-source --port 80 --namespace source`

4. From the `nginx-source` Pod in the `source` Namespace, reach out to the `nginx-target` Pod in the `target` Namespace
(note: we are using the Service DNS record)

`kubectl exec nginx-source --namespace source -it -- curl nginx-target.target.svc.cluster.local | head -n 4`

Expected output:
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

That works! The `nginx-source` Pod in the `source` Namespace can reach the `nginx-target` Pod in the `target` Namespace because there are no Network Policies in place.

### Network Policies
Create a default deny all Network Policy for `target` Namespaces then create Network Policies to allow specific ingress and egress to the `target` Namespace.

5. Create a default deny all Network Policy for the target Namespace:
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

6. Run the same test , from the `nginx-source` Pod try to reach the `nginx-target` Pod:
`kubectl exec nginx-source --namespace source -it -- curl nginx-target.target.svc.cluster.local | head -n 4`

We will see that the command times out.
Expected output:
```shell
curl: (28) Failed to connect to 10.244.1.63 port 80 after 128918 ms: Couldn't connect to server
command terminated with exit code 28
```

With the deny all Network Policy for the `target` Namespace, traffic is denied to Pods in the `target` Namespace.

7. Let's create a Network Policy to allow ingress from the `source` Namespace to the `target` Namespace

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

8. Run the same test. From the `nginx-source` Pod try to reach the `nginx-target` Pod:
`kubectl exec nginx-source --namespace source -it -- curl --max-time 60 nginx-target.target.svc.cluster.local | head -n 4`

Expected output:
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

That works! With a Network Policy in place that allows traffic from the `source` Namespace to the `target` Namespace. The `nginx-source` Pod can reach the `nginx-target` Pod.

9. Let's test if ingress is allowed from a new namespace
Create a `new-source` Namspace:
`kubectl create namespace new-source`

10. Create a workload Pod called `nginx-new-source` in the new-source Namespace:
`kubectl run nginx-new-source --namespace new-source --image nginx:1.27.0`

11. Run a test, from `nginx-new-source` Pod try to reach the `nginx-target` Pod:
`kubectl exec nginx-new-source --namespace new-source -it -- curl --max-time 60 nginx-target.target.svc.cluster.local | head -n 4`

Expected output:
```shell
curl: (28) Connection timed out after 60001 milliseconds
command terminated with exit code 28
```
Workloads from the `new-source` Namespace can not reach the `target` Namespace.

12. Repeat the test from the `source` Namespace to verify the ingress Network Policy works. From the `nginx-source` Pod try to reach `nginx-target` Pod:
`kubectl exec nginx-source --namespace source -it -- curl --max-time 60 nginx-target.target.svc.cluster.local | head -n 4`

Expected output:
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

We confirmed that ingress from Namespaces other than the `source` Namespace are allowed into the `target` Namespace.

13. Since we have a default deny Network Policy for both ingress and egress for the `target` Namespace and only have an ingress Network Policy from the `source` Namespace to the `target` Namespace. Egress traffic from the `target` Namespace to anywhere including the `source` Namespace should be blocked -- let's test this.

From the `nginx-target` Pod, try to reach the `nginx-source` Pod:
`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 nginx-source.source.svc.cluster.local | head -n 4`

Expected output:
```shell
curl: (28) Connection timed out after 60001 milliseconds
command terminated with exit code 28
```

14. Create an egress policy for the `target` Namespace to allow egress to the `source` Namespace:

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

Expected output:
```shell
networkpolicy.networking.k8s.io/egress-target-to-source created
```

15. Repeat egress test from the `nginx-target` Pod to the `nginx-source` Pod:
(note: we are using the `nginx-source` Service DNS)
`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 nginx-source.source.svc.cluster.local | head -n 4`

Expected output:
```shell
curl: (28) Connection timed out after 60001 milliseconds
command terminated with exit code 28
```
The time out is not expected unless we realize how the workloads lookup DNS records. 

16. Let's repeat the egress test but with the `nginx-source` Pod's virtual IP.
We can find the virtual IP of the `nginx-source` Pod with:
`kubectl get pods -o wide --namespace source`

Expected output:
```shell
NAME           READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
nginx-source   1/1     Running   0          1h   10.244.1.156   kind-worker   <none>           <none>
```

In this scenario, the `nginx-source` Pod's virtual IP is 10.244.1.156. Your virtual IP will vary.

17. Use the `nginx-source` Pod's virutal IP to try to reach it from the `nginx-target` Pod:

`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 10.244.1.156 | head -n 4`

Expected output:
```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```
It works! So using the Pod's virtual IP works but using the Service's DNS entry does not work but has worked when we did not have Network Policies.

18. Pods need access to the kube-system namespace to get Kubernetes cluster DNS records.

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

19. Repeat egress test from the `nginx-target` Pod to the `nginx-source` Pod using the `nginx-source` Service's DNS record:

`kubectl exec nginx-target --namespace target -it -- curl --max-time 60 nginx-source.source.svc.cluster.local | head -n 4`

Expected output:

```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

To summarize we created a default deny all for ingress and egress for the `target` Namespace. 
To allow traffic from the `source` Namespace to the `target` Namespace, we created a Network Policy in the `target` Namespace to allow ingress from the `source` Namespace. 
To allow traffic from the `target` Namespace to the `source` Namespace we created a Network Policy in the `target` Namespace to allow egress to the `source` Namespace. 
But to use Kubernets cluster DNS records, we also needed a Network Policy in the `target` Namespace to allow igress and egress to the `kube-system` Namespace to lookup cluster DNS records.