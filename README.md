# Kubernetes Cluster

Kubernetes es uno de los sistemas de orquestación de contenedores de código abierto más populares. Se utiliza para administrar toda la vida de las aplicaciones en contenedores, incluida la implementación, el escalado, la actualización, etc.

# Diseño de arquitectura

| Nombre del server  | Dirrecion IP |
|---|---|
| master01  | 192.168.100.4  |
| master02  |  192.168.100.5 |

# Prerrequisitos

- Cada servidor tiene al menos 2 cursos de CPU / vCPU, 4 GB de RAM y 10 GB de espacio en disco.
- Todos los servidores deben tener acceso a Internet para descargar paquetes de software.
- El sistema operativo en ellos es CentOS7 con usuario root habilitado.

# Configuracion inicial

### Desactiva el Selinux
- $ setenforce 0
- $ sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux

### Desactivar Firewalld
- $ systemctl disable firewalld
- $ systemctl stop firewalld

### Desactivar intercambio
- $ swapoff -a
- $ sed -i 's/^.*swap/#&/' /etc/fstab  

### Habilite el reenvío
- $ iptables -P FORWARD ACCEPT
```
$ cat <  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
```
- $ sysctl --system     

Edite el archivo / etc / hosts para que contenga lo siguiente:

- master01 192.168.100.4
- master02 192.168.100.5

### Instalar y configurar Docker

```
$ wget https://download.docker.com/linux/static/stable/x86_64/docker-17.03.2-ce.tgz
$ tar -zxvf docker-17.03.2-ce.tgz
$ cd docker
$ cp * /usr/local/bin
```

Cambie el contenido de /etc/systemd/system/docker.service a lo siguiente

```
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io
[Service]
ExecStart=/usr/local/bin/dockerd 
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
```
Habilite el servicio Docker y vuelva a cargar la configuración ejecutando los siguientes comandos

- $ systemctl daemon-reload 
- $ systemctl enable docker 
- $ systemctl restart docker

#  Instale kubeadm, kubectl y kubelet

```
$ cat <  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

$ yum makecache fast && yum install -y kubelet-1.10.0-0 kubeadm-1.10.0-0 kubectl-1.10.0-0

$ cat <  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
EOF

$ systemctl daemon-reload 
$ systemctl enable kubelet 
$ systemctl restart kubelet

```

# Configuración del clúster de Kubernetes

## Generando archivos de configuración maestra
- $ curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
- $ curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
- $ chmod +x /usr/local/bin/cfssl*

Cree un archivo de configuración de certificado /opt/ssl/*

- ca-config.json
- ca-csr.json
- etcd-csr.json

- $ cd /opt/ssl/ca-config.json
- $ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
- $ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

## Copiar archivos de configuración a la carpeta de instalación

Para copiar los archivos de configuración generados en el último paso a la carpeta de instalación, debe realizar las siguientes tareas en los 2 nodos maestros.

Crea la carpeta del certificado

* $ mkdir -p /etc/etcd/ssl

Crea el director de datos etcd

* $ mkdir -p /var/lib/etcd

Copie certificados etcd del nodo maestro "Master01"

* $ scp -rp 192.168.100.5:/opt/ssl/*.pem /etc/etcd/ssl/

### Instalar etcd

* $ wget https://github.com/coreos/etcd/releases/download/v3.3.4/etcd-v3.3.4-linux-amd64.tar.gz
* $ tar -zxvf etcd-v3.3.4-linux-amd64.tar.gz
* $ cp etcd-v3.3.4-linux-amd64/etcd* /usr/local/bin/

# Configuración del clúster ETCD

### Master 01

Cree el archivo /etc/systemd/system/etcd.service con el siguiente contenido
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
--name=master01 \
--cert-file=/etc/etcd/ssl/etcd.pem \
--key-file=/etc/etcd/ssl/etcd-key.pem \
--peer-cert-file=/etc/etcd/ssl/etcd.pem \
--peer-key-file=/etc/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
--initial-advertise-peer-urls=https://192.168.100.04:2380 \
--listen-peer-urls=https://192.168.100.04:2380 \
--listen-client-urls=https://192.168.100.04:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://192.168.100.04:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=master01=https://192.168.100.04:2380,master02=https://192.168.100.05:2380\
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Iniciar servicio etcd

- $ systemctl daemon-reload && systemctl enable etcd && systemctl start etcd

### Master 02

Cree el archivo /etc/systemd/system/etcd.service con el siguiente contenido
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
--name=master01 \
--cert-file=/etc/etcd/ssl/etcd.pem \
--key-file=/etc/etcd/ssl/etcd-key.pem \
--peer-cert-file=/etc/etcd/ssl/etcd.pem \
--peer-key-file=/etc/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
--initial-advertise-peer-urls=https://192.168.100.05:2380 \
--listen-peer-urls=https://192.168.100.05:2380 \
--listen-client-urls=https://192.168.100.05:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://192.168.100.05:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=master01=https://192.168.100.04:2380,master02=https://192.168.100.05:2380\
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Iniciar servicio etcd

- $ systemctl daemon-reload && systemctl enable etcd && systemctl start etcd
