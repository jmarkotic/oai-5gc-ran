# OpenAirInterface 5GC / O-RAN 

## Project description
Deploy e2e lab with 5GC SA and O-RAN in split mode in multi-cluster K8s environment.
Project is WIP and will be expanded with more compex scenarios (Networrkm Slicing, non-RT/near-RT RIC use cases, etc).


## Project scope

Objectives for project:

*   Deploy o-ran components in different O-RAN deployment models
*   deploy similar network conditions as used in production networks (multiple netowrk interfaces to support highspeec/low latency traffic)
*   use SR-IOV, DPDK, multus
*   upgrade with additional scenarios like network slicing and RIC use cases
*   etc.

## Components used in lab:

*   TCA 2.3.0 and accompanied TKG
*   Harbor 2.x
*   OpenAirIinterface (OAI) 5GC 1.5 (upgraded to 2.1.0) 
*   OpenAirInterface (OAI) O-RAN components (CU-CU, CU-UP, DU, NR UE)

Note: It was planned to use open5gs as 5GC SA, but there were certian issues when connecting OAI NR-UE to open5gs core (OAI NR-UE sends some non-cleartext IEs in Registration Request as cleartext, which is rejected by open5gs, Still investigated).


## Helm charts and CSAR repository

Helm charts, csar and yaml manifests can be found at:

[https://github.com/jmarkotic/oai-5gc-ran](https://github.com/jmarkotic/oai-5gc-ran)


# Network setup

Network model is balancing between flexibility (leveraging real multus network interfaces which enable us to run workload on multiple K8s clusters) and simplicity.

Most of the interfaces are mapped to same portgroups/VLANs  (only portgroup vlan 10 and 11 are used). When different network elements are communicating, communication is mostly contained within same vlan, so that routing is fairly simple on pod side, (so I don't need to add specific routes in pods, when multus interface is attached. When specific routes needs to be added on pods, one must be careful not to interfere with default route (via eth0). in some cases service discovery would not work properly.

## Basic network setup

Notes:

*   default interface eth0 (antrea cni eth0) is mostly used on 5gc clusters, between 5GC components (SBI interfaces)
*   Most of the communication between 5gc and RAN elements is using multus interfaces (AMF and CU-CP N2, UPF and CU-UP N3 network, F1/F1C, F1U, E2 between CU and DU, ...)
*   I am connecting most of those network interfaces instantiated via late binding to same portgroup (attached to vlan 11, or vlan 10). But each network is using its own network subnet, simulating more complex networking
*   Most networks (N2, N3, E1, F1, ...) are having all IPs directly reachable (all on same L3/segment) so that specific routes are not needed inside pod
*   All host interface are added via csar as vmxnet3 interface, but can be easily adopted to sr-iov if required (by editing csar and modifying network-attachment-definition file). Sr-ipv connection woule make sense on DU side and optionally on CU-UP and UPF (both carry userplane traffic). 

![Network Diagram](/images/1-net-diagram.jpg)

Figure: Network Diagram

# Preparations

## 3.1. Csar and helm artifacts

Both helm and csar files needs to be onboarded on Harbor registry and TCA.

Specific note for TCA

TCA will complain on helm validity when fetching hel file (will complain on regcred secret encoded in helm templates). This secret is used to guarantee proper download of container images form docker hub.

I chose simpler option to disregard those warnings by disabling helm validity check (or have it enabled, but do not stop in case of error reported).

This is not mandatory, but i have created manually on all K8s clusters, secret for docker registry (one must use name it regcred, since that one is reflected in all helm charts).

If to be used, pls replace with your own docker hub credentials.

```
$ kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=myusername --docker-password="xyz" --docker-email="email@email.org"
❯ kubectl get secret

NAME      TYPE                             DATA   AGE
regcred   kubernetes.io/dockerconfigjson   1      3d14h

## good to use when pulling images from docker-hub
imagePullSecrets:
  - name: regcred
```

It is not mandatory to have it crerated, should work without it as well (one needs to check if docker images are properly pulled from docker hub).

## K8s clusters setup

In this specific setup all 5gc components will run on single cluster (since 5gc is not in focus).

RAN components will be split betwen multiple clusters. For simplicity and constrained lab resources, i balanced between ahving different nodepools for workload on single cluster

![CaaS setup](/images/2-CaaS-setup.jpg)

Following addons are added on all clusters:

![CaaS addons](/images/3-CaaS-addons.jpg)

All clusters are having labels set to define which workload should run on it (and later used with nodeSelector option inhelm files).

Example for ran1 cluster which should run CU components (cu-cp, cu-up).

![CaaS labels](/images/4-CaaS-labels.jpg)

Labels will be set per clusters / nodepools:

<table><tbody><tr><td>Cluster</td><td>Nodepool</td><td>Label</td></tr><tr><td>ran1</td><td>np1</td><td>workload=cucp</td></tr><tr><td>ran1</td><td>np2</td><td>workload=cuup</td></tr><tr><td>ran2</td><td>np1</td><td>workload=du</td></tr><tr><td>ran2</td><td>np2</td><td>workload=ue</td></tr></tbody></table>

# OAI 5GC SA setup

## OAI 5GC instantiation

Instantiate OAI 5gc instance from prepared oai5gc csar.

![CNF 5GC](/images/5.1-cnf-5gc.jpg)

Make sure that helm validiation is either not run on just ignore failures

Failure is coming from secret which is optionally used to log into public docker hub. This can be encoded in csar

![CNF 5GC](/images/5.2-cnf-5gc.jpg)

![CNF 5GC](/images/5.3-cnf-5gc.jpg)

Values file provide many options to finutune setup. prepared helm values\_oai5gc.yaml reflects my specific lab.

Ie one can pick and choose ip addresses and interfaces to be used.

![CNF 5GC](/images/5.4-cnf-5gc.jpg)

Notes:

*   GTP is using separate interface G3
*   NGAP for AMF is listening on multus interface (not on LB exposed service)
*   in this lab different host intrafces are to be created with late binding (N2, N3). In lab I will be assigning to those diffewrent vlans/portgroups. I will be using mostly vlan 11. Only fot GTP traffic (N3) I will be using vlan 10
*   helm charts are recommended to be installed with help spray plugin (wit which one can orchestrate sequential line per require order ad maku sure to wait with next chart, umtil previous workload is up). I this case we are using basic helm, used by TCA, which meand that pods are started at same time. One can observe process where pods will be goint thru Error/Chrash states until reuqired pods are up. Takes some time for al pods to be in running state

Check if 5gc pods are successfully installed and started

```

❯ k get pods -n oai
NAME                                             READY   STATUS             RESTARTS      AGE
oai-5g-basi-efe22-awv5u-mysql-5657fdfb59-qnmhh   1/1     Running            0             2m22s
oai-amf-8498bdb5d8-k2dlh                         1/2     CrashLoopBackOff   3 (26s ago)   2m22s
oai-ausf-5fbfc868ff-k87tx                        0/1     CrashLoopBackOff   3 (40s ago)   2m22s
oai-nrf-5786955b46-ggqck                         1/1     Running            0             2m22s
oai-smf-c5fd4fc6d-84b88                          0/1     Running            3 (33s ago)   2m22s
oai-spgwu-tiny-7c44746c4d-rxfwf                  2/2     Running            0             2m22s
oai-udm-7bb6d55bdc-s58kl                         0/1     CrashLoopBackOff   3 (33s ago)   2m22s
oai-udr-6bc4b5dcf7-r9nwf                         1/1     Running            0             2m22s
```

Wait until all pods are in running state (with multiple restarts expected until all aligned in Running state):

```

❯ k get pods -n oai
NAME                                             READY   STATUS    RESTARTS        AGE
oai-5g-basi-efe22-awv5u-mysql-5657fdfb59-qnmhh   1/1     Running   0               4m50s
oai-amf-8498bdb5d8-k2dlh                         2/2     Running   5 (116s ago)    4m50s
oai-ausf-5fbfc868ff-k87tx                        1/1     Running   5 (2m14s ago)   4m50s
oai-nrf-5786955b46-ggqck                         1/1     Running   0               4m50s
oai-smf-c5fd4fc6d-84b88                          0/1     Running   5 (91s ago)     4m50s
oai-spgwu-tiny-7c44746c4d-rxfwf                  2/2     Running   0               4m50s
oai-udm-7bb6d55bdc-s58kl                         1/1     Running   4 (3m1s ago)    4m50s
oai-udr-6bc4b5dcf7-r9nwf                         1/1     Running   0               4m50s
```

AMF is key component and need sto be verified for proper function:

```

❯ k logs -f -n oai -c amf pod/oai-amf-8498bdb5d8-k2dlh
...

[2023-09-22 13:05:32.404] [config] [info] - AMF NAME.................: OAI-AMF
[2023-09-22 13:05:32.404] [config] [info] - GUAMI (MCC, MNC, Region ID, AMF Set ID, AMF pointer):
[2023-09-22 13:05:32.404] [config] [info]     (001, 01, 128, 1, 0)
[2023-09-22 13:05:32.404] [config] [info] - Served Guami List:
[2023-09-22 13:05:32.404] [config] [info]     (001, 01, 128 , 1, 0)
[2023-09-22 13:05:32.404] [config] [info] - Relative Capacity .......: 30
[2023-09-22 13:05:32.404] [config] [info] - PLMN Support:
[2023-09-22 13:05:32.404] [config] [info]     MCC, MNC ..............: 001, 01
[2023-09-22 13:05:32.404] [config] [info]     TAC ...................: 1
[2023-09-22 13:05:32.404] [config] [info]     Slice Support .........:
[2023-09-22 13:05:32.404] [config] [info]         SST ...............: 1
[2023-09-22 13:05:32.404] [config] [info]         SST, SD ...........: 1, 1 (0x1)
```

\-N2 interface created with multus:

```
[2023-09-22 13:05:32.404] [config] [info] - N2 Networking:
[2023-09-22 13:05:32.404] [config] [info]     Iface .................: n2
[2023-09-22 13:05:32.404] [config] [info]     IP Addr ...............: 172.16.11.21
[2023-09-22 13:05:32.404] [config] [info]     Port ..................: 38412
```

\-AMF will continuously show logs on how many gNB and Ues are registered:

```
|----------------------------------------------------------------------------------------------------------------
|----------------------------------------------------gNBs' information-------------------------------------------
|    Index    |      Status      |       Global ID       |       gNB Name       |               PLMN
|      -      |          -       |           -           |           -          |               -
|----------------------------------------------------------------------------------------------------------------

|----------------------------------------------------------------------------------------------------------------
|----------------------------------------------------UEs' information--------------------------------------------
| Index |      5GMM state      |      IMSI        |     GUTI      | RAN UE NGAP ID | AMF UE ID |  PLMN   |Cell ID
|----------------------------------------------------------------------------------------------------------------
```

\-n2 interface attached in AMF pod:

```

❯ k exec -it -n oai pod/oai-amf-8498bdb5d8-k2dlh -c amf -- ip a
…

3: n2@if4: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether fa:15:b1:86:af:fd brd ff:ff:ff:ff:ff:ff
    inet 172.16.11.21/24 brd 172.16.11.255 scope global n2
       valid_lft forever preferred_lft forever
    inet6 fe80::f815:b1ff:fe86:affd/64 scope link

```

\-check also for upf pod for n3 interface::

```

❯ k exec -it -n oai pod/oai-spgwu-tiny-7c44746c4d-rxfwf -- ip a
…
3: n3@n3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether a6:84:90:53:5e:c8 brd ff:ff:ff:ff:ff:ff
    inet 172.16.10.101/24 brd 172.16.10.255 scope global n3
       valid_lft forever preferred_lft forever
    inet6 fe80::a484:90ff:fe53:5ec8/64 scope link
       valid_lft forever preferred_lft forever

4: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP qlen 500
    link/[65534]
    inet 12.1.1.1/24 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::9793:cb72:59ce:f332/64 scope link
       valid_lft forever preferred_lft forever
```

**This means 5gc is running correctly.** 

# OAI O-RAN components install

components will be installed in split model, where we have CU-CP (CU Control Plane) and CU-UP (CU user plane) and DU (Distributed Unit) communicating between via F1 and E1 interfaces.

![O-RAN interfaces](/images/6-oran-intf.jpg)

I will be using separate cluster to host CU components (ie CU hosted in regional DC, core beign hosted in central DC). This can eihter be separate cluster per CU-CP and CU-UP, or as in this. case, I will be using dedicated nodepools for each.

In this way I will be honoring recommendation that single CNF is run per nodepool, so that late binding can do customization for that specific CNF (doing multiple customizations per different CNFs can  create unwanted situations).

## CU-CP setup (Central Unit - Control Plane componentd of gNB)

CU-CP component is 1st one that needs to be instantiated. This will register to AMF on behalf of gNB.

CU-CP will have 3 intefaces:

*   N2 - to reach AMF exposed interface (for NGAP / SCTP protocol)
*   F1C (on host instantiated as F1) - to communicate with DU via SCTP
*   E1 - SCTP communicaiton between CU-CP and CU-UP

When CU-CP is instantiated, communication if primarily going toward AMF  (gNB gets registered to AMF, 5gc), we want to monitor traffic on AMF side to catch traffic coming from cu-cp.

### CU-CP instantiation from cu-cp csar

We will be instantiating cu-cp instance on ran1 cluster, on np1 nodepool.

Image 7.1 oran cucp

![CU-CP instantiate](/images/7.1-oran-cucp.jpg)

\-currently we are selecting np1 nodepool for selection. Later we will make sure that cu-cp pode lands on that exact nodepool.

![CU-CP instantiate](/images/7.2-oran-cucp.jpg)

Again, disable helm verification or just ignore them (can this be confgiured as default per csar as default option ?)

![CU-CP instantiate](/images/7.3-oran-cucp.jpg)

In order to provide max isolation of all resources, we will be using both different nodepools and different namespaces.

![CU-CP instantiate](/images/7.4-oran-cucp.jpg)

![CU-CP instantiate](/images/7.5-oran-cucp.jpg)

![CU-CP instantiate](/images/7.6-oran-cucp.jpg)

CU-CP helm values file values\_gnb\_cu-cp.yaml is provided and that one reflect this specific setup. One can finetune setup by changing proivided values.yaml file

In values yaml file, node selector is used to land pod on nodepool np1 by selecting proper label.

Start cu-cp instantiation:

![CU-CP instantiate](/images/7.7-oran-cucp.jpg)

Before starting instantiation of cu-cp, we will prepare network traffic capture on amf, specifically on tcpdump sidecar when monitoring is enabled (tcpdump sidecar pod can be enabled via helm values, for most of pods).

```

❯ k exec -it -n oai pod/oai-amf-8498bdb5d8-k2dlh -c tcpdump -- sh

# tcpdump -i any -n -s1520 -w /tmp/amf-cucp-start.pcap
```

![CU-CP instantiate](/images/7.8-oran-cucp.jpg)

### cu-cp verification

cu-cp is started, check if pod is running on proper noodepool np1:

```

❯ k get pods -n oai -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP           NODE
oai-gnb-cu-cp-6cf98d85d5-x5wn5   2/2     Running   0          2m25s   100.96.1.4   ran1-np1-b85d666b-q5shg
```

Fetch pcap file for verification on laptop.

```

❯ kubectl cp -n oai  -c tcpdump oai-amf-8498bdb5d8-k2dlh:/tmp/amf-cucp-start.pcap amf-cucp-start.pcap
```

![CU-CP traffic](/images/8-oran-cucp-pcap.jpg)

AMF should show in logs that cu-cp registered.

\-check logs on amf pod to verifiy cu-cp is registered with name (still no Ues):

```

|----------------------------------------------------------------------------------------------------------------|
|----------------------------------------------------gNBs' information-------------------------------------------|
|    Index    |      Status      |       Global ID       |       gNB Name       |               PLMN             |
|      1      |    Connected     |         0x0         |         oai-cu-cp             |            001, 01
|----------------------------------------------------------------------------------------------------------------|

|----------------------------------------------------------------------------------------------------------------|
|----------------------------------------------------UEs' information--------------------------------------------|
| Index |      5GMM state      |      IMSI        |     GUTI      | RAN UE NGAP ID | AMF UE ID |  PLMN   |Cell ID|
|----------------------------------------------------------------------------------------------------------------|
```

## CU-UP setup (Central Unit - User Plane component of gNB)

CU-UP component needs to be started after cu-cp.

CU-UP will have 3 intefaces:

*   N3 - to transfer subscriber traffic from gNB to designated UPF (GTP)
*   F1U (on host instantiated as F1) - to communicate with DU via GTP (to carry subscriber traffic)
*   E1 - SCTP communicaiton between CU-CP and CU-UP

Prepration for traffic monitoring

*   we will be monitoring logs on cu-cp and upf.
*   When CU-UP is instantiated, communication if primarily going toward CU-CP  via E1AP protocol. There is no yet DU component.

### Instantiate from cu-up csar

We will be instantiating cu-cp instance on ran1 cluster, on np2 nodepool to keep it separated from cu-cp. Since this is still identical cluster, we will be using different namespace to further separate all rewsoures (i..e multus NAD resources).

Again, in this phase we are just selecting nodepool to do late binding cusotmizations, but later we will select via nodeselector / labels where cu-up pod will run.

We are using namespace oai2 and that needs to be consistently carried thru all steps.

Same as before, we will be using prepared helm values file values\_gnb\_cu-up.yaml which is set for this specific setup.

In helm values file we are stating on which nodepool cu-up will run (via labels assigned to np2 nodepool).

```
nodeSelector: { "workload": "cuup" }
```

Before instantiation, we need to prepare logs collection and traffic capture.

We want primarily to take logs and traffic on cu-cp, and perhaps logs on upf pod.

\-on cu-cp:

```
❯ k logs -f -n oai pod/oai-gnb-cu-cp-6cf98d85d5-x5wn5
```

\-on cu-cp:

```
❯ k exec -it -n oai pod/oai-gnb-cu-cp-6cf98d85d5-x5wn5 -c tcpdump -- sh

# tcpdump -I any -n -s1520 -w /tmp/cucp-cuup-start.pcap
```

\-on cluster wc1 on upf pod (no logs there, gtp session not yet crrated):

```
❯ k logs -f -n oai -c spgwu pod/oai-spgwu-tiny-7c44746c4d-rxfwf
```

### Verify cu-up 

Verify cu-up running on nodepool np2 in namespace oai2:

```

❯ k get pods -n oai2 -o wide
NAME                             READY   STATUS    RESTARTS   AGE    IP           NODE                        NOMINATED NODE   READINESS GATES
oai-gnb-cu-up-584c4d5f48-6dkzf   1/1     Running   0          3m1s   100.96.2.3   ran1-np2-6d6766cc56-wgwj6
```

CU-CP logs:

```
17851.356152 [E1AP] I CUCP received SCTP_NEW_ASSOCIATION_IND for instance 0
17851.356189 [E1AP] I CUCP received SCTP_NEW_ASSOCIATION_RESP for instance 0
17851.357963 [E1AP] I CUCP received SCTP_DATA_IND for instance 0
17851.358007 [E1AP] I Calling handler with instance 0
17851.358055 [E1AP] I CUCP received E1AP_SETUP_RESP for instance 0
17851.358074 [E1AP] I e1ap_send_SETUP_RESPONSE: Sending ITTI message to SCTP Task
```

Traffic on cu-cp:

```
❯ kubectl cp -n oai  -c tcpdump oai-gnb-cu-cp-6cf98d85d5-x5wn5:/tmp/cucp-cuup-start.pcap cucp-cuup-start.pcapwireshark check:

❯ Wireshark cucp-cuup-start.pcap&
```

![CU-UP traffic](/images/9-oran-cuup-pcap.jpg)

## DU setup (Distributed Unit of gNB)

DU needs to be instantiated after CU components.

DU will have  only 1 multus interface in lab

*   F1 - communicating toward CU-CP and CU-UP (sending both cp an up traffic over single interface)
*   Interface toward RU  (frontahul interafce) - in this case we don't have RU but we are simulating Radio. For this interface we will be using existing pod interface eth0. Straffic between du and Ue will be via antrea interface inside cluster

### Instantiate from du csar

We wil make sure that DU is run on cluster ran2 on nodepool np1 (and Ue will run on ran2-np2).

\-du will run on nodepool1 so thi is where we need late binding customizations (mainly create additional host interfaces).

We will be using prrepared helm values file values\_gnb\_du.yaml with required configuration for this setup.

To run du on ran2 cluster on np1, we are selecting it with nodeSelector/labels assigned to nodepool np1.

```
nodeSelector: { "workload": "du" }
```

### 5.3.2. Prepration for log and traffic capture

Traffic from DU is directed toward CU-CP and CU-UP.

We will be using combination of tcpdumo for traffic capture and also collecting logs.

\-on cucp we collect logs:

```

❯ k logs -f -n oai pod/oai-gnb-cu-cp-6cf98d85d5-x5wn5
...

22712.267023 [F1AP] I In F1AP connection, don't start GTP-U, as we have also E1AP
22712.268417 [F1AP] I Received Cell in 1 context
22712.268481 [NR_RRC] I Received F1 Setup Request from gNB_DU 3584 (oai-du-rfsim)
22712.268504 [NR_RRC] W instance 0 sib1 length 102
22712.271021 [F1AP] I Cell Configuration ok (assoc_id 3)
```

\-on cu-cp we capture traffic:

```
❯ k exec -it -n oai pod/oai-gnb-cu-cp-6cf98d85d5-x5wn5 -c tcpdump -- sh

/ # tcpdump -i any -n -s1520 -w /tmp/cucp-du-start.pcap
```

\-on cu-up we just take logs (no logs):

❯ k logs -f -n oai2 pod/oai-gnb-cu-up-584c4d5f48-6dkzf

\-downlod pcap file:

```
❯ kubectl cp -n oai  -c tcpdump oai-gnb-cu-cp-6cf98d85d5-x5wn5:/tmp/cucp-du-start.pcap cucp-du-start.pcap
```

\-wireshark decode:

```
❯ Wireshark cucp-du-start.pcap&
```

Image 10-oran-du

![DU traffic](/images/10-oran-du-pcap.jpg)

\-check du logs after du is up:

```

❯ k logs -f -n oai pod/oai-gnb-du-6fdbf5c65c-llgzc
…

22699.284112 [GNB_APP] I ngran_gNB_DU: Allocating ITTI message for F1AP_SETUP_REQ
22699.284234 [GNB_APP] I F1AP: gNB_DU_id[0] 3584
22699.284259 [GNB_APP] I F1AP: gNB_DU_name[0] oai-du-rfsim
22699.284264 [GNB_APP] I F1AP: tac[0] 1
22699.284268 [GNB_APP] I F1AP: mcc[0] 1
22699.284272 [GNB_APP] I F1AP: mnc[0] 1
22699.284277 [GNB_APP] I F1AP: mnc_digit_length[0] 2
22699.284280 [GNB_APP] I F1AP: nr_cellid[0] 12345678
22699.284283 [GNB_APP] I F1AP: CU_ip4_address in DU 172.16.13.24
22699.284287 [GNB_APP] I FIAP: CU_ip4_address in DU 0x7fd138004309, strlen 12
22699.284290 [GNB_APP] I F1AP: DU_ip4_address in DU 172.16.13.26
22699.284293 [GNB_APP] I FIAP: DU_ip4_address in DU 0x7fd138004349, strlen 12
22699.284305 [GNB_APP] I ngran_gNB_DU: Waiting for basic cell configuration
…

22699.413591 [HW] W rfsim: sample_rate 61440000.000000
22699.413615 [HW] I rfsimulator: running as server waiting opposite rfsimulators to connect
22699.413791 [HW] I [RAU] has loaded RFSIMULATOR device.
22699.413808 [PHY] I RU 0 Setting N_TA_offset to 800 samples (factor 2.000000, UL Freq 3600120, N_RB 106, mu 1)
22699.413815 [PHY] I Signaling main thread that RU 0 is ready, sl_ahead 6
22699.413846 [PHY] I RUs configured
22699.413856 [PHY] I init_eNB_afterRU() RC.nb_nr_inst:1
22699.413859 [PHY] I RC.nb_nr_CC[inst:0]:0x7fd14627c010
22699.413862 [PHY] I [gNB 0] phy_init_nr_gNB() About to wait for gNB to be configured
22699.425334 [GNB_APP] I Received F1AP_SETUP_RESP: associated ngran_gNB_CU oai-cu-cp with 0 cells to activate
22699.425358 [GNB_APP] I cells_to_activate 0, RRC instances 1
22699.426356 [GNB_APP] I Received F1AP_GNB_CU_CONFIGURATION_UPDATE: associated ngran_gNB_CU (null) with 1 cells to activate
```

\-verify du is functioning properly on np1 nodepool:

```

❯ k get pod -n oai -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE                        NOMINATED NODE   READINESS GATES
oai-gnb-du-6fdbf5c65c-llgzc   2/2     Running   0          5m14s   100.96.1.5   ran2-np1-75c4dc548d-cbj2t
```

## NR-UE setup (User Endopoint) instantiation

Finally we instantiate UE to emulatr sibscriber devide and geenrate some traffic thru RAN and 5gc.

*   UE will not have any specific network interface generated via late bindign process
*   usually we could egenrate fronthaul interface to communicate between UE/RU to DU. int his case UE pod will use default . In this case we don't have RU but we are simulating Radio. For this interface we will be using existing pod interface eth0. Straffic between du and Ue will be via antrea interface inside cluster

### Instantiate UE from ue csar

We wil make sure that UE is to run on cluster ran2 on nodepool np2, to have some separation from du (running on nodepool np1).

We will be using helm values file values\_ue.yaml with values set for this specifric setup. One can fine tune values to support different setup.

Logging and traffic capture when UE is registering:

we will be collecting traffic on cucp/amf for NGAP messages (UE registering to 5gc via AMF)

\-collecting logs on cu-cp:

```

❯ k logs -f -n oai pods/oai-gnb-cu-cp-6cf98d85d5-x5wn5

24939.735733 [F1AP] I Adding a new UE with RNTI 4938 and cu/du ue_f1ap_id 18744
…
24939.798401 [NGAP] I [gNB 0] Chose AMF 'OAI-AMF' (assoc_id 1) through selected PLMN Identity index 0 MCC 1 MNC 1
…
24941.458083 [E1AP] I CUCP received E1AP_BEARER_CONTEXT_SETUP_REQ for instance 0
…
nr_rrc_gNB_process_GTPV1U_CREATE_TUNNEL_RESP tunnel (2432060865) bearer UE context index 0, id 10, gtp addr len 4
24941.478252 [GTPU] E try to get a gtp-u not existing output
```

\-tcpdump cpature on cu-cp:

```
❯ k exec -it -n oai pod/oai-gnb-cu-cp-6cf98d85d5-x5wn5 -c tcpdump -- sh

/ # tcpdump -i any -n -s1520 -w /tmp/cucp-ue-start.pcap
```

\-collect logs on cu-up:

```

❯ k logs -f -n oai2 pods/oai-gnb-cu-up-584c4d5f48-6dkzf

...

[E1AP]   CUUP received SCTP_DATA_IND for instance 0
…
[GTPU]   [97] Created tunnel for UE ID 0, teid for incoming: 90f64dc1, teid for outgoing 1 to remote IPv4: 172.16.10.101, IPv6 ::
[PDCP]   ../../../openair2/LAYER2/nr_pdcp/nr_pdcp_e1_api.c:e1_add_drb:37: added DRB for UE ID 0
[GTPU]   [96] Created tunnel for UE ID 0, teid for incoming: 93b73104, teid for outgoing ffff to remote IPv4: 0.0.0.0, IPv6 ::
[E1AP]   e1apCUUP_send_BEARER_CONTEXT_SETUP_RESPONSE: Sending ITTI message to SCTP Task
[E1AP]   CUUP received SCTP_DATA_IND for instance 0
[E1AP]   Calling handler with instance 0
[E1AP]   Bearer context setup number of IEs 3
[GTPU]   [96] Tunnel Outgoing TEID updated to 7b5693cb and address to 1a0d10ac
```

\-collect logs on amf to show ue registered

```

❯ k logs -f -n oai -c amf pod/oai-amf-8498bdb5d8-k2dlh

|----------------------------------------------------------------------------------------------------------------|
|----------------------------------------------------gNBs' information-------------------------------------------|
|    Index    |      Status      |       Global ID       |       gNB Name       |               PLMN             |
|      1      |    Connected     |         0x0         |         oai-cu-cp             |            001, 01
|----------------------------------------------------------------------------------------------------------------|
|----------------------------------------------------------------------------------------------------------------|
|----------------------------------------------------UEs' information--------------------------------------------|
| Index |      5GMM state      |      IMSI        |     GUTI      | RAN UE NGAP ID | AMF UE ID |  PLMN   |Cell ID|
|      1|       5GMM-REGISTERED|   001010000000100|               |               0|          1| 001, 01 |      0|
|----------------------------------------------------------------------------------------------------------------|
```

Wireshard decode of traffic from cu-cp:

```

❯ kubectl cp -n oai -c tcpdump oai-gnb-cu-cp-6cf98d85d5-x5wn5:/tmp/cucp-ue-start.pcap cucp-ue-start.pcap
```

![UE traffic](/images/11-oran-ue-pcap.jpg)

Veriify pods:

```

❯ k get pods -n oai -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE
oai-gnb-du-6fdbf5c65c-llgzc   2/2     Running   0          51m   100.96.1.5   ran2-np1-75c4dc548d-cbj2t
oai-nr-ue-6c945b77-ztgwh      1/1     Running   0          14m   100.96.2.2   ran2-np2-5544d5f88b-m7jnn
```

\-Ue has receiver ip address 12.1.1.100

```

❯ k exec -it -n oai pod/oai-nr-ue-6c945b77-ztgwh  -- bash
root@oai-nr-ue-6c945b77-ztgwh:/opt/oai-nr-ue# ip a
…

3: oaitun_ue1: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none
    inet 12.1.1.100/24 brd 12.1.1.255 scope global oaitun_ue1
       valid_lft forever preferred_lft forever
    inet6 fe80::a46b:985e:8008:2a2e/64 scope link stable-privacy
```

# 6\. End-to-end verification

Generate traffic for 2e2 verification (Radi. Fala kurcu !!!!!!!)

```
Generate traffic in UE pod toward internet (trafifc must be pinned to UE created interface):

root@oai-nr-ue-6c945b77-ztgwh:/opt/oai-nr-ue# ping -I oaitun_ue1 8.8.8.8

PING 8.8.8.8 (8.8.8.8) from 12.1.1.100 oaitun_ue1: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=40.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=112 time=50.5 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=112 time=35.5 ms
```

\-collecting logs on different elements when traffic is generated

```

❯ k logs -f -n oai -c spgwu pod/oai-spgwu-tiny-7c44746c4d-rxfwf❯ k logs -f -n oai2 pods/oai-gnb-cu-up-584c4d5f48-6dkzf
…
[PDCP]   deliver_pdu_drb() (drb 1) sending message to gtp size 87
[PDCP]   deliver_pdu_drb() (drb 1) sending message to gtp size 87
[PDCP]   deliver_pdu_drb() (drb 1) sending message to gtp size 87
```

\-capture traffic on upf:

```

❯ k exec -it -n oai pod/oai-spgwu-tiny-7c44746c4d-rxfwf -c tcpdump -- sh

/ # tcpdump -i f1 -n -s1520 -w /tmp/du-ue-traffic.pcap
```

\-capture traffic on du on f1 interface:

```

❯ k exec -it -n oai pod/oai-gnb-du-6fdbf5c65c-llgzc -c tcpdump -- sh

/ # tcpdump -i f1 -n -s1520 -w /tmp/du-ue-traffic.pcap
```

\-collect pcap files from upf and du:

```

❯ kubectl cp -n oai -c tcpdump oai-gnb-du-6fdbf5c65c-llgzc:/tmp/du-ue-traffic.pcap du-ue-traffic.pcap
❯ kubectl cp -n oai -c tcpdump oai-spgwu-tiny-7c44746c4d-rxfwf:/tmp/upf-ue-traffic.pcap upf-ue-traffic.pcap
```

\-GTP-U between CU-UP/gNB and 5GC UPF:

![GTP traffic\(/images/12.1-oran-gtp-pcap.jpg)

\-gtp traffic between DU and CU-UP:

![GTP traffic](/images/12.2-oran-gtp-pcap.jpg)

**Fronthual traffic (between UE/RU and DU)**

FH traffic is usually between physical RU and DU (ie eCPRI protocol).

In this we have Rfsimulator, simulating traffic between UE and DU (since there is no RU). All RAN protocol layers are emulated.

This traffic is hight-thruput traffic which can be captures by tcpdump. Problem is that one can't easily decode it in wireshart, since there is no wireshark protocol dissector.

Still, one can get a sense of protocol messages exchanged over RF simulator

```

❯ k logs -f -n oai pod/oai-gnb-du-6fdbf5c65c-llgzc
…
29742.896222 [NR_MAC] I Frame.Slot 0.0
UE RNTI 4938 (1) PH 0 dB PCMAX 0 dBm, average RSRP -44 (16 meas)
UE 4938: CQI 0, RI 1, PMI (0,0)
UE 4938: dlsch_rounds 20077/0/0/0, dlsch_errors 0, pucch0_DTX 0, BLER 0.00000 MCS 9
UE 4938: dlsch_total_bytes 2467010
UE 4938: ulsch_rounds 173850/0/0/0, ulsch_DTX 0, ulsch_errors 0, BLER 0.00000 MCS 9
UE 4938: ulsch_total_bytes_scheduled 20166600, ulsch_total_bytes_received 20166600
UE 4938: LCID 1: 627 bytes TX
UE 4938: LCID 4: 147903 bytes TX
UE 4938: LCID 4: 148464 bytes RX
29746.307269 [NR_MAC] I Frame.Slot 128.0
UE RNTI 4938 (1) PH 0 dB PCMAX 0 dBm, average RSRP -44 (16 meas)
UE 4938: CQI 0, RI 1, PMI (0,0)
UE 4938: dlsch_rounds 20090/0/0/0, dlsch_errors 0, pucch0_DTX 0, BLER 0.00000 MCS 9
UE 4938: dlsch_total_bytes 2468609
UE 4938: ulsch_rounds 173978/0/0/0, ulsch_DTX 0, ulsch_errors 0, BLER 0.00000 MCS 9
UE 4938: ulsch_total_bytes_scheduled 20181448, ulsch_total_bytes_received 20181448
UE 4938: LCID 1: 627 bytes TX
UE 4938: LCID 4: 147903 bytes TX
UE 4938: LCID 4: 148464 bytes RX
29749.889501 [NR_MAC] I Frame.Slot 256.0
UE RNTI 4938 (1) PH 0 dB PCMAX 0 dBm, average RSRP -44 (16 meas)
UE 4938: CQI 0, RI 1, PMI (0,0)
UE 4938: dlsch_rounds 20103/0/0/0, dlsch_errors 0, pucch0_DTX 0, BLER 0.00000 MCS 9
UE 4938: dlsch_total_bytes 2470208
UE 4938: ulsch_rounds 174106/0/0/0, ulsch_DTX 0, ulsch_errors 0, BLER 0.00000 MCS 9
UE 4938: ulsch_total_bytes_scheduled 20196296, ulsch_total_bytes_received 20196296
UE 4938: LCID 1: 627 bytes TX
UE 4938: LCID 4: 147903 bytes TX
UE 4938: LCID 4: 148464 bytes RX
```

\-logs from NR UE pod:

```
❯ k logs -f -n oai pod/oai-nr-ue-6c945b77-ztgwh

30976.714262 [NR_PHY] I ============================================
30976.714306 [NR_PHY] I Harq round stats for Downlink: 24461/0/0
30976.714317 [NR_PHY] I ============================================
30978.702185 [NR_PHY] I ============================================
30978.702213 [NR_PHY] I Harq round stats for Downlink: 24471/0/0
30978.702220 [NR_PHY] I ============================================
30980.521758 [NR_PHY] I ============================================
30980.521801 [NR_PHY] I Harq round stats for Downlink: 24480/0/0
30980.521808 [NR_PHY] I ============================================
30982.634002 [NR_PHY] I ============================================
30982.634032 [NR_PHY] I Harq round stats for Downlink: 24492/0/0
30982.634041 [NR_PHY] I ============================================
30984.566771 [NR_PHY] I ============================================
30984.566809 [NR_PHY] I Harq round stats for Downlink: 24501/0/0
30984.566819 [NR_PHY] I ============================================

```

All clusters/pods in running sttaus

EU generating traffic over RAN, 5GC to Internet and back (ping)

![Final test](/images/13-terminal-final.jpg)

# TODO

Todo list:

*   near-RT RIC (E2 interface test)
*   Network slicing (TCA ?)

# Issues & observations

*   Removed ...
