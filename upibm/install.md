## Install helper node.

### Create a machine/vm with the following minimum configuration.


* CentOS/RHEL 7 or 8
* 50GB HD
* 4 CPUs
* 8 GB of RAM


Install EPEL

```
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E %rhel).noarch.rpm
```

Install ansible and git and clone this repo

```
yum -y install ansible git
git clone https://github.com/RedHatOfficial/ocp4-helpernode
cd ocp4-helpernode
```

copy and modify vars.yaml for your deployment.

```
cp docs/examples/vars.yaml .
vi vars.yaml
$ cat vars.yaml
---
disk: vda
helper:
  name: "helper"
  ipaddr: "10.10.10.2"
dns:
  domain: "example.com"
  clusterid: "ocp4"
  forwarder1: "8.8.8.8"
  forwarder2: "8.8.4.4"
dhcp:
  router: "10.10.10.1"
  bcast: "10.10.10.255"
  netmask: "255.255.255.0"
  poolstart: "10.10.10.10"
  poolend: "10.10.10.30"
  ipid: "10.10.10.0"
  netmaskid: "255.255.255.0"
bootstrap:
  name: "bootstrap"
  ipaddr: "10.10.10.20"
  macaddr: "aa:bb:cc:dd:ee:06"
masters:
  - name: "master0"
    ipaddr: "10.10.10.21"
    macaddr: "aa:bb:cc:dd:ee:01"
  - name: "master1"
    ipaddr: "10.10.10.22"
    macaddr: "aa:bb:cc:dd:ee:02"
  - name: "master2"
    ipaddr: "10.10.10.23"
    macaddr: "aa:bb:cc:dd:ee:03"
workers:
  - name: "worker0"
    ipaddr: "10.10.10.11"
    macaddr: "aa:bb:cc:dd:ee:04"
  - name: "worker1"
    ipaddr: "10.10.10.12"
    macaddr: "aa:bb:cc:dd:ee:05"
setup_registry:
  deploy: true
  autosync_registry: true
  registry_image: docker.io/library/registry:2
  local_repo: "ocp4/openshift4"
  product_repo: "openshift-release-dev"
  release_name: "ocp-release"
  release_tag: "4.8.2-x86_64"
```

### Run the playbook

Run the playbook to setup your helper node

```
ansible-playbook -e @vars.yaml tasks/main.yml
```

After it is done run the following to get info about your environment and some install help

```
/usr/local/bin/helpernodecheck

```

Check OCP images in the local registry

```
curl -u admin:admin -k -X GET https://registry.ocp4.example.com:5000/v2/ocp4/tags/list 
```



## Create Ignition Configs

Now you can start the installation process. Create an install dir.

```
mkdir ~/ocp4
cd ~/ocp4
```

Create a place to store your pull-secret

```
mkdir -p ~/.openshift
```

