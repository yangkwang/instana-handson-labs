# QOTD 

## Download the package from 
# https://gitlab.com/quote-of-the-day/quote-of-the-day/-/packages/12720271

```sh 

#Adding QOTD chart to the local repository
helm repo add qotd https://gitlab.com/api/v4/projects/26143345/packages/helm/stable

helm repo update

kubectl create namespace qotd
kubectl create namespace qotd-load

# optionally set the default namespace to qotd
kubectl config set-context --current --namespace=qotd

helm install qotd qotd/qotd -n qotd --set useNodePort=true --set instanaReportingUrl=https://52.118.170.209.nip.io:446/eum/ --set instanaEnumMinJsUrl=https://52.118.170.209.nip.io:446/eum/eum.min.js --set instanaKey=KzN3uHbyScaOqs0ZdXaBDw --set enableInstana=true

# The route can be used to access the main application UI can be found with the following command:
echo http://localhost:$(kubectl get --namespace qotd -o jsonpath="{.spec.ports[0].nodePort}" services qotd-web) ; echo
# The Anomaly Generator is deployed to the qotd-load namespace, its route can be found with the following command:
echo http://localhost:$(kubectl get --namespace qotd-load -o jsonpath="{.spec.ports[0].nodePort}" services qotd-usecase) ; echo

socat TCP4-LISTEN:80,fork TCP4:172.18.0.3:30332

```

