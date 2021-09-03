# Intro Storage Persistent Volumes
### Storage Class
oc get sc
oc describe sc gp2

### Persistent volume example en AWS
oc get pv
oc get pvc
oc apply -f OCPonOSP/storage/config-pvc-example.yml
oc get pv
oc get pvc
oc apply -f OCPonOSP/storage/deployment-pvc-example.yml
oc get pv
oc get pvc
oc get pods
oc exec -it rbd-write-workload-generator-cache-869cb6887d-shj58 bash
$ df -h /mnt/pv


### Persistent volume example en OSP

