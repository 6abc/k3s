# Monitoring Setup on k3s

🎯 **Goal**

* All services run as `ClusterIP`
* Access Grafana via an Ingress
* Hostname: `grafana.lab`
* Shared NFS storage: `192.168.0.11:/nfs/nfs_share/monit`

Since you’re on **k3s**, you already have the built‑in Traefik ingress controller
unless you disabled it.

---

## 1. Set all services to ClusterIP

When installing the chart, override the service types:

### kube‑prometheus‑stack

```sh
helm upgrade --install kind-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=ClusterIP \
  --set prometheus.service.type=ClusterIP \
  --set alertmanager.service.type=ClusterIP
```

Node exporter is already ClusterIP internally (DaemonSet, no need to expose).

Loki
```sh
helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --set service.type=ClusterIP \
  --set persistence.enabled=true \
  --set persistence.storageClassName=nfs-monitoring \
  --set persistence.size=50Gi
```

Promtail

Promtail does NOT need a service exposed externally.
Default is fine (it pushes to Loki internally).

---

## 2. Create Ingress for grafana.lab

Create file: grafana-ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
spec:
  ingressClassName: traefik
  rules:
  - host: grafana.lab
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kind-prometheus-grafana
            port:
              number: 80
```

Apply:

```sh
kubectl apply -f grafana-ingress.yaml
```

✅ 3️⃣ Add Local DNS Entry

On your PC (or local DNS server), add:
```sh
192.168.0.50   grafana.lab
```
(Use your control-plane IP)

Linux/macOS: edit /etc/hosts with sudo.

Windows: edit C:\Windows\System32\drivers\etc\hosts.

✅ 4️⃣ Configure Grafana Root URL (IMPORTANT)

Grafana must know it's behind ingress.

Upgrade with:

```sh
helm upgrade kind-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.grafana.ini.server.root_url=http://grafana.lab \
  --set grafana.grafana.ini.server.serve_from_sub_path=false
```

🏗 Final Architecture
Browser → grafana.lab
        ↓
   Traefik Ingress (k3s built-in)
        ↓
   Grafana (ClusterIP)
        ↓
Prometheus / Loki (ClusterIP)
        ↓
NFS Storage (192.168.0.11)
