# Istio Demo

## Install istion on Minikube

References:
- https://istio.io/latest/docs/setup/getting-started/
- https://istio.io/latest/docs/setup/platform-setup/minikube/
- https://istio.io/latest/docs/setup/additional-setup/config-profiles/
- https://istio.io/latest/docs/examples/bookinfo/

### Start Minikube
Refer to [minikube start](https://minikube.sigs.k8s.io/docs/start/) install Minikube firstly if not yet.

```bash
# stop previous running minikube (optional)
minikube stop

# Assume you have installed virtualbox
# Start minikube with 16384 MB of memory and 4 CPUs
minikube start --driver=virtualbox --memory=16384 --cpus=4 --kubernetes-version=v1.26.3

# provide a load balancer 
# run below commdand in a separate terminal
minikube tunnel

# open minikube dashboard
# run below commdand in a separate terminal
minikube dashboard
```

###  Download istio

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Or

```bash
curl -LO https://raw.githubusercontent.com/istio/istio/master/release/downloadIstioCandidate.sh
bash downloadIstioCandidate.sh
```

Or for mac, download istio from https://github.com/istio/istio/releases directly.

```bash
# for mac
curl -LO https://github.com/istio/istio/releases/download/1.17.2/istio-1.17.2-osx.tar.gz
tar -xzf istio-1.17.2-osx.tar.gz
```

Check istio directory:

```bash
cd istio-1.17.2
```
The installation directory contains:

Sample applications in samples/
The istioctl client binary in the bin/ directory.

Add the istioctl client to your path:

```bash
cp bin/istioctl /usr/local/bin/

# verify
# need enable istioctl in macos security & privacy
istioctl version
```

### Install Istio

```bash
# use demo profile in this case
istioctl install --set profile=demo -y
```

Verify
```bash
kubectl get pods -n istio-system
```

不同profile的区别参见：
- https://istio.io/latest/docs/setup/additional-setup/config-profiles/

## Deploy and access the sameple application

### Deploy the application
BookInfo demo:
- https://istio.io/docs/guides/bookinfo.html
- https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml

The yaml also in `samples/bookinfo/platform/kube/bookinfo.yaml`.

```bash
# create an new namespace
kubectl create namespace bookinfo

# set the namespace
kubectl config set-context --current --namespace=bookinfo

# use istio sidecar injection
kubectl label namespace bookinfo istio-injection=enabled

# deploy the application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

Verify:
```bash
# check deployments
kubectl get deployment

# check services
kubectl get svc

# check pods are running and ready
kubectl get pods -w
```

### Open the application to outside traffic

```bash
# create istio ingress gateway
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# verify
istioctl analyze
```

### Determining the ingress IP and ports

For Minikube:

```bash
# ensure minikube tunnel is running in a separate terminal
minikube tunnel

#Set the ingress host and ports
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

# set GATEWAY_URL
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

# verify gateway url
echo $GATEWAY_URL
```

### Verify extenal access

```bash
#  retrieve the external address of the Bookinfo application
echo "http://$GATEWAY_URL/productpage"
```

Open the URL in browser, you should see the Bookinfo application.

## View the service mesh dashboard

```bash
# Install Kiali and other addons (jaeger, prometheus, grafana) on istio-system namespace
kubectl apply -f samples/addons

# verify
kubectl get pods -n istio-system -w

# access the Kiali dashboard in a separate terminal
# open graph, and choose namespace bookinfo
# change refresh period as Every 10 seconds
# open Display, and check Traffic Animation
istioctl dashboard kiali
```

Send requests to the application:
```bash
# use curl in a for loop
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done

# or use hey performance tool
hey -n 100 -c 2 "http://$GATEWAY_URL/productpage"

# or more requests for a longer time
hey -n 1000 -c 5 "http://$GATEWAY_URL/productpage"
```

## Next steps

Use bookinfo application to test istio features.

- [Test istio features by bookinfo](./bookinfo.md)

## Troubleshooting

### Failed to connect to raw.githubusercontent.com port 443

https://github.com/hawtim/hawtim.github.io/issues/10
