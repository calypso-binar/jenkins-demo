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
If that's not the case, you will be logged in with root at startup.  

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

To clean up the user, you can run the following command:
```bash
sudo userdel ubuntu
sudo rm -rf /home/ubuntu
```

## Wireless 

> [!NOTE]
> This part might already be obsolete, depending on how you configured the installed OS on the micro sd card.   
Raspberry PI imager lets you configure Wi-Fi for the OS even before it is written to the micro sd card.  
Nevertheless, one might have forgotten to configure the Wi-Fi / might want to reconfigure it,   
which is why this part will be here.

For this configuration, you will need to connect your Raspberry PI to a monitor with the micro HDMI cable.  
You will also need a keyboard attached to the Raspberry PI.

```bash
# find your Wi-Fi adapter name, ex. wlan0
ls /sys/class/net
# generate a password from SSID (name of your Wi-Fi access point) and your Wi-Fi password 
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
Now your Raspberry PI should be connected to your Wi-Fi.

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
# replace the old hostname with new one (some examples above are provided if you need one)
# hostname must match with the one in /etc/hostname

# check if you have cloud-init installed
cloud-init status
# if you have cloud-init installed, then it will display something like "status: done"
# also if cloud-init is installed, then you will need to perform the following step:
sudo vi /etc/cloud/cloud.cfg
# change `preserve_hostname: false` to `preserve_hostname: true` 
```

Reboot to verify the hostname sticks.

# SSH Server

To make Raspberry PI secure, we will:
* configure SSH access through an SSH key
* disable SSH login through username / password

Install openssh-client (might already be installed)  
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install openssh-client --yes
```
Remember before when we created the ubuntu user? Now you can connect from your PC to raspberry pi with that user:
```bash
ssh ubuntu@raspberry-pi-master
```
or with ip:
```bash
ssh ubuntu@192.168.1.100
```


## Generate an ssh key-pair
On your PC, generate an ssh key pair by executing the following command in PowerShell:
```powershell
cd $env:USERPROFILE\.ssh
ssh-keygen
```
This command will start your ssh key generation process. Initial name and path will be `~/.ssh/id_rsa`.  
You should rename it to the Raspberry PI-s hostname, so you know which one to use it with,  
example `$env:USERPROFILE\.ssh\raspberry-pi-master`, `$env:USERPROFILE\.ssh\raspberry-pi-worker-1`, etc.   

The process will also ask you for a password. It is advised to use a password for your ssh keys,  
even though technically you can leave the password empty. Leaving the password empty will be a security risk though.  

You should generate separate ssh keys for every raspberry pi you have with different passwords.

## Add public key to authorized keys on the Raspberry PI
Here you will add the public key's content from your PC to the `/home/ubuntu/.ssh/authorized_keys` on the Raspberry PI.
This can be done either manually by copying the contents of the previously created public key and pasting the content
into the `/home/ubuntu/.ssh/authorized_keys` file, or by command line. It's your choice.    
Command line in PowerShell:  
```powershell
type $env:USERPROFILE\.ssh\raspberry-pi-5-master-1.pub | ssh ubuntu@raspberry-pi-5-master-1 "cat >> .ssh/authorized_keys"
```

## Check SSH connection works
To log in with your SSH key, you can perform the following command in PowerShell:
```powershell
ssh -i $env:USERPROFILE\.ssh\raspberry-pi-5-master ubuntu@raspberry-pi-5-master
```
This will match the private key against the public key on the Raspberry PI in `/home/ubuntu/.ssh/authorized_keys`.  
You will also have to supply the password for the SSH key if you generated it with a password.
If all goes well, you will be logged in on the Raspberry PI with the ubuntu user.

## Disable Username / Password Login

```bash
sudo vi /etc/ssh/sshd_config
# add to the end of the file:
PasswordAuthentication no
# :wq
sudo service ssh restart
```

# Firewall

TODO ufw
https://serverastra.com/docs/Tutorials/Setting-Up-and-Securing-SSH-on-Ubuntu-22.04%3A-A-Comprehensive-Guide

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

Everything was copy-pasted from here: https://docs.docker.com/engine/install/ubuntu/  

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
## add ubuntu user to the docker group

```bash
sudo usermod -a -G docker ubuntu
```

# Kubernetes

https://kubernetes.io/docs/setup/production-environment/container-runtimes/  

## OS Configuration

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without a reboot
sudo sysctl --system
# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay
# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

```bash
# Configure containerd so that it starts using systemd as cgroup, and disabled_plugins is empty.
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
```

## Installation
```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Reconfigure crictl endpoint to use containerd runtime instead of dockershim:
```bash
sudo crictl config --set runtime-endpoint=unix:///var/run/containerd/containerd.sock --set image-endpoint=unix:///var/run/containerd/containerd.sock
```

