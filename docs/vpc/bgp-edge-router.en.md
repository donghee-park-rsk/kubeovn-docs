# Bgp Edge Router

**Bgp Edge Router** is a routing solution that operates at the network boundary of Kube-OVN VPC, performing dynamic routing with external networks through BGP (Border Gateway Protocol) and has the following features:

- Advertises VPC internal subnet routes to external networks via BGP through bgp-edge-router-advertisements
- Learns routes from external BGP peers and adds them to the routing table
- Handle the routes that learn from and advertise to external BGP peer through gobgp-configs

At the same time, Bgp Edge Router has the following limitations:

- Uses macvlan for underlying network connectivity, requiring [Underlay support](../start/underlay.en.md#environment-requirements) from the underlying network

## Implementation Details

Each Edge Router consists of multiple Pods with multiple network interfaces. Each Pod has two network interfaces: one joins the virtual network for communication within the VPC, and the other connects to the underlying physical network via Macvlan for external network communication. Virtual network traffic ultimately accesses the external network through FIB within the Edge Router instances.

BGP Edge Router consists of the following components:

bgp-edge-routers
- Manages BGP sessions and handles route advertisement/learning
- Implements BGP protocol based on gobgp
- Establishes and maintains sessions with external BGP peers
- Adds learned routes to the routing table
- Performs actual traffic routing

bgp-edge-router-advertisements
- Adds advertising subnet
- Adds advertising subnet route to VPC

gobgp-configs
- Add routing policy per peer

## Requirements

The Bgp Edge Router is the same as the VPC NAT Gateway in that it requires [Multus-CNI](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md){: target = "_blank" }.

> No ConfigMap needs to be configured to use Bgp Edge Router.

## Usage

### Creating a Network Attachment Definition

The Bgp Edge Router uses multiple NICs to access both the VPC and the external network, so you need to create a Network Attachment Definition to connect to the external network. An example of using the `macvlan` plugin with IPAM provided by Kube-OVN is shown below:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: eth1
  namespace: default
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "kube-ovn",
        "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
        "provider": "eth1.default"
      }
    }'
---
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: macvlan1
spec:
  protocol: IPv4
  provider: eth1.default
  cidrBlock: 172.17.0.0/16
  gateway: 172.17.0.1
  excludeIps:
    - 172.17.0.0..172.17.0.10
```

> You can create a Network Attachment Definition with any CNI plugin to access the corresponding network.

For details on how to use multi-nic, please refer to [Manage Multiple Interface](../advance/multi-nic.en.md).

### Create Custom VPC, subnet
```yaml
kind: Vpc
apiVersion: kubeovn.io/v1
metadata:
  name: vpc-test
spec:
  bfdPort:
    enabled: true
    ip: 10.255.255.255
---
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: subnet99
spec:
  cidrBlock: 10.99.0.0/16
  gateway: 10.99.0.1
  protocol: IPv4
  provider: subnet99.kube-system.ovn
  vpc: vpc-dh
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: subnet99
  namespace: kube-system
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "kube-ovn",
      "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
      "provider": "subnet99.kube-system.ovn"
    }'
```

### Creating a Bgp Edge Router

Create a Bgp Edge Router resource as shown in the example below:

```yaml
apiVersion: kubeovn.io/v1
kind: BgpEdgeRouter
metadata:
  name: ber
  namespace: default
spec:
  vpc: vpc-test
  replicas: 2
  internalSubnet: subnet99
  externalSubnet: macvlan1
  internalIPs:
    - 10.99.0.10
    - 10.99.0.11
  externalIPs:
    - 172.17.0.10
    - 172.17.0.11
  bfd:
    enabled: true
    minRX: 300
    minTX: 300
    multiplier: 3
  policies:
    - snat: false
      subnets:
        - subnet100
  bgp:
    edgeRouterMode: true
    enabled: true
    image: kubeovn/kube-ovn:v1.15.0
    asn: 65010
    remoteAsn: 65000
    neighbors:
      - 10.101.0.10
    # if null, router ID set using external IP automatically
    routerId: ""
    enableGracefulRestart: true
```

The above resource creates a Bgp Edge Router named ber for VPC `vpc-test` under the default namespace, and all Pods under the `subnet99` subnet (10.99.0.0/16) within `vpc-test` VPC will access the external network via the `macvlan1` .

After the creation is complete, check out the Bgp Edge Router resource:

```shell
$ kubectl get ber
NAME   VPC        REPLICAS   BFD ENABLED   EXTERNAL SUBNET   PHASE       READY   AGE
ber    vpc-test   2          true          macvlan1          Completed   true    11s
```

To view the workload:

```shell
$ kubectl get deployment -l ovn.kubernetes.io/bgp-edge-router=ber
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
ber    2/2     2            2           4d

