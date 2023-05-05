# Lab 2.2 – Website Monitoring

## 2. Install Robot-Shop website

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

