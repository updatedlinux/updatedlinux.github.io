# [Manual de Instalacion de DashCore Ultima Version](https://updatedlinux.github.io)

 * Objetivo: Instalacion Limpia de DashCore V0.13.1 en entornos Linux

 * Scope: Fedora 29

 ## Let's start step by step

Dash Core 0.13.0 Se lanzo el 14 de enero del 2019, esta ultima version no es compatible con algunas librerias de sistemas EL7 (CentOS / RHEL) ni Fedora 27

## Consideraciones del S.O:

Sistema Fedora 29 x64
4 VCPU
16 GB RAM
Enlace de 5 GBits
Particion / de 60 GB
Particion /dashdata de 300 GB

## Paso 1: Entonacion del S.O

```bash
dnf update -y && dnf clean all
dnf install wget


useradd dash
usermod -d /dashdata dash -m
```

Temporalmente le daremos sudoers amplio a este usuario, puedes hacerlo desde visudo o creando un archivo en /etc/sudoers.d/dash con la linea:
```bash
dash        ALL=(ALL)       NOPASSWD: ALL
```
```bash
reboot
```

Ejecutamos: 

```bash
su - dash
wget https://github.com/dashpay/dash/releases/download/v0.13.1.0/dashcore-0.13.1.0-aarch64-linux-gnu.tar.gz
mkdir ~/.dashcore
tar xfv dashcore-0.13.1.0-aarch64-linux-gnu.tar.gz


sudo install -m 0755 -o dash -g dash -t /usr/local/bin ~/dashcore-0.13.1/bin/dashd
sudo install -m 0755 -o dash -g dash -t /usr/local/bin ~/dashcore-0.13.1/bin/dash-cli
```



## Paso 2: Edicion de dash.conf

vi ~/.dashcore/dash.conf
```bash
#----
rpcuser=XXXXXXXXXXXXX
rpcpassword=XXXXXXXXXXXXXXXXXXXXXXXXXXXX
rpcallowip=127.0.0.1
#----
listen=1
server=1
daemon=1
maxconnections=64
#----
masternode=0
masternodeprivkey=XXXXXXXXXXXXXXXXXXXXXXX
externalip=XXX.XXX.XXX.XXX
#----

```
Guardar cambios.
Nota: Cambiar solamente los datos rpcuser y rpcpassword bajo demanda, si deseas testnet, solo debes anadir: testnet=1 antes de rpcuser

Ejecutar

```bash
dashd
```

En este punto encontraras un mensaje tal como:
```bash
Dash Core server starting.
```
Esto quiere decir que la instalacion fue hecha satisfactoriamente.

Ejecutando ss -ltn conseguimos:

```bash
[dashu@ip-172-50-30-37 ~]$ ss -ltn
State                Recv-Q               Send-Q                             Local Address:Port                               Peer Address:Port
LISTEN               0                    128                                      0.0.0.0:9999                                    0.0.0.0:*
LISTEN               0                    128                                      0.0.0.0:22                                      0.0.0.0:*
LISTEN               0                    128                                            *:9998                                          *:*
LISTEN               0                    128                                         [::]:9999                                       [::]:*
LISTEN               0                    128                                         [::]:22                                         [::]:*
```

el puerto 9998 es el listening rpc

Para certificar el correcto funcionamiento del servicio dash, puedes ejecutar:

dash-cli getblockchaininfo 

Obteniendo como respuesta algo como esto:

```bash
[dashu@ip-172-50-30-37 ~]$ dash-cli getblockchaininfo
{
  "chain": "main",
  "blocks": 699728,
  "headers": 1024547,
  "bestblockhash": "00000000000018a2dc327f0d36cd06281314aa08e58d08ecbe7ec8392b195ca1",
  "difficulty": 390818.3908941554,
  "mediantime": 1499508591,
  "verificationprogress": 0.3198670774877285,
  "chainwork": "0000000000000000000000000000000000000000000000043a573eda7b8bc65f",
  "pruned": false,
  "softforks": [
    {
      "id": "bip34",
      "version": 2,
      "reject": {
        "status": true
      }
    },
    {
      "id": "bip66",
      "version": 3,
      "reject": {
        "status": true
      }
    },
    {
      "id": "bip65",
      "version": 4,
      "reject": {
        "status": true
      }
    }
  ],
  "bip9_softforks": {
    "csv": {
      "status": "active",
      "startTime": 1486252800,
      "timeout": 1517788800,
      "since": 622944
    },
    "dip0001": {
      "status": "defined",
      "startTime": 1508025600,
      "timeout": 1539561600,
      "since": 0
    },
    "dip0003": {
      "status": "defined",
      "startTime": 1546300800,
      "timeout": 1577836800,
      "since": 0
    },
    "bip147": {
      "status": "defined",
      "startTime": 1524477600,
      "timeout": 1556013600,
      "since": 0
    }
  }
}
```

