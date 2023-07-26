# openBalena - Helm chart

> This Helm Chart is an unofficial chart and isn't created by the openBalena maintainers.

This openBalena - Helm Chart is an unofficial Kubernetes chart, but will allow you to run openBalena in a Kubernetes cluster.  

# Dependencies
One of these:

- [HAProxy Ingress v0.13.7](https://github.com/jcmoraisjr/haproxy-ingress/tree/v0.13.7)
- alternatively Traefik / k3d

> Throughout, we assume that you are located in the `open-balena-helm` directory. So do a `cd open-balena-helm` right now.

## Using k3d as a local cluster

If you don't have a cluster ready, you can quickly spin up a cluster that contains traefik by doing:
```sh
k3d cluster create openbalena -c ./k3d-default.yaml
helm dependency update ./open-balena

```
# Installing the chart

## Configuring your helm chart

You can either use open-balena's `quickstart` script or **use the jobs in this repo** to generate the keys, certificates and tokens.


### (option a) Using openbalena's quickstart
First, you've to generate a new configuration of openBalena using the `quickstart`. This will generate the `docker-compose` values as well as a `kubernetes.yaml` file, which contains everything from the config but in the Chart format. Example of running the quickstart below.

> Keep in mind, all commands are from the repository's root directory

```sh
$ ./scripts/quickstart -U <email@address> -P <password> -d mydomain.com
```

Now that the configuration is generated, we can install the Helm chart, just like every other Helm chart.


### (option b) Using the provided jobs to generate the configuration

If you don't want to use the quickstart, you can also use the provided jobs to generate the configuration. For this you can just use the provided `kubernetes.yaml` file. Note that the key, token and cert related values are empty. This is because the jobs will generate them for you.
If you leave these empty, there is nothing to be done and the jobs will take of creating the needed secrets.

---
## SSL Certificates
By default, SSL is used for every subdomain. The TLS rules are also added in the Kubernetes Chart. However, there's no certificate manager installed by default. This means, every SSL certificate is not signed and thus SSL errors occur. It is a well-considered decision to not include a cert manager, because of different needs per Kubernetes cluster and problems that may occur.    



### Using cert-manager
It is, however, recommended to use signed certificates using a certificate manager. A widely used certificate manager is [cert-manager](https://cert-manager.io/docs/). Installation of cert-manager [can be found here](https://cert-manager.io/docs/installation/kubernetes/). Using regular manifests will suffice.  

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
# the command below takes a while to finish:
helm install --atomic cert-manager jetstack/cert-manager --namespace default --version v1.11.0 --set installCRDs=true
```

> Now configure one of the issuers below.
>
> We provide examples for both Let's Encrypt and self-signed certificates.



#### **(option a)  Using Let's Encrypt**
After installing [cert-manager](https://cert-manager.io/docs/), a (cluster-)issuer should be installed ([more information](https://cert-manager.io/docs/concepts/issuer/)). Here's an example of such an issuer as a yaml file. Apply this file to your cluster after the needed changes.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: openbalena-certificate-issuer
  namespace: default # Use same namespace as installation
spec:
  acme:
    email: <email@address> # Will notify you if something goes wrong with the certificates
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: openbalena-certificate-key
    solvers:
    - http01:
        ingress:
          class: openbalena-haproxy # HAProxy Ingress class of openbalena
```

Next, you've to instruct the Ingress resources that they've to use this issuer. Go to your `kubernetes.yaml` file in the `config/` directory, and add the following lines to the bottom of the file.

```yaml
ingress:
  annotations:
    cert-manager.io/issuer: openbalena-certificate-issuer
```

Apply the changes using the [upgrading the chart instructions](#Upgrading-the-chart), and if everything is configured right, you'll see some pods starting next to the openBalena pods. Those pods will wait for the HTTP01 challenge to complete.


#### **(option b) Using self-signed certificates (air-gapped setups)**

For e.g. air-gapped setups you can also use self-signed certificates. We supply an example for this in the `open-balena/cert-manager-self-signed-CA.yaml` file. You can apply this file with:

```sh
kubectl apply -f open-balena/cert-manager-self-signed-CA.yaml
```

----
## Starting the cluster and selecting an ingress controller

You can use either of these ingress controllers: 
  - (a) HAproxy
  - (b) Traefik

### (option a) Using HAproxy
You've to add the HAProxy Ingress repository to your Helm first, if you've done that yet.

```sh
$ helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
$ helm repo add open-balena https://open-balena.helm.nexelit.io/charts
$ helm dependency update ./open-balena
$ helm install --atomic openbalena ./open-balena -f ./config/kubernetes.yaml --debug > install.log
```

The openBalena server will now be installed on your cluster on the `default` namespace.   
More options while installing can be found in the [Helm documentation](https://helm.sh/docs/).

> After installing, you've to delete the HAProxy Ingress chart once, because of an issue with the TCP TLS secret update.  
> If you don't do this, the 'tunnel' endpoint will not work

### (option b) Using Traefik

Comment out the dependency to haproxy-ingress in open-balena/Chart.yaml

```sh
# if you are using k3d you can skip this step
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm dependency update ./open-balena

helm install traefik traefik/traefik
# stop skipping here

kubectl apply -f traefik-enable-access-logs-and-add-tunnel-entrypoint.yml
# todo: check if they are actually required
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.9/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml

kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.9/docs/content/reference/dynamic-configuration/kubernetes-crd-rbac.yml

# optionally for traefik dashboard
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name --all-namespaces)  -n $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=json --all-namespaces | jq -r '.items[].metadata.namespace') 9000:9000

helm install --atomic openbalena ./open-balena -f kubernetes.yaml --debug > install.log
```

---

# Upgrading the chart
If you've made any changes to the `kubernetes.yaml` chart, upgrading is as simple as installing, only changing the word 'install' with 'upgrade'.

```sh
helm upgrade --atomic openbalena ./open-balena -f kubernetes.yaml --debug > install.log
```

# Testing the installation

You can test the installation by using the balena CLI to deploy a qemu based test fleet to balena.
## Obtain the CA if you are using a self signed certificate
```sh
kubectl get secret root-secret -o jsonpath="{.data['ca\.crt']}" | base64 --decode > ca.crt
```

## Configure balena cli 
```sh
export NODE_EXTRA_CA_CERTS=$(pwd)/ca.crt
# reference the balena server by the ingress hostname found in the kubernetes.yaml file
echo "balenaUrl: '192.168.99.107.nip.io'" > ~/.balenarc.yml
```

## Login
```sh
# Use the admin credentials you set in your kubernetes.yml file

balena login

# select `Credentials`
# enter your username and password defined in the kubernetes.yaml file

```

## Create a fleet
```sh
balena fleet create qemu
# Select `QEMU X86 64bit`
# Note the slug of the created fleet: e.g. `admin/qemu
```
## Configure a qemu emulated device
Inspired by https://docs.balena.io/learn/getting-started/qemux86-64/python/ we can download a balena qemu base image and run that using balena cli.

1. Download the qemu 64 development build at https://www.balena.io/os#download-os and unzip `unzip ~/Downloads/qemux86-64-2.83.18+rev5-dev-v12.10.3.img.zip`
2. configure it: `balena os configure qemux86-64-2.83.18+rev5-dev-v12.10.3.img --fleet admin/qemu` Note the slug of the fleet you created earlier.
3. run it `qemu-system-x86_64 -device ahci,id=ahci -drive file=qemux86-64-2.83.18+rev5-dev-v12.10.3.img,media=disk,cache=none,format=raw,if=none,id=disk -device ide-hd,drive=disk,bus=ahci.0 -net nic,model=virtio -net user -m 512 -nographic -machine type=pc,accel=kvm -smp 4 -cpu host` The system now boots.
4. Wait for the system to fully boot and the login prompt to be visible. Use username `root`, there is no password.
5. You are now inside the emulated device. Check the status of the balena supervisor: `balena logs -f balena_supervisor`
6. Check the openvpn tunnel: `systemctl status openvpn`
7. terminate the qemu: `shutdown now`

After the qemu has started you can check on the host machine if it has connected correctly: `balena devices`