Visit [try.openshift.com](https://cloud.redhat.com/openshift/install) and select "Bare Metal". Download your pull secret and save it under `~/.openshift/pull-secret`

```shell
# ls -1 ~/.openshift/pull-secret
/root/.openshift/pull-secret
```

The playbook creates an sshkey for you; it's under `~/.ssh/helper_rsa`. You can use this key or create/user another one if you wish.

```shell
# ls -1 ~/.ssh/helper_rsa
/root/.ssh/helper_rsa
```

> :warning: If you want you use your own sshkey, please modify `~/.ssh/config` to reference your key instead of the one deployed by the playbook

Coppy install-config template and modify as needed.

```
# cp OCPonOSP/upibm/install-config.yaml ~/ocp4
```

Because we want to use a local registry we have to add to install-config several bits, updated pull secret with local registry, Certs of the local Registry, and mirror config, check info on the generated file ocp4-helpernode/postrun-local-registry-info

We need to add updated pull secret from  ~/.openshift/pull-secret-updated to the install-config yaml.
```
pullSecret: '$(< ~/.openshift/pull-secret-updated)'
```

Also the additionalTrustBundle from /opt/registry/certs/domain.crt to the install-config yaml.

```
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  -----END CERTIFICATE-----
````

And finally the mirror info that we can get from the following file ocp4-helpernode/postrun-local-registry-info :

```
imageContentSources:
- mirrors:
  - registry.ocp4.example.com:5000/ocp4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.ocp4.example.com:5000/ocp4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

Next, generate the ignition configs

```
openshift-install create ignition-configs
```

Finally, copy the ignition files in the `ignition` directory for the websever

```
cp ~/ocp4/*.ign /var/www/html/ignition/
restorecon -vR /var/www/html/
chmod o+r /var/www/html/ignition/*.ign
```

## Install Instances

PXE boot into your Instances and they should load up the right configuration based on the MAC address. The DHCP server is set up with MAC address filtering and the PXE service is configured to load the right config to the right machine (based on mac address).

Boot/install the VMs/Instances in the following order

* Bootstrap
* Masters
* Workers

## Wait for install

The boostrap VM actually does the install for you; you can track it with the following command.

```
openshift-install wait-for bootstrap-complete --log-level debug
```

at this point you can delete/poweroff the bootstrap server.


### Finish Install

First, login to your cluster

```
export KUBECONFIG=/root/ocp4/auth/kubeconfig

Your install may be waiting for worker nodes to get approved. Normally the `machineconfig node approval operator` takes care of this for you. However, sometimes this needs to be done manually. Check pending CSRs with the following command.

```
oc get csr
```

You can approve all pending CSRs in "one shot" with the following

```
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
```

You may have to run this multiple times depending on how many workers you have and in what order they come in. Keep a `watch` on these CSRs

```
watch oc get csr
```

In order to setup your registry, you first have to set the `managementState` to `Managed` for your cluster

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

For PoCs, using `emptyDir` is okay (to use PVs follow [this](https://docs.openshift.com/container-platform/latest/installing/installing_bare_metal/installing-bare-metal.html#registry-configuring-storage-baremetal_installing-bare-metal) doc)

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```


To finish the install process, run the following

```
openshift-install wait-for install-complete
```



## Disconnected  Operator 

Download tools
```
curl -O https://github.com/fullstorydev/grpcurl/releases/download/v1.8.2/grpcurl_1.8.2_linux_x86_64.tar.gz
tar -zxvf grpcurl_1.8.2_linux_x86_64.tar.gz bin/
curl -O https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.8/opm-linux.tar.gz
tar -zxvf opm-linux.tar.gz bin/
```

Disable OperatorHub Sources
```
$ oc patch OperatorHub cluster --type json \
    -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```

Run the source index image that you want to prune in a container.  use grpcurl to list all available operators.

```
podman run -p50051:50051 \
     -it registry.redhat.io/redhat/redhat-operator-index:v4.8

grpcurl -plaintext localhost:50051 api.Registry/ListPackages > packages.out
```

Inspect the packages.out file and identify which package names from this list you want to keep in your pruned index. 

In this example we are only going to install the web-terminal pkg/operator.

```
opm index prune     -f registry.redhat.io/redhat/redhat-operator-index:v4.8     -p web-terminal -t registry.ocp4.example.com:5000/ocp4/redhat-operator-index:v4.8
```
Push the pruned operator index image to the local registry:

```
podman push registry.ocp4.example.com:5000/ocp4/redhat-operator-index:v4.8
```

Export the auth.json podman/docker file that will give the mirror command access to the local registry 

```
REG_CREDS=${XDG_RUNTIME_DIR}/containers/auth.json
```

```
oc adm catalog mirror   -a ${REG_CREDS}   --manifests-only   --index-filter-by-os='.*'  registry.ocp4.example.com:5000/ocp4/redhat-operator-index:v4.8  registry.ocp4.example.com:5000/ocp4
cd manifests-redhat-operator-index-1631538294
oc image mirror   --skip-multiple-scopes=true   -a ${REG_CREDS}   --filter-by-os='.*'   -f mapping.txt
```

Create  imageContentSourcePolicy and catalogSource with the help of the yaml files created with the oc adm catalog mirror command

```
cd manifests-redhat-operator-index-1631538294
oc create -f imageContentSourcePolicy.yaml
oc create -f catalogSource.yaml
oc get pods -n openshift-marketplace
oc get catalogsource -n openshift-marketplace
oc get packagemanifest -n openshift-marketplace
```

Now we can install our webterminal operator package.


## Upgrading OCP version on a disconnected ENV.
Download the new OCP 4.8.10 images into the local registry
```
oc adm release mirror  -a .openshift/pull-secret-updated --from=quay.io/openshift-release-dev/ocp-release:4.8.10-x86_64 --to=registry.ocp4.example.com:5000/ocp4 --apply-release-image-signature
```






