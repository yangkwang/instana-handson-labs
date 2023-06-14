# Part 3. Advanced Topics

# Lab 3.1 – Events, analytics and troubleshooting

**Key Concepts**

Event

In Instana, there are 3 types of events: incidents, issues and changes.
- Incidents yield the highest severity level, which are created when edge services accessed by end-users are impacted or there is an imminent risk of impact;

- An Issue is an event that gets created if an application, service or any part of it gets unhealthy.

<picture>
  <img alt="image3" src="./assets/images/icons.png">
</picture>

Goal

1. To understand the mechanisms Instana offers for anomaly detection
2. To walk through a typical troubleshooting process
3. To understand how to customize Event rules

Steps

1. The mechanisms Instana offers to highlight anomalies

In the landing page of Instana, it will highlight if there are any incidents:

<picture>
  <img alt="image3" src="./assets/images/incident.png">
</picture>







## 4. Let’s purposely “inject” some issues

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
            value: "1"                     # enable it now
        image: robotshop/rs-load:latest
EOF
```






# Lab 3.5 – Custom Metrics (e.g. Cert Expiry Check)

## 1. Enable built-in statsd collector

Let's work in the "manage-to" Host VM.

Firtly, log into the Ubuntu "VM" powered by `footloose`:

```sh
$ footloose ssh root@ubuntu-0 -c footloose.yaml
```

Now, let's work in this Ubuntu "VM":

```
# Enable agent's built-in statsd collector deamon
root@ubuntu-0:~# cat > /opt/instana/agent/etc/instana/configuration-statsd.yaml <<EOF
com.instana.plugin.statsd:
  enabled: true
  ports:
    udp: 8125
    mgmt: 8126
  bind-ip: "0.0.0.0" # all IPs by default
  flush-interval: 10 # in seconds
EOF

root@ubuntu-0:~# netstat -an|grep 8125
```

## 2. Let’s have a quick try

```
root@ubuntu-0:~# apt-get install netcat -y
```

```
root@ubuntu-0:~# echo "hits:1|c" | nc -u -w1 127.0.0.1 8125
root@ubuntu-0:~# echo "custom.metrics.my_metric_name:10|g|#host:ubuntu-0" | nc -u -w1 127.0.0.1 8125
```

## 3. Create a simple script

```
root@ubuntu-0:~# cat > check-tls-cert-expiry.sh <<'EOF'
#!/bin/bash

TARGET="google.com:443";

echo "checking when the certificate of $TARGET expires";

date_cmd="date"
if [[ "$OSTYPE" == "darwin"* ]]; then
    if ! command -v gdate &> /dev/null
    then
        echo "GNU date command is required. You may install it by: brew install coreutils"
        exit
    fi
    date_cmd="gdate"
fi

# Example: Not After : Aug 29 08:29:45 2022 GMT
expiry_date="$( echo | openssl s_client -servername $TARGET -connect $TARGET 2>/dev/null \
                               | openssl x509 -noout -dates \
                               | grep 'notAfter' \
                               | cut -d "=" -f 2 )"
echo "expiry date: $expiry_date"

# Expire in seconds
expire_in_seconds=$( $date_cmd -d "$expiry_date" '+%s' ); 
echo "expire in seconds: $expire_in_seconds"

# Expire in days
expire_in_days=$((($expire_in_seconds-$(date +%s))/86400));
echo "expire in days: $expire_in_days"

# Send the generated metrics to Instana agent
echo "metrics generated: CertExpiresInDays:$expire_in_days|g"
echo "CertExpiresInDays:$expire_in_days|g" | nc -u -w1 127.0.0.1 8125
EOF
```

## 4. Run the script

```
root@ubuntu-0:~# chmod +x check-tls-cert-expiry.sh

root@ubuntu-0:~# while true; do ./check-tls-cert-expiry.sh; sleep 5; done
```

