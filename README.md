# PAO-SR-IOV-Operators

This document will guide you how to deploy Performance Add-On and SRI-IOV operators from OpenShift. Then Deploy SRIOV test pods and checking results.

## General Prerequisites
The following items are needed in order to be able to complete:
- An openshift cluster with at least one physical worker e.g SNO.
- oc installed and configured with KUBECONFIG env variable.
- SR-IOV capable NICs Intel or Mellanox, follow your BM server BIOS to enable SRI-OV parameters
```c
lspci|grep Ether
19:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
19:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
86:00.0 Ethernet controller: Intel Corporation Ethernet Controller XXV710 for 25GbE SFP28 (rev 02)
86:00.1 Ethernet controller: Intel Corporation Ethernet Controller XXV710 for 25GbE SFP28 (rev 02)
```
## Installation

## Installing the Performance Addon Operator using the CLI

  you can follow this link here
[PAO](https://docs.openshift.com/container-platform/4.9/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html#installing-the-performance-addon-operator_cnf-master) or using files is already prepared here.

### Get OpenShift Channel version 
```diff
+ oc version -o yaml | grep openshiftVersion |grep -o '[0-9]*[.][0-9]*' | head -1
"4.9"
```
### Update pao/install.yml
```bash
spec:
  channel: "4.9" 
```
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-performance-addon-operator
  annotations:
    workload.openshift.io/allowed: management
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-performance-addon-operator
  namespace: openshift-performance-addon-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-performance-addon-operator-subscription
  namespace: openshift-performance-addon-operator
spec:
  channel: "4.9" 
  name: performance-addon-operator
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
```
```diff
+ oc create -f pao/install.yml
```
## Expected Output:
```bash
namespace/openshift-performance-addon created
operatorgroup.operators.coreos.com/openshift-performance-addon-operatorgroup created
subscription.operators.coreos.com/performance-addon-operator-subscription created
```
```diff
+ oc get csv
NAME                                DISPLAY                      VERSION   REPLACES   PHASE
performance-addon-operator.v4.9.2   Performance Addon Operator   4.9.2                Succeeded
```
```diff
+ oc get po -n openshift-performance-addon-operator -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   
performance-operator-6ff7b477d9-6p7mn   1/1     Running   0          29h  
```
```diff
+ oc get csv|grep performance
performance-addon-operator.v4.9.2   Performance Addon Operator   4.9.2                Succeeded
```
## How to use/prepare Performance Profile
The performance addon operator deploys a custom resource (CR) named performanceprofile that we can use to configure the nodes. Creating such profile will trigger the creation of underlying objects such as machineconfigs or tuned objects, with handling done by more lowlevel operators.

The full list of attributes handled by the operator can be see [here](https://github.com/openshift-kni/performance-addon-operators/blob/master/api/v2/performanceprofile_types.go)

## The setup goes as following

**Note:** for SNO cluster, please use master MCP that is already created and skip this MCP creation
```diff
+ oc get mc
50-performance-master

+ oc get mcp
master   rendered-master-fb2e9e2d5bb26a745879cffb9c00053b   True      False      False      1              1                   1            
```
**Note:** for NON-SNO cluster, please skip this node label step

Label the workers that need specific configuration. We will use the commonly used label worker-cnf as hard-coded on files then update role and labels from files accordingly. But you can label to any name e.g. worker-hiperf= 
- label your worker node to worker-cnf=

```diff
+ oc label node your_worker node-role.kubernetes.io/worker-cnf='' --overwrite
```
- Create machine config pools that matches previously set labels. As previous task, it’s admin’s responsability to create those assets to match its platforms
```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker-cnf
  labels:
    machineconfiguration.openshift.io/role: worker-cnf
spec:
  machineConfigSelector:
    matchExpressions:
      - {
          key: machineconfiguration.openshift.io/role,
          operator: In,
          values: [worker-cnf, worker],
        }
  paused: false
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker-cnf: ""
```
```diff
+ oc create -f pao/mcp.yaml
```
```diff
+ oc get mcp
NAME         CONFIG                                                 UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
worker-cnf   rendered-worker-cnf-06a415a499dedcb8755706edcfed1d9a   True      False      False      0              0                   0                     0                       2d23h
````
- Create Performance Profile for NON-SNO cluster
```yaml
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
  name: worker-cnf
spec:
  additionalKernelArgs:
  - nohz_full=26-51
  - rcupdate.rcu_normal_after_boot=0
  - idle=poll
  cpu:
    isolated: 26-51
    reserved: 0-25
  globallyDisableIrqLoadBalancing: true
  hugepages:
    defaultHugepagesSize: 1G
    pages:
    - count: 0
      node: 0
      size: 1G
    - count: 0
      node: 1
      size: 1G
  nodeSelector:
    node-role.kubernetes.io/worker-cnf: ""
  numa:
    topologyPolicy: single-numa-node
  realTimeKernel:
    #set it as true for 4.9.6+ to enable the rt kernel
    enabled: true
```
```diff

```
- Create Performance Profile for SNO cluster
```yaml
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
  name: master    
spec:
  cpu:
    isolated: 0-8
    reserved: 9-103
  hugepages:
    defaultHugepagesSize: "1G"
    pages:
    - size: "1G"
      count: 16
      node: 0
  realTimeKernel:
    enabled: true
  nodeSelector:
    node-role.kubernetes.io/master: ""
    
+ oc create -f pao/pp.yaml
```
**Observation on PP creation:**
- if realTimeKernel is enabled, then node will reboot for this setup
- if ISOLATE or other Hi-Perf parameters are defined, there is a second reboot after


***Thanks to Jose for the info on Roles and pools***
https://github.com/jgato/jgato/blob/main/random_docs/Openshift%20Operators.md#Roles-and-pools

## Installing the SR-IOV Network Operator Using CLI

- To create the openshift-sriov-network-operator namespace, OperatorGroup CR and To create a Subscription CR for the SR-IOV Network Operator
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-sriov-network-operator
  annotations:
    workload.openshift.io/allowed: management
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: sriov-network-operators
  namespace: openshift-sriov-network-operator
spec:
  targetNamespaces:
  - openshift-sriov-network-operators
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: sriov-network-operator-subscription
  namespace: openshift-sriov-network-operator
spec:
  channel: "4.9"
  name: sriov-network-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

```diff
+ oc create -f sriov/install.yml
```
```diff
+ oc get po -n openshift-sriov-network-operator
NAME                                      READY   STATUS    RESTARTS       AGE
network-resources-injector-b4nvb          1/1     Running   0              2d5h
operator-webhook-pdt8c                    1/1     Running   0              2d5h
sriov-device-plugin-sgtcg                 1/1     Running   0              122m
sriov-network-config-daemon-vfwbm         3/3     Running   0              2d5h
sriov-network-operator-75dd9b8576-vc4lz   1/1     Running   0              2d5h
```
```diff
+ oc get csv -n openshift-sriov-network-operator
NAME                                        DISPLAY                      VERSION              REPLACES   PHASE
performance-addon-operator.v4.9.2           Performance Addon Operator   4.9.2                           Succeeded
sriov-network-operator.4.9.0-202111151318   SR-IOV Network Operator      4.9.0-202111151318              Succeeded
```
```diff
+ oc get mutatingwebhookconfigurations
NAME                                WEBHOOKS   AGE
network-resources-injector-config   1          2d5h
sriov-operator-webhook-config       1          2d5h
```

```diff
+ oc get validatingwebhookconfigurations
NAME                                                 WEBHOOKS   AGE
sriov-operator-webhook-config                        1          2d5h
vwb.performance.openshift.io-f26c9                   1          3d2h
```
### How to use
After the operator gets installed, We have the following CRS:
``
SriovNetworkNodeState
SriovNetwork
SriovNetworkNodePolicy
``
- SriovNetworkNodeState CRS are readonly and provide information about SR-IOV capable devices in the cluster. 
- List details of capable devices from your server that discovery by OCP SRIOV operator
```diff
+ oc get sriovnetworknodestates.sriovnetwork.openshift.io -n openshift-sriov-network-operator  -o yaml
```
```yaml
apiVersion: v1
items:
- apiVersion: sriovnetwork.openshift.io/v1
  kind: SriovNetworkNodeState
  metadata:
    creationTimestamp: "2021-11-30T16:20:31Z"
    generation: 3
    name: cnfdf06.ran.dfwt5g.lab
    namespace: openshift-sriov-network-operator
    ownerReferences:
    - apiVersion: sriovnetwork.openshift.io/v1
      blockOwnerDeletion: true
      controller: true
      kind: SriovNetworkNodePolicy
      name: default
      uid: 15766c2c-3eda-45bd-83aa-0e08efd3fe46
    resourceVersion: "3083675"
    uid: 7d6d1903-380a-4769-aeda-7a4007f54fe0
  spec:
    dpConfigVersion: "2471659"
    interfaces:
    - linkType: eth
      name: ens5f0
      numVfs: 6
      pciAddress: 0000:86:00.0
      vfGroups:
      - deviceType: netdevice
        policyName: sriov-network-node-policy
        resourceName: vuy_sriovnic
        vfRange: 0-5
  status:
    interfaces:
    - deviceID: "1015"
      driver: mlx5_core
      linkSpeed: 25000 Mb/s
      linkType: ETH
      mac: 0c:42:a1:bc:68:1c
      mtu: 1500
      name: eno1
      pciAddress: "0000:19:00.0"
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      linkSpeed: 25000 Mb/s
      linkType: ETH
      mac: 0c:42:a1:bc:68:1d
      mtu: 1500
      name: eno2
      pciAddress: "0000:19:00.1"
      vendor: 15b3
    - Vfs:
      - deviceID: 154c
        driver: iavf
        mac: 2a:fb:49:58:65:35
        mtu: 1500
        name: ens5f0v0
        pciAddress: 0000:86:02.0
        vendor: "8086"
        vfID: 0
      - deviceID: 154c
        driver: iavf
        mac: fa:d6:76:f8:cf:bb
        mtu: 1500
        name: ens5f0v1
        pciAddress: 0000:86:02.1
        vendor: "8086"
        vfID: 1
      - deviceID: 154c
        driver: iavf
        pciAddress: 0000:86:02.2
        vendor: "8086"
        vfID: 2
      - deviceID: 154c
        driver: iavf
        pciAddress: 0000:86:02.3
        vendor: "8086"
        vfID: 3
      - deviceID: 154c
        driver: iavf
        mac: 32:45:ab:d9:2c:87
        mtu: 1500
        name: ens5f0v4
        pciAddress: 0000:86:02.4
        vendor: "8086"
        vfID: 4
      - deviceID: 154c
        driver: iavf
        mac: 1e:55:b3:77:fe:9f
        mtu: 1500
        name: ens5f0v5
        pciAddress: 0000:86:02.5
        vendor: "8086"
        vfID: 5
      deviceID: 158b
      driver: i40e
      linkSpeed: 25000 Mb/s
      linkType: ETH
      mac: 40:a6:b7:2b:1f:40
      mtu: 1500
      name: ens5f0
      numVfs: 6
      pciAddress: 0000:86:00.0
      totalvfs: 64
      vendor: "8086"
    - deviceID: 158b
      driver: i40e
      linkSpeed: 25000 Mb/s
      linkType: ETH
      mac: 40:a6:b7:2b:1f:41
      mtu: 1500
      name: ens5f1
      pciAddress: 0000:86:00.1
      totalvfs: 64
      vendor: "8086"
    - deviceID: 158b
      driver: i40e
      linkSpeed: -1 Mb/s
      linkType: ETH
      mac: 40:a6:b7:2b:24:80
      mtu: 1500
      name: ens7f0
      pciAddress: 0000:88:00.0
      totalvfs: 64
      vendor: "8086"
    - deviceID: 158b
      driver: i40e
      linkSpeed: -1 Mb/s
      linkType: ETH
      mac: 40:a6:b7:2b:24:81
      mtu: 1500
      name: ens7f1
      pciAddress: 0000:88:00.1
      totalvfs: 64
      vendor: "8086"
```
### Create SriovNetworkNodePolicy CR
- Create Sriov Network Node Policy
```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: sriov-network-node-policy
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  isRdma: false
  nicSelector:
    pfNames:
      - ens5f0
  nodeSelector:
    node-role.kubernetes.io/worker-cnf: ""
  numVfs: 6
  resourceName: vuy_sriovnic
```
- deviceType can be netdevice or vfio-pci, it depends on your physical NIC capabilities.
- resourceName can be named anything you like
- pfNames is the physical NIC interface name from SriovNetworkNodeState output
- numVfs define # base on your application or test pod need.
- nodeSelecor must match with your worker role name e.g. worker-cnf
- isRdma, some NICs do not have this feature supported, enabled it only if NIC is supported. e.g. Check BIOS

**Note:** when I applied this first time, the worker node got reboot!

- Checking worker node capability related to SRIOV

```diff
+ oc create -f sriov/sriov-nnp.yaml
```
```diff
oc get no cnfdf06.ran.dfwt5g.lab -o json |jq -r '.status.allocatable'
{
  "cpu": "103500m",
  "ephemeral-storage": "431110364061",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "management.workload.openshift.io/cores": "104k",
  "memory": "96397884Ki",
  "openshift.io/vuy_sriovnic": "6",
  "pods": "250"
}
```
- Check SRIOV VF
```diff
+ ip a l|grep ens
2: ens5f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
3: ens5f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
100: ens5f0v0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
101: ens5f0v1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
104: ens5f0v2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
105: ens5f0v3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
106: ens5f0v4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
107: ens5f0v5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000

```

### Creating test pods
- Create namespace or project for SRIOV testing pods
```diff
oc create ns sriov-testing
```

- Creating Network Attach Definition
```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: sriov-network
  namespace: openshift-sriov-network-operator
spec:
  resourceName: vuy_sriovnic
  networkNamespace: sriov-testing
  ipam: |-
    {
      "type": "host-local",
      "subnet": "10.56.217.0/24",
      "rangeStart": "10.56.217.171",
      "rangeEnd": "10.56.217.181",
      "gateway": "10.56.217.1"
    }
```
```diff
+ oc create -f create-sriov-network.yaml 
sriovnetwork.sriovnetwork.openshift.io/sriov-network created

+ oc get net-attach-def
NAME             AGE
sriov-network    11s
```
- Create test PODs
```diff
+ oc create -f create-sriov-pod1.yaml 
pod/sriovpod1 created

+ oc get po
NAME        READY   STATUS    RESTARTS   AGE
sriovpod1   1/1     Running   0          5s
```
```diff
+ oc exec -it sriovpod1 -- ip a
3: eth0@if703: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:80:00:f0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.0.240/23 brd 10.128.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::812:ffff:fed6:2db0/64 scope link 
       valid_lft forever preferred_lft forever
99: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 8e:88:ad:15:73:96 brd ff:ff:ff:ff:ff:ff
    inet 10.56.217.171/24 brd 10.56.217.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::8c88:adff:fe15:7396/64 scope link 
       valid_lft forever preferred_lft forever
```
```diff
+ oc exec -it sriovpod1 -- env|grep PCI
PCIDEVICE_OPENSHIFT_IO_VUY_SRIOVNIC=0000:86:02.0

+ oc exec -it sriovpod1 -- lspci|grep Virtual
86:02.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.1 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.2 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.3 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.4 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.5 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
```
```diff
+ oc exec -it sriovpod2 -- ip a
3: eth0@if705: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:80:00:f2 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.0.242/23 brd 10.128.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1c09:aaff:fef4:62b7/64 scope link 
       valid_lft forever preferred_lft forever
90: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether f6:9f:34:f2:4b:f8 brd ff:ff:ff:ff:ff:ff
    inet 10.56.217.172/24 brd 10.56.217.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::f49f:34ff:fef2:4bf8/64 scope link 
       valid_lft forever preferred_lft forever

+ oc exec -it sriovpod2 -- env|grep PCI
PCIDEVICE_OPENSHIFT_IO_VUY_SRIOVNIC=0000:86:02.3
```
### Testing basic connectivity but it is not relevant since I have only one worker node and ip addresses of two PODs are on same SNO.
- get one of test POD container PID ID
```diff
+ sudo crictl ps pods |grep sriovpod
4782a4a9dd636       d6927021c8bbbd9c4cc71aeca51c4411325d3fa643ef0d0ce45f3d0ee15ef916  3 minutes ago  Running  sriovpod  0    4a8aeb1349ec5
71cb0da2246f6       d6927021c8bbbd9c4cc71aeca51c4411325d3fa643ef0d0ce45f3d0ee15ef916  8 minutes ago  Running  sriovpod  0    09958fd4681e3

+ Pid=$(sudo crictl inspect --output go-template --template '{{.info.pid}}' 4782a4a9dd636)
2933371
```
- Using nsenter to PING other POD IP
```diff
+ sudo nsenter -t $Pid -n -- ping -I net1 10.56.217.171
PING 10.56.217.171 (10.56.217.171) from 10.56.217.172 net1: 56(84) bytes of data.
64 bytes from 10.56.217.171: icmp_seq=1 ttl=64 time=0.054 ms
64 bytes from 10.56.217.171: icmp_seq=2 ttl=64 time=0.136 ms
```

         
```diff
+ [core@worker0 ~]$ sudo cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/ostree/rhcos-d4e4425aa2d5a3bc9de392cffa293dba5084d76c7be5b0edf28cd457be89050d/vmlinuz-4.18.0-305.25.1.rt7.97.el8_4.x86_64 random.trust_cpu=on console=tty0 console=ttyS0,115200n8 ignition.platform.id=metal ostree=/ostree/boot.1/rhcos/d4e4425aa2d5a3bc9de392cffa293dba5084d76c7be5b0edf28cd457be89050d/0 root=UUID=0d85b452-8f24-4a99-ad78-f3f620d48a84 rw rootflags=prjquota skew_tick=1 nohz=on rcu_nocbs=26-51 tuned.non_isolcpus=000000ff,ffffffff,fff00000,03ffffff intel_pstate=disable nosoftlockup tsc=nowatchdog intel_iommu=on iommu=pt isolcpus=managed_irq,26-51 systemd.cpu_affinity=0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100,101,102,103 default_hugepagesz=1G nohz_full=26-51 rcupdate.rcu_normal_after_boot=0 idle=poll
```
```bash
+ worker0 ~]$ uname -a 
Linux worker0.upitest.lab.eng.rdu2.redhat.com 4.18.0-305.25.1.rt7.97.el8_4.x86_64

```

**SNO cluster:**
```diff
+ oc get no cnfdf06.ran.dfwt5g.lab -o json |jq -r '.status.allocatable'
{
  "cpu": "9",
  "ephemeral-storage": "467784684Ki",
  "hugepages-1Gi": "16Gi",
  "hugepages-2Mi": "0",
  "management.workload.openshift.io/cores": "104k",
  "memory": "79641972Ki",
  "openshift.io/vuy_sriovnic": "6",
  "pods": "250"
}
+ [core@cnfdf06 ~]$ cat /proc/cmdline 
BOOT_IMAGE=(hd0,gpt3)/ostree/rhcos-d4e4425aa2d5a3bc9de392cffa293dba5084d76c7be5b0edf28cd457be89050d/vmlinuz-4.18.0-305.25.1.rt7.97.el8_4.x86_64 random.trust_cpu=on console=tty0 console=ttyS0,115200n8 ignition.platform.id=metal ostree=/ostree/boot.0/rhcos/d4e4425aa2d5a3bc9de392cffa293dba5084d76c7be5b0edf28cd457be89050d/0 ip=eno1:dhcp root=UUID=8cb433b8-b5cf-4548-8ace-ac26a25084fd rw rootflags=prjquota intel_iommu=on iommu=pt skew_tick=1 nohz=on rcu_nocbs=0-8 tuned.non_isolcpus=000000ff,ffffffff,ffffffff,fffffe00 intel_pstate=disable nosoftlockup tsc=nowatchdog intel_iommu=on iommu=pt isolcpus=managed_irq,0-8 systemd.cpu_affinity=9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56
```
**Useful Links:**

https://access.redhat.com/solutions/3875421

https://docs.openshift.com/container-platform/4.7/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.9/html/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes

https://infohub.delltechnologies.com/l/deployment-guide-dell-technologies-red-hat-openshift-reference-architecture-for-telecom-1/performance-profile-deployment-for-low-latency-3

## License
