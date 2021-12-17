



# OpenHPC en Rocky Linux

Receta de instalación basada en:

https://github.com/openhpc/ohpc/wiki/2.X

-----------------------------------------------------------------------------------
## Etiqueta de acceso a IP y hostname del master

El nombre del host (SMS) debe de poder ser resuelto localmente. 

Dependidendo en como se haya instalado el OS, debe habería existir una etiqueta en /etc/hosts. En caso contrario, aplicar lo siguiente para poder identificar al host:

```sh
echo ${sms_ip} ${sms_name} >> /etc/hosts
echo 192.168.122.1 rockygab2.edu.pe >> /etc/hosts
```
Obtener los nombres y dirección ip del master con:

```sh
hostnamectl
hostname -I
```

## Descactivar SELinux debido a problemas de compatibilidad, incluso en modo permisivo.

Revisar el estado de SELinux:

```sh
getenforce
sestatus
```
Desahibilitar. Abrir el archivo en /etc/selinux/config  y cambiar la opción de SELINUX a "disabled". En:
```sh
vi /etc/selinux/config...
```
## Deshabilitar firewall

Revisar el estado del firewall:
```sh
systemctl status firewalld 
```
Detener:
```sh
systemctl stop firewalld
```
Desactivar:
```sh
systemctl disable firewalld
```
Revisar si está desactivado:
```sh
systemctl status firewalld 
```
Opcionalmente, usar:
```sh
systemctl disable firewalld
systemctl stop firewalld
```


## Instalar componentes de OpenHPC

### Habilitar repositorio de OpenHPC para uso local

