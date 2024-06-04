# [Instalacion de SQL Server en HA (2 nodos, activo-pasivo) con modelo FCI via iscsi (targetcli)](https://updatedlinux.github.io)

 * Objetivo: Se desea instalar un Cluster Activo-Pasivo utilizando el modelo FCI de Microsoft, donde se garantiza alta disponibilidad utilizando almacenamiento compartido en iSCSI.

 * Requerimiento:

Para SQL Server
16 vCPU
16 GB de memoria Ram
100 GB en disco base
1 NIC

Para iSCSI Target Server
8 vCPU
16 GB de memoria Ram
100 GB de disco base
800 GB de Disco adicional para almacenamiento via iSCSI. 
1 NIC

## Paso 1: Instalar Targetcli y crear las WWN y LUN en el Target iSCSI Server. 

Importante: para este ejemplo ya tenemos la configuracion de VG y LV Creadas, con 800 GB en /dev/vgdata/lvdata, este paso es previo a cualquier configuracion de aqui en adelante. 

Toma en cuenta que para esta receta, deberemos tener 2 servidores "gemelos" que serviran para SQL Server, ambos deben tener los mismos recursos y ser instalaciones limpias de Redhat Enterprise Linux 8.8 (Funciona para Centos 8 y derivados). Cada servidor tendra como hostname: node1 y node2 respectivamente, si quieres nombres diferentes, deberas cambiar las configuraciones a continuacion en targetcli, corosync y pacemaker. 

En el iSCSI Target Server ejecutar:

```bash
yum -y install targetcli
```
entrar en la consola de iscsi con el comando: targetcli

```bash
cd /backstores/block
create dev=/dev/vgdata/lvdata name=SharedDisk
cd /iscsi

create wwn=iqn.2020-08.com.contoso:servers
cd iqn.2020-08.com.contoso:servers/tpg1/acls 

create wwn=iqn.2020-08.com.contoso:node1
create wwn=iqn.2020-08.com.contoso:node2

cd /iscsi/iqn.2020-08.com.contoso:servers/tpg1/luns
create /backstores/block/SharedDisk

 cd /
 saveconfig
 exit
```

## Paso 2: Configuracion de initiator en el nodo1 de SQL Server 

Para este paso, debemos tener los 2 Servidores SQL Server listos y limpios con Rhel 8.8 Instalado, vamos a seleccionar el nodo 1 para realizar los siguientes pasos, a menos que indique lo contrario, por ahora SOLAMENTE tocaremos el nodo 1. 

Para efectos practicos, los hostname de cada servidor seran node1 y node2 respectivamente, si tu hostname es diferente, deberas cambiarlo en los pasos siguientes. 

En el nodo 1 ejecutar:

```bash
yum install iscsi-initiator-utils -y
```

Edita el archivo: /etc/iscsi/initiatorname.iscsi y reemplaza por la siguiente configuracion: 

InitiatorName=iqn.2020-08.com.contoso:node1

Guarda cambios y valida que tengas configuracion con las siguientes comanderias. 

```bash
iscsiadm -m discovery -t st -p <ip del iscsi target server>
```

Debes recibir una respuesta como esta:

[root@node1 ~]# iscsiadm -m discovery -t st -p x.x.x.x
x.x.x.x:3260,1 iqn.2020-08.com.contoso:servers

Ahora debemos autenticar contra el iscsi target: 

```bash
iscsiadm --mode node --targetname iqn.2020-08.com.contoso:servers  --login
```
Debemos recibir una respuesta como esta:

[root@node1 ~]# iscsiadm --mode node --targetname iqn.2020-08.com.contoso:servers  --login
Logging in to [iface: default, target: iqn.2020-08.com.contoso:servers, portal: x.x.x.x,3260]
Login to [iface: default, target: iqn.2020-08.com.contoso:servers, portal: x.x.x.x,3260] successful.

Nota: Si esto falla, valida que via firewalld estes permisando el puerto 3260 desde el iSCSI Target Server. No es necesario aplicar reglas de selinux. 

En este punto ya el disco iSCSI esta presentado en el nodo1 de tu servidor SQL Server, lo puedes validar ejecutando el comando lsblk

```bash
[root@node1 ~]# lsblk 
NAME                   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                      8:0    0  100G  0 disk 
├─sda1                   8:1    0    1G  0 part /boot/efi
├─sda2                   8:2    0    1G  0 part /boot
└─sda3                   8:3    0   92G  0 part 
  ├─rhel-root          253:0    0   10G  0 lvm  /
  ├─rhel-swap          253:1    0    8G  0 lvm  [SWAP]
  ├─rhel-opt           253:2    0    8G  0 lvm  /opt
  ├─rhel-var_log       253:3    0   10G  0 lvm  /var/log
  ├─rhel-home          253:4    0   10G  0 lvm  /home
  ├─rhel-var           253:5    0   15G  0 lvm  /var
  ├─rhel-tmp           253:6    0    8G  0 lvm  /tmp
  ├─rhel-var_log_audit 253:7    0   15G  0 lvm  /var/log/audit
  └─rhel-var_tmp       253:8    0    8G  0 lvm  /var/tmp
sdb                      8:16   0  800G  0 disk 
sr0                     11:0    1 1024M  0 rom  
[root@node1 ~]# 
```

