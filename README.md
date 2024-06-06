Group: ACS A

1. M. Aditya Kurniawan
2. Ivan Indrastata Ramadhan
3. Aliefya Fikri
4. M. Dyenta
5. Rain Elgratio

# Apache Cloudstack 4.18 on Ubuntu 22.04

## Network/System Cofiguration
Home network: 192.168.104.0/24

Gateway address: 192.168.104.1

Subnet mask: 255.255.255.0

Management IP address: 192.168.104.12

System IP address: 192.168.104.

System IP address: 192.168.104.

=============================================================

### Set static IP for management server
#### Network configuration with netplan
Create backup for all existing configurations by adding .bak

root@acsaserver:~$ cat /etc/netplan/01-netcfg.yaml
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.104.12/24]
      routes:
        - to: default
          via: 192.168.104.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [eno1]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```
#### Apply network configuration
```
sudo -i
netplan generate
netplab apply
reboot
```
#### Update system and install relevant tools
```
apt update & upgrade
apt install htop lynk duf -y
apt install bridge-utils
```
#### Configure logical volume management or LVM (Optional)
not required unless logical volume is already used
```
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
#### Install SSH server and other tools
```
apt-get-install opentd openssh-server sudo vim htop tar -y
apt-get install intel-microcode -y
passwd root
# change it to Pa$$w0rd
```
#### Enable root login
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```
#### Set timezone
Time and date must be synchronized in order to gain access
```
timedatectl set-timezone Asia/Jakarta

# if time & date is still not synced, configure it yourself
timedatectl "yyyy-mm-dd hh:mm:ss"
```
=============================================================
## Apache Cloudstack installation
### Cloudstack management server and everything in one machine
#### Reference

1. https://rohityadav.cloud/blog/cloudstack-kvm/
2. https://github.com/maradens/apachecloudstack

#### Cloudstack management server setup from SHAPEBLUE ACS 4.18
Shapeblue provides a repository of pre-built Cloudstack packages that are tested and maintained for production use. Using their repository simplifies the installation process and ensures that you are using a reliable and up-to-date version of Cloudstack.
```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

# update and install cloudstack management and mysql
# this process takes a very long time

apt-get update -y
apt-get install cloudstack-management mysql-server
```
#### Cloudstack usage and billing (Optional)
The Cloudstack usage server is an optional component that provides detailed tracking and reporting of resource usage within a Cloudstack environment, it is primarily useful for billing, chargeback, and detailed resource monitoring.
```
apt-get install cloudstack-usage 
```
#### Configure database --> next time using sed
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# paste below code to mysqld.cnf under [mysqld] block section

[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'

# restart mysql service

systemctl restart mysql
```
#### Deploy database as root and create user "cloud" with password "cloud"
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.12
```
#### Configure primary and secondary storage
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
#### Configure NFS (Network File System) server
NFS is a protocol that allows a computer to share directories and files with others over a network, so they can be accessed on remote systems as if they were local.
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```
### Configure cloudstack host with KVM hypervisor
#### Install KVM (Kernel-based Virtual Machine) host and Cloudstack agent
KVM is a virtualization solution for Linux on x86 hardware containing virtualization extensions (Intel VT or AMD-V). The CloudStack agent is required for integrating the KVM hypervisor with the CloudStack management system.
```
apt-get install qemu-kvm cloudstack-agent
```
#### Configure Qemu KVM virtualisation managemnet (libvirtd)
Qemu (Quick Emulator) is a generic and open-source machine emulator and virtualizer. When used with KVM, it can achieve near-native performance for virtual machines.
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# on Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

# configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
#### More configurations to support docker and other services
```
# on certain hosts where you may be running docker and other services, you may need to add the following in /etc/sysctl.conf
# then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
#### Generate unique host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
#### Configure firewall iptables (Optional)
```
NETWORK=192.168.101.0/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
# just answer yes for everything
```
#### Disable apparmour on libvirtd
Apparmour is a Linux security module that provides mandatory access control by restricting the capabilities of programs based on a predefined policy.

Disabling apparmour is necessary to avoid potential compatibility issues, performance overhead, and to simplify the management of virtual machines in environments where its restrictions are too limiting or redundant.
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```
#### Launch management server and start cloud service
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
# wait until all services is running successfully
```
#### Open Cloudtack dashboard
```
# after management server is UP, proceed to http://192.168.104.12(i.e. the cloudbr0-IP):8080/client 
#and log in using the default credentials - username admin and password password.
# you might find trouble in Microsoft Edge browser because it requires secure https protocol

http://192.168.104.12:8080/client
```
=============================================================
### Enable XRDP (Optional)
XRDP (X Remote Desktop Protocol) allows users to remotely access the graphical desktop of a Linux system using the Microsoft remote desktop protocol.

The following will not work in newer Ubuntu releases
#### Reference
```
https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04
```
#### Install desktop XFCE environment and remote desktop XRDP
```
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
```
#### Allow tcp ipv4 to listen to 3389
```
netstat -tulpn | grep xrdp

sed -i.bak 's/^\(port=\).*/\1tcp:\/\/:3389/' /etc/xrdp/xrdp.ini
systemctl restart xrdp
systemctl status xrdp
```
=============================================================
## Continue with instalation until you can access the dashboard
(insert image here)

### Register ISO and add instance
(insert image here)

=============================================================
## Install additional KVM host
```
sudo -i
apt update
apt upgrade

mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

apt-get update -y
```
### Install KVM host and Cloudstack agent
```
apt-get install qemu-kvm cloudstack-agent
```
### Configure Qemu KVM virtualisation management (libvirtd)
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# on Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

# configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
### More configuration to support Docker and other services
```
# on certain hosts where you may be running docker and other services, 
# you may need to add the following in /etc/sysctl.conf
# and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
### Generate unique host id
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
### Disable firewall for easier process
```
# make sure it's inactive
ufw status
```
### Disable apparmour on libvirtd
 ```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Setup storage for additional primary and secondary
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

### Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

### Enable root login (PermitRootLogin)
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config

# restart ssh service
service ssh restart
```