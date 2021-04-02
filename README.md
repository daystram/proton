# :cyclone: Proton

Proton is a gateway which houses all daystram's applications at [daystram.com](https://daystram.com). It runs a Kubernetes cluster (the gateway runs the master node), allowing the addition of other worker nodes hosted on other premises to increase computing capacity while also keeping the VPS cost low. [K3s](https://k3s.io/) distribution is selected for its lightweight resource requirements.

Proton also acts as a [WireGuard](https://www.wireguard.com/) VPN server, as the worker nodes will attach to the cluster via this virtual network. This allows the worker nodes to lie behind a NAT'd network (e.g. homelabs or home servers) and lose the requirement to have a public IP or to expose any ports.

## Kubernetes Agent Setup Walkthrough

This guide will setup a master node (tailored to be setup in a VPS) and worker nodes, joined via a WireGuard VPN tunnel.

### Master Node

#### 1. Install and Setup WireGuard

Install WireGuard.

```shell
$ apt install wireguard resolvconf
```

Ensure IPv4 forwarding is enabled.

```shell
$ sysctl -w net.ipv4.ip_forward=1
```

Use https://www.wireguardconfig.com/ to easily generate key pairs. Proton networks use the IP range `10.7.7.0/24`. Save the configuration into `/etc/wireguard/wg0.conf`

> :bulb: Note the `wg0` interface name. This is kept consistent in the next steps.

Add the following to the `[Interface]` block in the `wg0.conf`, as described at https://www.reddit.com/r/WireGuard/comments/fqqqxz/connection_problem_with_wireguard_and_kubernetes/. (tl;dr: K8s kube-proxy in iptables mode messes with the IP table, this line masks K8s' configuration)

```ini
[Interface]
# ...
FwMark = 0x4000
```

Add `PersistentKeepalive` to prevent VPN shutting off on idle traffic.

```ini
[Peer]
# ...
PersistentKeepalive = 25
```

Enable the service.

```shell
$ systemctl enable wg-quick@wg0
```

Start VPN.

```shell
$ wg-quick up wg0
```

Ensure that port `51820` is opened on the VPS firewall.

#### 2. Install K3s Server (without Traefik Ingress Controller)

Conntrack is required for kube-proxy to work (implicit in their docs).

```shell
$ apt install conntrack
```

We'll use K3s, ensure that Traefik is disabled, we will install it separately to get v2 (default installed is Traefik v1).

```shell
$ curl -sfL https://get.k3s.io | K3S_NODE_NAME=proton sh -s - --disable traefik --disable-cloud-controller
```

We add the disable countroller flag to prevent K3s from running its own _dummy_ CCM, which requires a large amount of resource. To further reduce the control plane resource requirement (at the cost of performance), `GOGC=10` environment variable can be added to the K3s service at `/etc/systemd/system/k3s.service.env` (write permission restricted).

Set `KUBECONFIG` variable at `/etc/profile` for other tools (including Helm) to default to.

```shell
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Use this configuration to use access the cluster from a client machine by putting it into `~/.kube/config` file, and settting the cluster IP from localhost to `10.7.7.1` (as configured on the WireGuard server configuration).

> :warning: This `config` file has admin access to the cluster.

From here onwards, the setup can be done on a client machine connected to the WireGuard VPN.

#### 3. Install Helm

Helm is a package manager for Kubernetes. daystram's applications are also deployed via Helm charts, using the repository at https://charts.daystram.com. See https://github.com/daystram/helm-charts/ for more info daystram's Helm charts. See https://helm.sh/ for more info about Helm.

Install Helm.

```shell
$ wget https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz
$ tar -zxvf helm-v3.5.2-linux-amd64.tar.gz
$ mv linux-amd64/helm /usr/local/bin/helm
```

#### 4. Install Traefik v2 Ingress Controller

See https://github.com/traefik/traefik-helm-chart for more info about installing Traefik using their Helm chart.

Add the repository.

```shell
$ helm repo add traefik https://helm.traefik.io/traefik
$ helm repo update
```

Install Traefik. This also install the [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) by default.

```shell
helm -n ingress-traefik install traefik traefik/traefik --create-namespace --values ingress-traefik/values.yaml
```

Note that we are overriding some of the default values from the original chart. Adjust `service.spec.loadBalancerIP` to the external IP of your the traffic entrypoint node.

We set `externalTrafficPolicy` for the LoadBalancer service to `Local` (from the default `Cluster`), because LoadBalancers will be SNAT'd (Source NAT) by default to allow cross-node requests (somehow still works in this use case), setting this to `Local` disables this action and thus preserves the client IP. We also have to ensure that the Traefik controller pod runs on the node where the traffic comes in, on our case its the master node, Proton -- otherwise, the control pod will read the incoming IPs already being NAT'd. Thus, we can edit the deployment configuration as follows:

```shell
$ kubectl -n ingress-traefik edit deployment/traefik
```

and add the following to the template spec:
```yaml
# ...
spec:
  # ...
  template:
    # ...
    spec:
      # ...
      nodeSelector:
        kubernetes.io/hostname: proton
```

Also ensure that the LoadBalancer is provisioned to use the external IP of the node where yout traffic is coming from.

See https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-loadbalancer for more info.

> :bulb: Note the `traefik-cert-manager` ingress class, this is used when we setup cert-manager in the next steps.

More Traefik endpoints are also added in this values file, if required.

To view the [Traefik dashboard](http://localhost:9000/dashboard/#/), proxy it to your local machine.

```shell
$ kubectl -n ingress-traefik port-forward $(kubectl -n ingress-traefik get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000
```

Create a VPN whitelist Middleware. This is useful if we want to expose certain internal applications only to the VPN clients.

```shell
$ kubectl -n ingress-traefik apply -f ingress-traefik/vpn-whitelist.yaml
```

Note the IP range is set to the VPN client IP range, keep these consistent. This Middleware will only work if Traefik's LoadBalancer's `externalTrafficPolicy` is set to `Local`.

#### 5. Install cert-manager

cert-manager helps with issuing new X509 certificates for applications that require them. By creating Certificate objects (a CRD from cert-manger), a new certificate will be claimed from [Let's Encrypt](https://letsencrypt.org/) using the HTTP solver. This certificate is then used by Traefik's `websecure` endpoint. See https://cert-manager.io/docs/ for more info about cert-manager.

Add the repository.

```shell
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
```

Install cert-manager, also install [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

```shell
$ helm -n cert-manager install cert-manager jetstack/cert-manager --create-namespace --version v1.1.0 --set installCRDs=true
```

Install the ClusterIssuer.

```shell
kubectl apply -f cert-manager/letsencrypt.yaml
```

`ISRG Root X1` chain is used due to Let's Encrypt deprecating the old chain in late 2021. Note the `traefik-cert-manager` ingress class. This tells cert-manager to use Traefik's ingress as the endpoint when performing auto certificate retrieval using the HTTP solver.

#### 6. K8s Dashboard

Install the dashboard.

```shell
$ kubectl -n kubernetes-dashboard apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

Create a ServiceAccount with its ClusterRoleBinding.

> :warning: This service account has admin access to the cluster.

```shell
$ kubectl -n kubernetes-dashboard apply -f kubernetes-dashboard/serviceaccount.yaml
```

Get the access token from the secret.

```shell
$ kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/daystram -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

To open the dashboard, open an API proxy using `kubectl proxy`, then visit http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ and authenticate using token from above.

## Worker Nodes

#### 1. Install WireGuard

Same method as installing WireGuard on the master node. See [1. Install and Setup WireGuard](#1-install-and-setup-wireguard).

Use the one of the client configurations generated above.

#### 2. Install K3s Agent

Conntrack is required for kube-proxy to work (implicit in their docs).

```shell
$ apt install conntrack
```

Ensure VPN is connected and master node is setup and server is on `10.7.7.1`. Get the token from the master node at `/var/lib/rancher/k3s/server/token`. Install K3s worker node.

```shell
$ curl -sfL https://get.k3s.io | K3S_URL=https://10.7.7.1:6443 K3S_TOKEN=TOKEN_FROM_MASTER K3S_NODE_NAME=tambun sh -s - --flannel-iface wg0
```

Note the interface name `wg0` from what's set before. This affixes the IP bindings for flannel CNI to the VPN's interface (it defaults to the host's default interface, e.g. `eth0`), which wouldn't work since this node lies behind NAT.

See https://rancher.com/docs/k3s/latest/en/installation/install-options/server-config/#agent-networking for more info about K3s agent networking configuration.