## Control Plane Initialization
```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 #pod network cidr is important for flannel later
# Output should be something like:
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.100:6443 --token 86h57g.2cm8b0u9dc49ao8a \
        --discovery-token-ca-cert-hash sha256:757a340b5de4e10aff15424d34b75aed5ea794c346b22db612e961ba60409b53

```

## Highly Available Control Plane (Optional)

### Theory
> [!INFO]
> This part is optional. 
> In this doc we will be only using a single control plane since we concentrate on 4 Raspberry PIs.
> If you want highly available control planes you will need more than 4 Raspberry PIs.
>
**In theory,** to achieve the best scalability, the least resource utilisation, security and availability,  
the Kubernetes cluster **must have at least three control planes**, where:
* control plane and etcd server should reside on their own separate nodes
* the load balancer should be on its own separate node
  * load balancer should also be highly available

Why three?  
Because Kubernetes will not work with only two control planes. It is as good as having only one.  
The following table illustrates how many control planes are necessary to tolerate failure:

| Cluster Size | Majority | Failure Tolerance |
|--------------|----------|-------------------|
| 1            | 1        | 0                 |
| 2            | 2        | 0                 |
| 3            | 2        | 1                 |
| 4            | 3        | 1                 |
| 5            | 3        | 2                 |
| 6            | 4        | 2                 |
| 7            | 4        | 3                 |

#### Option 1: External etcd Cluster

```
          ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    
          │ Worker Node │ │ Worker Node │ │ Worker Node │ │ Worker Node │ │ Worker Node │  
          └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘    
                 │               │               ↓               │               │
                 │               └──>┌────────────────────────┐<─┘               │
                 └──────────────────>│ Load Balancer (master) │<─────────────────┘
                                     └──┬─────────┬────────┬──┘
                                        │         │        │ ↑     ┌─────────────────────────┐  
                                        │         │        │ └─────│ Load Balancer (Standby) │  
                                        │         │        │       └─────────────────────────┘
                 ┌──────────────────────┘         │        └─────────────────────┐                   
    ┌────────────│──────────────┐   ┌─────────────│─────────────┐   ┌────────────│──────────────┐
    │     Control│Plane Node    │   │      Control│Plane Node   │   │     Control│Plane Node    │
    │   ┌────────┴───────────┐  │   │   ┌─────────┴──────────┐  │   │   ┌────────┴───────────┐  │
    │ ┌─┤     Api Server     │  │   │ ┌─┤     Api Server     │  │   │ ┌─┤     Api Server     │  │
    │ │ └────────────────────┘  │   │ │ └────────────────────┘  │   │ │ └────────────────────┘  │
    │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │
    │ │ │ Controller Manager │  │   │ │ │ Controller Manager │  │   │ │ │ Controller Manager │  │
    │ │ └────────────────────┘  │   │ │ └────────────────────┘  │   │ │ └────────────────────┘  │
    │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │
    │ │ │     Scheduler      │  │   │ │ │     Scheduler      │  │   │ │ │     Scheduler      │  │
    │ │ └────────────────────┘  │   │ │ └────────────────────┘  │   │ │ └────────────────────┘  │
    └─│─────────────────────────┘   └─│─────────────────────────┘   └─│─────────────────────────┘ 
    ┌─│───────────────────────────────│───────────────────────────────│─────────────────────────┐  
    │ │  external etcd cluster        │                               │                         │
    │ │ ┌────────────────────┐        │ ┌────────────────────┐        │ ┌────────────────────┐  │
    │ └>│     etcd host      │        └>│     etcd host      │        └>│     etcd host      │  │
    │   └────────────────────┘          └────────────────────┘          └────────────────────┘  │
    └───────────────────────────────────────────────────────────────────────────────────────────┘
```
#### Option 2: Stacked etcd Topology
There is also the possibility to put the etcd host on the control plane directly (stacked etcd topology).  
Drawback is that if one of the control plane dies, it takes the etcd host with it, compromising redundancy.  
You can mitigate this risk by using a minimal of three control-planes. 
```
          ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    
          │ Worker Node │ │ Worker Node │ │ Worker Node │ │ Worker Node │ │ Worker Node │  
          └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘    
                 │               │               ↓               │               │
                 │               └──>┌────────────────────────┐<─┘               │
                 └──────────────────>│ Load Balancer (master) │<─────────────────┘
                                     └──┬─────────┬────────┬──┘
                                        │         │        │ ↑     ┌─────────────────────────┐  
                                        │         │        │ └─────│ Load Balancer (Standby) │  
                                        │         │        │       └─────────────────────────┘
                 ┌──────────────────────┘         │        └─────────────────────┐                   
    ┌────────────│──────────────┐   ┌─────────────│─────────────┐   ┌────────────│──────────────┐
    │     Control│Plane Node    │   │      Control│Plane Node   │   │     Control│Plane Node    │
    │   ┌────────┴───────────┐  │   │   ┌─────────┴──────────┐  │   │   ┌────────┴───────────┐  │
    │ ┌─┤     Api Server     │  │   │ ┌─┤     Api Server     │  │   │ ┌─┤     Api Server     │  │
    │ │ └────────────────────┘  │   │ │ └────────────────────┘  │   │ │ └────────────────────┘  │
    │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │
    │ │ │ Controller Manager │  │   │ │ │ Controller Manager │  │   │ │ │ Controller Manager │  │
    │ │ └────────────────────┘  │   │ │ └────────────────────┘  │   │ │ └────────────────────┘  │
    │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │
    │ │ │     Scheduler      │  │   │ │ │     Scheduler      │  │   │ │ │     Scheduler      │  │
    │ │ └────────────────────┘  │   │ │ └────────────────────┘  │   │ │ └────────────────────┘  │  
    │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │   │ │ ┌────────────────────┐  │
    │ └>│     etcd host      │  │   │ └>│     etcd host      │  │   │ └>│     etcd host      │  │
    │   └────────────────────┘  │   │   └────────────────────┘  │   │   └────────────────────┘  │
    └───────────────────────────┘   └───────────────────────────┘   └───────────────────────────┘   
```

