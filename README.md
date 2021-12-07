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
+ oc version -o yaml | grep openshiftVersion |grep -o '[0-9]*[.][0-9]*' | head -1)
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
- Create Performance Profile
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
+ oc create -f pao/pp.yaml
```

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
+ oc create -f sriov/install.yaml
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
### Create SriovNetworkNodePolicy CR,
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

```diff
+ oc create sriov/sriov-nnp.yaml
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

+ lspci|grep Virtual
86:02.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.1 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.2 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.3 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.4 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
86:02.5 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
```

### Creating test pods

- Creating Network Attach Definition
-
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

##https://access.redhat.com/solutions/3875421

##https://docs.openshift.com/container-platform/4.7/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html

##https://access.redhat.com/documentation/en-us/openshift_container_platform/4.9/html/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes

##https://infohub.delltechnologies.com/l/deployment-guide-dell-technologies-red-hat-openshift-reference-architecture-for-telecom-1/performance-profile-deployment-for-low-latency-3

## License
