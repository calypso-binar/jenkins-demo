# TODO

# Flannel
TODO

# Metallb
TODO

# nfs-subdir-external-provisioner
TODO

# Ingress-Nginx

```bash
# if you have an SSL Certificate and Key
kubectl create namespace ingress-nginx
kubectl create secret tls no-ip-ssl-cert --key calypso-binar.key --cert calypso-binar_com.pem -n ingress-nginx
kubectl -n ingress-nginx create secret generic no-ip-ssl-cert --from-file=./calypso-binar_com.pem
```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
# use controller.extraArgs.default-ssl-certificate only if you have a certificate installed on Kubernetes as a secret.
# otherwise Kubernetes will provide a self-signed certificate
helm upgrade --install --force ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace \
--set controller.extraArgs.default-ssl-certificate="ingress-nginx/no-ip-ssl-cert"
```

# Metallb 
TODO

# NFS with NAS (Network Attached Storage) with USB
Mount an USB Stick (or external SSD) to `/media/usb`  
example with vfat file system usb stick:  
```
sudo mount -t vfat /dev/sda1 /media/usb -o uid=1000,gid=1000,utf8,dmask=027,fmask=137
```


https://ubuntu.com/server/docs/service-nfs
Install nfs-kernel-server on a machine.
Export the /media/usb to /etc/exports on an IP range allowed to access it:
```
/media/usb 192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)
```

restart the nfs server:
```
systemctl restart nfs-kernel-server
```

If you want to access the files then install nfs-common:
sudo apt-get install nfs-common
Mount the USB form the nfs:
```
mkdir -p /media/usb
mount -t nfs 192.168.1.200:/media/usb /media/usb
```

# Use NFS in Kubernetes
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master

helm upgrade --install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
 --namespace nfs --create-namespace \
 --set nfs.server=192.168.1.200 \
 --set nfs.path=/media/usb \
 --set storageClass.defaultClass=true \
 --set storageClass.accessModes=ReadWriteMany

 # Monitoring

create a values.yaml file:
```yaml
grafana:
  persistence:
    enabled: true
    storageClassName: nfs-client
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/rewrite-target: /$1
      nginx.ingress.kubernetes.io/use-regex: "true"
    path: /grafana/?(.*)
    pathType: ImplementationSpecific
    hosts:
      - calypso-binar.com
  grafana.ini:
    server:
      root_url: https://calypso-binar.com/grafana # this host can be localhost
      datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://kube-prometheus-stack-prometheus:9090
        isDefault: true  # Only one datasource should have this set to true
      - name: Loki
        type: Loki
        access: proxy
        url: http://loki-stack:3100
        isDefault: false

prometheus:
  ingress:
    enabled: true
  monitor:
    relabelings:
      - action: replace
        replacement: raspberrypi-master
        targetLabel: cluster
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/rewrite-target: /$2
      nginx.ingress.kubernetes.io/use-regex: "true"
    paths:
      - "/prometheus(/|$)(.*)"
    pathType: ImplementationSpecific
    hosts:
      - calypso-binar.com
        #  prometheusSpec:
        #    externalUrl: "https://calypso-binar.com/prometheus"
        #    routePrefix: "/prometheus"
        #  prometheusSpec:
        #    additionalScrapeConfigs:
        #    - job_name: 'fluentd'
        #      static_configs:
        #        - targets:
        #          - 'fluentd.tracing:24231'

alertmanager:
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/rewrite-target: /$1
      nginx.ingress.kubernetes.io/use-regex: "true"
    paths:
      - /alertmanager/?(.*)
    pathType: ImplementationSpecific
    hosts:
      - calypso-binar.com

kubelet:
  serviceMonitor:
    cAdvisorRelabelings:
      - action: replace
        replacement: raspberrypi-master
        targetLabel: cluster
      - targetLabel: metrics_path
        sourceLabels:
          - "__metrics_path__"
      - targetLabel: "instance"
        sourceLabels:
          - "node"

defaultRules:
  additionalRuleLabels:
    cluster: raspberrypi-master

kube-state-metrics:
  prometheus:
    monitor:
      relabelings:
        - action: replace
          replacement: raspberrypi-master
          targetLabel: cluster
        - targetLabel: "instance"
          sourceLabels:
            - "__meta_kubernetes_pod_node_name"

prometheus-node-exporter:
  prometheus:
    monitor:
      relabelings:
        - action: replace
          replacement: raspberrypi-master
          targetLabel: cluster
        - targetLabel: "instance"
          sourceLabels:
            - "__meta_kubernetes_pod_node_name"
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
helm repo update  
helm upgrade --install --force -n monitoring --create-namespace kube-prometheus-stack prometheus-community/kube-prometheus-stack -f values.yaml
```

# Tracing

## Loki-Stack
https://github.com/grafana/helm-charts/tree/main/charts/loki-stack

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install --force -n monitoring --create-namespace loki-stack grafana/loki-stack -f loki-stack-values.yaml
```

loki-stack-values.yaml:  
```yaml
loki:
  config:
    auth_enabled: false
```

# MariaDB
https://www.datree.io/helm-chart/mariadb-bitnami

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install --force -n mariadb --create-namespace mariadb bitnami/mariadb -f mariadb-values.yaml
```

mariadb-values.yaml:  
```yaml
auth:
  rootPassword: "YOUR PW HERE"
```

# cert-manager
TODO