> [!INFO]
> There are multiple other options where you can combine different setups, 
> but these two are the recommended ones by Kubernetes. 

##### Option 3: Stacked etcd Topology with keepalived and haproxy in the Control Plane Cluster as Static Pods

Install keepalived and haproxy:
```bash
apt-get update
apt-get upgrade 
sudo apt-get install keepalived haproxy --yes
# The next part is necessary, since we want haproxy and keepalived to run on the control-planes, not in the OS.
# You can run the haproxy, keepalived in the OS too if you want. 
# In that case, skip the disabling, and skip the static pod manifests.
systemctl disable haproxy --now
systemctl disable keepalived --now
```

Configure keepalived:
```bash
vi /etc/keepalived/keepalived.conf

global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state ${STATE}
    interface ${INTERFACE}
    virtual_router_id ${ROUTER_ID}
    priority ${PRIORITY}
    authentication {
        auth_type PASS
        auth_pass ${AUTH_PASS}
    }
    virtual_ipaddress {
        ${APISERVER_VIP}
    }
    track_script {
        check_apiserver
    }
}
# ${STATE} is MASTER for one and BACKUP for all other hosts, hence the virtual IP will initially be assigned to the MASTER.
# ${INTERFACE} is the network interface taking part in the negotiation of the virtual IP, e.g. eth0, wlan0.
# ${ROUTER_ID} should be the same for all keepalived cluster hosts while unique amongst all clusters in the same subnet. 
# Many distros pre-configure its value to 51.
# ${PRIORITY} should be higher on the control plane node than on the backups. Hence 101 and 100 respectively will suffice.
# ${AUTH_PASS} should be the same for all keepalived cluster hosts, e.g. 42
# ${APISERVER_VIP} is the virtual IP address negotiated between the keepalived cluster hosts.# 
# :wq
```
```bash
# create the shell script for the keepalived config:
mkdir -p /usr/local/etc/keepalived/
vi /etc/keepalived/check_apiserver.sh
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

APISERVER_DEST_PORT='6443'
APISERVER_VIP='192.168.1.150'

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi
# :wq
chmod +x /etc/keepalived/check_apiserver.sh
```

Configure haproxy:
```bash
vi /etc/haproxy/haproxy.cfg
# /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:${APISERVER_DEST_PORT}
    mode tcp
    option tcplog
    default_backend apiserverbackend


#---------------------------------------------------------------------
# stats frontend which serves metrics from HAProxy
#---------------------------------------------------------------------
frontend stats
   bind *:8404
   http-request use-service prometheus-exporter if { path /metrics }
   stats enable
   stats uri /stats
   stats refresh 10s

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserverbackend
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server ${HOST1_ID} ${HOST1_ADDRESS}:${APISERVER_SRC_PORT}check inter 1s
        # [...]

# Placeholders to expand:  
# ${APISERVER_DEST_PORT} the port through which Kubernetes will talk to the API Server. example: 6443
# ${APISERVER_SRC_PORT} the port used by the API Server instances example: 6443
# ${HOST1_ID} a symbolic name for the first load-balanced API Server host  example: raspberry-pi-5-master-1
# ${HOST1_ADDRESS} a resolvable address (DNS name, IP address) for the first load-balanced API Server host. example: 192.168.1.100
additional server lines, one for each load-balanced API Server host
```

