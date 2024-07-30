# Authentication and Authorization 

## kube-controller-manager is running with `--use-service-account-credentials` enabled
Check /etc/kubernetes/kube-controller-manager.yaml

kubectl -n kube-system get pod kube-controller-manager

#gets kube-controller-manager pod name
kubectl -n kube-system get pods -o json | jq -r '.items[] | select(.metadata.name | contains("kube-controller-manager")).metadata.name'

#best one, gets the command
kubectl -n kube-system get pod $(kubectl -n kube-system get pods -o json | jq -r '.items[] | select(.metadata.name | contains("kube-controller-manager")).metadata.name') -o custom-columns='Pod:.metadata.name,Container:.spec.containers[].name,Command:.spec.containers[].command' 