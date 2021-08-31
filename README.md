# Basic commands to deploy OCP on OSP

## Repo git
```
git clone https://github.com/likid0/OCPonOSP.git
```

## Openshift installer 
```
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-install-linux.tar.gz
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
mkdir bin/
curl https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane --output bin/butane
chmod +x bin/butane
tar -zxvf openshift-install-linux.tar.gz -C bin/
tar -zxvf openshift-client-linux.tar.gz -C bin
```

## OSP pre-requisites.

```
cat /home/lab-user/.config/openstack/clouds.yaml
openstack floating ip list
openstack server list
openstack port list
openstack quota show
openstack security group list
openstack flavor list | grep 4c16g
openstack server group list
```

## Install OCP on OSP

```
mkdir cluster-blx99
openshift-install create install-config --dir cluster-blx99
openshift-install create  cluster --dir cluster-blx99
cat cluster-blx99/.openshift_install.log
export KUBECONFIG=/home/lab-user/cluster-blx99/auth/kubeconfig
```

## Map FIP to Ingress
```
openstack port list  | grep ingress
openstack floating ip  set --port  f5c336db-8aa7-4461-ad02-eac8ed292007 150.238.220.146
```
## Access cluster with OC
```
export KUBECONFIG=/home/lab-user/cluster-blx99/auth/kubeconfig
oc get nodes
oc get clusterversion
oc get co
oc get sc
```

## Añadir nodos de tipo Infra Dia 2
```
openstack server group create --policy soft-anti-affinity --os-compute-api-version 2.15 cluster-blx99-hbb6h-infra
cp OCPonOSP/day-two/mco/machineset-infra.yaml .
sed -i -e 's/blx99-hbb6h/$CLUSTER-ID/g' machineset-infra.yaml
#Also Modify serverGroupID: with the new sever group created in the previous step
oc apply -f machineset-infra.yaml
oc get machineset -A
```

## Crear MCP para nodos infra
```
oc apply -f OCPonOSP/day-two/mco/mcp.yaml
oc get mcp
```
## Configurar NTP con MC dia 2
```
export CHRONY_CONF_B64=$(base64 -w0 OCPonOSP/day-two/mco/chrony.conf)
envsubst < OCPonOSP/day-two/mco/mc-infra.yaml | oc apply -f -
oc get mc
```

## Acceso SSH a nodo
```
openstack floating ip create e0ddc3ba-1707-44e1-a341-7c7006d60f45
openstack server list -f value -c Name | grep master-0
openstack server add security group  cluster-gmg42-vnwdm-master-0  gmg42-bastion_sg
openstack server add floating ip cluster-gmg42-vnwdm-master-0  150.239.20.165
ssh -i .ssh/gmg42key.pem core@150.239.20.165
```

## Modificar NTP durante el install
```
openshift-install create ignition-configs --dir cluster-blx99
openshift-install create  manifests --dir cluster-blx99
butane chrony.bu  -o 99-worker-ntp.yaml
butane OCPonOSP/butane/chrony.bu  -o cluster-gmg42/openshift/99-worker-ntp.yaml
```

## Mover los routers a los nodos de infra y scale a 3
```
oc get machinesets
oc scale --replicas=3 machinesets/cluster-gmg42-vnwdm-infra-0 -n openshift-machine-api
oc apply -f OCPonOSP/ingress-controller/default.yml
```
