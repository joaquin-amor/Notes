# Pasos para rehacer la configuración MSDP over iSCSI and object storage:

**Requerimiento previo**: Detener los servicios de Netbackup:
```
#netbackup stop
```

1) Desmontar los File System utilizados por el MSDP.
```
#umount /msdp/vol[0-3]
```
2) Parar servicios iscsid y tgtd responsables de exportar los discos.
```
#systemctl stop iscsid
#systemctl status iscsid
#systemctl stop tgtd
#systemctl status tgtd
```
3) Acceder a los file system del almacenamiento de objeto y borrar los ficheros sparse sobre los que se crean los FS iSCSI.
```
#rm -f /ring/fs.7683088460454356516/NBU/data/block[1-4]/*
```
4) Crear de nuevo los ficheros sparse, cada uno en una carpeta diferente dentro del path del almacenamiento de objeto con el fin de evitar problemas de concurrencia.
```
#dd of=/ring/fs.7683088460454356516/NBU/data/block1/vol0.sparse bs=1k seek=32G count=0
#dd of=/ring/fs.7683088460454356516/NBU/data/block2/vol1.sparse bs=1k seek=32G count=0
#dd of=/ring/fs.7683088460454356516/NBU/data/block3/vol2.sparse bs=1k seek=32G count=0
```
5) Configurar en /etc/tgt/targets.conf el nuevo path a los ficheros sparse creados.
```
default-driver iscsi

<target 10.5.132.201:nbu.target02>
    backing-store /ring/fs0/NBU/data/block1/vol0.sparse
    incominguser iscsi 123
    initiator-address 10.5.132.201
</target>

<target 10.5.132.201:nbu.target03>
    backing-store /ring/fs1/NBU/data/block2/vol1.sparse
    incominguser iscsi 123
    initiator-address 10.5.132.201
</target>

<target 10.5.132.201:nbu.target01>
    backing-store /ring/fs2/NBU/data/block4/large.sparse
    incominguser iscsi 123
    initiator-address 10.5.132.201
</target>
```
6) Comprobar que el demonio tgtd exporta correctamente los ficheros
```
#systemctl start tgtd
#tgtadm --mode target --op show
```
7) Comprobar que iscsid ve correctamente los ficheros.
```
#systemctl start iscsid
#iscsiadm --mode discovery -t sendtargets --portal  10.5.132.201
```
8) Realizar el login contra los dispositivos iscsid.
```
#iscsiadm --mode node --targetname 10.5.132.201:nbu.target01 --portal 10.5.132.201 --login
#iscsiadm --mode node --targetname 10.5.132.201:nbu.target02 --portal 10.5.132.201 --login
#iscsiadm --mode node --targetname 10.5.132.201:nbu.target03 --portal 10.5.132.201 --login
```
9) Ejecutar un Fdisk de los discos
```
#partprobe
#fdisk -l
Disk /dev/sdd: 35184.4 GB, 35184372088832 bytes, 68719476736 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4194304 bytes
I/O size (minimum/optimal): 4194304 bytes / 4194304 bytes


Disk /dev/sdf: 35184.4 GB, 35184372088832 bytes, 68719476736 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4194304 bytes
I/O size (minimum/optimal): 4194304 bytes / 4194304 bytes


Disk /dev/sde: 35184.4 GB, 35184372088832 bytes, 68719476736 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4194304 bytes
I/O size (minimum/optimal): 4194304 bytes / 4194304 bytes
```
10) Crear un FS de tipo ext4 o xfs sobre los dispositivos de bloque recien reconocidos.
```
#mkfs.ext4 del dispositivo
#mkfs.ext4 /dev/sdf
```
11) Modificar el uuid de los discos en el fichero /etc/fstab para los rearranques
```
#blkid  (identificamos el UUID de los discos)
#blockdev --report  (comprobamos el tamaño y partición de los discos recien asignados)
#vi /etc/fstab
    UUID=2e940ff1-3765-49e2-9856-9ce9f7f91b3c       /msdp/vol0      ext4 rw,relatime,noquota,_netdev   0 0
    UUID=c9e40ba9-1f7b-439b-9e60-441a000e6aa3       /msdp/vol1      ext4 rw,relatime,noquota,_netdev   0 0
    UUID=ada2d1b3-041c-449c-8c9f-7becc576366a       /msdp/vol2      ext4 rw,relatime,noquota,_netdev   0 0
```
Se añaden el parametro "\_netdev" al fstab de la maquina para que estos dispositivos se monten una vez que el sistema haya arrancado la red.

12) Montar los dispositivos recien formateados
```
#mount -a
Aug 31 11:32:27 netbackup-master-1 kernel: EXT4-fs (sde): mounted filesystem with ordered data mode. Opts: noquota
Aug 31 11:32:27 netbackup-master-1 systemd: Unit msdp-vol2.mount is bound to inactive unit dev-disk-by\x2duuid-0596158a\x2daed7\x2d45d5\x2d914f\x2dee9984e7b6b0.device. Stopping, too.
Aug 31 11:32:27 netbackup-master-1 systemd: Stopped target Remote File Systems.
Aug 31 11:32:27 netbackup-master-1 systemd: Stopping Remote File Systems.
Aug 31 11:32:27 netbackup-master-1 systemd: Unmounting /msdp/vol2...
Aug 31 11:32:27 netbackup-master-1 systemd: Unmounted /msdp/vol2.
```
