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

prometheus:
  ingress:
    enabled: true
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
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
helm repo update  
helm upgrade --install --force -n monitoring --create-namespace kube-prometheus-stack prometheus-community/kube-prometheus-stack -f values.yaml
```

# Tracing

## FluentBit

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm upgrade --install --force -n tracing --create-namespace fluent-bit fluent/fluent-bit
```

## Fluentd

### Helm Chart Installation
https://github.com/fluent/helm-charts/tree/main/charts/fluentd

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm upgrade --install --force -n tracing --create-namespace fluentd fluent/fluentd
```

### OS Installation
This part describes how to install fluentd as OS service.

```bash
vi /etc/security/limits.conf
# add the following lines and reboot
root soft nofile 65536
root hard nofile 65536
* soft nofile 65536
* hard nofile 65536
# :wq
vi /etc/sysctl.conf
# add the following lines and reboot
net.core.somaxconn = 1024
net.core.netdev_max_backlog = 5000
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_wmem = 4096 12582912 16777216
net.ipv4.tcp_rmem = 4096 12582912 16777216
net.ipv4.tcp_max_syn_backlog = 8096
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 10240 65535
# If forward uses port 24224, reserve that port number for use as an ephemeral port.
# If another port, e.g., monitor_agent uses port 24220, add a comma-separated list of port numbers.
# net.ipv4.ip_local_reserved_ports = 24220,24224
net.ipv4.ip_local_reserved_ports = 24224
# :wq
```

### Installation as service
Tested on Ubuntu 23.10, where no official installer was found, so ruby was used as per documentation suggestion.  
https://docs.fluentd.org/installation/install-by-gem
```bash
sudo apt-get install ruby-full
gem install ruby_dev
gem install fluentd --no-doc
# verify installation:
fluentd --setup ./fluent
fluentd -c /home/ubuntu/fluent/fluent.conf -vv &
echo '{"json":"message"}' | fluent-cat debug.test
```

### Create fluentd service
```bash
vi /etc/systemd/system/fluentd.service
# paste the following
[Unit]
Description=Fluentd Daemon
After=network.target

[Service]
ExecStart=/usr/local/bin/fluentd -c /home/ubuntu/fluent/fluent.conf -vv
Restart=always
User=root
Group=root

[Install]
WantedBy=multi-user.target
# :wq
# start the service
sudo systemctl daemon-reload
sudo systemctl start fluentd
```




# cert-manager
TODO