Create the static pod manifest for keepalived:
```bash
vi /etc/kubernetes/manifests/keepalived.yaml
# paste the following content:
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: keepalived
  namespace: kube-system
spec:
  containers:
  - image: angelnu/keepalived
    name: keepalived
    resources:
      limits:
        cpu: "100m"      # max 1 CPU
        memory: "1Gi" # max 1GiB of RAM
      requests:
        cpu: "100m"  # request 0.5 CPU
        memory: "500Mi" # request 0.5GiB of RAM
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_BROADCAST
        - NET_RAW
    volumeMounts:
    - mountPath: /etc/keepalived/keepalived.conf
      name: config
    - mountPath: /etc/keepalived/check_apiserver.sh
      name: check
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/keepalived/keepalived.conf
    name: config
  - hostPath:
      path: /etc/keepalived/check_apiserver.sh
    name: check
status: {}
# :wq
```
Create the static pod manifest for haproxy:
```bash
vi /etc/kubernetes/manifests/haproxy.yaml
# paste the following content:
apiVersion: v1
kind: Pod
metadata:
  name: haproxy
  namespace: kube-system
spec:
  containers:
  - image: haproxy:2.9.1
    name: haproxy
    resources:
      limits:
        cpu: "100m"
        memory: "1Gi" # max 1GiB of RAM
      requests:
        cpu: "100m"
        memory: "500Mi" # request 0.5GiB of RAM
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: localhost
        path: /healthz
        port: 8443
        scheme: HTTPS
    volumeMounts:
    - mountPath: /usr/local/etc/haproxy/haproxy.cfg
      name: haproxyconf
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
    name: haproxyconf
status: {}
# placeholder needs to be filled in: ${APISERVER_DEST_PORT}
# :wq
```

Init the control planes:
```bash
sudo kubeadm init --apiserver-cert-extra-sans=calypso-binar.com --control-plane-endpoint "192.168.1.150:6443" --upload-certs --pod-network-cidr 10.244.0.0/16
```

Join the other control planes:
```bash
# it can happen that the main control-plane already deleted the certificates. They are available only 2 hours.
# In this case, execute the command on the main control-plane, which will print the certificate-key:
sudo kubeadm init phase upload-certs --upload-certs
# still on the main control-plane get a join command. get the token and the ca-cert-hash:
kubeadm token create --print-join-command
# on the secondary control planes:
kubeadm join 192.168.1.150:6443 --token [token] --discovery-token-ca-cert-hash [ca-cert-hash] --control-plane --certificate-key [certificate-key]
```

> [!NOTE]
> There is another way to configure HA control plane with kube-vip, but as of now arp mode seems to be bugged.

## Copy admin config

There is one kubeconfig file at creation time.
That one kubeconfig file can do anything in Kubernetes.  
For restricted connections, look further down where users are created.
To enable access for the ubuntu user we'll copy the admin config to ubuntu's home directory:
```bash
mkdir -p /home/ubuntu/.kube
sudo cp /etc/kubernetes/admin.conf /home/ubuntu/.kube/config
sudo chown ubuntu:ubuntu /home/ubuntu/.kube/config
```

# Creating Users for Kubernetes

In this section, a demonstration will be made how to create a user john 
and give it rights to create, list and get pods from the default namespace.  
For more information, consult the 
[Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) documentation.

This task consists of the following:
* Generate certificates for the user. 
* Create a certificate signing request (CSR). 
* Sign the certificate using the cluster certificate authority. 
* Create a configuration specific to the user. 
* Add RBAC rules for the user or their group.

For the sake of simplicity, a user named John will be created.

```bash
# generate the private key:
openssl genpkey -out john.key -algorithm Ed25519
# generate the certificate signing request, like common name (CN) and organization (O) 
openssl req -new -key john.key -out john.csr -subj "/CN=john,/O=acmeorg"
cat john.csr | base64 | tr -d "\n"
# Content will be something like:
# LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlHaE1GVUNBUUF3SWpFT01Bd0dBMVVFQXd3RmFtOW(...)
```
Now copy the CertificateSigningRequest for Kubernetes:  
```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  request: <PASTE BASE64 CONTENT FROM BEFORE HERE>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400 # one day. increase as needed
  usages:
  - client auth
EOF
```

Now approve the certificate:
```bash
kubectl certificate approve john
```

Double-check the status of John's certificate status:
```bash
kubectl describe certificatesigningrequest/john 
```
Next, we will create a configuration specific to John:
```bash
kubectl get csr/john -o jsonpath="{.status.certificate}" | base64 -d > john.crt
```

Extract the certificate from a CertificateSigningRequest (CSR) named “john” using kubectl and jsonpath:
```bash
kubectl --kubeconfig john-kube-config config set-credentials john --client-key john.key --client-certificate john.csr --embed-certs=true
# output:
# User "john" set.
```

