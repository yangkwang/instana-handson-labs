# Part 2. Core Capabilities & Use Cases

## Lab 2.1 – A Quick Tour

A good way to learn new things is to say “hello” to it, with a quick tour.

**Key Concepts**

Deployment pattern
Instana offers full flexibility in terms of deployment: SaaS, or on-prem.

Tenant Unit (TU)
Instana is designed for multi-tenancy. Each Instana installation supports multiple tenants with each tenant consisting of one or multiple tenant units (TU). Please note that multi-tenancy support is only available on SaaS or on-prem when deploying on Kubernetes / OpenShift.

Guided Tour
Instana provides a “guided tour” here: https://play-with.instana.io. Through it you can quickly walk Instana through too in your pace.

Goal
- To quickly walk through what functionalities Instana offers and how Instana organizes them
- To understand how Instana is designed from an end-user perspective

Steps
1. Landing page
The URL for “normal” tenant, other than those “special” ones like “play-with”,
will be structured like: https://<Tenant Unit>-<Tenant Instance>.instana.io

Once we’ve logged into it and we can see the landing page, where it’s the
default dashboard with below information:
- The list of “Websites & Mobile Apps”
- The list of “Application”
- The list of “Kubernetes” clusters
- The list of “Infrastructure”
- The list of “Events”

Please note though, the landing page of the “default” dashboard can be set to your custom dashboard too if you want, so you can have a landing page from that.

<picture>
  <img alt="image3" src="./assets/images/defaultDashboard.png">
</picture>


2. Menu items

<picture>
  <img alt="image3" src="./assets/images/menus.png">
</picture>

3. Commonly shared components

Instana provides a consistent UX and there are some reusable components on different UIs.

<picture>
  <img alt="image3" src="./assets/images/consistentUX.png">
</picture>

Let’s go through some of them one by one:

<picture>
  <img alt="image3" src="./assets/images/UX2.png">
</picture>

**Takeaways**

As you could see from this lab, Instana provides a fine-grained, simplified and consistent user experience while offering highly integrated modern Application Performance Management (APM) features.

We will focus on its core capabilities to dive deeper throughout the labs, one by one.


# Lab 2.2 – Website Monitoring

Instana supports website monitoring by analyzing actual browser request times and route loading times. This allows detailed insights into the web browsing experience of end-users, as well as deep visibility into application call paths. The Instana website monitoring solution works by means of a lightweight JavaScript agent which is embedded into the monitored website.

**Key Concepts**
End-User Monitoring (EUM) or Real-User Monitoring (RUM)
Website monitoring, often called End-User Monitoring (EUM) or Real-User Monitoring (RUM), is an important tool to understand digital user experience.

Goal
- To understand how to enable monitoring for (microservices-based) website
- To walk through the built-in dashboards and metrics
- To understand the value of website monitoring

Steps

1. Start by defining a website on Instana

<picture>
  <img alt="image3" src="./assets/images/defineWebsite.png">
</picture>

Key in the website name, like “Student-{n} Robot Shop Website” is fine, where the {n} is the trainee’s identifier.
Now Instana will automatically generate some code snippet for us which can be easily embedded into our application.


<picture>
  <img alt="image3" src="./assets/images/codeSnippet.png">
</picture>

Note:
  1. We can always check out this info within the website’s configuration anytime later.
  2. The are some important elements we will use, like the reporting URL and key.


## 2. Install Robot-Shop website

Well, a “website” is simply an application with customer-facing UI, which typically may include multiple services at the backend, with databases for data persistence.
Let’s use the Robot Shop app here, which is a “typical” cloud-native application built with different technologies. It’s maintained by Instana team and the OSS community.


### On Kubernetes

