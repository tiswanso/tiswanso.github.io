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