OpenHPC provides an ohpc-release package that includes GPG keys for package signing and enabling the
repository.
```sh
yum install http://repos.openhpc.community/OpenHPC/2/EL_8/x86_64/ohpc-release-2-1.el8.x86_64.rpm
```
For Rocky 8.5, the requirements are to
have access to the BaseOS, Appstream, Extras, PowerTools, and EPEL repositories for which mirrors are
freely available online:
• Rocky-8 (e.g. http://download.rockylinux.org/pub/rocky/8/ )
• EPEL 8 (e.g. http://download.fedoraproject.org/pub/epel/8/ )

The PowerTools repository is typically disabled in a standard install, but
can be enabled from EPEL as follows:
```sh
yum install dnf-plugins-core
yum config-manager --set-enabled powertools
```


## Add provisioning services on master node

This repository provides a number of aliases that group logical components together in order to help aid in this process.
To add support for provisioning services, the following commands illustrate addition of a common base package followed by the Warewulf provisioning system.
```sh
yum -y install ohpc-base
yum -y install ohpc-warewulf
```

Habilitar booteo por PXE networking



## NTP service: 

HPC systems rely on synchronized clocks throughout the system and the NTP protocol can be used to facilitate this synchronization. To enable NTP services on the SMS host with a specific server ${ntp server}, and allow this server to serve as a local time server for the cluster, issue the following:

Activar y configurar servicio NTP:
```sh
sudo dnf update -y
timedatectl list-timezones
sudo timedatectl set-timezone America/Lima
```
Confirmar timezone:
```sh
timedatectl
ls -l /etc/localtime
ls -l /usr/share/zoneinfo 
```
Instalar Chrony:
```sh
sudo dnf install chrony -y
```
Inicializar Chrony:
```sh
sudo systemctl start chronyd
sudo systemctl enable chronyd
systemctl enable chronyd.service
```
Verificar estado:
```sh
sudo systemctl status chronyd
```
Editar el archivo de configuación para acceder a un servidor NTP para la sincronización:
```sh
sudo vi /etc/chrony.conf
```
Agregar debajo de "pool 2.pool.ntp.org iburst":
```sh
server 0.pool.ntp.org
```
Además, en "# Allow NTP client access from local network",  permitir acceso de todos los servidores de la red local:
```sh
allow all
```
Guardar el archivo.

Opcionalmente modificar solo con:
```sh
echo "server ${ntp_server}" >> /etc/chrony.conf
echo "allow all" >> /etc/chrony.conf
```

Finalmente,
```sh
sudo timedatectl set-ntp true
sudo systemctl restart chronyd ; watch chronyc tracking
```


## Add resource management services on master node

OpenHPC provides multiple options for distributed resource management. The following command adds the Slurm workload manager server components to the chosen master host.

Instalar el servidor de slurm
```sh
yum -y install ohpc-slurm-server
```
Aplicar la configuración de slurm brindada por ohpc
```sh
cp /etc/slurm/slurm.conf.ohpc /etc/slurm/slurm.conf
```
Identificar el hostname o master host del resource manager

Ver sms_hostname con hostnamectl, "static hostname" (Elegido en la instalación de Centos localhost name):
rockygab2.edu.pe
```sh
perl -pi -e "s/ControlMachine=\S+/ControlMachine=${sms_name}/" /etc/slurm/slurm.conf
perl -pi -e "s/ControlMachine=\S+/ControlMachine=rockygab2.edu.pe/" /etc/slurm/slurm.conf
```
Modificar el archivo de configuración de Slurm
```sh
sudo vi /etc/slurm/slurm.conf
```
Al final, cambiar número de sockets, cores y threads


## Complete basic Warewulf setup for master node

Update several configuration files in order to allow Warewulf to work with Rocky 8.5 and to support local provisioning using a second private interface

Revisar el nombre de sms_eth_internal en configuraciones y con:
```sh
ip addr show
```
eno1

# Configure Warewulf provisioning to use desired internal interface
```sh
perl -pi -e "s/device = eth1/device = ${sms_eth_internal}/" /etc/warewulf/provision.conf
```
```sh
perl -pi -e "s/device = eth1/device = eno1/" /etc/warewulf/provision.conf
```
# Enable internal interface for provisioning

Usar ifconfig y hostname -I para las variables
```sh
ip link set dev ${sms_eth_internal} up
ip link set dev eno1 up
ip address add ${sms_ip}/${internal_netmask} broadcast + dev ${sms_eth_internal}
ip address add 192.168.122.1/255.255.255.0 broadcast + dev eno1
```

Restart/enable relevant services to support provisioning
```sh
systemctl enable httpd.service
systemctl restart httpd
systemctl enable dhcpd.service
systemctl enable tftp.socket
systemctl start tftp.socket
```

## Define compute image for provisioning one or more compute nodes

With the provisioning services enabled, the next step is to define and customize a system image that can subsequently be used to provision one or more compute nodes.

Build initial BOS image

The OpenHPC build of Warewulf includes specific enhancements enabling support for Rocky 8.5. The following steps illustrate the process to build a minimal, default image for use with Warewulf. We begin by defining a directory structure on the master host that will represent the root filesystem of the compute node. The default location for this example is in /opt/ohpc/admin/images/rocky8.5.

Define chroot location
```sh
export CHROOT=/opt/ohpc/admin/images/rocky8.5
```
Build initial chroot image
```sh
wwmkchroot -v rocky-8 $CHROOT
```
Enable OpenHPC and EPEL repos inside chroot
```sh
dnf -y --installroot $CHROOT install epel-release
cp -p /etc/yum.repos.d/OpenHPC*.repo $CHROOT/etc/yum.repos.d
```
Add OpenHPC components
The wwmkchroot process used in the previous step is designed to provide a minimal Rocky 8.5 configuration. Next, we add additional components to include resource management client services, NTP support, and other additional packages to support the default OpenHPC environment. This process augments the chroot-based install performed by wwmkchroot to modify the base provisioning image and will access the BOS and OpenHPC repositories to resolve package install requests. We begin by installing a few common base
packages.

Install compute node base meta-package
```sh
yum -y --installroot=$CHROOT install ohpc-base-compute
```
To access the remote repositories by hostname (and not IP addresses), the chroot environment needs to be updated to enable DNS resolution. Assuming that the master host has a working DNS configuration in place, the chroot environment can be updated with a copy of the configuration as follows:
```sh
cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf
```
Now, we can include additional required components to the compute instance including resource manager client, NTP, and development environment modules support.

Copy credential files into $CHROOT to ensure consistent uid/gids for slurm/munge at install. Note that these will be synchronized with future updates via the provisioning system.
```sh
cp /etc/passwd /etc/group $CHROOT/etc
```
Add Slurm client support meta-package and enable munge
```sh
yum -y --installroot=$CHROOT install ohpc-slurm-client
chroot $CHROOT systemctl enable munge
```
Register Slurm server with computes (using "configless" option)
```sh
echo SLURMD_OPTIONS="--conf-server ${sms_ip}" > $CHROOT/etc/sysconfig/slurmd
```
Add Network Time Protocol (NTP) support
```sh
yum -y --installroot=$CHROOT install chrony
```
Identify master host as local NTP server
```sh
echo "server ${sms_ip}" >> $CHROOT/etc/chrony.conf
```
Add kernel drivers (matching kernel version on SMS node)
```sh
yum -y --installroot=$CHROOT install kernel-`uname -r`
```
Include modules user environment
```sh
yum -y --installroot=$CHROOT install lmod-ohpc
```

Revisar pagina 13.

Comandos finales: 
```sh
echo "192.168.122.1:/home /home nfs nfsvers=3,nodev,nosuid 0 0" >> $CHROOT/etc/fstab

echo "192.168.122.1:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0" >> $CHROOT/etc/fstab

wwsh -y file set ifcfg-ib0.ww --path=/etc/sysconfig/network-scripts/ifcfg-ib0
```
Veriables: 
```sh
${eth_provision}
eth_provision=$sms_eth_internal # Provisioning interface for computes
${c_name[i]}
c_name=( n001 n002 n003 n004 n005 n006 n007 n008 n009 n010 ) # Host names for computes
${c_ip[i]}
c_ip=( 172.27.34.10 172.27.34.11 172.27.34.12 172.27.34.13 172.27.34.14 172.27.34.15 172.27.34.16 172.27.34.17 172.27.34.18 172.27.34.19 ) 
${c_mac[i]
${kargs}
${compute_regex}
compute_regex=n* # Regex matching all compute node names (e.g. ?c*?)
${c_ipoib[$i]}
c_ipoib=( 10.155.5.10 10.155.5.11 10.155.5.12 10.155.5.13 10.155.5.14 10.155.5.15 10.155.5.16 10.155.5.17 10.155.5.18 10.155.5.19 ) # # Node 
${ipoib_netmask}
ipoib_netmask=255.255.255.0 # IPoIB netmask
${internal_netmask
internal_netmask=255.255.255.0 # Subnet netmask for internal network
```
Para este caso: 
```sh
eth_provision=(eth0)
c_name1=(c1)
c_ip1=(172.27.34.10)
c_mac1=(E0-69-95-C9-80-C9)
compute_regex=(c*)
```


Set provisioning interface as the default networking device
```sh
echo "GATEWAYDEV=eth0" > /tmp/network.$$
wwsh -y file import /tmp/network.$$ --name network
wwsh -y file set network --path /etc/sysconfig/network --mode=0644 --uid=0
```
Add nodes to Warewulf data store
```sh
wwsh -y node new c1 --ipaddr=172.27.34.10 --hwaddr=E0:69:95:C9:80:C9 -D eth0
```
Define provisioning image for hosts
```sh
wwsh -y provision set "c* --vnfs=rocky8.5 --bootstrap=`uname -r` \
--files=dynamic_hosts,passwd,group,shadow,munge.key,network
```
Restart dhcp / update PXE
```sh
systemctl restart dhcpd
wwsh pxe update
```

Variables:
```sh
${num_computes} = 1
${c_bmc[$i]} = 172.27.33.10 
${bmc_username = admin
${bmc_password} = admin
```
```sh
ipmitool -E -I lanplus -H 172.27.33.10 -U admin -P admin chassis power reset
```
En caso de error de contraseña:
```sh
IPMI_PASSWORD=admin ipmitool -E -I lanplus -H 172.27.33.10 -U root chassis bootdev pxe options=persistent
```
O simplemente, con el PXE inicializado en el nodo:
```sh
IPMI_PASSWORD=admin
```

Solucionar el error:
Error: Unable to establish IPMI v2 / RMCP+ session
En:
https://stackoverflow.com/questions/51948745/error-unable-to-establish-ipmi-v2-rmcp-session


Opiones de clusters OS y Rocks Linux: 
```sh
opinion: Rocks is barely maintained, switch to xcat with or without openhpc
```
Alternativas:

Provisioning: Warewulf, xCAT
Resource management: Slurm, Munge, PBS Professional

QLUSTAR

https://www.reddit.com/user/HPC_Whisperer/comments/

Qlustar / Bright

