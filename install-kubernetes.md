# kind k8s Kubernetes to go

[Kind k8s](https://kind.sigs.k8s.io/) is a tool for running local Kubernetes clusters using Docker container "nodes".

## installing necessary packages and binaries

### system preparation

for the Workshop, we are going to deploy the Cluster as the root user. The only requirement we have to the System is podman being installed
~~~
dnf install -y podman
~~~

### kind k8s binary installation

To install, download the binary, then rename it to kind and place this into your $PATH at your preferred binary installation directory.

~~~
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64
chmod +x ./kind
install -o root -g root -m 0755 kind /usr/local/bin/kind
~~~

### Install kubectl binary with curl on Linux 

To install, download the binary, then rename it to kind and place this into your $PATH at your preferred binary installation directory.

~~~
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
~~~

### Install oc binary from Red Hat Downloads

oc is only available upstream as it's source. We do not want to compile it on our own so we utilize the Red Hat Downloads to retrieve the latest OpenShift Linux Client
https://access.redhat.com/downloads/content/290/ver=4.14/rhel---9/4.14.10/x86_64/product-software

Alternatively, you can as well create a bash alias for `kubectl` to `oc` as we do not utilize OCP specific functionality.
~~~
echo "alias oc=/usr/local/bin/kubectl" >> .bashrc 
source .bashrc
~~~

## Install the kind k8s Cluster

For the Workshop we require a Cluster with two nodes. The default setup will only deploy the control-plane node which makes it necessary to use a configuration file.
Furthermore, we want to be able to access our Cluster through HTTP which requires a PortMapping.
Please ensure to be able to use port `80` and `443`. If not you can choose a different port on your Host system and need to adjust curl commands from the Workshop accordingly.

~~~
cat <<EOF> kind-workshop.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: workshop
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
  - containerPort: 30443
    hostPort: 443
- role: worker
- role: worker
EOF

kind create cluster --config kind-workshop.yml
~~~

### initial configuration for your Workshop Cluster

to be able to utilize Routes we need to execute following command after the Cluster has deployed and is ready to be used
~~~
oc apply -k router
~~~

## Removing the Workshop Cluster

After you have finished the Workshop, you can remove the Cluster with following command
~~~
kind delete clusters workshop
~~~

After removing the installed binaries in `/usr/local/bin/` decomissioning is completed
~~~
rm -f /usr/local/bin/{oc,kind,kubectl}
~~~