$ kubectl get deployment -l ovn.kubernetes.io/bgp-edge-router=ber -owide
NAME   READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS                IMAGES                                              SELECTOR
ber    2/2     2            2           4d    bfdd,bgp-router-speaker   kubeovn/kube-ovn:v1.15.0,kubeovn/kube-ovn:v1.15.0   app=bgp-edge-router,ovn.kubernetes.io/bgp-edge-router=ber
```

To view ip address and bgp setting in the Pod:

```shell
$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: net1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 06:b9:4c:77:80:a9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.10/16 brd 172.17.255.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::4b9:4cff:fe77:80a9/64 scope link 
       valid_lft forever preferred_lft forever
320: eth0@if321: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default 
    link/ether 66:38:5c:05:fe:91 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.99.0.10/16 brd 10.99.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::6438:5cff:fe05:fe91/64 scope link 
       valid_lft forever preferred_lft forever

$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- gobgp global
AS:        65010
Router-ID: 172.17.0.10
Listening Port: -1, Addresses: 0.0.0.0, ::

$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- gobgp neighbor
Peer            AS     Up/Down State       |#Received  Accepted
10.101.0.10 65000 4d 00:23:12 Establ      |        2         2
```

To view address set which use spec.policy and logical router policy:
```shell
$ kubectl ko nbctl lr-policy-list vpc-dh
Routing Policies
     31000                           ip4.dst == 10.100.0.0/16           allow
     31000                           ip4.dst == 10.101.0.0/16           allow
     31000                            ip4.dst == 10.99.0.0/16           allow
     29100                  ip4.src == $BER.3bb7b07e93aa.ipv4         reroute                10.99.0.10, 10.99.0.11               bfd
     29100                   ip4.src == $BER.3bb7b07e93aa_ip4         reroute                10.99.0.10, 10.99.0.11               bfd
     29090                  ip4.src == $BER.3bb7b07e93aa.ipv4            drop
     29090                   ip4.src == $BER.3bb7b07e93aa_ip4            drop

$ kubectl ko nbctl list address_set
_uuid               : b18ba940-2dba-446f-a973-cd37ae82dd3e
addresses           : ["10.100.0.0/16"]
external_ids        : {af="4", bgp-edge-router="default/ber", vendor=kube-ovn}
name                : BER.3bb7b07e93aa.ipv4
```


### Creating a Bgp Edge Router Advertisement
```yaml
apiVersion: kubeovn.io/v1
kind: BgpEdgeRouterAdvertisement
metadata:
  name: ber-adv
  namespace: default
spec:
  bgpEdgeRouter: "ber"
  subnet:
    - subnet100
    - subnet99
```
After the advertisement creation is complete, check out the Bgp Edge Router resource whether advertised properly:
```shell
$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- gobgp global rib
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.100.0.0/16        172.17.0.10                               00:04:49   [{Origin: i}]
*> 10.99.0.0/16         172.17.0.10                               00:04:49   [{Origin: i}]
```

Advertised subnet added to address set which used for logical route policy that bgp edge router created:
```shell
$ kubectl ko nbctl list address_set
_uuid               : b18ba940-2dba-446f-a973-cd37ae82dd3e
addresses           : ["10.99.0.0/16", "10.100.0.0/16"]
external_ids        : {af="4", bgp-edge-router="default/ber", vendor=kube-ovn}
name                : BER.3bb7b07e93aa.ipv4
```

### Creating a gobgp config
```yaml
apiVersion: kubeovn.io/v1
kind: GobgpConfig
metadata:
  name: ber-config
  namespace: default
spec:
  bgpEdgeRouterInfo:
    name: ber      
    namespace: default
  neighbors:
    - address: "10.101.0.10"
      toAdvertise:
        allowed:
          mode: "all" 
          prefixes:
            - "10.100.0.0/16"
      toReceive:
        allowed:
          mode: "all"
```
After the config creation is complete, check out the Bgp Edge Router resource whether policy applied properly:
Before the config created, the default action is accept all
```shell
$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- gobgp global policy
Import policy:
    Default: ACCEPT
Export policy:
    Default: ACCEPT
```

After the config created, only policy accepted
```shell
$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- gobgp global policy
Import policy:
    Default: REJECT
    Name policy-10.101.0.10-in:
        StatementName stmt-10.101.0.10-in:
          Conditions:
            PrefixSet: any prefix-10.101.0.10-in 
            NeighborSet: any neighbor-10.101.0.10
          Actions:
             accept
