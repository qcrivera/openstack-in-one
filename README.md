#Deploy Mirantis OpenStack in one server

##Assumptions:

1. You have a bare-metal server with at least 64 gigs of RAM and 10 CPU cores.
2. The server has a public IP address that has access to Internet
3. The public IP address is assigned to a bond1 interface

##Instructions:

###1- Prepare your environment

Provision a Ubuntu 14.04 LTS Trusty 64 bit server

In your linux PC add the packacge 'virt-manager' 
```
sudo apt-get install virt-manager
```
open virt-manager
```
virt-manager --no-fork
```
add a connection to the host

  a- File/Add Connection/Conect to remote host/fill in hostname/click connect
  
  b- go to the terminal screen were you started virt manager and type the root password

#### the following steps are to be executed in your server

###2- install some base packages 

```
sudo apt-get install -y puppet-common qemu-kvm libvirt-bin python-swiftclient git virtinst
```

###3- after cloning this repo go to the files directory

```
sed -e "s/Host_ipaddr=/Host_ipaddr=$(facter ipaddress_bond1)/g" qemu > /etc/libvirt/hooks/qemu
chmod 777 /etc/libvirt/hooks/qemu
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
virsh net-autostart private-ne
virsh net-define tagged-net.xml
virsh net-start tagged-net
virsh net-autostart tagged-net
```

###4- Prepare the environment with the needed isos

```
mkdir /iso
```

* mv your-git-clone-location/ipxe.iso /iso
  
* download MOS to /iso

Can be downloaded from

https://software.mirantis.com/openstack-download-form/

###5- Create the fuel VM

```
mkdir /vms
sudo virt-install -n fuel-master -r 8192 \
-f /vms/fuel-master.qcow2 -s 150 \
-c /iso/MirantisOpenStack-8.0.iso \
--network network=pxe-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=4
```
using virt-manager you must go into the installtion fuelmenu and edit the PXE network to have 10.20.0.1 as the gateway and not the default .2

Wait until the installation is done and login to the fuel node

*after installation is done you might need to go into virt-manager and manully start the VMs, the rest of the process will continue automatically

```
systemctl list-jobs
```

the list should be empty

###6- Apply NAT rules to access fuel

We will activate the confiuration performed by the /etc/libvirt/hooks/qemu script from step 3

```
service libvirt-bin restart
virsh destroy fuel-master
virsh start fuel-master
```

Make sure fuel-master boots correctly and wait for the  'systemctl list-jobs' to return no jobs 

```
iptables-save
```

the output should have rule similar to the following

```
-A PREROUTING -d 169.54.100.74/32 -p tcp -m tcp --dport 443 -j DNAT --to-destination 10.20.0.2:443
-A PREROUTING -d 169.54.100.74/32 -p tcp -m tcp --dport 8443 -j DNAT --to-destination 10.20.0.2:8443
-A FORWARD -d 10.20.0.2/32 -p tcp -m state --state NEW -m tcp --dport 8443 -j ACCEPT
-A FORWARD -d 10.20.0.2/32 -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
```

###7- create the OS VMs

####fuel has to be up an running before you go ahead with this step

```
sudo virt-install -n compute1 -r 32768 
-f /vms/compute1.qcow2 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=8

sudo virt-install -n compute2 -r 32768 \
-f /vms/compute2.qcow2 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=8

sudo virt-install -n compute3 -r 32768 \
-f /vms/compute3.qcow2 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=8

sudo virt-install -n controller -r 8192 \
-f /vms/controller.qcow2 -s 100 \
-c /iso/ipxe.iso \
--network network=pxe-net,model=virtio \
--network network=public-net,model=virtio \
--network network=private-net,model=virtio \
--network network=tagged-net,model=virtio \
--video=vmvga --graphics vnc,listen=0.0.0.0 --noautoconsole -v --vcpus=4

```

###7- Deploy Minantis Openstack

1. Select the 8192 RAM node as controller
2. Select the 3 32768 noses as compute
3. Configure the interface order according to the configs/interfa-order.png file
4. Configure networking according to the configs/Fuel-networks.png and configs/floating-network.png files
5. Deploy and wait 


*after Ubuntu is installed you might need to go into virt-manager and manully start the VMs, the rest of the process will continue automatically

###8- Add some configs for remote access

1. Login into the controller and verify the IP that it was assigned from the public network
2. If needed edit the files/iptables.sh script to replace the 172.30.200.4 address
3. Execute the ```iptables.sh``` scritp inside the files directory. 