```sh
# Create the namespace
$ kubectl create namespace robot-shop

# Clone the repo
$ git clone https://github.com/instana/robot-shop

# Then cd into it
$ cd robot-shop

# Deploy it by Helm 3
# NOTE: Use the right values generated in website config as the variables
$ INSTANA_EUM_REPORTING_URL="https://168.1.53.231.nip.io:446/eum/" && \
  INSTANA_EUM_KEY="xxxxxxxxxxxxxxxxx" && \
  helm install robot-shop K8s/helm \
    --namespace robot-shop \
    --set image.version=2.1.0 \
    --set nodeport=true \
    --set eum.url="${INSTANA_EUM_REPORTING_URL}" \
    --set eum.key="${INSTANA_EUM_KEY}"
```

```sh
# Check out the pods deployed (wait 5-8 min)
$ kubectl get pod -n robot-shop
```

```sh
# To fix bug in Robot Shop
# Firstly, log into the Web Pod:

kubectl exec -n robot-shop -it "`kubectl get pod -l service=web -n robot-shop -o jsonpath={..metadata.name}`"  -- bash

# Then print out the eum.url injected eum.html file:

cat /usr/share/nginx/html/eum.html

# Let’s copy the content and paste it into your editor of choice, 
# and add the tailing “/” at the end of the 'reportingUrl', from:

# From ineum('reportingUrl', 'https://168.1.53.216.nip.io:446/eum');
# To ineum('reportingUrl', 'https://168.1.53.216.nip.io:446/eum/');

# Then write it back to replace the original eum.html file. 
# Please note that there is no “vi” in this Pod 
# so we use “cat” command to write back the file:

cat > /usr/share/nginx/html/eum.html

# Then copy and paste the updated content to the prompt, 
# and then press “Control + c” and the file will be updated.

<!-- EUM include -->
<script> 
 (function(s,t,a,n){s[t]||(s[t]=a,n=s[a]=function(){n.q.push(arguments)}, 
 n.q=[],n.v=2,n.l=1*new Date)})(window,"InstanaEumObject","ineum");

 ineum('reportingUrl', 'https://<Instana Server IP>.nip.io:446/eum/'); 
 ineum('key', '<key>'); 
 ineum('trackSessions');
 ineum('page', 'splash');
</script>
<script defer crossorigin="anonymous" src="https://<Instana Server IP>.nip.io:446/eum/eum.min.js"></script>
<!-- EUM include end -->

# You may have a final check by this command and make sure 
# there is a ending slash in the 'reportingUrl'

cat /usr/share/nginx/html/eum.html


```

```sh
# Expose the app if you want – this is optional for the lab
# Open a terminal

# Install socat
sudo apt-get update
sudo apt-get install socat

# Get an IP address of the worker node
kubectl get node -o wide

# Get the nodeport for web service
kubectl get svc -n robot-shop

socat TCP4-LISTEN:80,fork TCP4: <node-ip>:<node-port>

```

### On OpenShift (Just for Information)

```sh
# Create the project
$ oc adm new-project robot-shop
$ oc project robot-shop

# Grant permissions
$ oc adm policy add-scc-to-user anyuid -z default -n robot-shop
$ oc adm policy add-scc-to-user privileged -z default -n robot-shop

# Clone the repo
$ git clone https://github.com/instana/robot-shop

# Then cd into it
$ cd robot-shop

# Deploy it by Helm 3
# NOTE: Use the right values generated in website config as the variables
$ INSTANA_EUM_REPORTING_URL="https://168.1.53.231.nip.io:446/eum/" && \
  INSTANA_EUM_KEY="xxxxxxxxxxxxxxxxx" && \
  helm install robot-shop K8s/helm \
    --namespace robot-shop \
    --set image.version=2.1.0 \
    --set eum.url="${INSTANA_EUM_REPORTING_URL}" \
    --set eum.key="${INSTANA_EUM_KEY}"
    --set openshift=true
    --set ocCreateRoute=true
```

```sh
# Check out the pods deployed
$ kubectl get pod -n robot-shop

# Check out the route
$ kubectl get route -n robot-shop
```

## 3. Install the “load-gen” app

