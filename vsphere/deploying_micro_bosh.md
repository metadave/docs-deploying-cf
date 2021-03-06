---
title: Deploying MicroBOSH
---

Installation of BOSH is done using MicroBOSH, which is a single VM that
includes all of the BOSH components.
If you want to play around with BOSH, or create a simple development setup, you
can install MicroBOSH using the BOSH Deployer.
If you would like to use BOSH in production to manage a distributed system, you
also use the BOSH Deployer, install MicroBOSH, and then use it as a means to
deploy the final distributed system on multiple VMs.

A good way to think about this two step process is to consider that BOSH is a
distributed system in itself.
Since BOSH's core purpose is to deploy and manage distributed systems, it makes
sense that we would use it to deploy itself.
On the BOSH team, we gleefully refer to this as [Inception](http://en.wikipedia.org/wiki/Inception).

## <a id="bootstrap"></a>BOSH Bootstrap ##

### <a id="prerequisites"></a>Prerequisites ###

We recommend that you run the BOSH bootstrap from Ubuntu since it is the
distribution used by the BOSH team, and has been thoroughly tested.

Plan to have around 8 GB of free disk space for the `bosh_cli` if you plan to use it to deploy CF releases.
You'll need around 3 GB free disk space in `/tmp`.

* Install some core packages on Ubuntu that the BOSH deployer depends on.

<pre class="terminal">
$ sudo apt-get -y install libsqlite3-dev genisoimage libxml2-dev libxslt-dev
</pre>

* Install Ruby and RubyGems. Refer to the [Installing Ruby](../common/install_ruby_rbenv.html) page for help with Ruby installation.

* Install the BOSH Deployer Ruby gem.

<pre class="terminal">
$ gem install bosh_cli
$ gem install bosh_cli_plugin_micro
</pre>

Once you have installed the deployer, you will be able to use `bosh micro`
commands.
To see help on these type `bosh help micro`

### <a id="config"></a>Configuration ###

BOSH deploys things from a subdirectory under a deployments directory.
So create both and name it appropriately.
In our example we named it micro01.

<pre class="terminal">
  mkdir deployments
  cd deployments
  mkdir micro01
</pre>

BOSH needs a deployment manifest for MicroBOSH.
It must be named `micro_bosh.yml`.
Create one in your new directory following the example below:

~~~yaml
---
name: MicroBOSH01

network:
  ip: <ip_address_you_want_for_microbosh>
  netmask: <netmask_for_the_subnet_you_are_deploying_to>
  gateway: <gateway_for_the_subnet_you_are_deploying_to>
  dns:
  # The micro-bosh VM has the following DNS entries in its /etc/resolv.com, allowing it to resolve, for example, IaaS FQDNs.
  - <ip_for_dns>
  cloud_properties:
    name: <network_name_according_to_vsphere>

resources: # this seems like good sizing for microbosh
   persistent_disk: 16384
   cloud_properties:
      ram: 8192
      disk: 16384
      cpu: 4
cloud:
  plugin: vsphere
  properties:
    agent:
      ntp:
        - <ntp_host_1>
        - <ntp_host_2>
    vcenters:
      - host: <vcenter_ip>
        user: <vcenter_userid>
        password: <vcenter_password>
        datacenters:
          - name: <datacenter_name>
            vm_folder: <vm_folder_name>
            template_folder: <template_folder_name>
            disk_path: <subdir_to_store_disks>
            datastore_pattern: <data_store_pattern>
            persistent_datastore_pattern: <persistent_datastore_pattern>
            allow_mixed_datastores: <true_if_persistent_datastores_and_datastore_patterns_are_the_same>
            clusters:
            - <cluster_name>:
              resource_pool: <resource_pool_name_optional>

apply_spec:
  properties:
    vcenter:
      host: <vcenter_ip>
      user: <vcenter_userid>
      password: <vcenter_password>
      datacenters:
        - name: <datacenter_name>
          vm_folder: <vm_folder_name>
          template_folder: <template_folder_name>
          disk_path: <subdir_to_store_disks>
          datastore_pattern: <data_store_pattern>
          persistent_datastore_pattern: <persistent_datastore_pattern>
          allow_mixed_datastores: <true_if_persistent_datastores_and_datastore_patterns_are_the_same>
          clusters:
          - <cluster_name>:
            resource_pool: <resource_pool_name_optional>
    dns:
      # The BOSH powerDNS contacts the following DNS server for serving DNS entries from other domains.
      recursor: <ip_for_dns>

logging:
  # If needed increase the default logging level to trace REST traffic with IaaS providers. Default is info
  level: debug
  # Default location is <deployment_dir>/bosh_micro_deploy.log
  # file :
~~~

The `apply_spec` block provides MicroBOSH with the vCenter settings in order for it
to deploy Cloud Foundry.
It is different than the vCenter you are using to deploy MicroBOSH because
MicroBOSH can deploy to a different vCenter than the one it was deployed to.

If you want to create a role for the BOSH user in vCenter, the privileges are
defined [here](./vcenter_user_privileges.html).

Before you can run MicroBOSH deployer, you have to create folders according to
the values in your manifest.

In the example below, `vm_folder` contains the running virtual machine
instance of MicroBOSH.
`template_folder` stores the BOSH Stemcells that are used to create the
virtual machine instances.
`disk_path` is the destination datastore for the MicroBOSH deployment.
`resource_pool` defines the VMs for MicroBOSH to use as collections of VMs
which are on the
same network, are the same kind of VM from the IaaS’s perspective, and are
built from the same stemcell.

1. Create `vm_folder`
1. Create `template_folder`
1. Create `disk_path` in the appropriate datastores
1. Create the `resource_pool` (optional)

![vcenter-folders](/images/vcenter-folders.png)

<br>

![vcenter-disk-path](/images/vcenter-disk-path.png)

* Datastore Patterns

The datastore pattern above could just be the name of a datastore or some
regular expression matching the datastore name.

If you have a datastore called "vc\_data\_store\_1" and you would like to use
this datastore for both persistent and non persistent disks, your config would
look like this:

~~~yaml
               datastore_pattern: vc_data_store_1
               persistent_datastore_pattern:  vc_data_store_1
               allow_mixed_datastores: true
~~~

If you have two datastores called "vc\_data\_store\_1" and
"vc\_data\_store\_2", and you would like to use both datastore for both
persistent and non persistent disks, your config would look like this:

~~~yaml
               datastore_pattern: vc_data_store_?
               persistent_datastore_pattern:  vc_data_store_?
               allow_mixed_datastores: true
~~~

If you have two datastores called "vnx:1" and "vnx:2", and you would like to
separate your persistent and non persistent disks, your config would look like
this:

~~~yaml
               datastore_pattern: vnx:1
               persistent_datastore_pattern: vnx:2
               allow_mixed_datastores: false
~~~

## <a id="deploy"></a>Deployment ##

Download a BOSH Stemcell:

BOSH Stemcells are virtual machine templates that BOSH clones to create virtual
machine instances.
After cloning, a BOSH Stemcell applies the specifications outlined for it in
the deployment manifest.
A BOSH vSphere Stemcell usually exceeds 500MB in size,
except for the `light` Stemcells.

You need Internet access for `bosh_cli` to download the stemcells.
You may need to temporarily set `http_proxy` and `https_proxy` if
you are behind a corporate firewall.
If so, remember to unset it before completing the following steps if your proxy
won't allow the newly-created `micro_bosh` VM to be contacted.

<pre class="terminal">
$ bosh public stemcells
+---------------------------------------------+
| Name                                        |
+---------------------------------------------+
| bosh-stemcell-1657-aws-xen-ubuntu.tgz       |
| bosh-stemcell-1657-aws-xen-centos.tgz       |
| light-bosh-stemcell-1657-aws-xen-ubuntu.tgz |
| light-bosh-stemcell-1657-aws-xen-centos.tgz |
| bosh-stemcell-1657-openstack-kvm-ubuntu.tgz |
| bosh-stemcell-1657-vsphere-esxi-ubuntu.tgz  |
| bosh-stemcell-1657-vsphere-esxi-centos.tgz  |
+---------------------------------------------+
$ bosh download public stemcell bosh-stemcell-XXXX-vsphere-esxi-ubuntu.tgz
</pre>

CD to the deployments directory and set the deployment.
This assumes you named the directory micro01.

<pre class="terminal">
$ cd deployments
$ bosh micro deployment micro01
Deployment set to '/var/vcap/deployments/micro01/micro_bosh.yml'
</pre>

Deploy a stemcell for MicroBOSH.

<pre class="terminal">
$ bosh micro deploy bosh-stemcell-XXXX-vsphere-esxi-ubuntu.tgz
</pre>


### <a id="verify"></a>Checking Status of a MicroBOSH Deploy ###

Target the MicroBOSH

<pre class="terminal">
bosh target <ip_address_from_your_micro_bosh_manifest:25555>
</pre>

Login with admin/admin.

The `status` command will show the persisted state for a given MicroBOSH
instance.

<pre class="terminal">
$ bosh micro status
Stemcell CID   sc-1744775e-869d-4f72-ace0-6303385ef25a
Stemcell name  bosh-stemcell-1341-vsphere-esxi-ubuntu
VM CID         vm-ef943451-b46d-437f-b5a5-debbe6a305b3
Disk CID       disk-5aefc4b4-1a22-4fb5-bd33-1c3cdb5da16f
MicroBOSH CID bm-fa74d53a-1032-4c40-a262-9c8a437ee6e6
Deployment     /home/user/cloudfoundry/bosh/deployments/micro_bosh/micro_bosh.yml
Target         https://192.168.9.20:25555 #IP Address of the Director
</pre>

### <a id="listing"></a>Listing Deployments ###

The `deployments` command prints a table view of `bosh-deployments.yml`:

<pre class="terminal">
$ bosh micro deployments
</pre>

The files in your current directory need to be saved if you later want to be
able to update your MicroBOSH instance.
They are all text files, so you can commit them to a git repository to make
sure they are safe in case your bootstrap VM goes away.


### <a id="send-message"></a>Sending Messages to the MicroBOSH Agent ###

The `bosh` CLI can send messages over HTTP to the agent using the `agent`
command.

<pre class="terminal">
$ bosh micro agent ping
"pong"
</pre>

