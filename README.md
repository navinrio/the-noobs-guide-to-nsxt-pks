# The Noob's Guide to a VMware NSX-T/PKS Home Lab

)

and 

![vsphere-clusters-virtual-lab](/content/images/2018/04/vsphere-clusters-virtual-lab.png)

As well as the __Networks__ involved.

![vsphere-networks](/content/images/2018/04/vsphere-networks.png)

> I also created an __NSX Network__ on VLAN 90 and bound it to the remaining nics on the virtual ESXi hosts and edge VM, but it likely isn't required and seems to have no direct mapping.


    #!/bin/bash

    ovftool \
    --name=vesxi-1 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --net:"Network 0=VLAN Trunk"
    --net:"Network 1=VM Network"
    --net:"Network 2=NSX Tunnel"
    --net:"Network 3=NSX Tunnel"
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --overwrite \
    --prop:guestinfo.hostname=vesxi-1 \
    --prop:guestinfo.ipaddress=192.168.1.81 \
    --prop:guestinfo.netmask=255.255.255.0 \
    --prop:guestinfo.gateway=192.168.1.1 \
    --prop:guestinfo.dns=192.168.1.10 \
    --prop:guestinfo.domain=idstudios.local \
    --prop:guestinfo.domain=pool.ntp.org \
    --prop:guestinfo.password=mysecret \
    --prop:guestinfo.ssh=True \
    ../../pks/nsxt/Nested_ESXi6.5u1_Appliance_Template_v1.0.ova \
    vi://administrator@idstudios.local:mysecret@vsphere.idstudios.local/?ip=192.168.1.246

    ovftool \
    --name=vesxi-2 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --net:"Network 0=VLAN Trunk"
    --net:"Network 1=VM Network"
    --net:"Network 2=NSX Tunnel"
    --net:"Network 3=NSX Tunnel"
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --overwrite \
    --prop:guestinfo.hostname=vesxi-2 \
    --prop:guestinfo.ipaddress=192.168.1.82 \
    --prop:guestinfo.netmask=255.255.255.0 \
    --prop:guestinfo.gateway=192.168.1.1 \
    --prop:guestinfo.dns=192.168.1.10 \
    --prop:guestinfo.domain=idstudios.local \
    --prop:guestinfo.domain=pool.ntp.org \
    --prop:guestinfo.password=mysecret \
    --prop:guestinfo.ssh=True \
    ../../pks/nsxt/Nested_ESXi6.5u1_Appliance_Template_v1.0.ova \
    vi://administrator@idstudios.local:mysecret@vsphere.idstudios.local/?ip=192.168.1.246

    ovftool \
    --name=vesxi-3 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --net:"Network 0=VLAN Trunk"
    --net:"Network 1=VM Network"
    --net:"Network 2=NSX Tunnel"
    --net:"Network 3=NSX Tunnel"
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --overwrite \
    --prop:guestinfo.hostname=vesxi-3 \
    --prop:guestinfo.ipaddress=192.168.1.83 \
    --prop:guestinfo.netmask=255.255.255.0 \
    --prop:guestinfo.gateway=192.168.1.1 \
    --prop:guestinfo.dns=192.168.1.10 \
    --prop:guestinfo.domain=idstudios.local \
    --prop:guestinfo.domain=pool.ntp.org \
    --prop:guestinfo.password=mysecret \
    --prop:guestinfo.ssh=True \
    ../../pks/nsxt/Nested_ESXi6.5u1_Appliance_Template_v1.0.ova \
    vi://administrator@idstudios.local:mysecret@vsphere.idstudios.local/?ip=192.168.1.246

## NSX-T 2.1 Install for Dummies

