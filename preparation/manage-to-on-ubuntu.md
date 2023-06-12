# Preparing the “manage-to” Host, on Ubuntu

# The Architecture
The architecture of these labs can be illustrated like this:

<picture>
  <img alt="image" src="./assets/images/architecture.png">
</picture>

**Each student will have two VMs running Ubuntu on IBM Cloud. Access to the VM is via ssh using a cert key**

**Hardware Specs**

**The Instana Server (Manage-from)**
It must be 16+ vCPU, 64+G RAM, 100+G. Higher is better but that minimum specs should be sufficient.
The Instana Server should expose below ports:
  - TLS/443: for the Instana UI
  - TLS/1444: for the Instana agents to connect with
  - TLS/446 and HTTP/86: for End User Monitoring (EUM) agents to connect with

Even the server can be provisioned in different supported Linux distributions, commands will vary while walking through in different Linux distributions.

**The “manage-to” VM**

To simulate a hybrid infrastructure, where Kubernetes and VMs are available, to start the observability journey in a VM is not trivial.

Preferably an Ubuntu (e.g. 18.04 / 20.04) VM with 8+ CPU, 32+G RAM, 100+G Disks, with some detailed requirements:

  1. The VM’s account should have root permission, or at least can perform "sudo" for necessary software installation
  2. The VM should be Internet-connected with/without a proxy
  3. We will install Docker within the VM, use Kubernetes in Docker (Kind) to spin up Kubernetes, and use footloose to spin up container-based “VMs” on Docker

Note: if you prefer your own Linux distribution, instead of documented Ubuntu, it should work too but please adjust the commands accordingly.


## Access Manage-to VM
```sh
ssh itzuser@<manage-to ip address> -p 2223 -i <ssh key file>
```

## Mount additional disk space
```sh
lsblk
```

<picture>
  <img alt="image" src="./assets/images/disk-dev.png">
</picture>

```sh
sudo fdisk /dev/xvde
```

<picture>
  <img alt="image2" src="./assets/images/fdisk.png">
</picture>

```sh
sudo mkfs -t ext4 /dev/xvde1
```

<picture>
  <img alt="image3" src="./assets/images/mkfs.png">
</picture>

```sh
sudo mount -t ext4 /dev/xvde1 /opt

vi /etc/fstab
     /dev/xvde1 /opt ext4 defaults,noatime 0 0

```
<picture>
  <img alt="image4" src="./assets/images/fstab.png">
</picture>


## Install Docker

Remove some legacy components, if any
```sh
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

It is O.K if you see "Unable to locate package docker-engine"

If there is a need to purge the previous failed Docker install
```sh
$ sudo apt-get purge docker-ce docker-ce-cli containerd.io
```

Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```sh
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg
```

Add Docker’s official GPG key:
```sh
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Use the following command to set up the repository:
```sh
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update the apt package index:
```sh
sudo apt-get update
```

To install a specific version of Docker Engine, start by listing the available versions in the repository:
```sh
apt-cache madison docker-ce | awk '{ print $3 }'
```

Select the desired version and install:
```sh
VERSION_STRING=5:20.10.23~3-0~ubuntu-focal
sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify that the Docker Engine installation is successful by running the hello-world image:
```sh
sudo docker run hello-world
```

Moving /var/lib/docker to disk with more space
```sh
sudo systemctl stop docker
sudo mv /var/lib/docker /opt/docker
sudo ln -s /opt/docker /var/lib/docker
```

Start Docker daemon process
```sh
sudo systemctl enable docker
sudo systemctl start docker
```

Have a try for Docker
```sh
sudo docker run hello-world
```

<picture>
  <img alt="image4" src="./assets/images/dockerSuccess.png">
</picture>

Add current user into docker group so that we can run Docker cli without the need of sudo
```sh
sudo usermod -aG docker $USER
```

> Note: Please re-login to the VM to take effect.

```sh
# Try running docker without sudo
docker run hello-world
```

## Install necessary tools

1. Install “kind” – it’s a tool for creating Kubernetes in Docker, that why it’s called “kind”:

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Check the version
Should see something like: 
kind v0.16.0 go1.19.1 linux/amd64
```sh
kind --version
```

2. Install “footloose” – it’s a tool to spin up “VMs” as Docker containers:

```sh
curl -Lo footloose https://github.com/weaveworks/footloose/releases/download/0.6.3/footloose-0.6.3-linux-x86_64
chmod +x footloose
sudo mv footloose /usr/local/bin/
```

Check the version
Should see something like: 
version: 0.6.3
```sh
footloose version
```

3. Other tools

kubectl
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Check the version
should see output like: 
Client Version: v1.25.2
Kustomize Version: v4.5.7
```sh
kubectl version --short --client
```

Helm CLI
```sh
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install -y helm
```


Check the version
should see output like: 
v3.9.4+gdbc6d8e
```sh
helm version --short
```

## Spin up Kubernetes cluster


Customize a kind-config.yaml file with 1 master 3 worker nodes
You may spin up a minium cluster simply by: kind create cluster
```sh
$ cat > kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
```

Create the cluster using the file
This make take a 2-10 minutes depending on your download speed
```sh
$ kind create cluster --config kind-config.yaml
```

Output:

```sh
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.2) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:
```
```sh
kubectl cluster-info --context kind-kind
```
```sh
Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```


Verify the cluster
Note: the nodes might be in “NotReady”, just wait for a while
```sh
$ kubectl get nodes
```
```sh
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   83s   v1.25.2
kind-worker          Ready    <none>          51s   v1.25.2
kind-worker2         Ready    <none>          51s   v1.25.2
kind-worker3         Ready    <none>          64s   v1.25.2
```

## Spin up “VM”s


Create a YAML to define our VMs
```sh
$ cat > footloose.yaml <<EOF
cluster:
  name: labs
  privateKey: labs-key
machines:
- count: 1
  spec:
    image: quay.io/footloose/ubuntu18.04
    name: ubuntu-%d
    networks:
    - footloose-cluster
    portMappings:
    - containerPort: 22
    privileged: true
    volumes:
    - type: volume
      destination: /var
- count: 1
  spec:
    image: quay.io/footloose/centos7
    name: centos-%d
    networks:
    - footloose-cluster
    portMappings:
    - containerPort: 22
    privileged: true
    volumes:
    - type: volume
      destination: /var
EOF
```

Create a dedicated Docker network
```sh
$ docker network create footloose-cluster
```

Spin up VMs
```sh
$ footloose create -c footloose.yaml
```
```sh
Output:

INFO[0000] Docker Image: quay.io/footloose/ubuntu18.04 present locally
INFO[0000] Docker Image: quay.io/footloose/centos7 present locally
INFO[0000] Creating machine: labs-ubuntu-0 ...
INFO[0000] Connecting labs-ubuntu-0 to the footloose-cluster network...
INFO[0001] Creating machine: labs-centos-0 ...
INFO[0001] Connecting labs-centos-0 to the footloose-cluster network...
```

Check it out
```sh
$ footloose show -c footloose.yaml
```

```sh
Output:

NAME            HOSTNAME   PORTS           IP   IMAGE                           CMD          STATE     BACKEND
labs-ubuntu-0   ubuntu-0   0->{22 49154}        quay.io/footloose/ubuntu18.04   /sbin/init   Running
labs-centos-0   centos-0   0->{22 49155}        quay.io/footloose/centos7       /sbin/init   Running
```

Log into any of the VMs
```sh
$ footloose ssh root@ubuntu-0 -c footloose.yaml
```

Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-144-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

```sh
root@ubuntu-0:~# exit

```