Now to configure the kubeconfig file:
```bash
# first get the name of the cluster:
kubectl config get-clusters
# Output:
# kubernetes
# Step 1: Set the cluster
kubectl --kubeconfig john-kube-config config set-cluster kubernetes \
  --embed-certs=true \
  --server=https://192.168.1.150:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt

# Step 2: Set the credentials
kubectl --kubeconfig john-kube-config config set-credentials john \
  --client-key=john.key \
  --client-certificate=john.csr \
  --embed-certs=true

# Step 3: Set the context
kubectl --kubeconfig john-kube-config config set-context kubernetes \
  --cluster=kubernetes \
  --user=john

# Step 4: Use the context
kubectl --kubeconfig john-kube-config config use-context kubernetes
```

Time to create a Role for John in the default namespace:
```bash
kubectl create clusterrole pod-manager --verb=create,list,get --resource=pods --namespace=default
```

Now bind the newly created role to John:
```bash
kubectl create clusterrolebinding john-pod-manager --clusterrole=pod-manager --user=john
# Output:
# clusterrolebinding.rbac.authorization.k8s.io/john-pod-manager created
```

You can verify what you can do with the role through the can-i statement:
```bash
kubectl auth can-i create deployments --namespace=default --as=john
# no 

kubectl auth can-i create secrets --namespace=default --as=john
# no

kubectl auth can-i get pods --namespace=default --as=john
# yes
```

With this done, John is able to create, list and get pods in the default namespace.

# Flannel
https://github.com/flannel-io/flannel#deploying-flannel-manually

```bash
# Needs manual creation of namespace to avoid helm error
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged

helm repo add flannel https://flannel-io.github.io/flannel/
helm repo update
helm  upgrade --install flannel --set podCidr="10.244.0.0/16" --namespace kube-flannel flannel/flannel
```

# Metallb
https://metallb.universe.tf/installation/  
Needs joined worker nodes.  

Create a `metallb-config.yaml` file:
```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-nginx-ingress-ip-addr-pool
  namespace: metallb-system
spec:
  addresses:
  - ${IP_ADDRESS}/32
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: metallb-nginx-ingress-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - metallb-nginx-ingress
  interfaces:
  - wlan0
  - eth0
```
Replace IP_ADDRESS with the control plane-s ip address.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm upgrade --install metallb --namespace metallb-system --create-namespace metallb/metallb -v
```

Once all of the metallb pods are up configure metallb with the `metallb-config.yaml`:
```bash
kubectl apply  -n metallb-system -f metallb-config.yaml
```

Now if you already have an ingress-nginx installed it should get an external-ip.  
You can check by running the following command:
```bash
kubectl get services -n ingress-nginx
# output should be something like below:
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.107.125.102   192.168.1.140   80:32599/TCP,443:31875/TCP   9m23s
ingress-nginx-controller-admission   ClusterIP      10.106.61.254    <none>          443/TCP                      9m22s
```

# Ingress-Nginx

## With SSL key and cert-chain

Get yourself a signed certificate from your paid domain name. You will also need the private key.

```bash
# if you have an SSL Certificate and Key
kubectl create namespace ingress-nginx
kubectl create secret tls no-ip-ssl-cert --key calypso-binar.key --cert calypso-binar_com.pem -n ingress-nginx
# kubectl -n ingress-nginx create secret generic no-ip-ssl-cert --from-file=./calypso-binar_com.pem
```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
# use controller.extraArgs.default-ssl-certificate only if you have a certificate installed on Kubernetes as a secret.
# otherwise, Kubernetes will provide a self-signed certificate
helm upgrade --install --force ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace \
--set controller.extraArgs.default-ssl-certificate="ingress-nginx/no-ip-ssl-cert" --set "controller.extraArgs.enable-ssl-passthrough="
```

## With cert-manager and let's encrypt certificate

For this to work you will need to install cert-manager on Kubernetes:  
https://cert-manager.io/docs/installation/helm/

create a `issuer-lets-encrypt-production.yaml`:
```yaml
# issuer-lets-encrypt-production.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <email-address> # ❗ Replace this with your email address
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
        ingress:
          name: web-ingress
```

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install --force cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
kubectl apply -f issuer-lets-encrypt-production.yaml
```

Configure cert-manager by creating an issuer custom resource:

issuer-lets-encrypt-production.yaml  
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <YOUR E-MAIL HERE!!!>
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

```bash
kubectl apply -f issuer-lets-encrypt-production.yaml
```

You will install ingress-nginx as you usually would:  

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install --force ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

Now, for the interesting part. When creating an ingress, you will have to annotate it with the cert-manager cluster-issuer annotation.  
You will also point to the letsencrypt secret, which contains the tls certificate.
Example:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
      - calypso-binar.com
      secretName: letsencrypt-production
  ingressClassName: nginx
  rules:
  - host: calypso-binar.com
    http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: web-app
            port:
              number: 80
```


