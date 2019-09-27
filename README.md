# Kubernetes in Digital Ocean

## Slides available

[Slides](https://j.mp/jv-k8s-do)

## Documentation

[DO - kubernetes docs](https://www.digitalocean.com/docs/kubernetes/)

[Octant](https://github.com/vmware/octant/releases)

[Kubectl install](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

[Sonobuoy](https://github.com/heptio/sonobuoy/releases)

## Setup

### Init

```shell
doctl auth init
```

### Cluster creation

This will take approximately 5 minutes.

```shell
time doctl kubernetes cluster create presentation \
--auto-upgrade \
--region=nyc1 \
--set-current-context=false \
--update-kubeconfig=false \
--version=1.15.3-do.2 \
--node-pool "name=workers;size=s-4vcpu-8gb;count=2;tag=workers" \
--node-pool "name=executors;size=s-1vcpu-2gb;count=1;tag=executors"
```

### Setup local connection with kubernetes cluster

```shell
doctl kubernetes cluster kubeconfig save presentation
```

## Setup environment variables used in this documentation

Domain variable - this one came from [freenom](https://freenom.com)

```shell
export DOMAIN_DO=k8sdo.ml
```

Create a file from the value copied from [token page](https://cloud.digitalocean.com/account/api/tokens)

```shell
echo export DO_PAT="<your-token>" >> ~/digitalocean-token.sh
```

If you have yq locally...

```shell
echo export DO_PAT=\"$(yq r ~/.config/doctl/config.yaml access-token)\" > ~/digitalocean-token.sh
```

Source the file with your token

```shell
source ~/digitalocean-token.sh
```

## Steps

### Helm setup

[Digital Ocean HELM setup](https://www.digitalocean.com/community/tutorials/how-to-install-software-on-kubernetes-clusters-with-the-helm-package-manager)

- Create the tiller serviceaccount

```shell
kubectl -n kube-system create serviceaccount tiller
```

- Bind the tiller serviceaccount to the cluster-admin role

```shell
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```

- helm init

Installs Tiller on the cluster with and downloading the stable repo details locally

```shell
helm init --service-account tiller
```

### External dns

External dns is used to manage the dns for us.
Instead of manually updating the dns this project does that for us.

- Create namespace external dns

```shell
kubectl create namespace externaldns
```

- Replace token from externaldns helm parameters

```shell
sed -e "s#your_api_token#${DO_PAT}#g" \
externaldns/externaldns-values.tpl.yaml \
> externaldns/externaldns-values.yaml
```

- Install ExternalDNS to your cluster by running the following command:

```shell
helm install stable/external-dns \
--namespace externaldns \
--name external-dns \
-f externaldns/externaldns-values.yaml
```

### Cert manager

[Cert manager](https://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html)

```shell
# Install the CustomResourceDefinition resources separately
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.10/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Label the cert-manager namespace to disable resource validation
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.10.0 \
  jetstack/cert-manager

# deploy echo prod issuer
sed -e "s#youremail@yourdomain.com#$(git config --get user.email)#g" \
< cert-manager/letsencrypt-clusterissuer-prod.tpl.yaml \
| kubectl apply -f - \
--namespace cert-manager
```

### Web

- Create namespace web

```shell
kubectl create namespace web
```

- Deploy echo

```shell
sed -e "s#yourdomain#${DOMAIN_DO}#g" \
< web/echo.tpl.yaml \
| kubectl apply -f - \
-n web
```

- Deploy hello server

```shell
kubectl run hello-server --image=gcr.io/google-samples/hello-app:1.0 --replicas=3 --port=80 -n web
kubectl expose deployment hello-server --type=LoadBalancer --name=hello-service --port 80 --target-port=80 -n web
kubectl annotate service hello-service "external-dns.alpha.kubernetes.io/hostname=hello.k8sdo.ml" -n web
kubectl annotate service hello-service "external-dns.alpha.kubernetes.io/ttl=30" -n web
```

### Deploy metrics server

[Metric server article](https://www.digitalocean.com/community/tutorials/how-to-autoscale-your-workloads-on-digitalocean-kubernetes)

- Deploy metrics server in kube-system namespace

```shell
helm install stable/metrics-server \
--name metrics-server \
--namespace kube-system
```

- Edit deployment

```shell
kubectl edit deployment metrics-server \
--namespace kube-system
```

- Add these flags in the `command` part

```yaml
- --metric-resolution=60s
- --kubelet-preferred-address-types=InternalIP
```

- Verify which pods are consuming most

```shell
kubectl top pod
```

### Deploy ECK

[ECK on K8s](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)

- Deploy elastic operator

```shell
kubectl apply -f https://download.elastic.co/downloads/eck/0.9.0/all-in-one.yaml
sleep 5
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

- Deploy the Elasticsearch

```shell
kubectl apply -f elastic/elasticsearch.yaml -n elastic-system
```

- Discover password used for this cluster

```shell
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' -n elastic-system | base64 --decode)
echo $PASSWORD
```

In another tab start the proxy for elastic search:

```shell
kubectl port-forward service/quickstart-es-http 9200 -n elastic-system
```

- Deploy kibana associated with your elasticsearch cluster deployed previously

```shell
kubectl apply -f elastic/kibana.yaml -n elastic-system
```

- Port foward kibana for local access

```shell
kubectl port-forward service/quickstart-kb-http 5601 -n elastic-system
```

- Print password for kibana elastic user

```shell
echo $(
    kubectl get secret quickstart-es-elastic-user \
    -o=jsonpath='{.data.elastic}' \
    -n elastic-system | \
    base64 --decode
)
```

### Sonobuoy

```shell
sonobuoy run --mode certified-conformance
sonobuoy status
results=$(sonobuoy retrieve)
sonobuoy e2e $results
```

Output

```text
failed tests: 0
```

## Cleanup

### Cluster removal

```shell
doctl kubernetes cluster delete presentation \
--update-kubeconfig=false
```
