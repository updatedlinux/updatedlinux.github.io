# [Creacion de Mirror NPM con Verdaccio + Proxy Reverso](https://updatedlinux.github.io)

 * Objetivo: Creacion de Servidor de Replicas de NPM con Verdaccio para Cache Local 

 * Ambiente: Servidores Fisicos y Virtuales en AWS, VMWare y OpenStack

 * Servidor: 
 Linux RHEL 7.0 Full Updated
 4 VCPU
 30GB Disco S.O (/dev/sda)
 100 GB Disco Data NPM (/dev/sdb)
 1 Interfaz de Red

 ## Let's start step by step

Iniciamos la entonacion a nivel de disco:
```bash
pvcreate /dev/sdb
vgcreate volgroup_npm /dev/sdb
vgscan
pvscan
lvcreate -L99G -n volumen_npm01 volgroup_npm
lvscan
mkfs.xfs -f -L npmdata /dev/volgroup_npm/volumen_npm01
mkdir -p /npmdata
cp /etc/fstab /etc/fstab.BAK.NO-ERASE
echo "/dev/mapper/volgroup_npm-volumen_npm01  /npmdata        xfs     defaults        0 0@" >> /etc/fstab
mount /npmdata
```
## Instalamos aplicaciones:

Para esta receta, utilizaremos los mirrors locales de nuestro DC para instalar: nginx y nodejs. 
```bash
cd /etc/yum.repos.d
wget http://172.16.88.210/mrepo/redhat-7-x86_64/yum.repos.d/nodejs10.repo
wget http://172.16.88.210/mrepo/redhat-7-x86_64/yum.repos.d/nginx.repo
```
Es importante que, si no estas utilizando tus mirrors locales bien sea por no tener los paquetes, o simplemente no tener replicas, recomiendo busques en internet maneras de instalar Nginx 1.15.7 y NodeJS 10 LTS, formas hay, te dejo esto de referencia ya que no tiene que ver con este recipe.

```bash
Nginx: https://www.nginx.com/resources/wiki/start/topics/tutorials/install/
NodeJS: https://computingforgeeks.com/installing-node-js-10-lts-on-centos-7-fedora-29-fedora-28/
```
## Continuamos:

```bash
yum clean all
yum -y update

yum install nginx nodejs

npm install pm2 -g
npm install verdaccio -g

useradd dangerous
usermod -d /npmdata dangerous -m

su - dangerous

```
Con este usuario, ejecutamos una vez Verdaccio para que cree la configuración por defecto:
```bash
verdaccio
```

Mostrará un mensaje de este tipo:

```bash
warn --- config file  - /npmdata/.config/verdaccio/config.yaml
warn --- http address - http://localhost:4873/ - verdaccio/2.7.4
```
Pulsamos Ctrl + C

Ahora debemos editar el config.yaml (/npmdata/.config/verdaccio/config.yaml)

En la ultima linea agregamos:

```bash
listen:
 - 0.0.0.0:4873
url_prefix: http://npm-mirror.wocex.dom/
```
Guardamos y cerramos:

Nota: Importante que tomes en cuenta, el nombre npm-mirror.wocex.dom no es mas que un FQDN que deberas adaptar a tu infraestructura. 

## Seguimos:

Iniciamos la entonacion de Nginx

```bash
vi /etc/nginx/conf.d/verdaccio.conf

server {
  listen 80;
  listen [::]:80;

  server_name npm-mirror.wocex.dom;

  location / {
      proxy_pass http://127.0.0.1:4873/;
  }
}
```

Guardamos y cerramos

```bash
systemctl restart nginx
setsebool httpd_can_network_connect 1 -P
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=4873/tcp --permanent

pm2 start `which verdaccio`
pm2 save
pm2 startup
```

En este punto podremos ingresar al url http://npm-mirror.wocex.dom 

Si necesitas que un cliente use los servicios de Verdaccio como Proxy NPM para manejo de cache, desde los clientes, ejecutar:

```bash
npm config set registry http://npm-mirror.wocex.dom
```

Para esta guia no se habilito logueo a nivel de NPM, ya que la logica de uso, es que permita que cualquier servidor dentro de la Infraestructura, pueda obtener los paquetes NPM desde este Mirror Centralizado, y no desde internet, para asi disminuir consumo de ancho de banda. 

Esto es Todo. -

Facil? comenta en mi linkedIn


- **By Jonathan Alexander Melendez Duran**
- **Caracas, Venezuela**
- **soyjonnymelendez AT gmail DOT com**
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)**