I followed the [NSX-T install document](https://docs.vmware.com/en/VMware-NSX-T/2.1/com.vmware.nsxt.install.doc/GUID-3E0C4CEC-D593-4395-84C4-150CD6285963.html) and it was fairly straightforward up to the point of the Edge and transport nodes.

> You should read this guide and be prepared to refer to it as I won't be repeating all of the information involved in the NSX-T setup, just the bits that aren't entirely clear from the docs.

No offense to VMware but the GUI web form installation guides are sort of ridiculous... type the info in twice and you are on the road to insanity.  __ovftool__ is still the best swiss army knife tool there is!

Here are the __ovftool__ scripts I used (several times over)...

### Step 1 - NSX Manager

    #!/bin/bash

    ovftool \
    --name=nsx-manager \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-m2 \
    --network="VM Network" \
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --prop:nsx_role=nsx-manager \
    --prop:nsx_ip_0=192.168.1.60 \
    --prop:nsx_netmask_0=255.255.255.0 \
    --prop:nsx_gateway_0=192.168.1.1 \
    --prop:nsx_dns1_0=192.168.1.10 \
    --prop:nsx_domain_0=idstudios.local \
    --prop:nsx_ntp_0=pool.ntp.org \
    --prop:nsx_isSSHEnabled=True \
    --prop:nsx_allowSSHRootLogin=True \
    --prop:nsx_passwd_0=SUP3rD^B3r_2!07 \
    --prop:nsx_cli_passwd_0=SUP3rD^B3r_2!07 \
    --prop:nsx_hostname=nsx-manager \
    ../../pks/nsxt/nsx-unified-appliance-2.1.0.0.0.7395503.ova \
    vi://administrator@idstudios.local:mysecret@vsphere.idstudios.local/?ip=192.168.1.245

Nothing complicated about this one.

Log into NSX Manager and poke around.

### Step 2 - NSX Controller(s)

Next up the NSX controllers. I deployed 2 controllers by mistake and really only needed one.  Three are recommended, but likely not needed in the lab.

    #!/bin/bash

    ovftool \
    --overwrite \
    --name=nsx-controller-1 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --network="VM Network" \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --prop:nsx_ip_0=192.168.1.61 \
    --prop:nsx_netmask_0=255.255.255.0 \
    --prop:nsx_gateway_0=192.168.1.1 \
    --prop:nsx_dns1_0=192.168.1.10,192.168.1.2,8.8.8.8 \
    --prop:nsx_domain_0=idstudios.local \
    --prop:nsx_ntp_0=pool.ntp.org \
    --prop:nsx_isSSHEnabled=True \
    --prop:nsx_allowSSHRootLogin=False \
    --prop:nsx_passwd_0=SUP3rD^B3r_2!07 \
    --prop:nsx_cli_passwd_0=SUP3rD^B3r_2!07 \
    --prop:nsx_cli_audit_passwd_0=SUP3rD^B3r_2!07 \
    --prop:nsx_hostname=nsx-controller-1 \
    ../../pks/nsxt/nsx-controller-2.1.0.0.0.7395493.ova \
    vi://administrator@idstudios.local:mysecret@vsphere.idstudios.local/?ip=192.168.1.246

At this point it is best to refer to the NSX-T installation guide instructions on creating management clusters and controller clusters and the like.  See [Join NSX Clusters with the NSX Manager](https://docs.vmware.com/en/VMware-NSX-T/2.1/com.vmware.nsxt.install.doc/GUID-05434745-7D74-4DA8-A68E-9FE17093DA7B.html) for details and follow the guide up to the NSX Edge installation.  It involves `ssh`ing into the manager and controller and executing a few arcane GUID based join commands that are common among clustering technology.

I tried to follow the documentation right to the end but was unable to sort out the transport settings.  It was only with the kind help of the VMware SE who guided me through the process, described below...

### Step 3 - NSX Edge

So you don't actually need to have the edge OVA file.  This is the manual way to install the Edge:

_(logged into NSX Manager)_

* Add your vCenter as a compute manager under __Fabric__>__Compute Managers__.
* Now you can deploy an Edge VM directly from within the UI via __Fabric__>__Nodes__.

You'll want to put your Edge VM in the management cluster (in my example __labs__).  See the example screenshots for configuration:

![deploy-edge-1](/content/images/2018/04/deploy-edge-1.png)

![deploy-edge-2](/content/images/2018/04/deploy-edge-2.png)

The networking is of particular importance.  It is important that the resulting network adapters and associated port groups match up with those shown in my diagram above.

### Step 4 - Transport Zones and Transport Nodes

This is complicated in the documentation, and out of sync with the logical flow.  It also isn't entirely clear how to configure your transport zones. Lam's __Part 3__ assumes you already have your __Host__ and __Edge__ transport nodes configured, along with your transport zones.

At this stage we basically need to:

1. Create two transport zones
2. Create two uplink profiles
2. Create a VTEP IP Pool

#### Transport Zones

My setup has two transport zones defined:

#### nsxlab-overlay-tz

![nxslab-overlay-tz](/content/images/2018/04/nxslab-overlay-tz.png)

and 

#### nsxlab-vlan-tz

![nsxlab-vlan-tz](/content/images/2018/04/nsxlab-vlan-tz.png)

> Note the __N-DVS__ logical switch names we assign (and create) here as part of our transport zones.  These will be references again when we configure our __Edge Transport Node__ a bit further on.


#### Uplink Profiles

Here is another area where the stock documentation will cause confusion.  In our configuration the settings for the uplink profiles are very specific with respect to the network adapter settings.

Create two uplink profiles under __Fabric__>__Profiles__:

1. host-uplink
2. edge-uplink

(You can guess how they will be respectively used)

Ensure the __host-uplink__ settings are as follows:

![profiles-host-uplink](/content/images/2018/04/profiles-host-uplink.png)

> Pay particular attention to the __Active Uplinks__ field as that must be set to __vmnic2__ as this is the __NSX Tunnel__ portgroup on our Virtual ESXi hosts.  It would be our dedicated phyiscal nic required by NSX if we were deploying to physical ESXi hosts.  Later on we will reference this when we setup our N-DVS in our host transport node.

Ensure the __edge-uplink__ settings are as follows:

![profiles-edge-uplink](/content/images/2018/04/profiles-edge-uplink.png)

> Pay particular attention to the __Active Uplinks__ field as that must be set to __fp-eth1__ as this is the internal name of the network interface within the Edge VM (??).  Later on we will reference this when we setup our N-DVS in our Edge transport node.

#### VTEP IP Pool

And a __VTEP_POOL__ IP Pool of __10.20.30.30__ to __10.20.30.60__.

![vtep-pool](/content/images/2018/04/vtep-pool.png)

> This IP Pool will be used to create virtual tunnel end points on each of the ESXi hosts transport nodes, as well as on the Edge transport node (on the Edge VM), as per the diagram.  

### Step 5 - Host Transport Nodes

In __NSX Manager__ under __Fabric__>__Nodes__>__Hosts__, select your compute manager from the __Managed By:__ drop down, and then expand your target cluster (in my example: __virtual-lab__).  If you select the checkbox at the cluster level, the option to __Configure Cluster__ will enable.

![fabric-node-create-cluster](/content/images/2018/04/fabric-node-create-cluster.png)

Note that we allocated the __VTEP_POOL__ IP Pool for this, and we will do it again when we configure the Edge transport node.

By setting up  __Configure Cluster__ it will now automatically create and configure any new ESXi hosts added to the __virtual-lab__ cluster as transport nodes.  And our __host__ transport nodes are now created.

### Step 6 - The Edge Transport Node

You'll notice in the diagram that in addition to the __VTEP__ addresses allocated to each of the __vesxi__ hosts, one is also allocated to the Edge VM transport node.

So now we must create the __Edge Transport Node__:

Under __Fabric__>__Nodes__>__Transport Nodes__ click __Add__.  Enter __edge-transport-node__ as the name, and for the __Node__ chose your __Edge VM__.

Select both of your __transport zones__.  Your _General__ tab should appear as shown below:

![edge-transport-general-1](/content/images/2018/04/edge-transport-general-1.png)

There will be two __N-DVS__ entries, one for each of the transport zones we created, and referencing the associated logical switches.

__nsxlab-overlay-switch__

![edge-transport-ndvs-overlay](/content/images/2018/04/edge-transport-ndvs-overlay.png)

> Pay careful attention to the __Virtual NICs__ mapping.  For the Overlay switch it should map from __fp-eth0__ to __fp-eth1__.

__nsxlab-vlan-switch__

![edge-transport-ndvs-vlan](/content/images/2018/04/edge-transport-ndvs-vlan.png)

> Pay careful attention to the __Virtual NICs__ mapping.  For the VLAN switch it should map from __fp-eth1__ to __fp-eth1__.

At this point the __Edge Transport Node__ should be configured and we can rejoin with Lam's guide on configuring NSX-T:

[Getting started with VMware Pivotal Container Service (PKS) Part 3: NSX-T](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-3-nsx-t.html)

Everything should be in place to proceed with his direction, with a few adjustments:

* Remember that his 3250 VLAN is really our primary 192.168.1.0 home network, and so we don't really need it.
* His pfSense router is really just our primary home gateway to the internet and the __VLAN 3251__ isn't actually needed in our configuration, we use __192.168.1.8__ on the main network as our __uplink-1__ uplink port associated with our __T0__ router:

![T0-uplink-1](/content/images/2018/04/T0-uplink-1.png)

> And this eliminates the need for the __VLAN 3251__.  However we do need to setup the static routes on our router to enable access to two k8s networks he uses - __10.10.0.0/24__ (the T1 k8s management router) and __10.20.0.0/24__ (the IP Pool assigned for pks loadbalancers).

At the end of the article there are some useful tips about validating the NSX-T setup.  

SSH into the __vesxi__ hosts one by one and ensure the following:

    esxcli network ip interface ipv4 get

You should see __vmk10__ and __vmk50__ appear similar to this:

    Name   IPv4 Address  IPv4 Netmask   IPv4 Broadcast   Address Type  Gateway  DHCP DNS
    -----  ------------  -------------  ---------------  ------------  -------  --------
    vmk0   192.168.1.83  255.255.255.0  192.168.1.255    STATIC        0.0.0.0     false
    vmk10  10.20.30.30   255.255.255.0  10.20.30.255     STATIC        0.0.0.0     false
    vmk50  169.254.1.1   255.255.0.0    169.254.255.255  STATIC        0.0.0.0     false

> Remember that those __10.20.30.x__ addresses came from our VTEP pool.

Verify that the logical switches are shown on our ESXi hosts:

    esxcli network ip interface list

    vmk10
      Name: vmk10
      MAC Address: 00:50:56:61:f0:f3
      Enabled: true
      Portset: DvsPortset-1
      Portgroup: N/A
      Netstack Instance: vxlan
      VDS Name: nsxlab-overlay-switch
      VDS UUID: 59 e6 25 ac 27 00 45 6f-a5 92 77 7d d4 ce 87 ae
      VDS Port: 10
      VDS Connection: 1523044214
      Opaque Network ID: N/A
      Opaque Network Type: N/A
      External ID: N/A
      MTU: 1600
      TSO MSS: 65535
      Port ID: 67108868

    vmk50
      Name: vmk50
      MAC Address: 00:50:56:68:e4:33
      Enabled: true
      Portset: DvsPortset-1
      Portgroup: N/A
      Netstack Instance: hyperbus
      VDS Name: nsxlab-overlay-switch
      VDS UUID: 59 e6 25 ac 27 00 45 6f-a5 92 77 7d d4 ce 87 ae
      VDS Port: c6029e74-9952-4960-8d3a-87caafaf4fa5
      VDS Connection: 1523044215
      Opaque Network ID: N/A
      Opaque Network Type: N/A
      External ID: N/A
      MTU: 1500
      TSO MSS: 65535
      Port ID: 67108869

> You can see the __nsxlab-overlay-switch__ we defined in our uplink profile and associated with our __edge transport node__ and __host transport nodes__.

And from any of the Virtual ESXi hosts you should be able to ping all the other VTEP addresses:

    vmkping ++netstack=vxlan 10.20.30.30 # vesxi-1
    vmkping ++netstack=vxlan 10.20.30.31 # vesxi-2
    vmkping ++netstack=vxlan 10.20.30.32 # vesxi-3
    vmkping ++netstack=vxlan 10.20.30.33 # edge vm

With NSX-T configured we can now proceed with the remainder of the series.

_Part 4_ involves setting up the Ops Manager and BOSH...

> __VERY IMPORTANT__ to make sure that the version of the Pivotal Ops Manager you install is version __Build 249__ which maps to version __2.0.5__.  The name of the OVA you want is __pcf-vsphere-2.0-build.249.ova__.  Anything newer won't work and you'll be left pulling out hair and `bosh ssh`ing around sifting through failure logs without the help of __journald__ because, well, the stemcells are __trusty__ and it's old stuff in there, and trust me... just get __Build 249__.

[Getting started with VMware Pivotal Container Service (PKS) Part 4: Ops Manager & BOSH](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-4-ops-manager-bosh.html)

_Part 5_ and _Part 6_ go fairly smoothly from here, as long as you made sure to install __Build 249__ of the Ops Manager.

If all goes well you'll be `kubectl`ing away with __PKS__.

But if you are like me you'll repeat this entire thing a few dozen times first :)