# Expose Robot-Shop 

## Get the list of worker node
```sh
sudo apt-get update
sudo apt-get install socat
socat TCP4-LISTEN:80,fork TCP4:172.18.0.3:30332

# The following docker way is not working
# select the IP address of one of the worker node
$ kubectl get node -o wide

# Check the NodePort for web service
$ kubectl get svc -n robot-shop

# Create a Dockerfile with the following
FROM alpine:latest
RUN apk --no-cache add socat
CMD ["sh", "-c", "socat TCP4-LISTEN:80,fork TCP4:<node-ip>:<node-port>"]

# Build the Docker image with the following command
$ docker build -t my-socat-image .


# Then run the container with the following command
$ docker build -t my-socat-image .

$ docker run -d --name my-socat-container -p 80:80 my-socat-image

```

