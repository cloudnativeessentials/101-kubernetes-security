# Authentication and Authorization 

## Check `system:masters` group is not used for user or authentication after bootstrapping

kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[].name | startswith("system:masters"))'

kubectl get clusterrolebindings cluster-admin -o custom-columns='ClusterRole:.metadata.name,Subject:.subjects[].name'

kubectl get clusterrolebindings cluster-admin -o jsonpath='{.metadata.name}'


kubectl get clusterrolebindings -o jsonpath='{.items[*].metadata.name}'

kubectl get clusterrolebindings $(kubectl get clusterrolebindings cluster-admin -o jsonpath='{.metadata.name}') -o custom-columns='ClusterRole:.metadata.name,Subject:.subjects[].name'


kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[].name | startswith("system:masters"))' |  jq -j '.metadata.name'

#works 
kubectl get clusterrolebindings -o json |
jq -r '.items[] | select(any(.subjects[]?;.name=="system:masters")).metadata.name'

#works
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.subjects[]?.name | startswith("system:masters")).metadata.name'

#best one
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.subjects[]?.name | contains("system:masters")).metadata.name'

#best one
kubectl get clusterrolebindings $(kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.subjects[]?.name | contains("system:masters")).metadata.name') -o custom-columns='ClusterRole:.metadata.name,Subject:.subjects[].name,Labels:.metadata.labels'
