# VM Notes

## Custom VM Bringup without Metadata Server

### Ubuntu Workflow
1. build ubuntu qcow image with `NoCloud` cloud-init data sources
1. for each VM instance:
   1. create iso disk with meta-data / user-data files
   1. virt-install VM
   1. profit

#### Create Ubuntu qcow

NOTE: the following assumes disk-image-builder is installed

```
cd $IMGBLDDIR
VMSZ=100
DIB_CLOUD_INIT_DATASOURCES="NoCloud,ConfigDrive, OpenStack" disk-image-create ubuntu vm --image-size $VMSZ -oubuntu_vm_nocloud.qcow2

# result ends up in ubuntu_vm_nocloud.qcow2

```

##### References
- [disk-image-builder](https://docs.openstack.org/diskimage-builder/latest/)
- [cloud-init-datasources](https://github.com/NaohiroTamura/diskimage-builder/tree/master/elements/cloud-init-datasources)

#### Creating iso with cloud-init meta-data/user-data

#### Examples

The following meta-data file has the network interface params for interfaces using MACs to map to specific bridges during the libvirt setup (virt-install).  The user-data has an SSH key and root user password settings (pass123)

`meta-data`
```
instance-id: myVM1
local-hostname: myVM1
network-interfaces: |
  auto eth0
  iface eth0 inet static
    hwaddress ether 52:54:00:c4:8c:92
    address 172.1.1.93
    network 172.1.1.0
    netmask 255.255.255.0
    broadcast 172.1.1.255
    
  auto eth1
  iface eth1 inet static
    hwaddress ether 52:54:00:94:aa:18
    address 10.1.1.203
    network 10.1.1.0
    netmask 255.255.255.0
    broadcast 10.1.1.255
    gateway 10.1.1.1
```

`user-data`
```
#cloud-config
users:
  - name: root
    ssh_pwauth: True
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDCpuvLLTPbqIJTukxbm7v+LpE4bEJP6eaD6Ayu0o6uaZnS99zMpEH+54nNav+2RP41xWbGbnRP2wbPAd6CKhMZYG4ZGp+ZrsZYf4Vku78ilOj19blahblahblah+5s8f2v9w9SCe2l5gnw96Ql1wO4VzXFFRh9fM/sh/zzNHgvNz4Z5e7+xrwDZcpu5hJTHOv2pPp11w+J6ziOOi9dqt5hngtNH544r1CTqpl165QRljKwDxQf1JLVukW/ATlhELj6zsP5TvAwFQo+16r/e/OPDFgMifMmAkRaHXi+G1qqMwLdUp+4Uk3CJB5YodmQVy5DbDHYo6vez0jVCGTTihp7L2Ukv
chpasswd:
  list: |
    root:pass123
  expire: False
runcmd:
  - ifdown eth0
  - ifup eth0
  - ifdown eth1
  - ifup eth1
```

*Create the cidata disk iso for this VM*
```
genisoimage -output cidata.iso -volid cidata -joliet -rock user-data meta-data
```

*Start the VM with virt-install*
- Note: the mapping to bridge to network interface params in meta-data is done by setting the MAC addresses.
- Note 2: `--cpu host-passthrough` requires nested KVM setup in ubuntu, you can remove it if your guest images aren't hosting VMs as well.

```
VM_NAME=myVM1
VM_IMG=ubuntu_vm_nocloud.qcow2
cd $VMINSTDIR/$VM_NAME
cp $IMGBLDDIR/ubuntu_vm_nocloud.qcow2 .

VM_RAM=24288
VM_DISK=100
VCPUS=8
VM_MGMT_MAC="52:54:00:c5:8c:92"
VM_API_MAC="52:54:00:95:aa:18"

virt-install -n $VM_NAME -r $VM_RAM --os-type=linux --disk $VMINSTDIR/$VM_NAME/$VM_IMG,device=disk,bus=virtio,size=${VM_DISK} --disk path=$VMINSTDIR/$VM_NAME/cidata.iso,device=cdrom  --network bridge=br_mgmt,model=virtio,mac=${VM_MGMT_MAC} --network bridge=br_api,model=virtio,mac=${VM_API_MAC}   --network bridge=br_tenant,model=virtio --graphics none --noautoconsole --import --cpu host-passthrough --vcpus $VCPUS

# You can access the VM console while it comes up via
virsh console $VM_NAME

# if cloud-init works you can log in with root/pass123

```
