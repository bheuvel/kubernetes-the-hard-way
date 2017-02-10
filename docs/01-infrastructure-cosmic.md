# Cloud Infrastructure Provisioning - Google Cloud Platform

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 6 virtual machines will be created.

After completing this guide you should have the following compute instances:

```
gcloud compute instances list
```

````
NAME         ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
controller0  us-central1-f  n1-standard-1               10.240.0.10  XXX.XXX.XXX.XXX  RUNNING
controller1  us-central1-f  n1-standard-1               10.240.0.11  XXX.XXX.XXX.XXX  RUNNING
controller2  us-central1-f  n1-standard-1               10.240.0.12  XXX.XXX.XXX.XXX  RUNNING
worker0      us-central1-f  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XXX  RUNNING
worker1      us-central1-f  n1-standard-1               10.240.0.21  XXX.XXX.XXX.XXX  RUNNING
worker2      us-central1-f  n1-standard-1               10.240.0.22  XXX.XXX.XXX.XXX  RUNNING
````

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## Networking
Gather information for creation of network(s)
```
cloudmonkey set profile <profile for your cloud>
cloudmonkey set display table

cloudmonkey list vpcofferings  filter=name,id
cloudmonkey list zones  filter=name,id
cloudmonkey list domains filter=name,id
cloudmonkey list networkofferings filter=name,id,displaytext forvpc=true guestiptype=Isolated state=Enabled
cloudmonkey list serviceofferings filter=name,id
# Perhaps get this one from GUI or other
cloudmonkey list templates templatefilter=all filter=name,id
```
Set variables for use in the commands
```
vpcname=k8s-THW
vpcofferingid=<id of vpcoffering>
zoneid=<id of zone>
domainid=<id of domain>
networkofferingid=<id of offering WITH load balancing>
myip=<0.0.0.0/0 for all, or your ip/32>
keypair=<name of SSH keypair configured in cloudstack>
templateid=<id of OS image template>
serviceofferingsid=<id of service offering>
```

Create VPC
```
res=$(cloudmonkey create vpc name=$vpcname displaytext=$vpcname cidr=10.0.0.0/8 vpcofferingid=$vpcofferingid zoneid=$zoneid)

vpcid=$(echo $res | grep -Po '(?<=vpc: id = )[0-9a-f-]*')
```

Create an ACL
```
aclres=$(cloudmonkey create networkacllist vpcid=$vpcid name=kubernetes description=kubernetes)

aclid=$(echo $aclres | grep -Po '(?<= id = )[0-9a-f-]*')
```

Create a network for the Kubernetes cluster:
Required info:

```
resnetwork=$(cloudmonkey create network vpcid=$vpcid networkofferingid=$networkofferingid zoneid=$zoneid displaytext=kubernetes name=kubernetes gateway=10.240.0.1 netmask=255.255.255.0 aclid=$aclid)

networkid=$(echo $resnetwork | grep -Po '(?<=network: id = )[0-9a-f-]*')
```

### Firewall Rules

Create ACLlist on the network


```

cloudmonkey create networkacl number=1 cidrlist=$myip action=Allow protocol=icmp icmptype=-1 icmpcode=-1 traffictype=Ingress aclid=$aclid
cloudmonkey create networkacl number=2 cidrlist=10.240.0.0/24 action=Allow protocol=all traffictype=Ingress aclid=$aclid
cloudmonkey create networkacl number=3 cidrlist=$myip action=Allow protocol=tcp startport=3389 endport=3389 traffictype=Ingress aclid=$aclid
cloudmonkey create networkacl number=4 cidrlist=$myip action=Allow protocol=tcp startport=22 endport=22 traffictype=Ingress aclid=$aclid
cloudmonkey create networkacl number=5 cidrlist=$myip action=Allow protocol=tcp startport=8080 endport=8080 traffictype=Ingress aclid=$aclid
cloudmonkey create networkacl number=6 cidrlist=$myip action=Allow protocol=tcp startport=6443 endport=6443 traffictype=Ingress aclid=$aclid
```


```
cloudmonkey list networkacls filter=cidrlist,traffictype,protocol,action,startport,endport  aclid=$aclid
```

```
NAME                         NETWORK     SRC_RANGES      RULES                         SRC_TAGS  TARGET_TAGS
kubernetes-allow-api-server  kubernetes  0.0.0.0/0       tcp:6443
kubernetes-allow-healthz     kubernetes  130.211.0.0/22  tcp:8080
kubernetes-allow-icmp        kubernetes  0.0.0.0/0       icmp
kubernetes-allow-internal    kubernetes  10.240.0.0/24   tcp:0-65535,udp:0-65535,icmp
kubernetes-allow-rdp         kubernetes  0.0.0.0/0       tcp:3389
kubernetes-allow-ssh         kubernetes  0.0.0.0/0       tcp:22

+-----------+---------+-----------+-------------+--------+----------+
| startport | endport |  cidrlist | traffictype | action | protocol |
+-----------+---------+-----------+-------------+--------+----------+
|    6443   |   6443  | 0.0.0.0/0 |   Ingress   | Allow  |   tcp    |
|    8080   |   8080  | 0.0.0.0/0 |   Ingress   | Allow  |   tcp    |
|     22    |    22   | 0.0.0.0/0 |   Ingress   | Allow  |   tcp    |
|    3389   |   3389  | 0.0.0.0/0 |   Ingress   | Allow  |   tcp    |
+-----------+---------+-----------+-------------+--------+----------+
+----------+---------------+-------------+--------+
| protocol |    cidrlist   | traffictype | action |
+----------+---------------+-------------+--------+
|   all    | 10.240.0.0/24 |   Ingress   | Allow  |
|   icmp   |   0.0.0.0/0   |   Ingress   | Allow  |
+----------+---------------+-------------+--------+
```