Como vez, tenemos los 800 GB presentados, ahora debemos crear los respectivos PV y VG dentro del nodo1 (Recuerda, SOLAMENTE nodo1)

Ejecutamos:

```bash
pvcreate    /dev/sdb 
vgcreate FCIDataVG1     /dev/sdb
lvcreate -l 100%FREE -n FCIDataLV1 FCIDataVG1
```

Verifica que todo este en orden ejecutando lsblk, deberias ver algo como esto:

```bash
[root@node1 ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0  100G  0 disk 
├─sda1                    8:1    0    1G  0 part /boot/efi
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   92G  0 part 
  ├─rhel-root           253:0    0   10G  0 lvm  /
  ├─rhel-swap           253:1    0    8G  0 lvm  [SWAP]
  ├─rhel-opt            253:2    0    8G  0 lvm  /opt
  ├─rhel-var_log        253:3    0   10G  0 lvm  /var/log
  ├─rhel-home           253:4    0   10G  0 lvm  /home
  ├─rhel-var            253:5    0   15G  0 lvm  /var
  ├─rhel-tmp            253:6    0    8G  0 lvm  /tmp
  ├─rhel-var_log_audit  253:7    0   15G  0 lvm  /var/log/audit
  └─rhel-var_tmp        253:8    0    8G  0 lvm  /var/tmp
sdb                       8:16   0  800G  0 disk 
└─FCIDataVG1-FCIDataLV1 253:9    0  800G  0 lvm  
sr0                      11:0    1 1024M  0 rom 
```
Ahora vamos a darle formato a ese filesystem 

```bash
mkfs.xfs /dev/FCIDataVG1/FCIDataLV1
```

## Paso 3: Configuracion en el nodo2 de SQL Server

Ejecutar en el node2:

```bash
yum install iscsi-initiator-utils -y
```

Editamos el archivo /etc/iscsi/initiatorname.iscsi y reemplazamos el contenido por el siguiente:

InitiatorName=iqn.2020-08.com.contoso:node2

Guardamos cambios y certificamos la conexion:

```bash
iscsiadm -m discovery -t st -p <ip del iscsi target server>
```

la respuesta debe ser la misma que recibimos en la prueba del nodo1

Hacemos el login en el iSCSI Target Server 

```bash
iscsiadm --mode node --targetname iqn.2020-08.com.contoso:servers  --login
```

NO haremos mas pasos de momento en el nodo 2, puedes ejecutar lsblk para certificar que el disco este presentado: 

```bash
[root@node2 ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0  100G  0 disk 
├─sda1                    8:1    0    1G  0 part /boot/efi
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   92G  0 part 
  ├─rhel-root           253:0    0   10G  0 lvm  /
  ├─rhel-swap           253:1    0    8G  0 lvm  [SWAP]
  ├─rhel-opt            253:2    0    8G  0 lvm  /opt
  ├─rhel-var_log        253:3    0   10G  0 lvm  /var/log
  ├─rhel-home           253:4    0   10G  0 lvm  /home
  ├─rhel-var            253:5    0   15G  0 lvm  /var
  ├─rhel-tmp            253:6    0    8G  0 lvm  /tmp
  ├─rhel-var_log_audit  253:7    0   15G  0 lvm  /var/log/audit
  └─rhel-var_tmp        253:8    0    8G  0 lvm  /var/tmp
sdb                       8:16   0  800G  0 disk 
└─FCIDataVG1-FCIDataLV1 253:9    0  800G  0 lvm  
sr0                      11:0    1 1024M  0 rom
```

Sin embargo, quien lo va a "operar" es el nodo 1 a traves de SQL Server, lo cual haremos en los pasos siguientes


## Paso 4: Instalacion de SQL Server en ambos nodos. 

Vamos a instalar SQL Server en ambos nodos (ejecuta estos pasos en node 1 y nodo 2, lo puedes hacer en simultaneo)

Ejecutar:

```bash
curl --insecure -o /etc/yum.repos.d/mssql-server.repo https://packages.microsoft.com/config/rhel/8/mssql-server-2022.repo
echo "sslverify=false" >> /etc/yum.repos.d/mssql-server.repo

yum install -y mssql-server
/opt/mssql/bin/mssql-conf setup
```

