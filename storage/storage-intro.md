# Intro Storage Persistent Volumes
### Storage Class
```
oc get sc
oc describe sc gp2
```
### Persistent volume example en AWS
```
oc get pv
oc get pvc
oc apply -f OCPonOSP/storage/pvc-example.yml
oc get pv
oc get pvc
oc apply -f OCPonOSP/storage/deployment-pvc-example.yml
oc get pv
oc get pvc
oc get pods
oc exec -it rbd-write-workload-generator-cache-869cb6887d-shj58 bash
$ df -h /mnt/pv
```

### Persistent volume example en OSP
```
openstack volume list
oc get sc
oc get pv
cp OCPonOSP/storage/*pvc-example.yml .
#Modify the yaml to change the SC
oc create -f pvc-example.yml
oc create -f deployment-pvc-example.yml
oc delete deploy 
oc delete pvc pv-example
```

### ODF/OCS 4.8 in AWS
```
# Add workers for OCS
oc get machinesets -n openshift-machine-api
CLUSTERID=$(oc get machineset -n openshift-machine-api -o jsonpath='{.items[0].metadata.labels.machine\.openshift\.io/cluster-api-cluster}')
echo $CLUSTERID
curl -s https://raw.githubusercontent.com/red-hat-storage/ocs-training/master/training/modules/ocs4/attachments/cluster-workerocs-us-east-2.yaml | sed -e "s/CLUSTERID/${CLUSTERID}/g" | oc apply -f -
oc get machines -n openshift-machine-api | egrep 'NAME|workerocs'
watch "oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'"
oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
# Install operator from GUI
oc project openshift-storage
watch oc -n openshift-storage get csv
oc get crd | grep cluster
watch 'oc get pods'
oc get sc 
oc get pvc

oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD
$ ceph -s


#Example APP RBD RWO
curl -s https://raw.githubusercontent.com/red-hat-storage/ocs-training/master/training/modules/ocs4/attachments/configurable-rails-app.yaml | oc new-app -p STORAGE_CLASS=ocs-storagecluster-ceph-rbd -p VOLUME_CAPACITY=5Gi -f -
#Example APP CephFS RWX
oc new-app openshift/php:7.2-ubi8~https://github.com/christianh814/openshift-php-upload-demo --name=file-uploader
oc logs -f bc/file-uploader
oc scale --replicas=3 deploy/file-uploader
oc set volume deploy/file-uploader --add --name=my-shared-storage  -t pvc --claim-mode=ReadWriteMany --claim-size=1Gi  --claim-name=my-shared-storage --claim-class=ocs-storagecluster-cephfs  --mount-path=/opt/app-root/src/uploaded
