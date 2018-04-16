# The Noob's Guide to a VMware NSX-T/PKS Home Lab

William Lam recently posted a tutorial walkthrough series for __NSX-T and PKS__ on [virtuallyghetto](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-1-overview.html).  I used this as the foundation for my home lab setup, but it was a struggle to get things working, and without the help of a very clever VMware SE, I probably wouldn't have been able to figure it out.  .

> __Warning:__ NSX-T is complicated.

This post is an attempt to fill in some of the gaps in the series with information that wasn't initially obvious to me - and may not be obvious to you.

> __Full disclosure:__ I am not a VMware certified anything.

_Part 1_ of the PKS series is relatively straightforward:

[Getting started with VMware Pivotal Container Service (PKS) Part 1: Overview](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-1-overview.html)

_Part 2_ involves prepping your PKS workstation VM.  The folks at PKS are fond of Ubuntu, and so the tutorial uses an Ubuntu VM, but any machine that can run the required tools will do.  I prepped both my Macbook Pro and my CentOS7 [cluster-builder](https://github.com/ids/cluster-builder) control station with the necessary tools, as per the article:

[Getting started with VMware Pivotal Container Service (PKS) Part 2: PKS Client](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-2-pks-client.html)

_Part 3_ is where things start to get deep into the weeds, and fast.  The article on __NSX-T__ makes reference to having a working _NSX-T 2.x_ lab, and references the [automated powershell script](https://www.virtuallyghetto.com/2017/10/vghetto-automated-nsx-t-2-0-lab-deployment.html) for setting one up.  It may work for you, but I wanted to understand how all the parts fit together and for me doing it by hand is the ~best~ only way to learn.

[Getting started with VMware Pivotal Container Service (PKS) Part 3: NSX-T](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-3-nsx-t.html)

> I was stuck on Step #3... for awhile.  A kindly VMware SE walked me through it, and with his help, it started to make sense.

So here I will augment Mr. Lam's fine work with my own noobs guide to installing an NSX-T/PKS home lab...

## Prep

* I am assuming you have downloaded all the depencencies from Lam's article.
* It would be wise to review the articles first to get a sense of all the steps.  This is only a few extra tips in addition to the "Lam Series" - I relied heavily on his step-by-steps at all stages.

## ** Warning **

These topics probably won't mean anything yet, but take note anyway!  It could save you hours:

* __Beware of the "Down"__.  I mistook the status of Down in the transport nodes as failed installation or configuration.  After wasting countless hours experimenting with variations on what appeared to be correctly configured, I was informed, this is __by design__.  The status won't go green until a cluster has been deployed and their are k8s nodes using the VTEP.

![Beware the Down](/content/images/beware-the-down.png)

* __>= 1600 MTU is required__.  Make sure to set all of the switches (VSS and VDS) involved to 1600 MTU or greater as jumbo frames is required for correct NSX-T operation.

![Switch Settings](/content/images/virtual-switch-mtu-security.png)

![MTU](/content/images/switch0-mtu.png)

* __The Edge needs to be LARGE__... one of the few gotchas I didn't hit, but not clearly called out in the article: __PKS requires the Edge to be 8 CPU and 16GB Ram__, it is a hard limit and when you get to the very end and try to install PKS, the NSX-T errand will stop you if your edge VM is too small.

Ok... keep those things in mind...

Now let's start with a diagram:

## Context: The Home Lab

![the-condo-datacenter](/content/images/condo-lab-diagram.png)

In this post I take the example scenario described in Lam's series and implement it specifically in my home lab setup.  When I first reviewed the [diagram](https://i0.wp.com/www.virtuallyghetto.com/wp-content/uploads/2018/03/1.-Getting-Started-With-PKS-Overview-0.png?ssl=1) from the article, it wasn't clear to me how the two VLANs (3250 and 3251) might relate to the physical network, so I am presenting the entire picture here in an attempt to clear that up for anyone else who finds it fuzzy or confusing.

> My friendly neighbourhood VMware SE simplified the configuration to better suit my lab setup ;)

What Mr. Lam references as his management network of __3251__ (I'm sure he has many), is actually just my primary home network of __192.168.1.0/24__.

His __3250__ network isn't actually needed in my setup.

Put simply (very simply):

* The overlay transport zone and dedicated port group support the overlay networks and traffic in the k8s environment.  This VTEP network allows all of the k8s VMs to communicate in their own network spaces on top of the ESXi hosts that are the "transport nodes" for this virtual network (that sits on top of vSpheres already virtual network - head hurt yet?)
* The VLAN transport zone and dedicated port group and uplink represent the client traffic entry point into both the NSX-T k8s dedicated load balancers and the k8s managmenet nodes network.

The __labs__ cluster is used for management and contains two physical ESXi hosts: __esxi-5__ and __esxi-6__.  These are the compute resources dedicated to the NSX-T/ESXi virtual lab.  It is referred to as the __management cluster__.

The __pks-lab__ cluster is made up of three nested ESXi hosts: __vesxi-1__, __vesxi-2__ and __vesxi-3__.  It is referred to as the __compute cluster__, and it is where PKS will place all the k8s nodes and related artifacts.

> Specs for each physical ESXi host: i7 - 7700, 64GB Ram, 512GB m2 Nvme SSD local datastores.  I run all three of my Virtual ESXi hosts on __esxi-6__ along with my __nsx-edge__.  My __vesxi__ virtual ESXi hosts all share the same single iSCSI based datastore, which is actually just a CentOS7 iSCSI target hosting up a VMDK from an SSD running on a Mac Mini, over my primary Gigabit ethernet (not even dedicated).  And still I average nearly 100 MB/s write speed.  Remarkable.

Before we dive into NSX-T we should tackle the matter of setting up some virtual ESXi hosts and a dedicated vSphere cluster:

## Nested ESXi for Dummies

> __Note__ that on _Day 2_ I came back to my Virtual ESXi cluster to find that my NSX-T/PKS environment had stopped functioning.  Immediately after creation it was working fine, and I was able to deploy and test a full application stack on the clusters.  However, when I came back the next day I found NSX-T unresponsive and was unable to create clusters.  Due to time contraints I decided to rebuild NSX-T/PKS in non-virtual ESXi mode and dedicated my __esxi-5__ host as my management cluster and my __esxi-6__ host as my dedicated compute cluster.  I will post more details on the variations in the future, but the fundamental configuration remains the same regardless of the number of ESXi hosts, and whether they are virtual or physical.  Since switching to a dedicated physical ESXi host I have not seen the same issues.  Updates to follow.

Deploying the virtual ESXi hypervisors as VMs is actually fairly straightforward until you get to the part about networking with VLANs.  If you use VLANs, and wish to make them available to your virtual ESXi hosts, make sure to follow this [tip](https://blog.idstudios.io/nested-esxi-and-vlans/).

The physical ESXi hosts need to have access to a dedicated __NSX Tunnel__ portgroup (It is an NSX requirement to have a dedicated vnic or pnic for this).  In my home lab I use standard __VSS__ switches on my physical ESXi boxes so I am not dependent on vSphere - but I used a vSphere __VDS__ for my __pks-lab__ cluster of VESXi hosts I am dedicating to NSX... so I had to create two __NSX Tunnel__ portgroups (one on each physical ESXi hosts in the local VSS __Switch0__, and one on my _VDS_ that I will use in my virtual ESXi hosts - VLAN 0.  The __NSX Tunnel__ basically just provides a level 2 dedicated portgroup to NSX upon which NSX can work it's layer 7 magic.

As per the diagram above setup your ESXi VMs so that their network adapters are as follows:

* Network Adapter 1: __VLAN Trunk__
* Network Adapter 2: __VM Network__
* Network Adapter 3: __NSX Tunnel__

These will then map to __vmnic0__, __vmnic1__ and __vmnic2__ respectively.

__This is important later on when mapping our uplink ports.__

> I like to setup my virtual ESXi hosts with __/etc/ssh/keys-root/authorized_keys__ for passwordless ssh, but this isn't required if you enjoy entering passwords.

Once the ESXi hosts have been uploaded and installed they should be added to a dedicated cluster, such as the __pks-lab__ in this example.

> Now would be a good time to enable __DRS__ on your newly created __pks-lab__ cluster.  If you forget to do this, as I did, you will regret it when you deploy, and then have to redeploy, your first k8s cluster with PKS, which leverages and depends on __DRS__ for balancing of the k8s node VMs over ESXi hosts.

The following is a screenshot of my vSphere setup for the __labs__ and __pks-lab__ cluster environments, as well as the __Networks__ involved.

![vsphere-layout](/content/images/clusters-and-networks-before.png)


    #!/bin/bash

    ovftool \
    --name=vesxi-1 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --network="VLAN Trunk" \
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --overwrite \
    --prop:guestinfo.hostname=vesxi-1 \
    --prop:guestinfo.ipaddress=192.168.1.81 \
    --prop:guestinfo.netmask=255.255.255.0 \
    --prop:guestinfo.gateway=192.168.1.1 \
    --prop:guestinfo.dns=8.8.8.8 \
    --prop:guestinfo.domain=onprem.idstudios.io \
    --prop:guestinfo.domain=pool.ntp.org \
    --prop:guestinfo.password=SuperDuckPassword \
    --prop:guestinfo.ssh=True \
    ../../../pks/nsxt/Nested_ESXi6.5u1_Appliance_Template_v1.0.ova \
    vi://administrator@vsphere.onprem.idstudios.io:SuperDuckPassword@vsphere.onprem.idstudios.io/?ip=192.168.1.246

    ovftool \
    --name=vesxi-2 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --network="VLAN Trunk" \
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --overwrite \
    --prop:guestinfo.hostname=vesxi-2 \
    --prop:guestinfo.ipaddress=192.168.1.82 \
    --prop:guestinfo.netmask=255.255.255.0 \
    --prop:guestinfo.gateway=192.168.1.1 \
    --prop:guestinfo.dns=8.8.8.8 \
    --prop:guestinfo.domain=onprem.idstudios.io \
    --prop:guestinfo.domain=pool.ntp.org \
    --prop:guestinfo.password=SuperDuckPassword \
    --prop:guestinfo.ssh=True \
    ../../../pks/nsxt/Nested_ESXi6.5u1_Appliance_Template_v1.0.ova \
    vi://administrator@vsphere.onprem.idstudios.io:SuperDuckPassword@vsphere.onprem.idstudios.io/?ip=192.168.1.246

    ovftool \
    --name=vesxi-3 \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --network="VLAN Trunk" \
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --overwrite \
    --prop:guestinfo.hostname=vesxi-3 \
    --prop:guestinfo.ipaddress=192.168.1.83 \
    --prop:guestinfo.netmask=255.255.255.0 \
    --prop:guestinfo.gateway=192.168.1.1 \
    --prop:guestinfo.dns=8.8.8.8 \
    --prop:guestinfo.domain=onprem.idstudios.io \
    --prop:guestinfo.domain=pool.ntp.org \
    --prop:guestinfo.password=SuperDuckPassword \
    --prop:guestinfo.ssh=True \
    ../../../pks/nsxt/Nested_ESXi6.5u1_Appliance_Template_v1.0.ova \
    vi://administrator@vsphere.onprem.idstudios.io:SuperDuckPassword@vsphere.onprem.idstudios.io/?ip=192.168.1.246

Remember to add two additional network adapters so you have the following configuration for each of the nested ESXi hosts:

* Network Adapter 1: __VLAN Trunk__
* Network Adapter 2: __VM Network__
* Network Adapter 3: __NSX Tunnel__

The Network Adapters on your Virtual ESXi hosts should look as follows:

![vesxi-networks](/content/images/vesxi-networks.png)

### Virtual SAN

You'll also need to setup some sort of shared datastore solution for the Virtual ESXi hosts. I stopped short of setting up __vSAN__ and opted for a simpler approach.  If you don't have access to a SAN in your home lab, you can create one fairly easily with a simple VM on a hypervisor.

Make sure it is on a seperate ESXi host as you can't host an iSCSI datastore from a VM running on the same hypervisor that intends to mount the target.

I used [this guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/storage_administration_guide/ch-iscsi#iscsi-target-setup) from RedHat to setup a CentOS7 iSCSI target in a VM.

You can then configure your iSCSI adapter on your ESXi host to see the LUNs made available from your iSCSI VM, format them with vmfs, and just like that you have a SAN.

Or you could use vSAN...

## NSX-T 2.1 Install for Dummies

I followed the [NSX-T install document](https://docs.vmware.com/en/VMware-NSX-T/2.1/com.vmware.nsxt.install.doc/GUID-3E0C4CEC-D593-4395-84C4-150CD6285963.html) and it was fairly straightforward up to the point of the Edge and transport nodes.

> You should read this guide as I won't be repeating all of the information involved in the NSX-T setup, just the bits that aren't entirely clear from the docs and are only covered off by the automated script referenced in Mr. Lam's article.

Here is the __ovftool__ script I used (several times over)... mostly taken right from the docs.

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

Next up the NSX controllers. You really only need one in the lab.

    #!/bin/bash

    ovftool \
    --overwrite \
    --name=nsx-controller \
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
    --prop:nsx_hostname=nsx-controller \
    ../../pks/nsxt/nsx-controller-2.1.0.0.0.7395493.ova \
    vi://administrator@idstudios.local:mysecret@vsphere.idstudios.local/?ip=192.168.1.246

At this point it is best to refer to the NSX-T installation guide instructions on creating management clusters and controller clusters and the like.  See [Join NSX Clusters with the NSX Manager](https://docs.vmware.com/en/VMware-NSX-T/2.1/com.vmware.nsxt.install.doc/GUID-05434745-7D74-4DA8-A68E-9FE17093DA7B.html) for details and follow the guide up to the NSX Edge installation.  It involves `ssh`ing into the manager and controller and executing a few arcane token based join commands that are common among clustering technology.

I tried to follow the documentation right to the end but was unable to sort out the transport settings.  It was only with the kind help of the VMware SE who guided me through the process, described below...

### Step 3 - NSX Edge

So you don't actually need to have the edge OVA file.  This is the manual way to install the Edge:

_(logged into NSX Manager)_

* Add your vCenter as a compute manager under __Fabric__>__Compute Managers__.
* Now you can deploy an Edge VM directly from within the UI via __Fabric__>__Nodes__.

However I prefer to install it with __ovftool__ because I can enable SSH right out of the box.  And doing anything more then once gets tiresome in web forms.

    #!/bin/bash

    ovftool \
    --name=nsx-edge \
    --deploymentOption=large \
    --X:injectOvfEnv \
    --X:logFile=ovftool.log \
    --allowExtraConfig \
    --datastore=datastore-ssd \
    --net:"Network 0=VM Network" \
    --net:"Network 1=NSX Tunnel" \
    --net:"Network 2=NSX Tunnel" \
    --net:"Network 3=NSX Tunnel" \
    --acceptAllEulas \
    --noSSLVerify \
    --diskMode=thin \
    --powerOn \
    --prop:nsx_ip_0=192.168.1.65 \
    --prop:nsx_netmask_0=255.255.255.0 \
    --prop:nsx_gateway_0=192.168.1.1 \
    --prop:nsx_dns1_0=8.8.8.8 \
    --prop:nsx_domain_0=onprem.idstudios.io \
    --prop:nsx_ntp_0=pool.ntp.org \
    --prop:nsx_isSSHEnabled=True \
    --prop:nsx_allowSSHRootLogin=True \
    --prop:nsx_passwd_0=SuperDuckPassword \
    --prop:nsx_cli_passwd_0=SuperDuckPassword \
    --prop:nsx_hostname=nsx-edge \
    ../../../pks/nsxt/nsx-edge-2.1.0.0.0.7395502.ova \
    vi://administrator@vsphere.onprem.idstudios.io:SuperDuckPassword@vsphere.onprem.idstudios.io/?ip=192.168.1.246

You'll want to put your Edge VM in the management cluster (in my example __labs__).

The networking is of particular importance.  It is important that the resulting network adapters and associated port groups match up with those shown:

![NSX Edge Nic Layout](/content/images/nsx-edge-nic-layout.png)

### Step 4 - Transport Zones and Transport Nodes

It was never clear to me intially from the documentation how you layout your transport zones. Lam's __Part 3__ assumes you already have your __Host__ and __Edge__ transport nodes configured, along with your transport zones, so it doesn't provide much guidance.  It is actually fairly simple.

At this stage we basically need to:

1. Create two transport zones
2. Create two uplink profiles
2. Create a VTEP IP Pool

#### Transport Zones

My setup has two transport zones defined:

#### pks-lab-overlay-tz

![pks-lab-vlan-tz](/content/images/pks-lab-vlan-tz.png)

and 

#### pks-lab-vlan-tz

![pks-lab-overlay-tz](/content/images/pks-lab-overlay-tz.png)

> Note the __N-DVS__ logical switch names we assign (and create) here as part of our transport zones.  These will be referenced again when we configure our __Edge Transport Node__ a bit further on.

#### Uplink Profiles

Here is another area where the stock documentation caused me confusion.  In this example configuration the settings for the uplink profiles are very specific with respect to the network adapter settings.

Create two uplink profiles under __Fabric__>__Profiles__:

1. host-uplink
2. edge-uplink

(You can guess how they will be respectively used)

Ensure the __host-uplink__ settings are as follows:

![Host Uplink Profile](/content/images/host-uplink.png)

> Pay particular attention to the __Active Uplinks__ field as that must be set to __vmnic2__ as this is the __NSX Tunnel__ portgroup on our Virtual ESXi hosts.  It would be our dedicated phyiscal nic required by NSX if we were deploying to physical ESXi hosts.  Later on we will reference this when we setup our N-DVS in our host transport node.

Ensure the __edge-uplink__ settings are as follows:

![Edge Uplink Profile](/content/images/edge-uplink.png)

> Pay particular attention to the __Active Uplinks__ field as that must be set to __fp-eth1__ as this is the internal name of the network interface within the Edge VM.  Later on we will reference this when we setup our N-DVS in our Edge transport node.

#### VTEP IP Pool

And a __VTEP_IP_POOL__ IP Pool of __10.20.30.30__ to __10.20.30.60__.

![vtep-ip-pool](/content/images/vtep-ip-pool.png)

> This IP Pool will be used to create virtual tunnel end points on each of the ESXi hosts transport nodes, as well as on the Edge transport node (on the Edge VM), as per the diagram.  Remember it is used as network transport for the overlay networks used by the k8s nodes that make up the kubernetes clusters.

### Step 5 - Host Transport Nodes

In __NSX Manager__ under __Fabric__>__Nodes__>__Hosts__, select your compute manager from the __Managed By:__ drop down, and then expand your target cluster (in my example: __pks-lab__).  If you select the checkbox at the cluster level, the option to __Configure Cluster__ will enable.

![fabric-node-create-cluster](/content/images/cluster-config.png)

Note that we allocated the __VTEP_IP_POOL__ for this, and we will do it again when we configure the Edge transport node.

By setting up  __Configure Cluster__ it will now automatically create and configure any new ESXi hosts added to the __pks-lab__ cluster as transport nodes.  And our __host__ transport nodes are now created.

### Step 6 - The Edge Transport Node

You'll notice in the diagram that in addition to the __VTEP__ addresses allocated to each of the __vesxi__ hosts, one is also allocated to the Edge VM transport node.

So now we must create the __Edge Transport Node__:

Under __Fabric__>__Nodes__>__Transport Nodes__ click __Add__.  Enter __edge-transport-node__ as the name, and for the __Node__ chose your __Edge VM__.

Select both of your __transport zones__.  Your _General_ tab should appear as shown below:

![edge-transport-general](/content/images/edge-transport-general.png)

There will be two __N-DVS__ entries, one for each of the transport zones we created, and referencing the associated logical switches.

__pks-lab-overlay-switch__

![edge-transport-ndvs-overlay](/content/images/edge-transport-overlay-ndvs.png)

> Pay careful attention to the __Virtual NICs__ mapping.  For the Overlay switch it should map from __fp-eth0__ to __fp-eth1__.

__nsxlab-vlan-switch__

![edge-transport-ndvs-vlan](/content/images/edge-transport-vlan-ndvs.png)

> Pay careful attention to the __Virtual NICs__ mapping.  For the VLAN switch it should map from __fp-eth1__ to __fp-eth1__.

At this point the __Edge Transport Node__ should be configured and we can rejoin with Lam's guide on configuring NSX-T:

[Getting started with VMware Pivotal Container Service (PKS) Part 3: NSX-T](https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-3-nsx-t.html)

Everything should be in place to proceed with his direction, with a few adjustments:

* Remember that his 3251 VLAN is really our primary 192.168.1.0 home network, and we don't really need both 3250 and 3251 (though it is likely a good practice).
* Remember to map all of his __172.30.51.0/24__ addresses to what we implement as __192.168.1.0/24__.  And that we __don't use 172.30.50.0/24__ at all.
* His pfSense router is really just our primary home gateway to the internet and the both VLANs aren't actually needed in our configuration, we use __192.168.1.8__ on the main network as our __uplink-1__ uplink port address associated with our __T0__ router:

![T0-uplink-1](/content/images/uplink-1.png)

> Although this eliminates the need for the __VLAN 3250__ we do need to setup the static routes on our router to enable access to two k8s networks he uses - __10.10.0.0/24__ (the T1 k8s management router) and __10.20.0.0/24__ (the IP Pool assigned for pks loadbalancers).

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

## Troubleshooting

__1 of 3 post-start scripts failed. Failed Jobs: ncp. Successful Jobs: bosh-dns, kubelet.__

If you get all the way to the end, and go to deploy a k8s cluster only to have this:

    Using environment '192.168.1.201' as client 'ops_manager'

    Task 22

    Task 22 | 22:36:11 | Preparing deployment: Preparing deployment (00:00:03)
    Task 22 | 22:36:17 | Preparing package compilation: Finding packages to compile (00:00:00)
    Task 22 | 22:36:17 | Creating missing vms: master/96cb1deb-bc4e-44b3-ac69-220cb0935bf8 (0)
    Task 22 | 22:36:17 | Creating missing vms: worker/033088fc-5f91-4b57-b9e3-3cc718031e3b (0)
    Task 22 | 22:36:17 | Creating missing vms: worker/5b8d175b-927e-48c1-a97c-f8b7ef099be5 (2)
    Task 22 | 22:36:17 | Creating missing vms: worker/d806a4f4-6a1e-4f9c-a2ba-08173699e830 (1)
    Task 22 | 22:37:21 | Creating missing vms: worker/5b8d175b-927e-48c1-a97c-f8b7ef099be5 (2) (00:01:04)
    Task 22 | 22:37:23 | Creating missing vms: worker/d806a4f4-6a1e-4f9c-a2ba-08173699e830 (1) (00:01:06)
    Task 22 | 22:37:28 | Creating missing vms: master/96cb1deb-bc4e-44b3-ac69-220cb0935bf8 (0) (00:01:11)
    Task 22 | 22:37:32 | Creating missing vms: worker/033088fc-5f91-4b57-b9e3-3cc718031e3b (0) (00:01:15)
    Task 22 | 22:37:32 | Updating instance master: master/96cb1deb-bc4e-44b3-ac69-220cb0935bf8 (0) (canary) (00:01:36)
    Task 22 | 22:39:08 | Updating instance worker: worker/033088fc-5f91-4b57-b9e3-3cc718031e3b (0) (canary) (00:03:49)
    Task 22 | 22:42:57 | Updating instance worker: worker/5b8d175b-927e-48c1-a97c-f8b7ef099be5 (2) (00:06:20)
    Task 22 | 22:49:17 | Updating instance worker: worker/d806a4f4-6a1e-4f9c-a2ba-08173699e830 (1) (00:07:15)
                      L Error: Action Failed get_task: Task 2b6efc6b-0a14-4cf5-7f1e-067d80563cce result: 1 of 3 post-start scripts failed. __Failed Jobs: ncp. Successful Jobs: bosh-dns, kubelet.__
    Task 22 | 22:56:32 | Error: Action Failed get_task: Task 2b6efc6b-0a14-4cf5-7f1e-067d80563cce result: 1 of 3 post-start scripts failed. Failed Jobs: ncp. Successful Jobs: bosh-dns, kubelet.

    Task 22 Started  Wed Apr 11 22:36:11 UTC 2018
    Task 22 Finished Wed Apr 11 22:56:32 UTC 2018
    Task 22 Duration 00:20:21
    Task 22 error

    Capturing task '22' output:
      Expected task '22' to succeed but state is 'error'

    Exit code 1

### Causes

* Not enabling the NSX-T Errand that tags everything (which is off by default because Flannal is the default networking stack for PKS).  This is set when configuring PKS itself in Ops Manager, in the PKS Tile settings under __Errands__ and  __NSX-T Validation errand__ should be set to "On" or NSX-T will require advanced manual tagging (that is as much as I know) and without it will not work out-of-the-box.  The errand is actually a requirement of NSX-T, unless you are a big brained VMware SE.

* Not using PCF Ops Manager for vSphere __build 249__.

### The Logs

If things go sideways and you are inclined to drill into why...

    bosh ssh -d <bosh deployment instance id> <worker/master/node id>

As mentioned, you won't get any help from __journald__, but you will find all the logs cleverly hidden at:

    /var/vcap/sys/log

> This PKS guide to [diagnostic tools](https://docs.pivotal.io/runtimes/pks/1-0/diagnostic-tools.html) has some good tips.

