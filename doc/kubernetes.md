# TODO


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

 # Prometheus

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
      nginx.ingress.kubernetes.io/rewrite-target: /$1
      nginx.ingress.kubernetes.io/use-regex: "true"

    paths:
      - /prometheus/?(.*)
    pathType: ImplementationSpecific
    hosts:
      - calypso-binar.com
  prometheusSpec: 
    externalUrl: "https://calypso-binar.com/prometheus" #TODO 
    routePrefix: "/prometheus" #TODO

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
 
 helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
 helm repo update
 helm upgrade --install --force -n monitoring --create-namespace kube-prometheus-stack prometheus-community/kube-prometheus-stack -f values.yaml