# NFS with NAS (Network Attached Storage) with USB
TODO: highly available NAS

# hardware
First we will mount a USB Stick (or external SSD) to `/media/usb`.
On one of the kubernetes nodes (you decide which0) put a USB stick / external SSD into a USB.

## nfs-kernel-server
This piece of software will be able to serve directories as network attached storage (NAS).  
In our case, the mounted usb device will be our NAS.

https://ubuntu.com/server/docs/service-nfs
Install nfs-kernel-server on a machine.
```bash
sudo apt install nfs-kernel-server network-manager --yes
```

To find out which device the USB is written:
```bash
lsblk
# output will be something like:
AME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0  49.1M  1 loop /snap/core18/2810
loop1         7:1    0  69.1M  1 loop /snap/core22/1035
loop2         7:2    0  68.5M  1 loop /snap/core22/867
loop3         7:3    0   9.6M  1 loop /snap/helm/374
loop4         7:4    0   169M  1 loop /snap/lxd/25953
loop5         7:5    0 134.8M  1 loop /snap/lxd/26202
loop6         7:6    0  35.5M  1 loop /snap/snapd/20298
sda           8:0    0 465.8G  0 disk
└─sda1        8:1    0 465.8G  0 part
mmcblk0     179:0    0  59.5G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0    59G  0 part /
```
In this case, the sda1 is the USB device. It has ~465 GB, therefore, it is my external SSD.

Now we will mount the USB device, example with ext4 file system:  
```bash
systemctl stop nfs-server
fdisk /dev/sda
# follow instructions here: https://www.redips.net/linux/create-fat32-usb-drive/
p # prints device info
d # deletes partition (delete all partitions if there are more)
n # new partition
p # primary partition
# leave everything on default, except Y for remove signature 
t # set partition type
83 # alias for Linux
w # write chnges

sudo mkfs.ext4 /dev/sda1
sudo mkdir -p /media/usb
lsblk -o NAME,FSTYPE,UUID

NAME        FSTYPE UUID
loop0
loop1
loop2
loop3
loop4
loop5
loop6
sda
└─sda1      ext4   a4974eea-9365-412e-8442-a5ed5eec538f
mmcblk0
├─mmcblk0p1 vfat   81C9-1C52
└─mmcblk0p2 ext4   d78ea5c7-a075-476e-acdd-a16cc2187931

vi /etc/fstab
# TABs are important
UUID=a4974eea-9365-412e-8442-a5ed5eec538f  /media/usb      ext4    defaults        0       0

# sudo mount -t ext4 /dev/sda1 /media/usb -o uid=1000,gid=1000,utf8,dmask=027,fmask=137
```

Now you can write to /media/usb whatever you want. 

Export the /media/usb to /etc/exports on an IP range allowed to access it:
```bash
vi /etc/exports
# paste at the end the following:
/media/usb 192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)
```

restart the nfs server:
```
systemctl restart nfs-kernel-server
```

## nfs-common
We want to access this USB on the worker nodes. Therefore, we will install nfs-common on all of them.
```bash
sudo apt-get install nfs-common network-manager --yes
sudo mkdir -p /media/usb
# Mount the USB form the nfs manually:
sudo mount -t nfs 192.168.1.100:/media/usb /media/usb
# boot mount:
vi /etc/fstab
192.168.1.100:/media/usb    /media/usb   nfs defaults 0 0
# :wq
```

Note: the IP address is the host IP of the usb device.

# Use NFS in Kubernetes
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
helm upgrade --install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
 --namespace nfs --create-namespace \
 --set nfs.server=192.168.1.100 \
 --set nfs.path=/media/usb \
 --set storageClass.defaultClass=true \
 --set storageClass.accessModes=ReadWriteMany
