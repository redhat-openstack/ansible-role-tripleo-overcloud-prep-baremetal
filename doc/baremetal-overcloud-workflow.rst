-------------------
Baremetal Overcloud
-------------------

tripleo-quickstart can deploy an overcloud on baremetal machine nodes rather than virtual machines.
Certain files and settings need to be customized to reflect the actual hardware in the system as well
as the networking setup. Depending on the machines used, additional steps may be required to set up
the environment for deployment.

###########################
Minimum System Requirements
###########################

By default, tripleo-quickstart requires 3 machines:

* 1 Undercloud (can be a Virtual Machine)
* 1 Overcloud Controller
* 1 Overcloud Compute

Commonly, deployments include HA (3 Overcloud Controllers) and multiple Overcloud Compute nodes.

Each Overcloud machine requires at least:

* 1 quad core CPU
* 8 GB free memory
* 60 GB disk space

The undercloud VM or baremetal machine requires:

* 1 quad core CPU
* 20 GB free memory
* 80 GB disk space

###########
Networking
###########

With a Virtual Environment, tripleo-quickstart sets up the networking as part of the workflow.
The networking arrangement needs to be set up prior to working with tripleo-quickstart.
The TripleO developer documentation at <http://docs.openstack.org/developer/tripleo-docs/environments/environments.html#networking>
provides an explanation of how the networking should be set up.

If using a VM for the undercloud, see <doc link to https://github.com/redhat-openstack/ansible-role-tripleo-baremetal-prep-virthost doc>
for the steps to set up the virthost machine for PXE forwarding.

..note:: When using baremetal nodes for the overcloud deployment, the ``overcloud_nodes``
         variable must overwritten and set to empty.

##############################
Customizing the configuration
##############################

Deploying to a baremetal overcloud requires customizing the following files to reflect the
baremetal environment and networking setup:

- undercloud.conf
- instackenv.json
- network-environment.yaml
- nic-configs files (optional)
- external network vlan (optional)

Customizing undercloud.conf
^^^^^^^^^^^^^^^^^^^^^^^^^^^
The undercloud.conf file is copied to the undercloud VM using a template where the system values
are variables. <https://github.com/openstack/tripleo-quickstart/blob/master/roles/tripleo/undercloud/templates/undercloud.conf.j2>.
The tripleo-quickstart defaults for these variables are suited to a virtual overcloud,
but can be overwritten by passing custom settings to tripleo-quickstart in a settings file
(--extra-vars @<file_path>). For example:

 ::

    undercloud_network_cidr: 10.0.5.0/24
    undercloud_local_ip: 10.0.5.1/24
    undercloud_network_gateway: 10.0.5.1
    undercloud_undercloud_public_vip: 10.0.5.2
    undercloud_undercloud_admin_vip: 10.0.5.3
    undercloud_local_interface: eth1
    undercloud_masquerade_network: 10.0.5.0/24
    undercloud_dhcp_start: 10.0.5.5
    undercloud_dhcp_end: 10.0.5.24
    undercloud_inspection_iprange: 10.0.5.100,10.0.5.120

Copying over customized instackenv.json, network-environment.yaml, nic-configs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``instackenv.json`` file is generated from a template in tripleo-quickstart:
<https://github.com/openstack/tripleo-quickstart/blob/master/roles/libvirt/setup/overcloud/tasks/main.yml#L91>.
A customized ``instackenv.json`` can be copied to the undercloud by overwriting the
``undercloud_instackenv_template`` variable with the path to the customized file.

See the TripleO developer documentation for an explanation of, and example of the ``instackenv.json`` file,
<http://docs.openstack.org/developer/tripleo-docs/environments/environments.html#instackenv-json>

Similarly, the ``network-environment.yaml`` file is generated from a template,
<https://github.com/openstack/tripleo-quickstart/blob/master/roles/tripleo/undercloud/tasks/post-install.yml#L32>
A customized ``network-environment.yaml`` file can be copied to the undercloud by overwriting the
`` network_environment_file`` variable with the path to the customized file.

By default, the virtual environment deployment uses the standard nic-configs files are there is no
ready section to copy custom nic-configs files.
The ``ansible-role-tripleo-overcloud-prep-config`` repo includes a task that copies the nic-configs
files if they are defined,
<https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-config/blob/master/tasks/main.yml#L15>

Customizing external network vlan
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If network-isolation is used in the deployment, tripleo-quickstart will, by default,
add a NIC on the external vlan to the undercloud,
<https://github.com/openstack/tripleo-quickstart/blob/master/roles/tripleo/undercloud/templates/undercloud-install-post.sh.j2#L88>.
When working with a baremetal overcloud, the vlan values must be customized with the correct
system-related values. The default vlan values can be overwritten in a settings file passed
to triple-quickstart as in the following example:

 ::

    undercloud_networks:
      external:
        address: 10.0.7.13
        netmask: 255.255.255.192
        device_type: ovs
        type: OVSIntPort
        ovs_bridge: br-ctlplane
        ovs_options: '"tag=102"'
        tag: 102


#########################################################
Additional steps preparing the environment for deployment
#########################################################
Depending on the parameters of the baremetal overcloud environment in use,
other pre-deployment steps may be needed to ensure that the deployment succeeds.
<https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-baremetal/tree/master/tasks>
includes a number of these steps. Whether each step is run, depends on variable values
that can be set per environment.

Some examples of additional steps are:

- Adding disk size hints
- Adjusting MTU values
- Rerunning introspection on failure


##############################################
Validating the environment prior to deployment
##############################################
In a baremetal overcloud deployment there is a custom environment and many related settings
and steps. As such, it is worthwhile to validate the environment and custom configuration
files prior to deployment.

A collection of validation tools is available in the 'clapper' repo:
<https://github.com/rthallisey/clapper/>.

An example of using one of these validation tools, validating the IPMI connections,
is already included in the baremetal overcloud playbook:
<https://github.com/redhat-openstack/ansible-role-tripleo-validate-ipmi/blob/master/templates/validate-overcloud-ipmi-connection.sh.j2>


###########################################
Baremetal overcloud workflows and playbooks
###########################################
The steps, and baremetal overcloud-specific considerations, are included in workflows as
executed by baremetal-related playbooks.

The playbook for using a VM undercloud, and deploying to a baremetal overcloud, is available at:
<https://github.com/redhat-openstack/ansible-role-tripleo-baremetal-prep-virthost/blob/master/playbooks/baremetal-virt-undercloud-tripleo.yml>

..note:: The baremetal overcloud playbook includes a step to validate the overcloud using the
         role: <https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-validate>.
         Again here, the variables that are used to set up the networking set by default,
         for the virtual overcloud environment and must be overwritten in a settings file.
         For example:
 ::

    # validate / tempest config
    public_network_type: vlan
    public_physical_network: datacentre
    public_segmentation_id: 102
    # overcloud network config
    floating_ip_cidr: 10.0.7.0/24
    public_net_pool_start: 10.0.7.45
    public_net_pool_end: 10.0.7.64
    public_net_gateway: 10.0.7.254

