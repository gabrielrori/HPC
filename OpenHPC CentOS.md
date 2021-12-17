



# OpenHPC con Slurm en CentOS 7


## Configuración de Maestro y nodos
```sh
Master
Compute1
Compute2
```
```sh
Hostname: master
enp0s3: private(internal) network 192.168.1.254
enp0s8: public network 192.168.11.6
```
```sh
Hostname: c1
enp0s3: private network 192.168.1.253 MAC 08:00:27:99:b3:4f
```
```sh
Hostname: c2
enp0s3: private network 192.168.1.252 MAC 08:00:27:99:b3:5f
```

## Ingresar al servidor linux para instalar el sistema
ssh root@192.168.11.6

## Preparar para instalar, editar los archivos del host
```sh
vi /etc/hosts
192.168.1.254 master
164.115.20.65 build.openhpc.community
```
Disable Firewall.
```sh
systemctl disable firewalld
systemctl stop firewalld
```
Update system.
```sh
yum -y update
yum -y install wget
```
## Instalación del Repositorio de openHPC
```sh
yum -y install http://build.openhpc.community/OpenHPC:/1.3/CentOS_7/x86_64/ohpc-
release-1.3-1.el7.x86_64.rpm
```
## Instalación del paquete básico de OpenHPC.
```sh
yum -y install ohpc-base
yum -y install ohpc-warewulf5.6. Edit Time Server configuration add servername time.haii.or.th.
systemctl enable ntpd.service
echo "server time.haii.or.th" >> /etc/ntp.conf
systemctl restart ntpd
```
## Instalación de Slurm para Job scheduling y workload management.
```sh
yum -y install ohpc-slurm-server
perl -pi -e "s/ControlMachine=\S+/ControlMachine=master/" /etc/slurm/slurm.conf
```
## Configure internal interface. Replace eth1 by enp0s3.
```sh
perl -pi -e "s/device = eth1/device = enp0s3/" /etc/warewulf/provision.conf
```
## Enable tftp service.
```sh
perl -pi -e "s/^\s+disable\s+= yes/ disable = no/" /etc/xinetd.d/tftp
```
## Up ip interface enp0s3.
```sh
ifconfig enp0s3 192.168.1.254 netmask 255.255.255.0 up
```
Restart services all service.
```sh
systemctl restart xinetd
systemctl enable mariadb.service
systemctl restart mariadb
systemctl enable httpd.service
systemctl restart httpd# systemctl enable dhcpd.service
```
## Define image location for compute node. Use /opt/ohpc/admin/images/centos7.4
```sh
export CHROOT=/opt/ohpc/admin/images/centos7.4
wwmkchroot centos-7 $CHROOT
```
## Install OpenHPC for compute node.
```sh
yum -y --installroot=$CHROOT install ohpc-base-compute
```
## Create resolv.conf file for compute node.
```sh
cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf
```
## Install slurm client in image compute node.
```sh
yum -y --installroot=$CHROOT install ohpc-slurm-client
```
## Install NTP for compute node.
```sh
yum -y --installroot=$CHROOT install ntp
```
## Install kernel for compute node.
```sh
yum -y --installroot=$CHROOT install kernel
```
## Install modules user environment for compute node.
```sh
yum -y --installroot=$CHROOT install lmod-ohpc
```
## Initialize warewulf database for OpenHPC.
```sh
wwinit database# wwinit ssh_keys
```
## Create NFS Client for compute node.
```sh
mount /home and /opt/ohpc/pub from 192.168.1.254(master)
echo "192.168.1.254:/home /home nfs nfsvers=3,nodev,nosuid,noatime 0 0" >> $CHROOT/etc/fstab
echo "192.168.1.254:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev,noatime 0 0" >> $CHROOT/etc/fstab
```
## Create NFS Server.
Share directory /home and /opt/ohpc/pub on 192.168.1.254(master)
```sh
echo "/home *(rw,no_subtree_check,fsid=10,no_root_squash)" >> /etc/exports
echo "/opt/ohpc/pub *(ro,no_subtree_check,fsid=11)" >> /etc/exports
exportfs -a
systemctl restart nfs-server
systemctl enable nfs-server
```
## Setup NTP on compute node.
Add NTP server 192.168.1.254(master) to compute node.
```sh
chroot $CHROOT systemctl enable ntpd
echo "server 192.168.1.254" >> $CHROOT/etc/ntp.conf
```
## Update basic slurm configuration.
Slurm client node have 2 node is c1 and c2. And vCPU is 2 core/nodes.
```sh
perl -pi -e "s/^NodeName=(\S+)/NodeName=c[1-2]/" /etc/slurm/slurm.conf
perl -pi -e "s/^PartitionName=normal Nodes=(\S+)/PartitionName=normal Nodes=c[1-2]/" /etc/slurm/slurm.conf
perl -pi -e "s/^Sockets=(\S+)/Sockets=1/" /etc/slurm/slurm.conf
perl -pi -e "s/^CoresPerSocket=(\S+)/CoresPerSocket=2/" /etc/slurm/slurm.conf
perl -pi -e "s/^ThreadsPerCore=(\S+)/ThreadsPerCore=1/" /etc/slurm/slurm.conf# perl -pi -e "s/^NodeName=(\S+)/NodeName=c[1-2]/" $CHROOT/etc/slurm/slurm.conf
perl -pi -e "s/^PartitionName=normal Nodes=(\S+)/PartitionName=normal Nodes=c[1-2]/" $CHROOT/etc/slurm/slurm.conf
perl -pi -e "s/^Sockets=(\S+)/Sockets=1/" $CHROOT/etc/slurm/slurm.conf
perl -pi -e "s/^CoresPerSocket=(\S+)/CoresPerSocket=2/" $CHROOT/etc/slurm/slurm.conf
perl -pi -e "s/^ThreadsPerCore=(\S+)/ThreadsPerCore=1/" $CHROOT/etc/slurm/slurm.conf
systemctl enable munge
systemctl enable slurmctld
systemctl start munge
systemctl start slurmctld
chroot $CHROOT systemctl enable slurmd
```
## Increase locked memory limits.
```sh
perl -pi -e 's/# End of file/\* soft memlock unlimited\n$&/s' /etc/security/limits.conf
perl -pi -e 's/# End of file/\* hard memlock unlimited\n$&/s' /etc/security/limits.conf
perl -pi -e 's/# End of file/\* soft memlock unlimited\n$&/s' $CHROOT/etc/security/limits.conf
perl -pi -e 's/# End of file/\* hard memlock unlimited\n$&/s' $CHROOT/etc/security/limits.conf
```
Enable slurm pam module
```sh
echo "account required
pam_slurm.so" >> $CHROOT/etc/pam.d/sshd
```
## rsyslog setup.
In compute node set rsyslog config to 192.168.1.254(master).
```sh
perl -pi -e "s/\\#\\\$ModLoad imudp/\\\$ModLoad imudp/" /etc/rsyslog.conf
perl -pi -e "s/\\#\\\$UDPServerRun 514/\\\$UDPServerRun 514/" /etc/rsyslog.conf
systemctl restart rsyslog
echo "*.* @192.168.1.254:514" >> $CHROOT/etc/rsyslog.conf
perl -pi -e "s/^\*\.info/\\#\*\.info/" $CHROOT/etc/rsyslog.conf# perl -pi -e "s/^authpriv/\\#authpriv/" $CHROOT/etc/rsyslog.conf
perl -pi -e "s/^mail/\\#mail/" $CHROOT/etc/rsyslog.conf
perl -pi -e "s/^cron/\\#cron/" $CHROOT/etc/rsyslog.conf
perl -pi -e "s/^uucp/\\#uucp/" $CHROOT/etc/rsyslog.conf
```
## Install Ganglia Monitor OpenHPC.
Change <sms> to master and gridname to MySite..
```sh
yum -y install ohpc-ganglia
yum -y --installroot=$CHROOT install ganglia-gmond-ohpc
cp /opt/ohpc/pub/examples/ganglia/gmond.conf /etc/ganglia/gmond.conf
perl -pi -e "s/<sms>/master/" /etc/ganglia/gmond.conf
cp /etc/ganglia/gmond.conf $CHROOT/etc/ganglia/gmond.conf
echo "gridname MySite.." >> /etc/ganglia/gmetad.conf
systemctl enable gmond
systemctl enable gmetad
systemctl start gmond
systemctl start gmetad
chroot $CHROOT systemctl enable gmond
systemctl try-restart httpd
```
## Add Clustershell.
Add role adm to masteradd role compute to ${compute_prefix}[1-${num_computes}] by 
```sh
compute_prefix = c and
num_computes = 2
yum -y install clustershell-ohpc
cd /etc/clustershell/groups.d
mv local.cfg local.cfg.orig
echo "adm: master" > local.cfg
echo "compute: c[1-2]" >> local.cfg
echo "all: @adm,@compute" >> local.cfg
cd
```
## Import files.
```sh
wwsh file list
wwsh file import /etc/passwd
wwsh file import /etc/group
wwsh file import /etc/shadow
wwsh file import /etc/slurm/slurm.conf
wwsh file import /etc/munge/munge.key
wwsh file list
```
## Assemble bootstrap image.
```sh
export WW_CONF=/etc/warewulf/bootstrap.conf
echo "drivers += updates/kernel/" >> $WW_CONF
echo "drivers += overlay" >> $WW_CONF
```
## Build bootstrap image.
```sh
wwbootstrap `uname -r`
```
## create Virtual Node File System(VNFS) image.
```sh
wwvnfs --chroot $CHROOT
```
## Define compute node from MAC Address.
```sh
GATEWAYDEV=enp0s8 is Public interface.
echo "GATEWAYDEV=enp0s8" > /tmp/network.$$
wwsh -y file import /tmp/network.$$ --name network
wwsh -y file set network --path /etc/sysconfig/network --mode=0644 --uid=0
wwsh file list
```
Have 2 compute node include c1 and c2. Add to warewulf data store.
```sh
wwsh -y node new c1 --ipaddr=192.168.1.253 --hwaddr=08:00:27:99:B3:4F -D enp0s8
wwsh -y node new c2 --ipaddr=192.168.1.252 --hwaddr=08:00:27:99:B3:5F -D enp0s8
wwsh node list
```
*if you need delete node. use command “wwsh node delete c1”.