## Paso 3: Firewalld

Importante que a nivel de firewall interno y externo, coloques las reglas necesarias para que solo el que realmente quieres que consuma los servicios RPC, sea el que le llegue al equipo, por ejemplo:
```bash
firewall-cmd --permanent --zone=public --add-rich-rule='
  rule family="ipv4"
  source address="172.50.30.137/32"
  port protocol="tcp" port="9998" accept'
```
Con esto garantizamos que solo el host 172.50.30.137 sea el que consuma los servicios del puerto 9998 del DashServer, Sigo recomendando utilizar soluciones como Tuneles SSH Inversos, esto anade una capa de encriptacion de data, y oculta del aplicativo la IP del nodo. 

Para aplicarlo:

 Nota: Esto aplicaria para Litecoin, pero es perfectamente adaptable a Dash cambiando las variables:

 Si se configura un Tunel SSH Inverso, se debe tomar en cuenta las premisas:

- Al usuario litecoin se le debe configurar la carpeta .ssh en su Home Directory, asignarle la clave id_rsa.pub del usuario que ejecuta la aplicación en el servidor cliente 
- Se debe crear el siguiente Script en el servidor de la aplicación en la ruta /usr/local/bin/, el owner debe ser el usuario que ejecuta la aplicacion, en este caso develop.develop



Script: 
```bash
#!/bin/bash
# Script for SSH tunnel elevation for cryptoactive management
# Created by the Technological Infrastructure Unit
#
#        Jonathan Melendez        - Technology Infrastructure Manager
#
# Date of Creation: January 22, 2019
#
#
# Updates:
#
#Starting Script:
##############################################################
PATH=$PATH:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

REMOTEUSER=tunelhc
REMOTEHOST=15.164.5.23
TUNNEL_REMOTEPORT=19998
TUNNEL_LOCALPORT=19998

amirunnign=`ps -ef |grep $TUNNEL_LOCALPORT|grep -v grep|wc -l`
    case $amirunnign in
    0)
        echo "Tunnel to $REMOTEHOST it's not up, raising please wait ..."
        sleep 3
        /bin/ssh -N $REMOTEUSER@$REMOTEHOST -L $TUNNEL_REMOTEPORT:localhost:$TUNNEL_LOCALPORT &
        sleep 3
        mypid=`ps -ef | grep $TUNNEL_LOCALPORT | grep -v grep | awk '{print $2}'`
        echo "Tunnel to $REMOTEHOST created Successfully at PID: $mypid"
                ;;
        *)
        mypid=`ps -ef | grep $TUNNEL_LOCALPORT | grep -v grep | awk '{print $2}'`
        echo "Tunnel to $REMOTEHOST created Successfully at PID: $mypid"
        ;;
    esac

#END
```

El Script se le debe personalizar las variables iniciales de usuario, ip, port remoto y local segun aplique, ejemplo:
```bash
REMOTEUSER=litecoin
REMOTEHOST=15.164.5.23
TUNNEL_REMOTEPORT=9332
TUNNEL_LOCALPORT=9332
```
Se le dan los permisos de ejecucion pertinentes (chmod +x y chown)

Se crea el siguiente Crontab 
```bash

*/1 * * * *   develop    /usr/local/bin/litecoinssh.sh >> /var/log/tunelssh/litecoin.log
```

Nota: el script lo nombramos como litecoinssh.sh y obviamente generamos la carpeta /var/log/tunelssh con develop como owner. 

Si deseas certificar el tunel, desde el servidor aplicativo ejecuta algo como esto:
```bash
curl  --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "listaccounts", "params": [] }' -H 'content-type: text/plain;' http://lit_user:d2rcGCMEMCwvDAVyzU2gA4Cak7GPxdVA@142.93.204.144:16321/
```
Donde el sintaxis http debe ser igual al rpcuser, rpcpassword e ip/fqnd del servidor dash/ltc/btc (esta solucion aplica para todos), ejemplo:
```bash
curl  --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "listaccounts", "params": [] }' -H 'content-type: text/plain;' http://usuariorpc:passwordrpc@ipdelnodo:puertodelnodo/
```
La respuesta de este curl, debe ser la misma que ejecutando el metodo listaccounts directamente en el nodo. 

Esto es todo. -

Documentacion Oficial: https://docs.dash.org/en/latest/masternodes/setup.html#install-dash-core

Facil? Comenta en mi Linkedin


- **By Jonathan Alexander Melendez Duran**
- **Caracas, Venezuela**
- **soyjonnymelendez AT gmail DOT com**
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)**