```

This will enable you to use a storage class named `nfs-client` in Kubernetes. To verify:
```bash
kubectl get storageclasses
# output:
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   78s
```

# Monitoring

create a `kube-prometheus-stack-values.yaml` file:
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
  envFromSecret: grafana-github
  grafana.ini:
    server:
      root_url: https://calypso-binar.com/grafana # this host can be localhost
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
    auth:
      disable_login_form: true
    auth.github:
      enabled: true
      allow_sign_up: true
      allow_assign_grafana_admin: true
      role_attribute_strict: true
      role_attribute_path: contains(groups[*], '@SpringStoreOrg/admin') && 'GrafanaAdmin' || contains(groups[*], '@SpringStoreOrg/dev') && 'Editor' || 'Viewer'
      scopes: user:email,read:org
      auth_url: https://github.com/login/oauth/authorize
      token_url: https://github.com/login/oauth/access_token
      api_url: https://api.github.com/user
      team_ids: admin
      allowed_organizations: SpringStoreOrg
              #      client_id: c363c8d4734b7a166c09
      #      client_secret: 3254b560314dd2858dde2a049e70c815d6a16d9d
      client_id: ${client-id}
      client_secret: ${client-secret}

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
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: haproxy
        static_configs:
        - targets: ['192.168.1.100:8404','192.168.1.101:8404','192.168.1.102:8404']
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

Please note, that the GitHub authentication is configured for SpringStoreOrg, change this as needed.  
SpringStoreOrg has admin, and dev groups. You will use your own groups as needed.

You will also need a kubernetes secret called grafana-github in the monitoring namespace 
configured with a GitHub client id and client secret.  
`grafana-github-secret.yaml`:  
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-github
  namespace: monitoring
type: Opaque
data:
  client-id: <GITHUB OAUTH APP CLIENT ID>
  client-secret: <GITHUB OAUTH APP CLIENT SECRET>
```

```bash
kubectl apply -f grafana-github-secret.yaml
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
helm repo update  
helm upgrade --install --force -n monitoring --create-namespace kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
```

# Tracing

## Loki-Stack
https://github.com/grafana/helm-charts/tree/main/charts/loki-stack

loki-stack-values.yaml:
```yaml
loki:
  config:
    auth_enabled: false
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install --force -n monitoring --create-namespace loki-stack grafana/loki-stack -f loki-stack-values.yaml
```


# MariaDB
https://www.datree.io/helm-chart/mariadb-bitnami

mariadb-values.yaml:
```yaml
auth:
  rootPassword: "YOUR PW HERE"
```

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install --force -n mariadb --create-namespace mariadb bitnami/mariadb -f mariadb-values.yaml
```


# Jenkins

## Master

https://github.com/jenkinsci/helm-charts
https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/values.yaml  
https://github.com/CenterForOpenScience/helm-charts/blob/master/jenkins/README.md
https://github.com/jenkinsci/configuration-as-code-plugin/blob/master/README.md#introduction

jenkins-values.yaml (it's a long one...):

```yaml
persistence:
  enabled: true
  storageClass: nfs-client
  size: 10Gi
controller:
  jenkinsUriPrefix: "/jenkins"
  ingress:
    enabled: true
    ingressClassName: nginx
    path: "/jenkins"
    hostName: calypso-binar.com
  JCasC:
    defaultConfig: false
    configScripts:
      jenkinscasc: |
        jenkins:
          agentProtocols:
            - "JNLP4-connect"
            - "Ping"
          authorizationStrategy:
            loggedInUsersCanDoAnything:
              allowAnonymousRead: false
          clouds:
            - kubernetes:
                containerCap: 10
                containerCapStr: "10"
                jenkinsTunnel: "jenkins-agent.default.svc.cluster.local:50000"
                jenkinsUrl: "http://jenkins.default.svc.cluster.local:8080/jenkins"
                name: "kubernetes"
                namespace: "default"
                podLabels:
                  - key: "jenkins/jenkins-jenkins-agent"
                    value: "true"
                serverUrl: "https://kubernetes.default"
                templates:
                  - containers:
                      - args: "^${computer.jnlpmac} ^${computer.name}"
                        envVars:
                          - envVar:
                              key: "JENKINS_URL"
                              value: "http://jenkins.default.svc.cluster.local:8080/jenkins"
                        image: "fractalwoodstories/jenkins-inbound-agent:latest"
                        name: "jnlp"
                        resourceLimitCpu: "4096m"
                        resourceLimitMemory: "4096Mi"
                        resourceRequestCpu: "2048m"
                        resourceRequestMemory: "2048Mi"
                        workingDir: "/home/jenkins/agent"
                    id: "923e6d6b3128baaa56764f4b69d4c62b61e55d00f8170e3428f011148767dc99"
                    label: "jenkins-jenkins-agent"
                    name: "default"
                    namespace: "default"
                    nodeUsageMode: NORMAL
                    podRetention: "never"
                    serviceAccount: "default"
                    slaveConnectTimeout: 100
                    slaveConnectTimeoutStr: "100"
                    volumes:
                    - nfsVolume:
                        mountPath: "/home/jenkins/.m2"
                        readOnly: false
                        serverAddress: "192.168.1.100"
                        serverPath: "/media/usb/.m2"
                    - hostPathVolume:
                        hostPath: "/var/run/docker.sock"
                        mountPath: "/var/run/docker.sock"
                        readOnly: false
                    yamlMergeStrategy: "override"
          crumbIssuer:
            standard:
              excludeClientIPFromCrumb: true
          disableRememberMe: false
          labelAtoms:
            - name: "built-in"
            - name: "jenkins-jenkins-agent"
          markupFormatter: "plainText"
          mode: NORMAL
          myViewsTabBar: "standard"
          numExecutors: 0
          primaryView:
            all:
              name: "all"
          projectNamingStrategy: "standard"
          quietPeriod: 5
          remotingSecurity:
            enabled: true
          scmCheckoutRetryCount: 0
          securityRealm:
            local:
              allowsSignup: false
              enableCaptcha: false
              users:
                - id: "${chart-admin-username}"
                  password: "${chart-admin-password}"
                  name: "Jenkins Admin"
                  properties:
                    - "apiToken"
                    - "mailer"
                    - "myView"
                    - preferredProvider:
                        providerId: "default"
                    - "timezone"
                    - "experimentalFlags"
          slaveAgentPort: 50000
          updateCenter:
            sites:
              - id: "default"
                url: "https://updates.jenkins.io/update-center.json"
          views:
            - all:
                name: "all"
          viewsTabBar: "standard"
        globalCredentialsConfiguration:
          configuration:
            providerFilter: "none"
            typeFilter: "none"
        security:
          apiToken:
            creationOfLegacyTokenEnabled: false
            tokenGenerationOnCreationEnabled: false
            usageStatisticsEnabled: true
          gitHooks:
            allowedOnAgents: false
            allowedOnController: false
          gitHostKeyVerificationConfiguration:
            sshHostKeyVerificationStrategy: "knownHostsFileVerificationStrategy"
        unclassified:
          buildDiscarders:
            configuredBuildDiscarders:
              - "jobBuildDiscarder"
          fingerprints:
            fingerprintCleanupDisabled: false
            storage: "file"
          location:
            adminAddress: "address not configured yet <nobody@nowhere>"
            url: "http://calypso-binar.com/jenkins/"
          mailer:
            charset: "UTF-8"
            useSsl: false
            useTls: false
          pollSCM:
            pollingThreadCount: 10
          scmGit:
            addGitTagAction: false
            allowSecondFetch: false
            createAccountBasedOnEmail: false
            disableGitToolChooser: false
            hideCredentials: false
            showEntireCommitSummaryInChanges: false
            useExistingAccountWithSameEmail: false
        tool:
          git:
            installations:
              - home: "git"
                name: "Default"
          mavenGlobalConfig:
            globalSettingsProvider: "standard"
            settingsProvider: "standard"