### Kubernetes Public Address

Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:

```
assoc_ip=$(cloudmonkey associate ipaddress vpcid=$vpcid)

ipaddressid=$(echo $assoc_ip | grep -Po '(?<= ipaddress: id = )[0-9a-f-]*')
ipaddress=$(echo $assoc_ip | grep -Po '(?<= ipaddress = )[0-9.]*')

```

```
cloudmonkey list publicipaddresses vpcid=4c4075af-4839-4668-882c-51ea4bb90a55 filter=ipaddress
```
```
count = 2
publicipaddress:
+--------------+
|  ipaddress   |
+--------------+
| w.x.y.z      |
| a.b.c.d      |
+--------------+
```

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Ubuntu 16.04 mainly because it runs a newish Linux Kernel that has good support for Docker.

### Virtual Machines

#### Kubernetes Controllers
```
res_con0=$(cloudmonkey deploy virtualmachine zoneid=$zoneid templateid=$templateid hypervisor=KVM serviceofferingid=$serviceofferingid domainid=$domainid keypair=$keypair iptonetworklist[0].networkid=$networkid iptonetworklist[0].ip=10.240.0.10 displayname=controller0 name=controller0)
controller0id=$(echo $res_con0 | grep -Po '(?<=jobresult: virtualmachine: id = )[0-9a-f-]*')

res_con1=$(cloudmonkey deploy virtualmachine zoneid=$zoneid templateid=$templateid hypervisor=KVM serviceofferingid=$serviceofferingid domainid=$domainid keypair=$keypair iptonetworklist[0].networkid=$networkid iptonetworklist[0].ip=10.240.0.11 displayname=controller1 name=controller1)
controller1id=$(echo $res_con1 | grep -Po '(?<=jobresult: virtualmachine: id = )[0-9a-f-]*')

res_con2=$(cloudmonkey deploy virtualmachine zoneid=$zoneid templateid=$templateid hypervisor=KVM serviceofferingid=$serviceofferingid domainid=$domainid keypair=$keypair iptonetworklist[0].networkid=$networkid iptonetworklist[0].ip=10.240.0.12 displayname=controller2 name=controller2)
controller2id=$(echo $res_con2 | grep -Po '(?<=jobresult: virtualmachine: id = )[0-9a-f-]*')
```

#### Kubernetes Workers
```
res_work0=$(cloudmonkey deploy virtualmachine zoneid=$zoneid templateid=$templateid hypervisor=KVM serviceofferingid=$serviceofferingid domainid=$domainid keypair=$keypair iptonetworklist[0].networkid=$networkid iptonetworklist[0].ip=10.240.0.20 displayname=worker0 name=worker0)
worker0id=$(echo $res_work0 | grep -Po '(?<=jobresult: virtualmachine: id = )[0-9a-f-]*')


res_work1=$(cloudmonkey deploy virtualmachine zoneid=$zoneid templateid=$templateid hypervisor=KVM serviceofferingid=$serviceofferingid domainid=$domainid keypair=$keypair iptonetworklist[0].networkid=$networkid iptonetworklist[0].ip=10.240.0.21 displayname=worker1 name=worker1)
worker1id=$(echo $res_work1 | grep -Po '(?<=jobresult: virtualmachine: id = )[0-9a-f-]*')


res_work2=$(cloudmonkey deploy virtualmachine zoneid=$zoneid templateid=$templateid hypervisor=KVM serviceofferingid=$serviceofferingid domainid=$domainid keypair=$keypair iptonetworklist[0].networkid=$networkid iptonetworklist[0].ip=10.240.0.22 displayname=worker2 name=worker2)
worker2id=$(echo $res_work2 | grep -Po '(?<=jobresult: virtualmachine: id = )[0-9a-f-]*')

```

Save specified variables to file for safe keeping
```
for v in vpcname vpcofferingid zoneid domainid networkofferingid myip keypair templateid serviceofferingid; do echo export $v=$(eval "echo \$$v") >>stat_envs.sh ; done
```
Safe variables of created objects to file
```
for v in vpcid aclid networkid ipaddressid ipaddress controller0id controller1id controller2id worker0id worker1id worker2id; do echo export $v=$(eval "echo \$$v") >>dyn_envs.sh ; done
```
