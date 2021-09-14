# Deploy UPI Baremetal in Disconnected Environment.

## Deploy OCP Disconnected.

### Install helper node. Create a machine/vm with the following minimum configuration.

* CentOS/RHEL 7 or 8
* 50GB HD
* 4 CPUs
* 8 GB of RAM

Onced the helper nodes is deployed login and Install EPEL

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

### [OPTIONAL only instructive]. Review Helper node playbook pre-requisite deployment on helper node.

### Httpd to host Boot images for PXE(live rootfs coreos) and Ignition files
```
[root@helper ~]# ps -ef | grep httpd
root        5041       1  0 08:22 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      5043    5041  0 08:22 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      5044    5041  0 08:22 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      5045    5041  0 08:22 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      5046    5041  0 08:22 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
root        5851    1885  0 08:35 pts/0    00:00:00 grep --color=auto httpd
[root@helper ~]# ls -lR /var/www/html/
/var/www/html/:
total 0
drwxr-xr-x. 2 root root 63 Sep 10 14:23 ignition
drwxr-xr-x. 2 root root 24 Sep 10 13:21 install

/var/www/html/ignition:
total 284
-rw-r--r--. 1 root root 280711 Sep 14 08:30 bootstrap.ign
-rw-r--r--. 1 root root   1718 Sep 14 08:30 master.ign
-rw-r--r--. 1 root root   1718 Sep 14 08:30 worker.ign

/var/www/html/install:
total 901988
-r-xr-xr-x. 1 root root 923635712 Sep 10 13:21 rootfs.img
```
### DHCP with fixed IPs for our OCP nodes, and PXE/Nextserver config.

```
[root@helper ~]# ps -ef | grep dhcp
dhcpd       5551       1  0 08:22 ?        00:00:00 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid
[root@helper ~]# cat /etc/dhcp/dhcpd.conf
authoritative;
ddns-update-style interim;
default-lease-time 14400;
max-lease-time 14400;

    option routers                  10.10.10.1;
    option broadcast-address        10.10.10.255;
    option subnet-mask              255.255.255.0;
    option domain-name-servers      10.10.10.2;
    option domain-name              "ocp4.example.com";
    option domain-search            "ocp4.example.com", "example.com";

    subnet 10.10.10.0 netmask 255.255.255.0 {
    interface eth0;
        pool {
            range 10.10.10.10 10.10.10.30;
        # Static entries
        host bootstrap { hardware ethernet aa:bb:cc:dd:ee:06; fixed-address 10.10.10.20; }
        host master0 { hardware ethernet aa:bb:cc:dd:ee:01; fixed-address 10.10.10.21; }
        host master1 { hardware ethernet aa:bb:cc:dd:ee:02; fixed-address 10.10.10.22; }
        host master2 { hardware ethernet aa:bb:cc:dd:ee:03; fixed-address 10.10.10.23; }
        host worker0 { hardware ethernet aa:bb:cc:dd:ee:04; fixed-address 10.10.10.11; }
        host worker1 { hardware ethernet aa:bb:cc:dd:ee:05; fixed-address 10.10.10.12; }
        # this will not give out addresses to hosts not listed above
        deny unknown-clients;

        # this is PXE specific
        filename "pxelinux.0";

        next-server 10.10.10.2;
        }
}
```
### Tftp. PXE provisioning.

