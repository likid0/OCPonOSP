# Initial review. Goes with slides.
### Login against OCP API
```
oc login  -u kubeadmin https://api.cluster-v9bbl.v9bbl.sandbox502.opentlc.com:6443
oc whoami
oc get projects
oc get clusteroperators
oc get clusterversion
oc get nodes
oc get pods -A
```

### Basics oc 
```
oc new-project demo
oc new-app  httpd-example
oc scale --replicas=3 rc/httpd-example-1
```

### Operators Example.
```
oc get co
#install ocs operator
oc project openshift-storage
oc get csv
oc get pods
```

### Machine config Operator
```
oc get mc
oc get mc rendered-worker-8c2f88c95e34765b573f32c6a6b10568 -o yaml | grep -C 6 'name: core'
```

### Computes
```
oc get nodes
oc get nodes --show-labels
oc describe node ip-10-0-177-157.us-east-2.compute.internal
oc debug node/ip-10-0-177-157.us-east-2.compute.internal
oc adm node-logs ip-10-0-177-157.us-east-2.compute.internal
```


## Networking
### Networking services
```
oc get svc
oc get endpoints
oc get rc
oc scale --replicas=3 rc/httpd-example-1
oc describe rc httpd-example-1
oc get pods
oc get svc
oc get endpoints
```

### Networking Routes/Ingress
```
oc get route
oc describe route httpd-example
oc delete route httpd-example
oc expose service httpd-example --hostname=curso.apps.cluster-rdnhg.rdnhg.sandbox81.opentlc.com
```

### Networking Ingress Nodeport
```
oc new-app -S mysql
oc new-app --template=mysql-ephemeral
oc delete svc/mysql
oc appy -f OCPonOSP/introduction/node-port.yml
oc get svc
oc describe svc 
oc exec -it mysql-1-5gws9 -- bash
$ /usr/bin/mysql -h 172.30.135.53 -P 3306 -u mysql
$ /usr/bin/mysql -h 10.0.138.57 -P 30036 -u mysql
```
# Services use Iptables controlled by kube-proxy, example On a worker node 
```
$ iptables -t nat -L
```

### DNS 
```
oc get dnses.operator.openshift.io/default -o yaml | grep cluster
oc get svc -n openshift-dns
oc get pod -n openshift-dns  -o wide
oc get ds -n openshift-dns
oc exec -ti dns-default-7xfzl -n openshift-dns bash
$ ps -ef 
$ cat /etc/coredns/Corefile
$ cat /etc/resolv.conf 
$ exit
oc rsh mysql-1-5gws9
$ cat /etc/resolv.conf
nslookup mysql.demo.svc.cluster.local
```
