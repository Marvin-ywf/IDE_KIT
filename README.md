# Test case overview

* [Multus-CNI](#Multus-CNI)
    * [bridge plugin](#bridgeplugin)
    * [macvlan plugin]()
    * [ipvlan plugin]()
    * [ptp plugin]()
* [NFD](#)
    * [node selector]()
    * [affinity]()
* [SRIOV plugin]()
    * [single VF allocated]()
    * [multiple VF allocated]()
* [CMK]()

## Multus-CNI
Multus CNI is a container network interface (CNI) plugin for Kubernetes that enables attaching multiple network interfaces to pods. Typically, in Kubernetes each pod only has one network interface (apart from a loopback) -- with Multus you can create a multi-homed pod that has multiple interfaces. This is accomplished by Multus acting as a "meta-plugin", a CNI plugin that can call multiple other CNI plugins.

### bridge plugin
#### Overview

With bridge plugin, all containers (on the same host) are plugged into a bridge (virtual switch) that resides in the host network namespace.
The containers receive one end of the veth pair with the other end connected to the bridge.
An IP address is only assigned to one end of the veth pair -- one residing in the container.
The bridge itself can also be assigned an IP address, turning it into a gateway for the containers.
Alternatively, the bridge can function purely in L2 mode and would need to be bridged to the host network interface (if other than container-to-container communication on the same host is desired).

The network configuration specifies the name of the bridge to be used.
If the bridge is missing, the plugin will create one on first use and, if gateway mode is used, assign it an IP that was returned by IPAM plugin via the gateway field.

#### Example configuration
```
{
    "cniVersion": "0.3.1",
    "name": "mynet",
    "type": "bridge",
    "bridge": "mynet0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.10.0.0/16"
    }
}
```

#### Example L2-only configuration
```
{
    "cniVersion": "0.3.1",
    "name": "mynet",
    "type": "bridge",
    "bridge": "mynet0",
    "ipam": {}
}
```

#### Network configuration reference

* `name` (string, required): the name of the network.
* `type` (string, required): "bridge".
* `bridge` (string, optional): name of the bridge to use/create. Defaults to "cni0".
* `isGateway` (boolean, optional): assign an IP address to the bridge. Defaults to false.
* `isDefaultGateway` (boolean, optional): Sets isGateway to true and makes the assigned IP the default route. Defaults to false.
* `forceAddress` (boolean, optional): Indicates if a new IP address should be set if the previous value has been changed. Defaults to false.
* `ipMasq` (boolean, optional): set up IP Masquerade on the host for traffic originating from this network and destined outside of it. Defaults to false.
* `mtu` (integer, optional): explicitly set MTU to the specified value. Defaults to the value chosen by the kernel.
* `hairpinMode` (boolean, optional): set hairpin mode for interfaces on the bridge. Defaults to false.
* `ipam` (dictionary, required): IPAM configuration to be used for this network. For L2-only network, create empty dictionary.
* `promiscMode` (boolean, optional): set promiscuous mode on the bridge. Defaults to false.
* `vlan` (int, optional): assign VLAN tag. Defaults to none.

*Note:* The VLAN parameter configures the VLAN tag on the host end of the veth and also enables the vlan_filtering feature on the bridge interface.

*Note:* To configure uplink for L2 network you need to allow the vlan on the uplink interface by using the following command ``` bridge vlan add vid VLAN_ID dev DEV```.

### macvlan plugin

#### Overview

[macvlan](http://backreference.org/2014/03/20/some-notes-on-macvlanmacvtap/) functions like a switch that is already connected to the host interface.
A host interface gets "enslaved" with the virtual interfaces sharing the physical device but having distinct MAC addresses.
Since each macvlan interface has its own MAC address, it makes it easy to use with existing DHCP servers already present on the network.

#### Example configuration

```
{
	"name": "mynet",
	"type": "macvlan",
	"master": "eth0",
	"ipam": {
		"type": "dhcp"
	}
}
```

#### Network configuration reference

* `name` (string, required): the name of the network
* `type` (string, required): "macvlan"
* `master` (string, optional): name of the host interface to enslave. Defaults to default route interface.
* `mode` (string, optional): one of "bridge", "private", "vepa", "passthru". Defaults to "bridge".
* `mtu` (integer, optional): explicitly set MTU to the specified value. Defaults to the value chosen by the kernel. The value must be \[0, master's MTU\].
* `ipam` (dictionary, required): IPAM configuration to be used for this network. For interface only without ip address, create empty dictionary.

#### Notes

* If are testing on a laptop, please remember that most wireless cards do not support being enslaved by macvlan.
* A single master interface can not be enslaved by both `macvlan` and `ipvlan`.

### ipvlan plugin

#### Overview

ipvlan is a new [addition](https://lwn.net/Articles/620087/) to the Linux kernel.
Like its cousin macvlan, it virtualizes the host interface.
However unlike macvlan which generates a new MAC address for each interface, ipvlan devices all share the same MAC.
The kernel driver inspects the IP address of each packet when making a decision about which virtual interface should process the packet.

Because all ipvlan interfaces share the MAC address with the host interface, DHCP can only be used in conjunction with ClientID (currently not supported by DHCP plugin).

#### Example configuration

```
{
	"name": "mynet",
	"type": "ipvlan",
	"master": "eth0",
	"ipam": {
		"type": "host-local",
		"subnet": "10.1.2.0/24"
	}
}
```

#### Network configuration reference

* `name` (string, required): the name of the network.
* `type` (string, required): "ipvlan".
* `master` (string, required unless chained): name of the host interface to enslave.
* `mode` (string, optional): one of "l2", "l3", "l3s". Defaults to "l2".
* `mtu` (integer, optional): explicitly set MTU to the specified value. Defaults to the value chosen by the kernel.
* `ipam` (dictionary, required unless chained): IPAM configuration to be used for this network.

#### Notes

* `ipvlan` does not allow virtual interfaces to communicate with the master interface.
Therefore the container will not be able to reach the host via `ipvlan` interface.
Be sure to also have container join a network that provides connectivity to the host (e.g. `ptp`).
* A single master interface can not be enslaved by both `macvlan` and `ipvlan`.
* For IP allocation schemes that cannot be interface agnostic, the ipvlan plugin
can be chained with an earlier plugin that handles this logic. If `master` is
omitted, then the previous Result must contain a single interface name for the
ipvlan plugin to enslave. If `ipam` is omitted, then the previous Result is used
to configure the ipvlan interface.

### ptp plugin

#### Overview
The ptp plugin creates a point-to-point link between a container and the host by using a veth device.
One end of the veth pair is placed inside a container and the other end resides on the host.
The host-local IPAM plugin can be used to allocate an IP address to the container.
The traffic of the container interface will be routed through the interface of the host.

#### Example network configuration

```
{
	"name": "mynet",
	"type": "ptp",
	"ipam": {
		"type": "host-local",
		"subnet": "10.1.1.0/24"
	},
	"dns": {
		"nameservers": [ "10.1.1.1", "8.8.8.8" ]
	}
}
```

#### Network configuration reference

* `name` (string, required): the name of the network
* `type` (string, required): "ptp"
* `ipMasq` (boolean, optional): set up IP Masquerade on the host for traffic originating from ip of this network and destined outside of this network. Defaults to false.
* `mtu` (integer, optional): explicitly set MTU to the specified value. Defaults to value chosen by the kernel.
* `ipam` (dictionary, required): IPAM configuration to be used for this network.
* `dns` (dictionary, optional): DNS information to return as described in the [Result](https://github.com/containernetworking/cni/blob/master/SPEC.md#result).


#### SRIOV-plugin
Some CNI plugins require specific device information which maybe pre-allocated by K8s device plugin. This could be indicated by providing `k8s.v1.cni.cncf.io/resourceName` annotaton in its network attachment definition CRD. The file [`examples/sriov-net.yaml`](./sriov-net.yaml) shows an example on how to define a Network attachment definition with specific device allocation information. Multus will get allocated device information and make them available for CNI plugin to work on.

In this exmaple (shown below), it is expected that an [SRIOV Device Plugin](https://github.com/intel/sriov-network-device-plugin/) making a pool of SRIOV VFs available to the K8s with `intel.com/sriov` as their resourceName. Any device allocated from this resource pool will be passed down by Multus to the [sriov-cni](https://github.com/intel/sriov-cni/tree/dev/k8s-deviceid-model) plugin in `deviceID` field. This is up to the sriov-cni plugin to capture this information and work with this specific device information.

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net-a
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/sriov
spec:
  config: '{
  "type": "sriov",
  "vlan": 1000,
  "ipam": {
    "type": "host-local",
    "subnet": "10.56.217.0/24",
    "rangeStart": "10.56.217.171",
    "rangeEnd": "10.56.217.181",
    "routes": [{
      "dst": "0.0.0.0/0"
    }],
    "gateway": "10.56.217.1"
  }
}'
```
The [net-resource-sample-pod.yaml](./net-resource-sample-pod.yaml) is an exmaple Pod manifest file that requesting a SRIOV device from a host which is then configured using the above network attachement definition.

>For further information on how to configure SRIOV Device Plugin and SRIOV-CNI please refer to the links given above.