```
[root@helper tftpboot]# ls -l /var/lib/tftpboot/pxelinux.cfg
total 24
-r-xr-xr-x. 1 root root 423 Sep 10 13:21 01-aa-bb-cc-dd-ee-01
-r-xr-xr-x. 1 root root 423 Sep 10 13:21 01-aa-bb-cc-dd-ee-02
-r-xr-xr-x. 1 root root 423 Sep 10 13:21 01-aa-bb-cc-dd-ee-03
-r-xr-xr-x. 1 root root 423 Sep 10 13:21 01-aa-bb-cc-dd-ee-04
-r-xr-xr-x. 1 root root 423 Sep 10 13:21 01-aa-bb-cc-dd-ee-05
-r-xr-xr-x. 1 root root 440 Sep 10 13:21 01-aa-bb-cc-dd-ee-06
[root@helper tftpboot]# ls -l /var/lib/tftpboot/rhcos/
total 96884
-r-xr-xr-x. 1 root root 89175364 Sep 10 13:21 initramfs.img
-r-xr-xr-x. 1 root root 10030400 Sep 10 13:21 kernel

[root@helper tftpboot]# cat /var/lib/tftpboot/pxelinux.cfg/01-aa-bb-cc-dd-ee-01
default menu.c32
 prompt 1
 timeout 9
 ONTIMEOUT 1
 menu title ######## PXE Boot Menu ########
 label 1
 menu label ^1) Install Master Node
 menu default
 kernel rhcos/kernel
 append initrd=rhcos/initramfs.img nomodeset rd.neednet=1 ip=dhcp coreos.inst=yes coreos.inst.install_dev=vda  coreos.live.rootfs_url=http://10.10.10.2:8080/install/rootfs.img  coreos.inst.ignition_url=http://10.10.10.2:8080/ignition/master.ign
```
### DNS configuration. Bind.
```
[root@helper named]# ps -ef | grep named
named       5600       1  0 08:22 ?        00:00:00 /usr/sbin/named -u named -c /etc/named.conf
[root@helper named]# cat /var/named/zonefile.db 
$TTL 1W
@	IN	SOA	ns1.ocp4.example.com.	root (
			2021091005	; serial
			3H		; refresh (3 hours)
			30M		; retry (30 minutes)
			2W		; expiry (2 weeks)
			1W )		; minimum (1 week)
	IN	NS	ns1.ocp4.example.com.
	IN	MX 10	smtp.ocp4.example.com.
;
; 
ns1	IN	A	10.10.10.2
smtp	IN	A	10.10.10.2
;
helper	IN	A	10.10.10.2
;
;
; The api points to the IP of your load balancer
api			IN	A	10.10.10.2
api-int		IN	A	10.10.10.2
;
; The wildcard also points to the load balancer
*.apps		IN	A	10.10.10.2
;
; Create entry for the local registry
registry	IN	A	10.10.10.2
;
; Create entry for the bootstrap host
bootstrap	IN	A	10.10.10.20
;
; Create entries for the master hosts
master0		IN	A	10.10.10.21
master1		IN	A	10.10.10.22
master2		IN	A	10.10.10.23
;
; Create entries for the worker hosts
worker0		IN	A	10.10.10.11
worker1		IN	A	10.10.10.12
;
```
###  Haproxy. just showing a snippet of the config file.
```
[root@helper bin]# ps -ef | grep haproxy
root        5460       1  0 08:22 ?        00:00:00 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
haproxy     5462    5460  0 08:22 ?        00:00:05 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
root        6116    1885  0 08:45 pts/0    00:00:00 grep --color=auto haproxy
[root@helper bin]# cat /etc/haproxy/haproxy.cfg
...
listen stats
    bind :9000
    mode http
    stats enable
    stats uri /
    monitor-uri /healthz


frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    option tcplog

backend openshift-api-server
    balance source
    server bootstrap 10.10.10.20:6443 check
    server master0 10.10.10.21:6443 check
    server master1 10.10.10.22:6443 check
    server master2 10.10.10.23:6443 check
...    
```


### Create Ignition Configs

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
unalias cp
cp -f ~/ocp4/*.ign /var/www/html/ignition/
restorecon -vR /var/www/html/
chmod o+r /var/www/html/ignition/*.ign
```

### Deploy RHCOS instances 

PXE boot into your Instances and they should load up the right configuration based on the MAC address. The DHCP server is set up with MAC address filtering and the PXE service is configured to load the right config to the right machine (based on mac address).

Boot/install the VMs/Instances in the following order

* Bootstrap
* Masters
* Workers

### Wait for install

The boostrap VM actually does the install for you; you can track it with the following command.

```
openshift-install wait-for bootstrap-complete --log-level debug
```

at this point you can delete/poweroff the bootstrap server.


### Finish Install

First, login to your cluster

```
export KUBECONFIG=/root/ocp4/auth/kubeconfig
```

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



## Disconnected Operator configuration 

### References
```
https://github.com/openshift-telco/ocp4-offline-operator-mirror
https://access.redhat.com/articles/5489341
```

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






