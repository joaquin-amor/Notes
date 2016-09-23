# Instalación de clúster de Ceph de 3 nodos:

###Requerimientos

- 3 hosts con sistema operativo RHEL 7.X instalado.
- Servidor NTP (es posible utilizar el admin-node como servidor de NTP).
- Dirección IP de un servidor proxy o acceso a internet.

###TIPS

* Es necesario tener configurada la resolución de nombres bien a través de un DNS o fichero _hosts_ en su defecto.
* Para modificar el mínimo nivel de réplicas en el fichero _ceph.conf_ bajo el campo "[global] - osd crush chooseleaf type = 0" (0 se define para permitir tener copias en el mismo host,1 para requerir un host diferente, etc...)
* Para modificar el número de réplicas, en el fichero _ceph.conf_ bajo el campo "[global] - osd pool default size = 2" (2 para utilizar mínimo 2 OSD, 3 para 3 OSD, etc...)
* Instalación de packstack https://www.rdoproject.org/install/quickstart/ (revisar manual de integración)

####PROCEDIMIENTO DE PRE-INSTALACION

**Nota:** Es necesario disponer de una correcta resolución de nombres tanto para la configuración ssh como para el propio comando de ceph-deploy. Se debe utilizar el resultado de la salida del comando "hostname -s"

1) Se debe generar el mismo usuario en cada uno de los nodos (i.e _ceph-user_). No debe usarse ni el usuario _root_ ni _ceph_

2) Deshabilitar _tty_ en el fichero _/etc/hosts_ 

```
Modificar "requiretty" a "!requiretty"
```
3) Configurar _sudo_ en modo _passwordless_ añadiendo la linea siguiente al fichero sudoers bajo una linea similar ya existente para el usuario _root_

```
add "{username} ALL = (root) NOPASSWD:ALL"
```
4) Configurar la conexión ssh _passwordless_ entre todos los nodos (utilizando los nombres de nodos devueltos por el comando _hostname -s_

* En el _admin node :_
```
ssh-keygen (no passphrase)
ssh-copy-id {username}@node1(& node2, node3, etc)
```
* Añadir el siguiente contenido al fichero ~/.ssh/config file:
```
Host node1
   Hostname node1
   User {username}
Host node2
   Hostname node2
   User {username}
Host node3
   Hostname node3
   User {username}
```

5) Deshabilitar _selinux_

```
sudo setenforce 0
```
O modificar en su defecto el fichero de configuración _/etc/selinux/config_

6) Configuración NTP

* En cada nodo ejecutar:
```
sudo yum install ntp ntpdate ntp-doc
```
* Sincronizar el reloj de cada servidor con el mismo ntp server
```
sudo service ntp stop
sudo ntpdate -s {ntpserver}
sudo service ntp start
```
7) Configuración del firewall para permitir la comunicación a través de los puertos 6789, 6800 y 7300

* En _firewalld_
```
sudo firewall-cmd --zone=public --add-port=6789/tcp --permanent
sudo firewall-cmd --zone=public --add-port=6800/tcp --permanent
sudo firewall-cmd --zone=public --add-port=7300/tcp --permanent
```
* En _iptables_
```
sudo iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} --dport 6789 -j ACCEPT
sudo iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} --dport 6800 -j ACCEPT
sudo iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} --dport 7300 -j ACCEPT
```
Si fuera posible, deshabilitar el firewall por completo.

8) Configuración de los repositorios

* Para RHEL 7, registrar los diferentes hosts con **suscription-manager**, verificar las subscripciones y habilitar los repositorios "Extras" para las dependencias de los paquetes (http://docs.ceph.com/docs/master/start/quick-start-preflight/#rhel-centos):
```
sudo subscription-manager repos --enable=rhel-7-server-extras-rpms
```
* Instalar y habilitar el repositorio EPEL (Extra PAckages for Enterprise Linux). 
```
sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
```
* Añadir el paquete al repositorio. A través de un editor de texto crear el siguiente fichero:
```
sudo vim /etc/yum.repos.d/ceph.repo
```
* Incluir la siguiente información en el fichero de repositorio de ceph. Sustituir `{ceph-release}` con la versión más actual de Ceph `{jewel}` por ejemplo y `{distro}` con la distribución de Linux que corresponda `{el7}` por ejemplo. Salvar el contenido del fichero como `/etc/yum.repos.d/ceph.repo`

```
[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-{ceph-release}/{distro}/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
```

* Actualizar el repositorio e instalar `ceph-deploy`

```
sudo yum update && sudo yum install ceph-deploy
```
#### PROCEDIMIENTO DE INSTALACION

1) Definir qué nodos serán los monitores iniciales. Para una configuración de tres nodos:
```
ceph-deploy new node1 node2 node3
```
2) Instalar el software de ceph en todos los nodos.
```
ceph-deploy install node1 node2 node3
```
3) Crear los monitores initiales.
```
ceph-deploy mon create-initial
```
4) Crear los OSDs.En el siguiente ejemplo se asume que no existe partición de journal en el OSD y que únicamente se asigna un OSD por nodo.
```
ceph-deploy osd create node1:/dev/sdb node2:/dev/sdb node3:/dev/sdb
```

5) Distribuir la keyring de admin a todos los nodos.
```
ceph-deploy admin node1 node2 node3
```