# Lab 1.1 – Install Instana Server Manually
## Access Instana-Server VM
```sh
ssh itzuser@<Instana Server ip address> -p 2223 -i <ssh key file>
```

## 2. Mount additional disk space
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



## 3. Check Prerequisites

```sh
# Test the connectivity
# Make sure the netcat is installed
$ sudo apt-get install netcat
$ nc -vz auth-infra.instana.io 443

# Mount or simply create some data folders for simplicity purposes
$ sudo mkdir /opt/{data,metrics,traces}
$ sudo mkdir /opt/log
$ sudo mkdir /opt/log/instana

# TLS
$ sudo curl -sLO https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64
$ sudo chmod +x mkcert-v1.4.3-linux-amd64 && sudo mv mkcert-v1.4.3-linux-amd64 /usr/local/bin/mkcert

# Create a TLS key pair with <INSTANA SERVER IP>.nip.io as its CN
# or skip this if you're going to use your key pair
# NOTE: PLEASE CHANGE TO YOUR IP
$ INSTANA_SERVER_IP=<Instana Server ip address> && \
  sudo mkcert -cert-file tls.crt -key-file tls.key "${INSTANA_SERVER_IP}.nip.io" "${INSTANA_SERVER_IP}"

# NOTE : take note of the path to tls.crt (signed certificate file) and tls.key (private key file) 
#        and the FQDN "${INSTANA_SERVER_IP}.nip.io"
```

## 4. Install Docker

```sh
# Remove some legacy components, if any
$ sudo apt-get remove docker docker-engine docker.io containerd runc

# It is O.K if you see "Unable to locate package docker-engine"

# Update the apt package index and install packages to allow apt to use a repository over HTTPS:
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg

# Add Docker’s official GPG key:
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Use the following command to set up the repository:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update the apt package index:
sudo apt-get update

# To install a specific version of Docker Engine, start by listing the available versions in the repository:
apt-cache madison docker-ce | awk '{ print $3 }'

# Select the desired version and install:
VERSION_STRING=5:20.10.23~3-0~ubuntu-focal
sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin

# Verify that the Docker Engine installation is successful by running the hello-world image:
sudo docker run hello-world

# Moving /var/lib/docker to disk with more space
sudo systemctl stop docker
sudo mv /var/lib/docker /opt/docker
sudo ln -s /opt/docker /var/lib/docker

# Start Docker daemon process
sudo systemctl enable docker
sudo systemctl start docker

# Have a try for Docker
sudo docker run hello-world

# Add current user into docker group so that we can run Docker cli without the need of sudo
sudo usermod -aG docker $USER
```

> Note: Please re-login to the VM to take effect.

## 5. Install Instana Server

```sh
#As root, run the following commands:
sudo -i
echo "deb [arch=amd64] https://self-hosted.instana.io/apt generic main" > /etc/apt/sources.list.d/instana-product.list
wget -qO - "https://self-hosted.instana.io/signing_key.gpg" | apt-key add -
apt-get update
apt-get install instana-console

# To avoid getting major updates during automated upgrades, run the following commands:
cat >/etc/apt/preferences.d/instana-console <<EOF
Package: instana-console
Pin: version 247-0-1
Pin-Priority: 1000
EOF

instana version

# Finally, let’s kick off the init process
sudo instana init
```
<picture>
  <img alt="image" src="./assets/images/Instana-init.png">
</picture>

## After around 30 min
<picture>
  <img alt="image" src="./assets/images/init-complete.png">
</picture>

## 6. First login

## Launch from browser: https://52.118.165.235.nip.io

## 7. License

```sh
# Download license
$ sudo instana license download

# Now import it
$ sudo instana license import -f license

# And verify it
$ sudo instana license verify
```

## 8. Post Actions

After installing the agent by running the generated one-liner command from Instana UI:

```sh
$ sudo systemctl is-enabled instana-agent

# Or we can enable and start it manually
$ sudo systemctl enable instana-agent
$ sudo systemctl start instana-agent

$ sudo systemctl status instana-agent

# Configure zone
sudo touch /opt/instana/agent/etc/instana/configuration-zone.yaml

INSTANA_ZONE="InstanaServer" && \
cat <<EOF | sudo tee /opt/instana/agent/etc/instana/configuration-zone.yaml
# Hardware & Zone
com.instana.plugin.generic.hardware:
  enabled: true
  availability-zone: "${INSTANA_ZONE}"
EOF

# (optional) Configure host, like tags
# Do change them accordingly
sudo touch /opt/instana/agent/etc/instana/configuration-host.yaml

cat <<EOF | sudo tee /opt/instana/agent/etc/instana/configuration-host.yaml
# Host
com.instana.plugin.host:
  tags:
    - 'poc'
    - 'instana'
EOF
```

```sh
# Set up the proper EUM endpoint which will proxy to the :2999 internal port
# DO REPLACE IT WITH YOUR ENDPOINT!!!
# Ref: https://www.ibm.com/docs/en/obi/current?topic=installer-configuring-end-user-monitoring
EUM_ENDPOINT="https://168.1.53.216.nip.io:446" && \
cat | sudo tee -a settings.hcl <<EOF
eum {
  tracking_base_url = "${EUM_ENDPOINT}/eum/"
}
EOF

# Then apply it
sudo instana update -f settings.hcl
```