Seleccionar la opcion 8, colocamos el serial enterprise y colocamos NO en la opcion de Software Assurance, aceptamos terminos y condiciones con un Yes, y setearemos una clave para el usuario SA, pon la que quieras mientras tenga 8 caracteres alfanumericos. 

Luego validaremos el estatus del servicio: 

```bash
systemctl status mssql-server
```

Permitimos las conexiones remotas al puerto 1433. 

```bash
firewall-cmd --zone=public --add-port=1433/tcp --permanent
firewall-cmd --reload
```

Instalamos algunas tools adicionales:

```bash
yum install unixODBC-devel.x86_64
```

En este punto, tienes 2 SQL Servers en Standalone, en los siguientes pasos realizaremos configuraciones adicionales para que se comporten como un cluster activo-pasivo dandole gobernanza a Corosync y Pacemarker.

## Paso 5: Configuracion del Cluster SQL Server

Los siguientes pasos los realizaremos UNICAMENTE en el nodo 1 a menos que se indique lo contrario. 

Ejecutaremos los siguientes querys desde el manejador de base de datos de SQL Server, debemos conectarnos al nodo 1. Puedes usar DBeaver, Azure Studio etc...

~~~~sql
CREATE LOGIN [sqlpackmaker] with PASSWORD= N'YourStrongP@ssword1'
ALTER SERVER ROLE [sysadmin] ADD MEMBER [sqlpackmaker]
~~~~

Esto crea un usuario para pacemaker dandole rol de admin. 

Ahora cambiaremos el default server name por uno nuevo que sea flotante entre ambos nodos del cluster SQL Server, De nuevo, estos pasos solo se ejecutan en el nodo 1

~~~~sql
exec sp_dropserver node1
exec sp_addserver 'sqlvirtualname1','local'
~~~~

Completado esto, en el nodo 1 vamos a reiniciar el servicio de SQL Server

```bash
systemctl stop mssql-server
systemctl restart  mssql-server
```

Espera 1 minuto (o menos) y luego desde tu manejador de SQL Server ejecuta el siguiente query:

~~~~sql
select @@servername, SERVERPROPERTY('ComputernamephysicalNetBIOS')
~~~~

Deberias ver 2 columnas:

sqlvirtualname1	node1

Luego de esto, vamos a copiar el machinekey y secrets del nodo 1 al 2 para que su configuracion sea "gemelo", tecnicamente esta configuracion de HA cada nodo es standalone del otro, solo que Corosync y Pacemaker hacen el trabajo respectivo de brindar Alta Disponibilidad en caso de caida de un nodo. 

Bajamos los servicios en AMBOS nodos:

```bash
systemctl stop mssql-server
```

Copia el siguiente archivo: /var/opt/mssql/secrets/machine-key del node1 al node2, formas hay muchas, desde SCP, hasta pendrive, elige la que tu quieras, pero recuerda, el dueño del archivo debe ser mssql.

Ahora, de nuevo, en el nodo1 (NODO1!!) ejecutar. 

```bash
mkdir /var/opt/mssql/tempdir
cp /var/opt/mssql/data/*  /var/opt/mssql/tempdir/
rm -f /var/opt/mssql/data/*

mount /dev/FCIDataVG1/FCIDataLV1 /var/opt/mssql/data
chown mssql:mssql /var/opt/mssql/data
chgrp mssql /var/opt/mssql/data
cp /var/opt/mssql/tempdir/* /var/opt/mssql/data
chown mssql:mssql /var/opt/mssql/data/*
```

Ahora vamos a asegurarnos de montar el Disco iSCSI en el fstab del servidor, para esto generaremos un blkid

```bash
blkid /dev/FCIDataVG1/FCIDataLV1
```
Tomaremos el valor UUID, para este ejemplo es:

UUID="cb22e46f-6944-405f-b90a-2bc4afac5d19"

y lo colocaremos en el fstab del servidor de AMBOS nodos (node1 y node2), importante: los pasos anteriores SOLAMENTE debistes ejecutarlos en el nodo 1, el UUID siempre sera el mismo para ambos nodos ya que es un disco compartido via iscsi, es decir, esta modificacion del Fstab debe hacerse en ambos servidores por unica vez. 

Agregar al fstab: 

```bash
UUID="cb22e46f-6944-405f-b90a-2bc4afac5d19"     TYPE="xfs"      /dev/FCIDataVG1/FCIDataLV1      _netdev 0       0
```

Guarda cambios y ejecuta: 

```bash
systemctl daemon-reload
```


## Paso 6: Configuracion del Cluster con Pacemaker y Corosync

En la tabla de host de ambos nodos (nodo 1 y nodo 2) colocar la ip de ambos nodos y sus nombres

/etc/hosts
```bash
<ip del nodo1> node1
<ip del nodo2> node2
```
Guarda cambios.

