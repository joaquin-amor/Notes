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
