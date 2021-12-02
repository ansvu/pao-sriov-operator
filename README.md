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
```bash
oc version -o yaml | grep openshiftVersion |grep -o '[0-9]*[.][0-9]*' | head -1)
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
```bash
oc create -f pao/install.yml
```
## Expected Output:
```bash
namespace/openshift-performance-addon created
operatorgroup.operators.coreos.com/openshift-performance-addon-operatorgroup created
subscription.operators.coreos.com/performance-addon-operator-subscription created
```
```python
oc get csv
NAME                                DISPLAY                      VERSION   REPLACES   PHASE
performance-addon-operator.v4.9.2   Performance Addon Operator   4.9.2                Succeeded
```
```bash
oc get po -n openshift-performance-addon-operator -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   
performance-operator-6ff7b477d9-6p7mn   1/1     Running   0          29h  

oc get csv|grep performance
performance-addon-operator.v4.9.2   Performance Addon Operator   4.9.2                Succeeded
```
## How to use/prepare PerformanceProfile
The performance addon operator deploys a custom resource (CR) named performanceprofile that we can use to configure the nodes. Creating such profile will trigger the creation of underlying objects such as machineconfigs or tuned objects, with handling done by more lowlevel operators.

The full list of attributes handled by the operator can be see [here](https://github.com/openshift-kni/performance-addon-operators/blob/master/api/v2/performanceprofile_types.go)

## The setup goes as follows

Label the workers that need specific configuration. We will use the commonly used label worker-cnf as hard-coded on files then update role and labels from files accordingly. But you can label to any name e.g. worker-hiperf= 
- label your worker node to worker-cnf=
```
oc label node your_worker node-role.kubernetes.io/worker-cnf='' --overwrite
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
```
oc create -f pao/mcp.yaml
```
```bash
oc get mcp
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
    enabled: false
```
```bash
oc create -f pao/pp.yaml
```

## License