```sh
# Deploy the load-gen App
$ kubectl -n robot-shop apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load
  labels:
    service: load
spec:
  replicas: 1
  selector:
    matchLabels:
      service: load
  template:
    metadata:
      labels:
        service: load
    spec:
      containers:
      - name: load
        env:
          - name: HOST
            value: "http://web:8080/"
          - name: NUM_CLIENTS
            value: "5"
          - name: SILENT
            value: "1"
          - name: ERROR
            value: "0"                  # disable the error calls first
        image: robotshop/rs-load:latest
EOF
```

```sh
# Deploy the Selenium-based EUM friendly load-gen
$ kubectl -n robot-shop apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rs-website-load
  labels:
    service: rs-website-load
spec:
  replicas: 1
  selector:
    matchLabels:
      service: rs-website-load
  template:
    metadata:
      labels:
        service: rs-website-load
    spec:
      containers:
      - name: rs-website-load
        env:
          - name: HOST
            value: "http://web:8080/"
        image: brightzheng100/rs-website-load:2.1.0
        imagePullPolicy: Always
EOF

# scaling down the load 
kubectl scale deployment rs-website-load -n robot-shop --replicas=0
```



# Lab 2.3 – Install & Manage Agents

## 6. Linux VM agent – Install agent

Run this in the Host VM:

```sh
# cd to the home folder
$ cd ~

# List out the footloose-powered VMs we have
$ footloose show -c footloose.yaml

# Log into the Ubuntu VM
$ footloose ssh root@ubuntu-0 -c footloose.yaml
```

Once SSH'ed into the Ubuntu "VM" powered by `footloose`:

```
root@ubuntu-0:~# apt-get update
root@ubuntu-0:~# apt-get install gpg apt-utils -y

root@ubuntu-0:~# curl -o setup_agent.sh https://setup.instana.io/agent && chmod 700 ./setup_agent.sh && sudo ./setup_agent.sh -a xxxxxxxxxxxxxxxxxx -d xxxxxxxxxxxxxxxxxx -t dynamic -e 168.1.53.231.nip.io:1444 -y 
```

```
# By default, the agent is not up and running after installation
root@ubuntu-0:~# systemctl status instana-agent
```

```
# If needed to uninstall the agent
apt list --installed | grep instana-agent
sudo apt-get purge <package_name>

```

```
# Log into the Centos VM
$ footloose ssh root@centos-0 -c footloose.yaml
yum install which

root@ubuntu-0:~# curl -o setup_agent.sh https://setup.instana.io/agent && chmod 700 ./setup_agent.sh && sudo ./setup_agent.sh -a xxxxxxxxxxxxxxxxxx -d xxxxxxxxxxxxxxxxxx -t dynamic -e 168.1.53.231.nip.io:1444 -y
```

```
# Configure zone
root@ubuntu-0:~# touch /opt/instana/agent/etc/instana/configuration-zone.yaml
root@ubuntu-0:~# INSTANA_ZONE="Student-1-Zone" && \
cat <<EOF | sudo tee /opt/instana/agent/etc/instana/configuration-zone.yaml
# Hardware & Zone
com.instana.plugin.generic.hardware:
  enabled: true
  availability-zone: "${INSTANA_ZONE}"
EOF

# (optional) Configure host, like tags
# Do change them accordingly
root@ubuntu-0:~# touch /opt/instana/agent/etc/instana/configuration-host.yaml
root@ubuntu-0:~# cat <<EOF | sudo tee /opt/instana/agent/etc/instana/configuration-host.yaml
# Host
com.instana.plugin.host:
  tags:
    - 'labs'
    - 'poc'
    - 'instana'
EOF
```

```
# Start it up
root@ubuntu-0:~# systemctl enable instana-agent
root@ubuntu-0:~# systemctl start instana-agent

# We can trace the logs too
root@ubuntu-0:~# journalctl -flu instana-agent
```

