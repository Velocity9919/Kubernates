
CLUSTER COMMANDS :
```
kubectl cluster-info                                                       # Display cluster information
kubectl get nodes -o wide                                                  # List all nodes and show IPs
```
POD COMMANDS :
```
kubectl get pods                                                           # Show pods with a specific label
kubectl get pods -o wide.                                                  # Show pods with extra details
kubectl get pods -1 <label>=<value>                                        # Filter pods by label
kubectl get pod <name>                                                     # Show a specific pod
kubectl describe pod <name>                                                # Show detailed pod info
kubectl logs <pod>                                                         # View logs for a pod
kubectl exec -it <pod> <command>                                           # Execute a command inside a pod
kubectl delete pod <name>                                                  # Delete a pod
kubectl explain pod <resource>                                             # Explain Pod resource fields
```
DEPLOYMENT COMMANDS :
```
kubectl create deployment <name> --image=<image>                            # Create a deployment with image
kubectl get deployments                                                     # List deployments
kubectl describe deployment <name>                                          # Show deployment details
kubectl scale deployment <name> --replicas=<number>                         # Scale deployment replicas
kubectl rollout restart deployment/<name>                                   # Restart rollout
kubectl rollout status deployment/<name>                                    # Show rollout status
kubectl create deployment <name> --image=<image> -o yaml                    # Dry-run to test YAML
kubectl create deployment <name> --image=<image> --dry-run-client -o yaml   # Output deployment YAML 
kubectl create deployment <name> --image=<image> -o yaml > name.yaml        # Save YAML to file
```
SERVICE  :
```
kubectl get services                                                        # List services
kubectl describe service <name>                                             # Describe a service
kubectl expose pod <name>                                                   # Expose a pod as a service
kubectl delete service <name>                                               # Delete a service
kubectl port-forward pod <pod> <local>:<remote>                             # Forward local port to pod
```
CONFIGMAP & SECRET COMMANDS :
```
kubectl create configmap <name> --from-literal=<key>=<value>                # Create a ConfigMap from literal
kubectl create secret generic <name> --from-literal=<key>=<value>           # Create a generic Secret
kubectl get configmaps.                                                     # List ConfigMaps
kubectl get secrets                                                         # List Secrets
kubectl describe configmap <name>                                           # Describe a ConfigMap
```
NAMESPACE COMMANDS :
```
kubectl get namespaces                                                      # List namespaces
kubectl create namespace <name>                                             # Create a namespace
kubectl delete namespace <name>                                             # Delete a namespace
kubectl config set-context --current--namespace=<name>                      # Set current context namespace
```
RESOURCE COMMANDS :
```
kubectl apply -f <file>                                                     # Apply configuration file
kubectl edit <type> <name>                                                  # Edit a live resource
kubectl delete -f <file>                                                    # Delete resources from file
kubectl get <type>                                                          # List resources of a type
kubectl describe <type> <name>                                              # Describe a resource
```
STATS & EVENT COMMANDS :
```
kubectl top nodes                                                           # Show node resource usage
kubectl top pods                                                            # Show pod resource usage
kubectl get events                                                          # List recent cluster events
kubectl describe events                                                     # Describe events in detail
```
PERMISSION COMMANDS :
```   
get roles                                                                   # List Roles
get rolebindings.                                                           # List RoleBindings
kubectl auth can-i <action> --namespace=<ns>                                # Check if action is allowed in namespace
```


