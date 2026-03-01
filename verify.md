kubectl get all -n gatus -o wide

# Persistance Volume
kubectl get pvc -n gatus -o wide

# Service
kubectl get svc -n gatus -o wide

# Ingress
kubectl get ingress -n gatus -o wide

# Describe/Log Check
kubectl describe pod <pod-NAME> -n gatus   `kubectl describe pod gatus-65f995f55d-9pnbt -n gatus`
kubectl logs <pod-NAME> -n gatus           `kubectl logs pod gatus-65f995f55d-s4kv6 -n gatus`
