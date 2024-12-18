## Binary Build Command
go mod vendor
make
## Container image Build Command
sudo podman build --build-arg TAG=v1.5.1-flannel2 --build-arg GOLANG_VERSION=1.21 -t cni-flannel-wyou:v1.5.1 -f Dockerfile

## Overview
This plugin is designed to work in conjunction with [flannel](https://github.com/coreos/flannel), a network fabric for containers.
When flannel daemon is started, it outputs a `/run/flannel/subnet.env` file that looks like this:
```bash
FLANNEL_NETWORK=10.1.0.0/16
FLANNEL_SUBNET=10.1.17.1/24
FLANNEL_MTU=1472
FLANNEL_IPMASQ=true
```

This information reflects the attributes of flannel network on the host.
The flannel CNI plugin uses this information to configure another CNI plugin, such as bridge plugin.

## Build

Run `go mod vendor` and then `make`. The resulting `flannel` binary would be under the bin/ directory

## Operation
Given the following network configuration file and the contents of `/run/flannel/subnet.env` above,
```json
{
	"name": "mynet",
	"type": "flannel"
}
```
the flannel plugin will generate another network configuration file:
```json
{
	"name": "mynet",
	"type": "bridge",
	"mtu": 1472,
	"ipMasq": false,
	"isGateway": true,
	"ipam": {
		"type": "host-local",
		"subnet": "10.1.17.0/24"
	}
}
```

It will then invoke the bridge plugin, passing it the generated configuration.

As can be seen from above, the flannel plugin, by default, will delegate to the bridge plugin.
If additional configuration values need to be passed to the bridge plugin, it can be done so via the `delegate` field:
```json
{
	"name": "mynet",
	"type": "flannel",
	"delegate": {
		"bridge": "mynet0",
		"mtu": 1400
	}
}
```

This supplies a configuration parameter to the bridge plugin -- the created bridge will now be named `mynet0`.
Notice that `mtu` has also been specified and this value will not be overwritten by flannel plugin.

Additionally, the `delegate` field can be used to select a different kind of plugin altogether.
To use `ipvlan` instead of `bridge`, the following configuration can be specified:

```json
{
	"name": "mynet",
	"type": "flannel",
	"delegate": {
		"type": "ipvlan",
		"master": "eth0"
	}
}
```

The `ipam` part of the network configuration generated for the delegated plugin, can also be customized by adding a base `ipam` section to the input flannel network configuration. This `ipam` element is then updated with the flannel subnet, a route to the flannel network and, unless provided, an `ipam` `type` set to `host-local` before being provided to the delegated plugin.

## Network configuration reference

* `name` (string, required): the name of the network
* `type` (string, required): "flannel"
* `subnetFile` (string, optional): full path to the subnet file written out by flanneld. Defaults to /run/flannel/subnet.env
* `dataDir` (string, optional): path to directory where plugin will store generated network configuration files. Defaults to `/var/lib/cni/flannel`
* `delegate` (dictionary, optional): specifies configuration options for the delegated plugin.
* `ipam` (dictionary, optional, Linux only): when specified, is used as basis to construct the `ipam` section of the delegated plugin configuration.

flannel plugin will always set the following fields in the delegated plugin configuration:

* `name`: value of its "name" field.
* `ipam`: based on the received `ipam` section if present, with a `type` defaulting to `host-local`, a `subnet` set to `$FLANNEL_SUBNET` and (Linux only) a `routes` element assembled from the routes listed in the received `ipam` element and a route to the flannel network. Other fields present in the input `ipam` section will be transparently provided to the delegate.

flannel plugin will set the following fields in the delegated plugin configuration if they are not present:
* `ipMasq`: the inverse of `$FLANNEL_IPMASQ`
* `mtu`: `$FLANNEL_MTU`

Additionally, for the bridge plugin, `isGateway` will be set to `true`, if not present.

The bridge plugin includes `isDefaultGateway` that when false allow adding back the routes to the cluster services and/or to the hosts. In that case, the bridge plugin does not install a default route and, as a result, only pod-to-pod connectivity would be available.

## Windows Support (Experimental)
This plugin supports delegating to the windows CNI plugins (overlay.exe, l2bridge.exe) to work in conjunction with [Flannel on Windows](https://github.com/coreos/flannel/issues/833). 
Flannel sets up an [HNS Network](https://docs.microsoft.com/en-us/virtualization/windowscontainers/container-networking/architecture) in L2Bridge mode for host-gw and in Overlay mode for vxlan. 

The following fields must be set in the delegated plugin configuration:
* `name` (string, required): the name of the network (must match the name in Flannel config / name of the HNS network)
* `type` (string, optional): set to `win-l2bridge` by default. Can be set to `win-overlay` or other custom windows CNI
* `ipMasq`: the inverse of `$FLANNEL_IPMASQ`
* `endpointMacPrefix` (string, optional): required for `win-overlay` mode, set to the MAC prefix configured for Flannel  
* `clusterNetworkPrefix` (string, optional): required for `win-l2bridge` mode, setup NAT if `ipMasq` is set to true

For `win-l2bridge`, the Flannel CNI plugin will set:
* `ipam`: "host-local" type will be used with "subnet" set to `$FLANNEL_SUBNET` and gateway as the .2 address in `$FLANNEL_NETWORK`

For `win-overlay`, the Flannel CNI plugin will set:
* `ipam`: "host-local" type will be used with "subnet" set to `$FLANNEL_SUBNET` and gateway as the .1 address in `$FLANNEL_NETWORK`

If IPMASQ is true, the Flannel CNI plugin will setup an OutBoundNAT policy and add FLANNEL_SUBNET to any existing exclusions.

All other delegate config e.g. other HNS endpoint policies in AdditionalArgs will be passed to WINCNI as-is.    

Example VXLAN Flannel CNI config
```json
{
	"name": "mynet",
	"type": "flannel",
	"delegate": {
		"type": "win-overlay",
		"endpointMacPrefix": "0E-2A"
	}
}
```

For this example, Flannel CNI would generate the following config to delegate to the windows CNI when FLANNEL_NETWORK=10.244.0.0/16, FLANNEL_SUBNET=10.244.1.0/24 and IPMASQ=true
```json
{
	"name": "mynet",
	"type": "win-overlay",
	"endpointMacPrefix": "0E-2A",
	"ipMasq": true,
	"ipam": {
		"subnet": "10.244.1.0/24",
		"type": "host-local"
	}
}
```
