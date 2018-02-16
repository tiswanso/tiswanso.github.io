# Bringup:  Openstack Magnum (devstack)

## Prerequisites
1. AIO or Compute node
   1. 10+GB RAM
   1. 60+GB disk
   1. support for 3+ vCPUs
1. FIP network:
   1. 4+ available IPs
   1. internet access (through SNAT GW is fine)

## References
1. [Magnum Dev Quickstart](https://docs.openstack.org/magnum/latest/contributor/quickstart.html)
1. [devstack systemd](https://docs.openstack.org/devstack/latest/systemd.html)
1. [Magnum Troubleshooting](https://docs.openstack.org/magnum/latest/admin/troubleshooting-guide.html)
1. [Magnum wiki](https://wiki.openstack.org/wiki/Magnum)

## AIO / controller local.conf

### Pre-setup
1. For devstack run on Ubuntu 16.04 (xenial) with `stable/pike`, the following magnum patch is needed:
   1. [496229](https://review.openstack.org/#/c/496229/)
      1. clone the repo to /opt/stack/ and fetch/cherry-pick/patch it in and the subsequent stack.sh
         run won't clobber it.
      1. see "Example Ansible Plays" section for a play to auto-patch it.

### Example with obfuscated IPs

The example below shows a devstack all-in-one setup where:
- magnum + UI, octavia, and the other default services are started and `stable/pike` version is used.
- 10.1.1.7 is the IP of the node & used for the openstack API
- enp9s0.1237 is a precreated interface with access to a public GW on 172.1.1.0/24
  - NOTE: after stacking it may be required to remove the IP for the public GW from
    this interface--for some reason devstack or l3-agent sometimes adds it.
- openstack FIP address pool of 172.1.1.130 - 150
- LIBVIRT_TYPE=kvm is to force nova to setup libvirt for kvm hypervisor usage
- max MTU for all networks = 1400 and advertised to VMs via DHCP


NOTE: you may want to remove `disable_service tempest` from your setup if you want the tests to run.


```shell
[[local|localrc]]
HOST_IP=10.1.1.7
ADMIN_PASSWORD=password
MYSQL_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SWIFT_HASH=$ADMIN_PASSWORD
SWIFT_TEMPURL_KEY=$ADMIN_PASSWORD

GIT_BASE=https://git.openstack.org

PUBLIC_INTERFACE=enp9s0.1237
FLOATING_RANGE=172.1.1.0/24
PUBLIC_NETWORK_GATEWAY=172.1.1.1
Q_FLOATING_ALLOCATION_POOL="start=172.1.1.130,end=172.1.1.150"

LIBVIRT_TYPE=kvm

disable_service tempest

# Enable barbican service and use it to store TLS certificates
# For details https://docs.openstack.org/developer/magnum/userguide.html#transport-layer-security
enable_plugin barbican https://git.openstack.org/openstack/barbican stable/pike

enable_plugin heat https://git.openstack.org/openstack/heat stable/pike

# Enable magnum plugin after dependent plugins
enable_plugin magnum https://git.openstack.org/openstack/magnum stable/pike

# Optional:  uncomment to enable the Magnum UI plugin in Horizon
enable_plugin magnum-ui https://github.com/openstack/magnum-ui stable/pike

VOLUME_BACKING_FILE_SIZE=20G

enable_plugin neutron-lbaas https://git.openstack.org/openstack/neutron-lbaas stable/pike
enable_plugin octavia https://git.openstack.org/openstack/octavia stable/pike

# Disable LBaaS(v1) service
disable_service q-lbaas
# Enable LBaaS(v2) services
enable_service q-lbaasv2
enable_service octavia
enable_service o-cw
enable_service o-hk
enable_service o-hm
enable_service o-api

[[post-config|/etc/neutron/neutron.conf]]
[DEFAULT]
advertise_mtu = True
global_physnet_mtu = 1400

```


## Example Ansible Plays
### Setup Magnum with Patches

NOTE: the `git config` portion is only needed with the local cherry-pick.  If you already have a git config on
the system then remove that task.

```
- name: bring in magnum patch needed for stable/pike
  hosts: controller
  tasks:
    - name: clone magnum under stack user
      git: repo=https://github.com/openstack/magnum
           dest=/opt/stack/magnum
           version={{ devstack_version }}
      become: yes
      become_user: stack
      tags: ["patch_magnum"]

    - name: setup git config so we can cherry-pick
      shell: >
             git config --global user.name "Tim Swanson" &&
             git config --global user.email "tiswanso@cisco.com"
      become: yes
      become_user: stack
      tags: ["patch_magnum"]

    - name: cherry-pick patch
      shell: >
             git fetch https://git.openstack.org/openstack/magnum {{ item }} && git cherry-pick FETCH_HEAD
      args:
        chdir: /opt/stack/magnum
      become: yes
      become_user: stack
      with_items:
        - refs/changes/29/496229/17
      tags: ["patch_magnum"]

```
### Setup controller node local.conf from templates

Example Playbook vars being set--template task is in role setup_devstack:

```
- name: setup devstack
  hosts: controller
  vars:
    devstack_version: stable/pike
    enable_lbaas: True
    local_conf_template: magnum_local.conf.j2
  roles:
    - { role: setup_devstack, tags: ["setup_devstack"] }
```


Task:

```
- name: Setup local.conf
  template:
    src: "{{ local_conf_template }}"
    dest: "{{ local_conf_dir }}/local.conf"
  become: yes
  become_user: stack
  when: not 'computes' in group_names
  tags: ["setup_devstack", "local_conf"]
```

`magnum_local.conf.j2`
```
{% include "lconf_base.j2" %}

LIBVIRT_TYPE=kvm

disable_service tempest

# Enable barbican service and use it to store TLS certificates
# For details https://docs.openstack.org/developer/magnum/userguide.html#transport-layer-security
enable_plugin barbican https://git.openstack.org/openstack/barbican {{ barbican_version | default(devstack_version) }}

enable_plugin heat https://git.openstack.org/openstack/heat {{ heat_version | default(devstack_version) }}

# Enable magnum plugin after dependent plugins
enable_plugin magnum https://git.openstack.org/openstack/magnum {{ magnum_version | default(devstack_version) }}

# Optional:  uncomment to enable the Magnum UI plugin in Horizon
enable_plugin magnum-ui https://github.com/openstack/magnum-ui {{ magnum_version | default(devstack_version) }}

VOLUME_BACKING_FILE_SIZE=20G

{% if enable_lbaas | default(False) %}
enable_plugin neutron-lbaas https://git.openstack.org/openstack/neutron-lbaas {{ neutron_version | default(devstack_version) }}
enable_plugin octavia https://git.openstack.org/openstack/octavia {{ octavia_version | default(devstack_version) }}

# Disable LBaaS(v1) service
disable_service q-lbaas
# Enable LBaaS(v2) services
enable_service q-lbaasv2
enable_service octavia
enable_service o-cw
enable_service o-hk
enable_service o-hm
enable_service o-api
{% endif %}
```

`lconf_base.j2`
```
[[local|localrc]]
HOST_IP={{ host_ip | default(ansible_default_ipv4['address']) }}
ADMIN_PASSWORD=password
MYSQL_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SWIFT_HASH=$ADMIN_PASSWORD
SWIFT_TEMPURL_KEY=$ADMIN_PASSWORD

GIT_BASE=https://git.openstack.org

PUBLIC_INTERFACE={{ public_base_intf }}{% if pub_vlan is defined %}.{{ pub_vlan }}{% endif %}

FLOATING_RANGE={{ fip_cidr }}
PUBLIC_NETWORK_GATEWAY={{ fip_cidr | ipaddr(1) | ipaddr('address') }}
Q_FLOATING_ALLOCATION_POOL="start={{ fip_cidr | ipaddr(fip_start) | ipaddr('address') }},end={{ fip_cidr | ipaddr(fip_end) | ipaddr('address') }}"
```

## Magnum Architecture (dated)

![magnum arch](https://wiki.openstack.org/wiki/File:Magnum_architecture.png)