```

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm upgrade --install --force --namespace jenkins --create-namespace jenkins jenkins/jenkins -f jenkins-values.yaml
```

# Argo CD

https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install --force --namespace argocd --create-namespace argocd argo/argo-cd
```


Install plugins:
https://plugins.jenkins.io/github/
https://plugins.jenkins.io/github-oauth/
https://plugins.jenkins.io/ansicolor/
# https://plugins.jenkins.io/blueocean/
https://plugins.jenkins.io/pipeline-stage-view/

Github app:
bb2aeb8440696e98400c
95fdca3b7bd1f9dbc401e34841bd9d26d8a703e4

# Sonatype Nexus 3

The docker image has to be built for Raspbery PI.
Clone:  https://github.com/sonatype/docker-nexus3
```bash
docker build . -t fractalwoodstories/nexus-arm64-3.64.0
docker push fractalwoodstories/nexus-arm64-3.64.0
```

create values file:
```yaml
image:
  repository: fractalwoodstories/nexus-arm64
  tag: 3.64.0
ingress:
  enabled: true
  hostPath: /nexus
  hostRepo: calypso-binar.com
nexus:
  env:
    - name: NEXUS_CONTEXT
      value: nexus
  livenessProbe:
    path: /nexus
  readinessProbe:
    path: /nexus
```


```bash
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
helm upgrade --install --force -n nexus --create-namespace nexus-rm sonatype/nexus-repository-manager --values nexus-values.yaml
```

# change / add new host to kubernetes certificates

https://stackoverflow.com/questions/64470897/generating-a-new-conf-files-on-kubeadm-1-13-12  
https://blog.scottlowe.org/2019/07/30/adding-a-name-to-kubernetes-api-server-certificate/  

```bash
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml  

kubeadm init phase certs apiserver --config kubeadm.yaml  
mv /etc/kubernetes/pki /etc/kubernetes/pki-backup  
mv /etc/kubernetes/admin.conf /etc/kubernetes/admin.conf.backup  
kubeadm init phase certs all  
kubeadm init phase kubeconfig admin  
crictl stopp $(crictl pods | grep kube-apiserver | cut -d' ' -f1) && crictl rmp $(crictl pods | grep kube-apiserver | cut -d' ' -f1)  

kubeadm init phase upload-config kubeadm --config kubeadm.yaml
```
