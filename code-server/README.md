# code-server

> :bulb: See https://github.com/cdr/code-server for more info about code-server.

To install [code-server](https://github.com/cdr/code-server) into the cluster, the chart at https://github.com/cdr/code-server/tree/v3.8.1/ci/helm-chart is utilized. To suit our own deployment, some templates have to be adjusted.

Ingress is updated to use our own Traefik ingress controller. Despite being exposed, it is attached the VPN middleware, making it accessible only via our VPN's local network.

The deployment template it also modified, adding a CIFS SMB volume mount on `/mnt/share/code-server`. This store is not ephemeral and persists across installation. By using our existing SMB server from `samba` at Proton node, this share can be accessed directly by our client devices connected to the VPN.

## Installation

Clone code-server repository:

```shell
$ git clone https://github.com/cdr/code-server.git
```

> :bulb: This guide uses code-server v3.8.1.

Replace values.yaml at `code-server/ci/helm-chart` as well as ingress.yaml and deployment.yaml at `code-server/ci/helm-chart/templates` with those in this repository. Adjust as required.

<!-- TODO: SMB guide -->

Using the SMB mount is optional. An SMB server is required, follow [this]() guide to install it on your master node.

Create namespace and CIFS credentials secret:

```shell
$ kubectl create namespace code-server
$ kubectl -n code-server create secret generic secret-cifs-proton \
  --from-literal=username=SMB_USERNAME \
  --from-literal=password=SMB_PASSWORD
```

Intall code-server:

```shell
$ helm -n code-server install code-server code-server/ci/helm-chart/ --create-namespace
```

code-server is now accessible from https://code.daystram.com, only when connected to the VPN network.