## Defind VNFS to compute node.
```sh
wwsh -y provision set "c1" --vnfs=centos7.4 --bootstrap=`uname -r` --files=dynamic_hosts,passwd,group,shadow,slurm.conf,munge.key,network
wwsh -y provision set "c2" --vnfs=centos7.4 --bootstrap=`uname -r` --files=dynamic_hosts,passwd,group,shadow,slurm.conf,munge.key,network
wwsh provision list
```
## Restart ganglia/dhcp services
```sh
systemctl restart gmond
systemctl restart gmetad# systemctl restart dhcpd
wwsh pxe update
```
## Create usersname test
```sh
useradd -m test
passwd test
wwsh file resync
updatenode compute -F
```
## Power On compute node have correct MAC Address and first boot fromnetwork.
Power On compute node and test by pdsh command on master node.
```sh
pdsh -w c1 uptime
pdsh -w c[1-2] uptime
```
## Resource Manager Startup.
```sh
systemctl restart munge
systemctl restart slurmctld
pdsh -w c[1-2] systemctl restart slurmd
```
Test mung by
```sh
munge -n | unmunge
munge -n | ssh c1 unmunge
munge -n | ssh c2 unmunge
```
Test slurm by
```sh
systemctl status slurmctld# ssh c1 systemctl status slurmd
ssh c2 systemctl status slurmd
```
```sh
Test resource by scontrol show nodes
```sh
See State=IDLE. 
```
Resource is good. else State Please run scontrol update NodeName=c1 State=Resume

## Compilers
Install mpicc compiler.
```sh
yum -y install openmpi3-gnu7-ohpc mpich-gnu7-ohpc lmod-defaults-gnu7-openmpi3-ohpc
```
## Test Interactive execution.
su to test user.
su - test
Compile MPI "hello world" example
$ mpicc -O3 /opt/ohpc/pub/examples/mpi/hello.c
Submit interactive job request. n=mpi tasks and N=nodes.
$ srun -n 2 -N 1 --pty /bin/bash
Use prun to launch executable
[test@c1 ~]$ prun ./a.out
[prun] Master compute host = c1
[prun] Resource manager = slurm
[prun] Launch cmd = mpirun ./a.out (family=openmpi3)
Hello, world (2 procs total)--> Process # 0 of 2 is alive. -> c1
--> Process # 1 of 2 is alive. -> c1

## Test Batch execution.
su to test user.
```sh
su - test
```
Copy sample job script from /opt/ohpc/pub/examples/pbspro/job.mpi
```sh
[test@master ~]$ cp /opt/ohpc/pub/examples/slurm/job.mpi .
```
Edit -n to 1 for mpi tasks and N to 2 for nodes.
```sh
[test@master ~]$ vi job.mpi
#!/bin/bash
#SBATCH -J test
```
Job name
```sh
#SBATCH -o job.%j.out # Name of stdout output file (%j expands to jobId)
#SBATCH -N 1 # Total number of nodes requested
#SBATCH -n 2 # Total number of mpi tasks requested
#SBATCH -t 01:30:00 # Run time (hh:mm:ss) - 1.5 hours
```
Launch MPI-based executable
```sh
prun ./a.out
```
Submit job for batch execution by sbatch
```sh
[test@master ~]$ sbatch job.mpi
Job status command. by squeue.
[test@master ~]$ squeueJob output.
[test@master ~]$ cat job.XX.out
[prun] Master compute host = c1
[prun] Resource manager = slurm
[prun] Launch cmd = mpirun ./a.out (family=openmpi3)
Hello, world (2 procs total)
--> Process # 0 of 2 is alive. -> c1
--> Process # 1 of 2 is alive. -> c1
Delete job = scancel [job id]
job status = scontrol show job
node status = scontrol show node
```