Export policy:
    Default: REJECT
    Name policy-10.101.0.10-out:
        StatementName stmt-10.101.0.10-out:
          Conditions:
            PrefixSet: any prefix-10.101.0.10-out 
            NeighborSet: any neighbor-10.101.0.10
          Actions:
             accept

$ kubectl exec ber-54ff969988-fvbmg -c bgp-router-speaker -- gobgp policy prefix
NAME                     PREFIX
prefix-10.101.0.10-in   0.0.0.0/0 0..32
prefix-10.101.0.10-out  10.100.0.0/16 16..16
```

### Configuration Parameters

#### BgpEdgeRouter

Spec :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `vpc` | `string` | Yes | Name of the default VPC (ovn-cluster) | VPC name where the BGP edge router will operate. | `vpc1` |
| `replicas` | `integer/int32` | Yes | `1` | Number of BGP edge router replicas. Must be between 0 and 10. | `2` |
| `prefix` | `string` | Yes | – | Prefix of the workload deployment name. This field is immutable. Must match DNS-label pattern. | `veg-` |
| `image` | `string` | Yes | – | The container image used by the workload deployment. | `docker.io/kubeovn/kube-ovn:v1.14.0-debug` |
| `internalSubnet` | `string` | Yes | Name of the default subnet within the VPC | Name of the subnet used to access the VPC network. | `subnet1` |
| `externalSubnet` | `string` | No | – | Name of the subnet used to access the external network. | `ext1` |
| `internalIPs` | `string array` | Yes | – | IP addresses used for accessing the VPC network. IPv4, IPv6, and dual-stack are supported. Must ≥ `replicas`. | `10.16.0.101,fd00::11` |
| `externalIPs` | `string array` | Yes | – | IP addresses used for accessing the external network. IPv4, IPv6, and dual-stack are supported. Must ≥ `replicas`. | `10.16.0.101,fd00::11` |
| `bfd` | `object` | Yes | – | BFD (Bidirectional Forwarding Detection) configuration settings. | – |
| `policies` | `object array` | Yes | – | Egress policies configuration. Must have at least one policy or selector; each policy needs ≥ one ipBlock or subnet. | – |
| `selectors` | `object array` | Yes | – | Configure egress by namespace/pod selectors. Must have ≥ one policy or selector. | – |
| `nodeSelector` | `object array` | Yes | – | Node selector for the workload. Supports matchLabels, matchExpressions, matchFields. | – |
| `trafficPolicy` | `string` | Yes | `Cluster` | Traffic policy: `Cluster` or `Local`. Effective only when BFD is enabled. | `Local` |
| `bgp` | `object` | Yes | – | BGP protocol configuration (ASN, neighbors, routing options). | – |

BFD Configuration :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `enabled` | `boolean` | Yes | `false` | Whether to enable BFD. | `true` |
| `minRX` | `integer/int32` | Yes | `1000` | BFD minimum receive interval (ms). | `500` |
| `minTX` | `integer/int32` | Yes | `1000` | BFD minimum transmit interval (ms). | `500` |
| `multiplier` | `integer/int32` | Yes | `3` | BFD detection multiplier (missed packets threshold). | `1` |

BGP Configuration :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `enabled` | `boolean` | Yes | `false` | Whether to enable BGP. | `true` |
| `edgeRouterMode` | `boolean` | Yes | `false` | Enable edge router mode. Automatically update pod FIB about received routes from neighbor | `true` |
| `routeServerClient` | `boolean` | Yes | `false` | Enable route server client mode. | `true` |
| `image` | `string` | Yes | – | Container image for the BGP speaker daemon. | `docker.io/osrg/gobgp:latest` |
| `asn` | `integer/int32` | Yes | – | BGP Autonomous System Number. Must be ≥ 0. | `65001` |
| `remoteAsn` | `integer/int32` | Yes | – | Remote BGP Autonomous System Number. Must be ≥ 0. | `65002` |
| `neighbors` | `string array` | Yes | – | List of BGP neighbor IPs. Supports IPv4 and IPv6. | `192.168.1.1,2001:db8::1` |
| `holdTime` | `string` | Yes | – | BGP hold time duration (`^[0-9]+[smhd]$`). | `180s` |
| `routerId` | `string` | Yes | – | BGP router ID (IPv4 or IPv6) or empty to use environment default. If null, it use pod's external IP automatically | `192.168.1.100` |
| `password` | `string` | Yes | – | BGP authentication password. | `secretpassword` |
| `enableGracefulRestart` | `boolean` | Yes | `false` | Enable BGP graceful restart capability. | `true` |
| `extraArgs` | `string array` | Yes | – | Additional command-line arguments for the BGP daemon. | `--log-level=debug` |

Selector Configuration :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `namespaceSelector` | `object` | Yes | – | Selector for namespaces. Must have ≥ one matchLabels or matchExpressions. | – |
| `podSelector` | `object` | Yes | – | Selector for Pods. Must have ≥ one matchLabels or matchExpressions. | – |

Label Selector :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `matchLabels` | `object` | Yes | – | Map of label key:value pairs for exact matching. | `app: web, tier: frontend` |
| `matchExpressions` | `object array` | Yes | – | List of label selector requirements. | – |

Match Expression :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `key` | `string` | No | – | Label key to match. | `environment` |
| `operator` | `string` | No | – | Relationship between key and values. Valid: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`. | `In` |
| `values` | `string array` | Yes | – | Array of values for the operator. | `production, staging` |

