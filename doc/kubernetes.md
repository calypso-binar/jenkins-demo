# TODO

# Hardware

This shopping list is the bare minimum for the project, you can always get more or get better ones:

- 4 x Raspberry PI 4B GGB RAM
- 4x Micro SD Cards: SanDisk Ultra Pro 64 GB
- 4x original Raspberry PI 4 power supplies
- Micro HDMI Cable
- Keyboard

Optional:
- 4x Raspberry PI coolers
- 4x Raspberry PI cases 

# OS
Download [Raspberry PI Imager](https://www.raspberrypi.com/software/) on your computer.  
Install Ubuntu 23.10 on the SD Cards.  
Assemble the Raspberry PIs and start them up.

# First User
Depending on how you installed the OS on the micro SD, it might already have a user other than root.  
I usually configure my OS with a preconfigured user "ubuntu".  
If that's not the case you will be logged in with root at startup.  

You can create a user named ubuntu with the following command:
```bash
sudo groupadd --gid 2000 ubuntu
sudo useradd \
  --uid 2000 \
  --gid ubuntu \
  --no-user-group \
  --base-dir /home/ubuntu \
  --home-dir /home/ubuntu \
  --create-home \
  --shell bash \
  --groups sudo
  ubuntu
sudo passwd ubuntu # type your password for ubuntu user 
```
This will create a user named "ubuntu" with its primary group also named "ubuntu",  
and the new user will also be able to perform sudo. 

To clean up the user you can run the following command:
```bash
sudo userdel ubuntu
sudo rm -rf /home/ubuntu
```

## Wireless 

> [!NOTE]
> This part might already be obsolete, depending on how you configured the installed OS on the micro sd card.   
Raspberry PI imager lets you configure wifi for the OS even before it is written to the micro sd card.  
Nevertheless, one might have forgotten to configure the wifi / might want to reconfigure it,   
which is why this part will be here.

For this configuration you will need to connect your Raspberry PI to a monitor with the micro HDMI cable.  
You will also need a keyboard attached to the Raspberry PI.

```bash
# find your wifi adapter name, ex. wlan0
ls /sys/class/net
# generate a password from SSID (name of your wifi access point) and your wifi's password 
wpa_passphrase SSID PASSWORD 
# get the psk value, you will need it as password in the 50-cloud-init.yaml file
sudo vi /etc/netplan/50-cloud-init.yaml
# paste the following value and replace the placeholders
network:
    version: 2
    wifis:
        renderer: networkd
        [wifi-adapter-from-/sys/class/net]:
            access-points:
                [ssid-name]:
                    password: [psk-value-from-before]
            dhcp4: true
            optional: true
# :wq
sudo netplan apply
```
Now your Raspberry PI should be connected to your wifi.

# Hostname

> [!NOTE]
> This part may already be obsolete, since it is possible to set the hostname 
> through Raspberry PI Imager even before writing the OS to the micro sd.
> Regardless, one may want to change the hostname even after the OS was booted. 

```bash
sudo vi /etc/hostname
# change content with preferred hostname, ex.: raspberry-pi-master, raspberry-pi-worker-1, raspberry-pi-worker-2, etc.
# :wq
sudo vi /etc/hosts
# replace the old hostname with new one (examples above if you need one)
# hostname must match with the one in /etc/hostname

# check if you have cloud-init installed
cloud-init status
# if you have cloud-init installed then it will display something like "status: done"
# also if cloud-init is installed then you will need to perform the following step:
sudo vi /etc/cloud/cloud.cfg
# change `preserve_hostname: false` to `preserve_hostname: true` 
```

Reboot to verify the hostname sticks.

# SSH Server

To make Raspberry PI secure we will:
* configure SSH access through SSH key
* disable SSH login through username / password

## Generate an ssh key-pair
On your PC generate an ssh key pair by executing the following command:
```bash
ssh-keygen
```
# TODO

Install openssh-client (might already be installed)  
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install openssh-client
```
Now you can connect from your PC to raspberry pi:
```bash
ssh ubuntu@
```

# Fixed IP 
Having a fixed IP will be important later on when the Kubernetes nodes will be communicating with each other.
Not only that, but it makes it easier to know to which Raspberry PI you are connecting yourself to through SSH.
You also have the possibility to connect to the Raspberry PI through its hostname which was configured before. 

They will always search for each other under the same IP address.  
This part is dependent on your router model, which is why you will have to   
read up on your router model, on how to assign the same IP address 
to the Raspberry PI's mac-address every time it connects to the router.

My model is a TP-Link Archer AX1500 Wi-Fi 6 Router.  
The UI can be accessed through http://192.168.1.1/.
The way I have to configure it is by logging in to the UI, then navigate to:  
Advanced > Network > DHCP Server  
Here under "Address Reservation" I have to add the MAC address of the Raspberry PI  
and the desired IP address, ex. 192.168.1.100.

# Docker


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
