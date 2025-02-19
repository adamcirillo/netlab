# Platform Caveats

```eval_rst
.. contents:: Table of Contents
   :depth: 1
   :local:
   :backlinks: none
```

(caveats-eos)=
## Arista EOS

* Routed VLANs cannot be used in EVPN MPLS VLAN bundles
* Arista EOS uses an [invalid value for the suboption 150 of the DHCP option 82](https://blog.ipspace.net/2023/03/netlab-vrf-dhcp-relay.html#vendor-interoperability-is-fun) when doing inter-VRF DHCPv4 relaying.
* The DHCP client on Arista EOS is finicky. When the DHCP state changes on one of the data-plane Ethernet interfaces, the management interface might lose its IPv4 address.
* You can set Arista cEOS serial number and system MAC address with the **eos.serialnumber** and **eos.systemmacaddr** node properties.
* Use **libvirt.uuid** node property to ensure a vEOS VM does not change its serial number every time you start the lab.
* Arista EOS does not support routed port-channel interfaces. Port channel interfaces can be used only as VLAN trunks or VLAN access interfaces.
* Anycast gateways and DHCP/DHCPv6 clients do not work on Arista cEOS Ethernet interfaces.
* cEOS MPLS data plane was introduced in release 4.32.1F.

The default name of the management interface is **Management0** on vEOS and **Management1** on cEOS. If you'd like to change the management interface name on cEOS:

* Create `intf_map` file in the topology directory with the following contents:

```
{
  "ManagementIntf": {
    "eth0": "Management0"
  }
}
```

* Map that file into the `/mnt/flash/EosIntfMapping_json` container file in the node **clab.binds** dictionary, for example:

```
nodes:
  r:
    device: eos
    clab.binds:
      intf_map: /mnt/flash/EosIntfMapping_json
```

(caveats-aruba)=
## Aruba AOS-CX

* Ansible automation of Aruba AOS-CX requires the installation of the [ArubaNetworks Ansible Collection](https://galaxy.ansible.com/arubanetworks/aoscx) with `ansible-galaxy collection install arubanetworks.aoscx`.
* The limitations of the Aruba AOS-CX Simulator can be found [here](https://feature-navigator.arubanetworks.com/), selecting *CX Simulator* as platform.
* It seems that the Aruba AOS-CX Simulator cannot generate ICMP Fragmentation-Needed messages.

### VRF and L3VPN Caveats

* OSPF processes can be only *1-63*. VRF indexes usually are > 100, so a device tweak will map every *vrfidx* to a different OSPF process ID. That means you cannot have more than 62 VRF using OSPF.
* On the Aruba AOS-CX Virtual version *10.11.0001*, the MPLS L3VPN forwarding plane seems broken (while the control plane works fine).

### VXLAN and EVPN Caveats

* ECMP is not supported.
* The VXLAN data plane (at least, on the virtual version) does not support VNI greater than 65535. If you set a higher value, an overflow will occur, and you may have overlapping VNIs. The workaround for this is to set `defaults.vxlan.start_vni: 20000` and `defaults.evpn.start_transit_vni: 10000` (especially on multi-vendor topologies).
* EVPN Symmetric IRB is supported only from the Aruba AOS-CX Virtual version *10.13*.
* CPU-generated traffic is not encapsulated in Symmetric IRB on the AOS-CX Simulator.
* Active-Gateway MAC Addresses shall be the same across all VTEPs in AOS-CX Simulator.
* Manually Setting Virtual-MAC for EVPN RT-5 is not working (but an autogenerated one is fine).
* AOS-CX doing [centralized VXLAN-to-VXLAN routing](https://github.com/ipspace/netlab/blob/dev/tests/integration/evpn/04-vxlan-central-routing.yml) does not work with Linux-based edge switches bridging from VXLAN to edge VLANs. It works flawlessly with Arista cEOS and Cisco Nexus OS as edge devices.

(caveats-bird)=
## BIRD Internet Routing Daemon

* You must build the BIRD container image with the **netlab clab build bird** command.
* BIRD is implemented as a pure control-plane daemon running on a Linux VM or as a container with a single external interface. You can set the node **role** to **router** to turn a BIRD instance into a more traditional networking device with a loopback interface.
* _netlab_ installs BIRD software in a container image or a VM running Ubuntu 22.04. The current version of BIRD shipping with Ubuntu 22.04 is 2.0.8.
* BIRD supports a single router ID used for BGP and OSPF.
* The VM or container running BIRD starts with static routes pointing to one of the adjacent routers (see [host routes on Linux](linux-routes)). After establishing routing adjacencies, BIRD copies BGP and OSPF into the kernel IP routing table.

### OSPF Caveats

* BIRD OSPF implementation has no *reference bandwidth*. The default OSPF cost is 10.

### BGP Caveats

* You must run OSPF on the BIRD daemon for the IBGP sessions to work.
* BIRD will not advertise (reflect) an IBGP route if it has an equivalent OSPF route.
* You cannot configure BGP community propagation on BIRD. All BGP communities are always propagated to all neighbors.

### IPv6 Caveats

* OSPFv3 does not advertise the prefix configured on the loopback interface even when the loopback interface is part of the OSPFv3 process.
* If the BGP next hop of a reflected IBGP route is reachable as an OSPF route, BIRD advertises a link-local address as one of the next hops of the IBGP IPv6 prefix, potentially resulting in broken IPv6 connectivity.

(caveats-asav)=
## Cisco ASAv Caveats

* Some ASAv versions use older SSH protocols. For more details, see the [Cisco IOSv SSH caveats](cisco-iosv-ssh).

(caveats-cat8000v)=
## Cisco Catalyst 8000v

* Apart from the VLAN configuration, Catalyst 8000v implementation uses the same configuration templates as CSR 1000v.
* Catalyst 8000v accepts CSR 1000v-based VXLAN configuration, but the validation tests fail. You cannot configure VXLAN on Catalyst 8000v with the current _netlab_ release.
* MPLS and SR-MPLS require a license that enables the advanced functionality after a reboot. That license is automatically enabled in recent _netlab_ and _vrnetlab_ releases, but you cannot apply it to a running lab; you have to rebuild the Catalyst 8000v Vagrant box or container.

See also [CSR 1000v](caveats-csr) and [Cisco IOSv](caveats-iosv) caveats.

(caveats-csr)=
## Cisco CSR 1000v

* Cisco CSR 1000v does not support interface MTU lower than 1500 bytes or IP MTU higher than 1500 bytes.
* The minimum VXLAN VNI accepted by Cisco CSR 1000v is 4096. Using lower VNI values triggers a configuration error that is not caught by Ansible, resulting in a weird failure of the **netlab initial** command.

See also [Cisco IOSv](caveats-iosv) SSH, OSPF, RIPng, and BGP caveats.

(caveats-iosv)=
## Cisco IOSv and IOSvL2

* Cisco IOSv release 15.x does not support unnumbered interfaces. Use Cisco CSR 1000v.
* BGP configuration is optimized for reasonable convergence times under lab conditions. Do not use the same settings in a production network.
* Multiple OSPFv2 processes on Cisco IOS cannot have the same OSPF router ID. By default, _netlab_ generates the same router ID for global and VRF OSPF processes, resulting in non-fatal configuration errors that Ansible silently ignores.
* It's impossible to configure RIPv2 on individual subnets on Cisco IOS. RIPv2 might be running on more interfaces than intended. _netlab_ configures those interfaces to be *passive*.
* Cisco IOS does not support passive interfaces in RIPng.
* Cisco IOS requires a *default metric* when redistributing routes into RIPv2. The RIPv2 configuration template sets the default metric to the value of the **netlab_ripv2_default_metric** node parameter (default: 5)
* You cannot use VLANs 1002 through 1005 with Cisco IOSvL2 image 

(cisco-iosv-ssh)=
### SSH Access to Cisco IOSv

Cisco IOSv SSH implementation uses RSA keys and older encryption algorithms that might not be allowed on newer Linux distributions.

Most users running recent Ansible versions won't notice the problem; Ansible uses the `ansible-pylibssh` package (installed together with Ansible) as its interface to `libssh` and adjusts the SSH algorithms as needed.

We added a similar mechanism to _netlab_ commands that use SSH to connect to network devices. These commands append group variable `netlab_ssh_args` (when defined) to the **ssh** command; the value of that variable for Cisco IOS/IOS-XE devices is set to:

```
group_vars:
  netlab_ssh_args: "-o KexAlgorithms=+diffie-hellman-group-exchange-sha1 -o PubkeyAcceptedKeyTypes=ssh-rsa -o HostKeyAlgorithms=+ssh-rsa"
```

You can change the additional SSH arguments with the node **netlab_ssh_args** parameter or with the **defaults.devices._device_.group_vars.netlab_ssh_args** [system default](defaults.md).

Additionally, you might have to execute `sudo update-crypto-policies --set LEGACY` on AlmaLinux/RHEL.

(caveats-iol)=
## Cisco IOS on Linux (IOL) and IOL Layer-2 Image

* The Cisco IOL and IOL L2 images work only as containers created with Roman Dodin's fork of [vrnetlab](https://github.com/hellt/vrnetlab/).
* You need Containerlab 0.59.0 or greater to run these images.
* You cannot use VLANs 1002 through 1005 with Cisco IOL layer-2 image

See also [common Cisco IOS](caveats-iosv) caveats.

(caveats-iosxr)=
## Cisco IOS XR

### Cisco IOS XRv

* netlab was tested with IOS XR release 7.4. Earlier releases might use a different management interface name. In that case, you must set **defaults.devices.iosxr.mgmt_if** parameter to the name of the management interface
* Copying Vagrant public insecure SSH key into IOS XR during the box building process is cumbersome. The vagrant configuration file uses a fixed SSH password.
* Maximum interface bandwidth on IOS XRv is 1 Gbps (1000000).
* It seems IOS XR starts an SSH server before it parses the device configuration[^WCPGW], and newer versions of Vagrant don't like that and will ask you for the password for user **vagrant**. Ignore that prompt and the subsequent error messages[^POT]; you might get a running lab in a few minutes[^MAS].

[^WCPGW]: Yeah, what could possibly go wrong?

[^POT]: You'll get plenty of those. Even when the IOS XR device is configured, and you can log into the console, it hates accepting SSH sessions.

[^MAS]: Hint: you have plenty of time to make coffee and a snack.

### Cisco XRd

* The `cisco.iosxr` Ansible Galaxy collection, used to initialize the device and push configurations, is currently incompatible with IOS XRd. A [pull request](https://github.com/ansible-collections/cisco.iosxr/pull/510) exists to fix this, but it has not been merged yet. Until then, you can utilize the following command to install the fork:

```shell
ansible-galaxy collection install git+https://github.com/jmussmann/ansible_collection_cisco.iosxr.git,issue/509
```

* The IOS XRd container seems to be a resource hog. If you experience errors during the initial device configuration, reduce the number of parallel configuration processes -- set the ANSIBLE_FORKS environment variable to one with `export ANSIBLE_FORKS=1`.

(caveats-nxos)=
## Cisco Nexus OS

* Nexus OS release 9.3 requires 6 GB of RAM (*netlab* system default).
* Nexus OS release 10.1 requires 8 GB of RAM and will fail with a cryptic message claiming it's running on unsupported hardware when it doesn't have enough memory.
* Nexus OS release 10.2 requires at least 10 GB of RAM and crashes when you try to run it as an 8 GB VM.
* To change the default amount of memory used by a **nxos** device, set the **defaults.devices.nxos.memory** parameter (in MB)[^DD]

[^DD]: See [](topology/hierarchy.md) for an in-depth explanation of why attributes with hierarchical names work in *netlab*

(caveats-cumulus)=
## Cumulus Linux

* The Cumulus VX 4.4.0 Vagrant box for VirtualBox is broken. *netlab* is using Cumulus VX 4.3.0 with *virtualbox* virtualization provider.

_netlab_ uses the VLAN-aware bridge paradigm to configure VLANs on Cumulus Linux. That decision results in the following restrictions:

* The _netlab_-generated Cumulus Linux VLAN configuration cannot use routed subinterfaces; *ifupdown2* version shipping with Cumulus Linux 4.4.0 refuses to create VLAN subinterfaces in combination with a VLAN-aware bridge.
* The _netlab_-generated Cumulus Linux VLAN configuration cannot use routed native VLAN; *ifupdown2* enslaves physical ports to the bridge and cannot configure IP addresses on physical ports.
* FRRouting version bundled with Cumulus Linux 4.4 cannot run OSPFv3 in VRFs and fails to advertise local IPv6 prefixes in other areas.

See also [other FRRouting caveats](caveats-frr).

(caveats-cumulus-clab)=
### Running Cumulus Linux in Containerlab

* *netlab* uses Cumulus VX 4.4 containers created by Michael Kashin and downloaded from his Docker Hub account. These containers are not tested as they cause occasional system crashes.
* *containerlab* could run Cumulus Linux as a [container or as a micro-VM with *firecracker*](https://containerlab.dev/manual/kinds/cvx/). The default used by *netlab* is to run Cumulus Linux as a container. Add the **clab.runtime** parameter to node data to change that.
* Cumulus Linux running as a container might report errors related to the DHCP client during initial configuration. In this case,  you might have to disable **apparmor** for the DHCP client. The hammer-of-Thor command to fix this problem is `sudo systemctl disable apparmor` followed by a reboot; your sysadmin friends probably have a better suggestion.

(caveats-cumulus-nvue)=
## Cumulus 5.x with NVUE

You could configure Cumulus Linux 5.x with configuration templates developed for Cumulus Linux 4.4 (use device type **cumulus** and specify desired device image), or with NVUE.

NVUE has several shortcomings that prevent *netlab* from configuring basic designs like IBGP on top of IGP. Don't be surprised if the labs that work with **cumulus** device don't work with **cumulus_nvue** device, and please create a GitHub issue whenever you find a glitch. We'd love to know (at least) what doesn't work as expected.

To run Cumulus Linux 5.x with **cumulus** device type, set the following default values in [lab topology](defaults-topology) or one of the [defaults files](defaults-user-file):

```
defaults.devices.cumulus.libvirt.image: CumulusCommunity/cumulus-vx:5.2.0
defaults.devices.cumulus.libvirt.memory: 2048
```

Other caveats:

* The default MTU value is 1500 to match the implementation defaults from other vendors and enable things like seamless OSPF peering.
* *netlab* uses Cumulus VX 5.3 containers created by Michael Kashin and downloaded from his Docker Hub account. These containers are severely out-of-date, are not tested in our integration tests, and might not work as expected.

(caveats-os10)=
## Dell OS10

* Dell OS10 uses a so-called *Virtual Network* interface to try to handle VLANs and VXLANs transparently in the same way. However, it seems that right now, it is **NOT** possible to activate OSPF on a *Virtual Network* (VLAN) SVI interface.
* Sadly, it's also **NOT** possible to use *VRRP* on a *Virtual Network* interface (but *anycast* gateway is supported).
* At the same time, the *anycast* gateway is not supported on plain *ethernet* interfaces, so you need to use *VRRP* there.
* Dell OS10 only allows configuring of the EVPN RD in the form `X.X.X.X:N.` By default, *netlab* uses `N:M` for L3VNI, so on this platform the L3VNI RD is derived from the Router-ID and the VRF ID as `router-id:vrf-id` (and the one generated by *netlab* is not used).

### VRRP Caveats
Netlab enables VRRPv3 by default on Dell OS10, overriding any platform defaults. If you need VRRPv2, check out the [vrrp.version](plugin-vrrp-version) plugin

(caveats-dnsmasq)=
## dnsmasq DHCP server

* You have to build the *dnsmasq* container image with the **netlab clab build dnsmasq** command.

(caveats-fortios)=
## Fortinet FortiOS

We're not testing Fortinet implementation as part of the regular integration tests; the configuration scripts might be outdated and might not work with recent Fortinet software releases. A _netlab_ user reported he got Fortinet devices running with the following software releases:

* Fortios v7.0.15 (Vagrant box built with [this recipe](https://github.com/mweisel/fortigate-vagrant-libvirt))
* Ansible 9.6.1 (Ansible core 2.16.7)
* **fortinet.fortios** Ansible Galaxy collection version 2.3.6

```{tip}
*FortiOS* VM images have a default 15-day evaluation license. The VM has [limited capabilities](https://docs.fortinet.com/document/fortigate-private-cloud/6.0.0/fortigate-vm-on-kvm/504166/fortigate-vm-virtual-appliance-evaluation-license) without a license file. It will work for 15 days from the first boot, at which point you must install a license file or recreate the vagrant box completely from scratch.
```

### OSPF Caveats

* Fortinet implementation of OSPF configuration module does not implement per-interface OSPF areas. All interfaces belong to the OSPF area defined in the node data.
* Fortinet configuration templates set OSPF network type based on number of neighbors, not based on **ospf.network_type** link/interface parameter.

(caveats-frr)=
## FRRouting

* Many FRR configuration templates are not idempotent -- you cannot run **netlab initial** multiple times. Non-idempotent templates include VLAN and VRF configurations.
* VM version of FRR is an Ubuntu VM. The FRR package is downloaded and installed during the initial configuration process.
* You can change the FRR default profile with the **netlab_frr_defaults** node parameter (`traditional` or `datacenter`, default is `datacenter`).
* **netlab collect** downloads FRR configuration but not Linux interface configuration.
* FRR containers need host kernel modules (drivers) to implement the data-plane functionality of *vrf*, *mpls*, and *vxlan* netlab modules. The kernel modules are automatically loaded (when available) during the **netlab up** processing.
* VRF and VXLAN kernel modules are usually bundled with a Linux distribution. If your Ubuntu distribution does not include the MPLS drivers, try installing them with `sudo apt install linux-generic`.
* You cannot load kernel modules in GitHub Codespaces and thus cannot use *vrf*, *mpls*, or *vxlan* modules on FRRouting nodes in that environment.
* FRR containers have a management VRF. Use `ip vrf exec mgmt <command>` to run a CLI command that needs access to the outside world through the management interface. To disable the management VRF, set the **netlab_mgmt_vrf** node parameter to *False*.
* FRR initial container configuration might fail if your Ubuntu distribution does not include the VRF kernel module. Install the VRF kernel module with the `sudo apt install linux-generic` and reboot the server.
* FRR 9.0 and later creates malformed IS-IS LSPs; the bug has been fixed in release 10.0.1 ([details](https://github.com/FRRouting/frr/issues/14514)). You cannot build an IS-IS network using Arista EOS and FRR if you're running an affected version of FRR.
* FRR configures BFD as part of OSPFv2/OSPFv3 configuration.
* STP is *disabled* on Linux bridges used to implement VLANs on this platform, so FRR devices cannot be used in topologies that include L2 loops. Cumulus (with FRR inside) may work better in that case

(caveats-junos)=
## Common Junos caveats

* Junos cannot have more than one loopback interface per routing instance. Using **loopback** links on Junos devices will result in configuration errors.
* Junos configuration template configures BFD timers within routing protocol configuration, not on individual interfaces

(caveats-vptx)=
## Juniper vPTX

* *netlab* release 1.7.0 supports only vJunosEvolved releases that do not require external PFE- and RPIO links. The first vJunosEvolved release implementing internal PFE- and RPIO links is the release 23.2R1-S1.8.

The rest of this section lists information you might find helpful if you're a long-time Junos user:

* vJunos Evolved (vJunos EVO, Juniper vPTX) uses Linux instead of BSD as the underlying OS. There are some basic differences from a "default" JunOS instance, including the management interface name, which is `re0:mgmt-0`.
* After the VM boots up, you need to wait for the *virtual FPC* to become *Online* before being able to forward packets. You can verify this with `show chassis fpc`. **NOTE**: You can see the network interfaces only after the *FPC* is online.
* It seems that the DHCP Client of the management interface does not install a default route, even if received by the DHCP server.
* The VM will complain about missing licenses. You can ignore that.

See also [](caveats-junos).

(caveats-vsrx)=
## Juniper vSRX in Containerlab

You can run Juniper vSRX as a container packaged by *vrnetlab*. See [_containerlab_ documentation](https://containerlab.dev/manual/kinds/vr-vsrx/) for further details.

The Juniper vSRX image in *vrnetlab* uses the network `10.0.0.0/24` for its own internal network, which conflicts with the default network used by **netlab** for the loopback addressing. See [](clab-vrnetlab) for details.

vSRX container built with *vrnetlab* uses **flow based forwarding**. You have two ways to use it:

* Configure security zones, and attach interfaces and rules to them;
* Change the mode to [**packet based forwarding**](https://supportportal.juniper.net/s/article/SRX-How-to-change-forwarding-mode-for-IPv4-from-flow-based-to-packet-based).

See also [](caveats-junos).

(caveats-vjunos-switch)=
## vJunos-Switch in Containerlab

You can run Juniper vJunos-switch as a container packaged by *vrnetlab*. See [_containerlab_ documentation](https://containerlab.dev/manual/kinds/vr-vjunosswitch/) for further details.

The *vrnetlab* containers use the IP subnet `10.0.0.0/24` for the internal network, which conflicts with the default network used by **netlab** for the loopback addressing. See [](clab-vrnetlab) for details.

See also [](caveats-junos).

(caveats-routeros6)=
## Mikrotik RouterOS 6

* Runs with the *CHR* image.
* The CHR free license offers full features with a 1Mbps upload limit per interface, upgradeable to an unrestricted 60-day trial by registering a free MikroTik account and using the ```/system license renew``` command.
* LLDP on Mikrotik CHR RouterOS is enabled on all the interfaces.
* A BGP VRF instance cannot have the same Router ID as the default one. The current configuration template uses the IP address of the last interface in the VRF as the VRF instance Router ID.

(caveats-routeros7)=
## Mikrotik RouterOS 7

* Runs with the *CHR* image.
* LLDP on Mikrotik CHR RouterOS is enabled on all the interfaces.
* The CHR free license offers full features with a 1Mbps upload limit per interface, upgradeable to an unrestricted 60-day trial by registering a free MikroTik account and using the ```/system license renew``` command.
* At the time of the build, testing is being performed with releases **7.14** (claimed as *stable*). With that release:
  * MPLS dataplane seems to have issues when using *virtio* networking, while the LDP and VPNv4 control plane work fine. With *e1000* everything works fine.
  * BGP-to-OSPF route leaking is working on the control plane, but not on the dataplane.
  * There's not an easy way to control the BGP community propagation.
  * Even if you configure the BGP RR cluster-id, this is not announced.
  * Route Reflection of inactive routes does not work.
  * There are still problems with VRFs and IPv6.

(caveats-srlinux)=
## Nokia SR Linux

* Only supported on top of *Containerlab*
* Supports SR Linux release 24.7.1 or later (due to YANG model changes)
* Requires `nokia.srlinux` Ansible Galaxy collection (minimum version 0.5.0). Use **ansible-galaxy collection install nokia.srlinux** command to install it.
* MPLS and LDP are only supported on 7250 IXR (clab.type in ['ixr6','ixr6e','ixr10','ixr10e']). You need a license to run these containers.
* Nokia SR Linux needs an EVPN control plane to enable VXLAN functionality. VXLAN ingress replication lists are built from EVPN Route Type 3 updates.
* Inter-VRF route leaking is supported only in combination with BGP EVPN
* SR Linux does not support configurable propagation of extended BGP communities.
* The SR Linux prefix filters cannot contain the **deny** action.
* The SR Linux configuration templates do not support additional routing policies on routing protocol route imports
* SR Linux needs a static default route (with low route preference) to implement OSPF **default-originate always** functionality.
* SR Linux does not set metrics on routes imported into OSPF. While you can specify the metric and metric type of the OSPF default route, those settings have no impact.

(caveats-sros)=
## Nokia SR OS
* Only supported on top of *Containerlab*, using VRNetlab (VM running inside container)
* Requires the latest `nokia.grpc` Ansible Galaxy collection and its dependencies to be installed from the git repo. You can also use the **netlab install grpc** command to install them

```
ansible-galaxy collection install git+https://github.com/nokia/ansible-networking-collections.git#/grpc/
python3 -m pip install grpcio protobuf==3.20.1
```

* As of May 2024, the `nokia.grpc` collection crashes Ansible versions between 4.10.0 and 9.5.1. We recommend upgrading to Ansible release 9.5.1 (also included as part of **netlab install grpc** script):

```
sudo pip3 install --upgrade 'ansible>=9.5.1'
```

(caveats-sonic)=
## Sonic

* Sonic implementation was tested with Azure sonic-vs VM image (release 2023-11) with FRR running in a container. Other Sonic distributions might use different approaches that would require significant modifications to the configuration deployment process.
* BGP is the only routing protocol running on Azure Sonic. The choice is hardcoded in FRR compilation flags.
* You cannot use IBGP as there's no IGP protocol to resolve IBGP next hops, unless you believe in running IBGP over EBGP.
* The Azure Sonic VM image has to be started with a preconfigured BGP AS number (specified in **config_db.json**); otherwise, it does not start the FRR container. That BGP process is removed during the initial BGP configuration and replaced with the actual BGP AS number specified in the lab topology.
* _netlab_ configures BGP on Sonic through vtysh, not through **config_db**.

(caveats-vyos)=
## VyOS

**netlab ** uses VyOS 1.5, which is currently a rolling release with daily builds. However, all the configurations should also work on the 1.4 LTS release (since it was tested just before it became the new LTS).

The use of a *rolling release* means potentially any build is broken or with regressions, even if the VyOS team is smart enough to perform some [automated smoke tests](https://github.com/vyos/vyos-1x/tree/current/smoketest/scripts/cli) and load [arbitrary configurations](https://github.com/vyos/vyos-1x/tree/current/smoketest/configs) to ensure there are no errors during config migration and system bootup.

Using the latest build published on [Vagrant Hub](https://app.vagrantup.com/vyos/boxes/current) should allow us to easily track and react to any configuration syntax change (which, anyway, is a very rare event). In any case, if you find a misalignment between the VyOS config and the **netlab** templates, feel free to *Open an Issue* or *Submit a PR*.

(vyos-clab)=
It looks like the official VyOS container is not updated as part of the daily builds; *netlab* uses a [third-party container](https://github.com/sysoleg/vyos-container) (`ghcr.io/sysoleg/vyos-container`) to run VyOS with *containerlab*.

Other VyOS caveats:

* VyOS configuration template configures BFD timers only at the global level
* Multi-topology IS-IS (assumed by the [IS-IS configuration module](module-isis)) cannot be configured with VyOS IS-IS CLI ([bug report](https://vyos.dev/T6332)).
* VyOS containers need host kernel modules (drivers) to implement the data-plane functionality of _vrf_, _mpls_, and _vxlan_ netlab modules. The kernel modules are automatically loaded (when available) during the **netlab up** processing.
* VRF and VXLAN kernel modules are usually bundled with a Linux distribution. If your Ubuntu distribution does not include the MPLS drivers, try installing them with `sudo apt install linux-generic`.
* You cannot load kernel modules in GitHub Codespaces and thus cannot use _vrf_, _mpls_, or _vxlan_ modules on VyOS nodes in that environment.
* While VyOS itself supports IPv6 transport for VXLAN, using static flooding with the **vxlan** module, this seems not to work with EVPN, where an IPv4 VTEP is always announced by **frr**.
* VyOS does not have a simple way to handle a management VRF on containerlab, so it will always have a default IPv4 route (`0.0.0.0/0`) on the default routing table. This can cause some problems if you want to originate a default only if received by other routers.