Policy Configuration :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `snat` | `boolean` | Yes | `false` | Enable Source Network Address Translation for matched traffic. | `true` |
| `ipBlocks` | `string array` | Yes | – | IP blocks to match (IPv4, IPv6, CIDR). Must have ≥ one ipBlock or subnet. | `192.168.0.0/16` / `10.0.0.0/8` |
| `subnets` | `string array` | Yes | – | Subnet names to match. Must have ≥ one ipBlock or subnet. | `subnet1,subnet2` |

Status :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `phase` | `string` | No | `Pending` | Current phase. Valid: `Pending`, `Processing`, `Completed`. | `Completed` |
| `ready` | `boolean` | No | `false` | Whether the BgpEdgeRouter is ready for operation. | `true` |
| `replicas` | `integer/int32` | Yes | – | Current number of replicas. | `2` |
| `labelSelector` | `string` | Yes | – | Label selector for the workload pods. | `app=bgp-edge-router` |
| `internalIPs` | `string array` | Yes | – | List of internal IPs assigned to the BgpEdgeRouter. | `10.16.0.101` / `10.16.0.102` |
| `externalIPs` | `string array` | Yes | – | List of external IPs assigned to the BgpEdgeRouter. | `192.168.1.101` / `192.168.1.102` |
| `workload` | `object` | Yes | – | Workload resource managing the BgpEdgeRouter (Deployment, Pods). | – |
| `conditions` | `object array` | No | – | List of conditions describing the current status. | – |

#### BgpEdgeRouterAdvertisement

Spec :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `subnet` | `string array` | No | – | List of subnet names to advertise via BGP. Must contain ≥ 1 item. Items must be unique. | `subnet1` / `subnet2` |
| `bgpEdgeRouter` | `string` | No | – | Name of the BgpEdgeRouter resource for advertisement. | `my-bgp-router` |

Status :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ready` | `boolean` | No | `false` | Whether the advertisement is ready. | `true` |
| `conditions` | `object array` | No | – | List of conditions describing the advertisement status. | – |


#### GobgpConfig

Spec :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `bgpEdgeRouterInfo` | `object` | No | – | Information about the BgpEdgeRouter to configure. | – |
| `neighbors` | `object array` | No | – | List of BGP neighbor configurations. Each must include `address`, `toAdvertise`, `toReceive`. | – |

BgpEdgeRouterInfo :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `name` | `string` | Yes | – | Name of the BgpEdgeRouter resource. | `my-bgp-router` |
| `namespace` | `string` | Yes | – | Namespace of the BgpEdgeRouter resource. | `kube-system` |

BGP Neighbor Configuration :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `address` | `string` | No | – | BGP neighbor IP address (IPv4 or IPv6). | `192.168.1.1` |
| `toAdvertise` | `object` | No | – | Policy configuration for routes to advertise to this neighbor. | – |
| `toReceive` | `object` | No | – | Policy configuration for routes to receive from this neighbor. | – |

Advertisement/Receive Policy :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `allowed` | `object` | No | – | Allowed routes policy. | – |

Policy Allowed Configuration :

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `mode` | `string` | No | – | Policy mode for filtering: `all`, `none`, `filtered`, etc. When `mode: filtered` only prefixes allowed | `filtered` |
| `prefixes` | `string array` | Yes | – | Prefix list when `mode: filtered`. Must be valid IPv4/IPv6 CIDRs. | `192.168.0.0/16` / `10.0.0.0/8` |

Status

| Fields | Type | Optional | Default Value | Description | Example |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ready` | `boolean` | No | `false` | Whether the GoBGP configuration is ready. | `true` |
| `conditions` | `object array` | No | – | List of conditions describing the configuration status. | – |