Ahora realizaremos las configuraciones respectivas de sesion de Pacemaker, como se comento en lineas anteriores, tecnicamente son 2 instancias Standalone, solo que pacemaker tiene usuario en la base de datos y tendra gobernanza sobre el disco iscsi presentado para, en caso de caida, por medio de su healthcheck definir que nodo sera el preferente, con esto tendremos un servicio HA activo-pasivo. 


Los siguientes pasos los realizaremos en ambos nodos. 

```bash
touch /var/opt/mssql/secrets/passwd
echo 'sqlpackmaker' >> /var/opt/mssql/secrets/passwd
echo 'YourStrongP@ssword1' >> /var/opt/mssql/secrets/passwd
chown root:root /var/opt/mssql/secrets/passwd
chmod 600 /var/opt/mssql/secrets/passwd
```

Activa el repositorio rhel-8-for-x86_64-rhel-8-for-x86_64-highavailability-rpms en ambos nodos. 

Instala: 
```bash
yum install pacemaker pcs fence-agents-all resource-agents
```

Ejecuta:

```bash
passwd hacluster
```

Utiliza la misma contraseña en ambos nodos 

Ejecuta en ambos nodos: 

```bash
systemctl enable pcsd
systemctl start pcsd
systemctl enable pacemaker
```

SOLAMENTE EN EL NODO1 ejecuta:

pcs host auth  node1 node2 

el usuario es hacluster y la clave la colocada anteriormente.

```bash
pcs cluster setup  sqlcluster  node1 node2
pcs cluster start --all
pcs cluster enable --all
pcs property set stonith-enabled=false
```

en ambos nodos (node 1 y node 2) instala:

```bash
yum install mssql-server-ha
```

Ahora crearmos los grupos de recursos de disco, SOLAMENTE EN EL NODO 1:

```bash
pcs resource create iSCSIDisk1 Filesystem device="/dev/FCIDataVG1/FCIDataLV1" directory="/var/opt/mssql/data" fstype="xfs" --group fci
```

Crearemos el grupo de recursos IP, con esto tendremos una IP Flotante (VIP de Servicio) que flotara entre ambos nodos (nodo 1 y nodo 2), en caso de failover, la ip se movera del nodo prefente, al nodo que quede activo. 

```bash
pcs resource create vip2 ocf:heartbeat:IPaddr2 ip=x.x.x.x nic=interfacename  cidr_netmask=24 --group fci
```

Nota: Es necesario que cambies el nombre de la interfaz y la ip, la ip debe ser una que este libre dentro de tu LAN, y el nombre de la interfaz, la que le da ip al node1, puede ser eth0, como ens192, o lo que sea. 

Por ultimo, crearemos el recurso FCI para SQL Server y con esto culmina la receta: 

```bash
pcs resource create sqlvirtualname1 ocf:mssql:fci   --group fci
```

## Pruebas de Failover

El servicio de SQL Server por defecto quedara arriba en el nodo 1, el cual sera el preferente, puedes hacer pruebas de failover ejecutando:

```bash
pcs resource move sqlvirtualname1 node2
```

Luego desde tu manejador de Base de Datos ejecuta el query

~~~~sql
select @@servername, SERVERPROPERTY('ComputernamephysicalNetBIOS')
~~~~

Nota: en este punto debiste cambiar la ip de conexion al manejador de base de datos a la VIP Flotante que definiste, obviamente acabas de mover los recursos al nodo 2, por lo cual el nodo 1 dejara de responder, al ejecutar el query, veras algo como esto en 2 columnas:

 sqlvirtualname1	node2

 Devuelve los recursos al nodo preferente (para este caso de prueba de failover, ejecuta desde el nodo 2): 

```bash
 pcs resource move sqlvirtualname1 node1
```

Si ejecutas pcs resource, veras el estado del cluster y en donde quedo levantado: 

```bash
[root@node1 ~]# pcs resource
  * Resource Group: fci:
    * iSCSIDisk1        (ocf::heartbeat:Filesystem):     Started node1
    * vip2      (ocf::heartbeat:IPaddr2):        Started node1
    * sqlvirtualname1   (ocf::mssql:fci):        Started node1
```

## Recomendaciones Finales: 

Como estas trabajando con una IP Flotante que tendra vida en cualquiera de los dos nodos que conforman el despliegue, siempre se recomienda que tus aplicaciones que hagan consumo a la Base de Datos se conecten directamente a dicha IP y no a las ip inviduales de los nodos, como tip usa records DNS para que todo sea transparente. 

Esto es todo. -

Creditos al autor original de la receta: https://sqlserverbang.blogspot.com/2020/08/tutorial-create-sql-fci-on-rhel.html

Facil? Comenta en mi Linkedin


- **By Jonathan Alexander Melendez Duran**
- **Caracas, Venezuela**
- **soyjonnymelendez AT gmail DOT com**
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)**

