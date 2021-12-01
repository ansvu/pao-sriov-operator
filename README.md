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

## Get OpenShift Channel version 
```bash
oc version -o yaml | grep openshiftVersion |grep -o '[0-9]*[.][0-9]*' | head -1)
"4.9"
```
### Update pao/install.yml
```bash
spec:
  channel: "4.9" 
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
NAME                                    READY   STATUS    RESTARTS   AGE   IP            NODE                     NOMINATED NODE   READINESS GATES
performance-operator-6ff7b477d9-6p7mn   1/1     Running   0          29h   10.128.0.26   cnfdf06.ran.dfwt5g.lab   <none>           <none>
```

## License
[MIT](https://choosealicense.com/licenses/mit/)
