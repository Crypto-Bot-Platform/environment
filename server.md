# Server setup
## Spec:

| Item    | Spec                 |
|---------|----------------------|
| Machine | HP Proliant 380p G8  |
| Memory  | 128GB                |
| CPU     | 2 x Intel Xeon 2.9GZ |
| OS      | RHEL 7.5             |


## Environment setup 
### zsh
```
sudo dnf install zsh
sudo dnf install wget curl
sudo yum update
sudo yum install git
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

chsh -s /bin/zsh root <--- do it for your user
```
### Oh-my-zsh
```
sudo dnf install wget git
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh
```

### tmux
```
sudo dnf -y install tmux
```
Useful commands:
* Ctrl-b + c - create nwe windows 
* Ctrl-b + n - next window 
* Ctrl-b + p - previous window 
* Ctrl-b + " - horizontal split 
* Ctrl-b + o - switch slits 

### speedtest 
I use it to check network speed on my server
```
curl -s https://install.speedtest.net/app/cli/install.rpm.sh | sudo bash
sudo yum install speedtest
```

### Java
```sudo yum install java-1.8.0-openjdk```

### Python
```sudo yum install python39```

### Docker 
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce --allowerasing
sudo systemctl enable --now docker
```

#### Docker Compose 
```
curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o docker-compose
sudo mv docker-compose /usr/local/bin && sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose 
```
### Kubernetes
https://thenewstack.io/how-to-install-a-kubernetes-cluster-on-red-hat-centos-8/

Open Ports 
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd –-reload
sudo modprobe br_netfilter
sudo echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
sudo usermod -aG docker $USER
```
Install containerd.io
```
sudo dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
sudo dnf install docker-ce
```
Install Kubernetes 
`sudo vim /etc/yum.repos.d/kubernetes.repo`
Add file content:
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```
Install kubectl, kubeadm, kubelet
```
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
su
vim /etc/sysctl.d/k8s.conf
```
Add the content
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
```sysctl --system```

Disable Swap
```sudo swapoff -a```

Edit `/etc/fstab` and comment swap line

Create daemon file
`sudo vim /etc/docker/daemon.json`

with content 
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```
`mkdir -p /etc/systemd/system/docker.service.d`

```
systemctl daemon-reload
systemctl restart docker

sudo kubeadm init
```

### Kafka
```
wget https://dlcdn.apache.org/kafka/3.1.0/kafka_2.12-3.1.0.tgz
tar xvf kafka_2.12-3.1.0.tgz
sudo mv kafka_2.12-3.1.0 /opt
sudo ln -s /opt/kafka_2.12-3.1.0 /opt/kafka
sudo useradd kafka
mkdir /tmp/kafka-logs
mkdir /tmp/zookeeper/
sudo chown -R kafka:kafka /opt/kafka*
sudo chown -R kafka:kafka /tmp/kafka-logs/
sudo chown -R kafka:kafka /tmp/zookeeper/
```
Edit file `sudo vim /opt/kafka/config/server.properties`:
* `listeners=PLAINTEXT://localhost:9092`

#### Configure Kafka as a service
* Create file `/etc/systemd/system/zookeeper.service`
```
[Unit]
Description=zookeeper
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple

User=kafka
Group=kafka

ExecStart=/usr/bin/bash /opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/usr/bin/bash /opt/kafka/bin/zookeeper-server-stop.sh

[Install]
WantedBy=multi-user.targetx
```
* Create file `/etc/systemd/system/kafka.service`
```
[Unit]
Description=Apache Kafka
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple

User=kafka
Group=kafka

ExecStart=/usr/bin/bash /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/usr/bin/bash /opt/kafka/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target
```
* Start the server 
```
sudo systemctl daemon-reload
sudo systemctl start zookeeper
sudo systemctl start kafka
```

### Elastic search
https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html

```
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
Create file `sudo vim /etc/yum.repos.d/elasticsearch.repo`:

```
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
```

Install `sudo dnf install --enablerepo=elasticsearch elasticsearch`

start as a service 
```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
```

Edit file `sudo vim /etc/elasticsearch/elasticsearch.yml`:
* `cluster.name: cbp`
* `path.data: /home/lib/elasticsearch` ( move existing data folder `mv /var/lib/elasticsearch /home/lib/` )
* `network.host: 0.0.0.0`
* `discovery.type: single-node`

Start service `sudo systemctl start elasticsearch.service`

Open port:
```
sudo firewall-cmd --add-port=9200/tcp --permanent
sudo firewall-cmd --reload 
```

### Kibana 

```
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

Create file `sudo vim /etc/yum.repos.d/kibana.repo`

with content:
```
[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

```
sudo dnf install kibana
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
```

Edit file `sudo vim /etc/kibana/kibana.yml`. Add line:
```
server.host: 0.0.0.0
```

### Mongo DB 

https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/

Create file `sudo vim /etc/yum.repos.d/mongodb-org-5.0.repo`

```
[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc
```

Install 

```
sudo yum install -y mongodb-org
```

Store Data in user directory:

```
sudo mv /var/lib/mongo /home/lib
sudo vim /etc/mongod.conf
```

Edit:
* storage:
  dbPath: /home/lib/mongo
* bindIp: 0.0.0.0

Open port:
```
sudo firewall-cmd --add-port=27017/tcp --permanent
sudo firewall-cmd --reload
```

Configure SELinux

```
sudo yum install checkpolicy
cat > mongodb_cgroup_memory.te <<EOF
module mongodb_cgroup_memory 1.0;
require {
      type cgroup_t;
      type mongod_t;
      class dir search;
      class file { getattr open read };
}
#============= mongod_t ==============
allow mongod_t cgroup_t:dir search;
allow mongod_t cgroup_t:file { getattr open read };
EOF

checkmodule -M -m -o mongodb_cgroup_memory.mod mongodb_cgroup_memory.te
semodule_package -o mongodb_cgroup_memory.pp -m mongodb_cgroup_memory.mod
sudo semodule -i mongodb_cgroup_memory.pp

cat > mongodb_proc_net.te <<EOF
module mongodb_proc_net 1.0;
require {
        type sysctl_net_t;
        type mongod_t;
        class dir search;
        class file { getattr open read };
}
#============= mongod_t ==============
#!!!! This avc is allowed in the current policy
allow mongod_t sysctl_net_t:dir search;
allow mongod_t sysctl_net_t:file open;
#!!!! This avc is allowed in the current policy
allow mongod_t sysctl_net_t:file { getattr read };
EOF

checkmodule -M -m -o mongodb_proc_net.mod mongodb_proc_net.te
semodule_package -o mongodb_proc_net.pp -m mongodb_proc_net.mod
sudo semodule -i mongodb_proc_net.pp

sudo semanage fcontext -a -t mongod_var_lib_t '/home/lib/mongo.*'
sudo chcon -Rv -u system_u -t mongod_var_lib_t /home/lib/mongo
sudo restorecon -R -v /home/lib/mongo

ausearch -c 'mongod' --raw | audit2allow -M my-mongod
semodule -X 300 -i my-mongod.pp
```
### Grafana

Edit file `sudo vim /etc/yum.repos.d/grafana.repo`. Add context:

```
[grafana]
name=grafana
baseurl=https://packages.grafana.com/enterprise/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

`sudo yum install grafana`

```
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server
```

Open port:
```
sudo firewall-cmd --add-port=3000/tcp --permanent
sudo firewall-cmd --reload
```




