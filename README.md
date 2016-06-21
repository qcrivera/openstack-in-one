#Deploy Mirantis Onpestack in one server

Instructions:

1- Provision a Ubuntu 14.04 LTS Trusty 64 bit server

2- install some base packages 

sudo apt-get install -y puppet-common qemu-kvm libvirt-bin python-swiftclient git virtinst

3- go to the files directory

chmod 777 qemu
mv qemu /etc/libvirt/hooks

virsh net-destroy default
virsh net-undefine default
virsh net-define pxe-net.xml
virsh net-start pxe-net
virsh net-autostart pxe-net

virsh net-define public-net.xml
virsh net-start public-net
virsh net-autostart public-net

virsh net-define private-net.xml
virsh net-start private-net
virsh net-autostart private-net

virsh net-define tagged-net.xml
virsh net-start taggedc-net
virsh net-autostart tagged-net

4- Prepare the environment with the needed isos

  a- mkdir /iso
  b- mv <your-git-clone-location>/iso/ipxe.iso /iso
  c- download MOS to /iso

5- Create the fuel VMs

mkdir /vms

sudo virt-install -n fuel-master -r 8192 \
-f /vms/fuel-master.qcow2 -s 300 \
-c /images/MirantisOpenStack-8.0.iso \
--network network=pxe-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=4

#you must go into the installtion fuelmenu and edit the PXE network to have 10.20.0.1 and not the default .2

6- create the OS VMs

#fuel has to be up an running before you go ahead with this step

sudo virt-install -n compute1 -r 32768 \
-f /vms/compute1.qcow1 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=8

sudo virt-install -n compute2 -r 32768 \
-f /vms/compute2.qcow1 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=8

sudo virt-install -n compute3 -r 32768 \
-f /vms/compute3.qcow1 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=8

sudo virt-install -n controller -r 8192 \
-f /vms/controller.qcow1 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=4