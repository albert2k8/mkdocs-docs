# Guía de instalación de REDIS

## Consideraciones iniciales

> [!WARNING]
> Consideraciones previas
> Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.
> * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
> *La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
> * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
> * La instalación es compatible con SELinux en modo `enforcing`.
> * La instalación es compatible con el uso del servicio `firewalld`.

## Rutas de instalación para módulos AVL

El paso previo a la instalación de Redis es crear el punto de montaje base para instalación de los diferentes módulos que vayan a convivir y el almacenamiento de logs.

Se va a comenzar por crear una nueva partición ya que el disco del sistema operativo dispone de `100GB` de los cuales solo se aprovisionan `56GB`.

```bash
# Creación de la particion sda4
echo -e "n\n4\n\n\nt\n\nE6D6D379-F507-44C2-A23C-238F2A3DF928\nw" | fdisk /dev/sda

# Comprobacíon de que la nueva partición se ha creado
fdisk -l | grep sda
```

Una vez creada la partición, se puede comenzar el proceso para crear los puntos de montaje `/AVL` y `/AVL/logs`

``` bash
# Creación del Physical Volume
pvcreate /dev/sda4
pvs | grep sda4
# Creación del Volume Group
vgcreate vg_avl /dev/sda4 
vgs | grep vg_avl
# Creación del Logical Volume
lvcreate -L 30G -n avl vg_avl
lvcreate -L 10G -n logs vg_avl
lvs | grep vg_avl
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_avl-avl
mkfs.xfs /dev/mapper/vg_avl-logs
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_avl-avl /AVL                    xfs     defaults        0 0" >> /etc/fstab
echo "/dev/mapper/vg_avl-logs /AVL/logs                    xfs     defaults        0 0" >> /etc/fstab
# Inclusión de los cambios realizados en /etc/fstab
systemctl daemon-reload
# Creación del punto de montaje /AVL 
mkdir -p /AVL
mount /dev/mapper/vg_avl-avl /AVL
df -h | grep vg_avl
# Creación del punto de montaje
mkdir -p /AVL/logs
mount /dev/mapper/vg_avl-logs /AVL/logs
df -h | grep vg_avl
```

El siguiente paso, será crear el grupo `avlgroup` y los usuarios `avlmodule` y `avladmin`. El primero será el usuario que levante los servicios, mientras que el segundo, será el usuario encargado de administrar las configuraciones.

``` bash
# Creación del grupo avlgruop
groupadd -g 5000 avlgroup
# Creación del usario avladmin que gestiona los servicios
useradd -u 5000 -d /AVL -c "AVL Admin Account" -g avlgroup -p $(openssl passwd -1 MMNUTXvCIm3@DCpjsrVg) avladmin
# Creación del usario avlmodule que ejecuta los servicios
useradd -u 5001 -d /AVL -c "AVL module runner" -g avlgroup -s /sbin/nologin avlmodule
```

El proceso finaliza creando los directorios necesarios para organizar los diferentes elementos y asignando los propietarios y permisos correspondientes. La ACL permitirá que a medida que se vayan creando ficheros, estos ya tengan los permisos adecuados.

``` bash
# Creación del árbol de directorios para el módulo AVL
mkdir -p /AVL/scripts
mkdir -p /AVL/sw_base
mkdir -p /AVL/module
mkdir -p /AVL/security
mkdir -p /AVL/installers

# Configuración de usuarios, permisos y ACL
chown -R avlmodule:avlgroup /AVL
chmod -R 750 /AVL
chmod -R 750 /AVL/logs
setfacl -PRdm u::rwx,g::rw,o::- /AVL
```

## Configuración del firewall

Dada la importancia que tiene la seguridad en el entorno, no se debe deshabilitar el firewall, por lo que será necesario habilitar el tráfico para los puertos en los que va a levantar tanto el servicio de Redis, como el servicio de Sentinel:

```bash
# Puerto asignado para redis
firewall-cmd --add-port=6379/tcp --permanent
# Puerto asignado para sentinel
firewall-cmd --add-port=26379/tcp --permanent
# Reinicio del servicio para que actualice los cambios
firewall-cmd --reload
# Verificacion de que los puertos se han añadido
firewall-cmd --list-all
```

## Instalación del software REDIS

> [!WARNING]
>  "IMPORTANTE"
> En este manual, el disco añadido es `/dev/sdb` en otros entornos no tiene por que existir ni ser el mismo.

Para la instalación de Redis se ha optado por emplear un disco adicional, lo que permite crear un módulo con un almacenamiento independiente, facilitando realizar modificaciones sobre el tamaño.
Tambien protege el sistema de ficheros, pues en caso de que se dispare el uso del disco, nunca podrá ocuparse más espacio del asignado, protegiendo el sistema operativo.

``` bash
# Creación de la particion sdb1
echo -e "n\np\n\n\n\nt\n8e\nw" | fdisk /dev/sdb

# Comprobacíon de que la nueva partición se ha creado
fdisk -l | grep sdb

# Creación del Physical Volume
pvcreate /dev/sdb1
pvs | grep sdb1
# Creación del Volume Group
vgcreate vg_avlredis /dev/sdb1
vgs | grep vg_avlredis
# Creación del Logical Volume
lvcreate -l 100%FREE -n avlredis vg_avlredis
lvs | grep vg_avlredis
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_avlredis-avlredis
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_avlredis-avlredis /AVL/module/avlredis                   xfs     defaults        0 0" >>/etc/fstab
systemctl daemon-reload
# Creación del punto de montaje
mkdir /AVL/module/avlredis
mount /dev/mapper/vg_avlredis-avlredis /AVL/module/avlredis
df -h | grep vg_avlredis
```

Para instalar redis, descargaremos la versión más reciente disponible. Podemos optar por descargar  la versión numerada, [redis-7.4.1](https://download.redis.io/releases/redis-7.4.1.tar.gz) o como la versión estable más reciente, [redis-stable](https://download.redis.io/redis-stable.tar.gz). En el caso de ser necesario, es posible descargar versiones anteriores de [redis](https://download.redis.io/releases/). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AVL/installers`.

``` bash
# Descarga de la version 7.4.1
wget https://download.redis.io/releases/redis-7.4.1.tar.gz -P /AVL/installers

# Descarga de la última versión estable
wget https://download.redis.io/redis-stable.tar.gz -P /AVL/installers
```

Para este manual se va a utilizar redis con su versión, lo que permitirá mantener un control de las versiones mas sencillo y cómodo.

``` bash
# Instalación de paquetes para compilar e instalar redis
dnf -y install gcc gcc-c++ make zlib-devel pcre-devel openssl-devel wget

# Creación de los directorios necesarios para instalar redis
mkdir -p /AVL/module/avlredis/software/redis-7.4.1
mkdir -p /AVL/module/avlredis/software/redis-7.4.1/conf
mkdir -p /AVL/module/avlredis/software/redis-7.4.1/dump
mkdir -p /AVL/module/avlredis/software/redis-7.4.1/aof

# Creación de directorios para el guardado de logs
mkdir -p /AVL/logs/avlredis

# Creacíon de un link simbólico para independizar las configraciones de la version instalada
ln -s /AVL/module/avlredis/software/redis-7.4.1 /AVL/module/avlredis/software/redis

# Extracción y compilación de los binarios de redis
tar -xvzf /AVL/installers/redis-7.4.1.tar.gz -C /AVL/installers/
cd /AVL/installers/redis-7.4.1
make

# Instalación de redis
make install redis-cli PREFIX=/AVL/module/avlredis/software/redis/

# Copia de los ficheros de configuración de redis y sentinel para utilizarlos como propios
cp -p /AVL/installers/redis-7.4.1/redis.conf /AVL/module/avlredis/software/redis/conf/avlredis.conf
cp -p /AVL/installers/redis-7.4.1/sentinel.conf /AVL/module/avlredis/software/redis/conf/avlsentinel.conf

# Copia de seguridad de los ficheros de configuración de redis
cp -p /AVL/module/avlredis/software/redis/conf/avlredis.conf /AVL/module/avlredis/software/redis/conf/avlredis.conf.orig
cp -p /AVL/module/avlredis/software/redis/conf/avlsentinel.conf /AVL/module/avlredis/software/redis/conf/avlsentinel.conf.orig

# Cambio del contecto de seguridad para poder ejecutar redis con SELinux activado
chcon -Rv -u system_u -t bin_t /AVL/module/avlredis/software/redis/bin/redis-server

# Creación de un link simbólico para poder utilizar redis-cli como comando
ln -s /AVL/module/avlredis/software/redis/bin/redis-cli /usr/bin/redis-cli

# Configuración de usuarios, permisos y ACL
chown -R avlmodule:avlgroup /AVL/module/avlredis
chown -R avlmodule:avlgroup /AVL/logs/avlredis

# Limpieza de paquetes
dnf -y remove gcc gcc-c++ make zlib-devel pcre-devel openssl-devel wget
```

### Configuración de redis-server

La configuración de Redis se aplicará al fichero `avlredis.conf` que se ha creado previamente para evitar modificar el fichero original. Para agilizar la configuración, se utilizará el comando `sed` para modificar las líneas correspondientes.
Tambien se añaden las correspondientes comprobaciones para ver que se han realizado adecuadamente los cambios.

``` bash
cd /AVL/module/avlredis/software/redis/conf/

sed -i 's/bind 127.0.0.1 -::1/bind <IP REDIS ADDRESS>/' avlredis.conf
sed -i 's/protected-mode yes/protected-mode no/' avlredis.conf
sed -i 's/daemonize no/daemonize yes/' avlredis.conf
sed -i 's|pidfile /var/run/redis_6379.pid|pidfile /AVL/module/avlredis/software/redis/bin/redis-server.pid|' avlredis.conf
sed -i 's|logfile ""|logfile "/AVL/logs/avlredis/avlredis.log"|' avlredis.conf
sed -i 's/# save 3600 1 300 100 60 10000/save 3600 1 300 100 60 10000/' avlredis.conf
sed -i 's/dbfilename dump.rdb/dbfilename avlredis_dump.rdb/' avlredis.conf
sed -i 's|dir ./|dir "/AVL/module/avlredis/software/redis/dump"|' avlredis.conf
sed -i 's/appendonly no/appendonly yes/' avlredis.conf
sed -i 's/appendfilename "appendonly.aof"/appendfilename "avlredis.aof"/' avlredis.conf

grep "bind <IP REDIS ADDRESS>" avlredis.conf 
grep "protected-mode no" avlredis.conf 
grep "daemonize yes" avlredis.conf 
grep "pidfile /AVL/module/avlredis/software/redis/bin/redis-server.pid" avlredis.conf 
grep 'logfile "/AVL/logs/avlredis/avlredis.log"' avlredis.conf  
grep "save 3600 1 300 100 60 10000" avlredis.conf
grep "dbfilename avlredis_dump.rdb" avlredis.conf 
grep 'dir "/AVL/module/avlredis/software/redis/dump"' avlredis.conf 
grep "appendonly yes" avlredis.conf 
grep 'appendfilename "avlredis.aof"' avlredis.conf
```

En caso de que se esté montando una arquitectura de maestro-esclavo, la siguiente configuración se aplicará en los nodos esclavos:

``` bash
cd /AVL/module/avlredis/software/redis/conf/

sed 's|# replicaof <masterip> <masterport>|replicaof <MASTER IP ADDRESS> 9379|' archivo.conf > archivo_modificado.conf

grep 'replicaof <MASTER IP ADDRESS> 9379' avlredis.conf
```

### Configuración de sentinel

El servicio de Sentinel será necesario siempre que se monte una arquitectura de maestro-esclavos, pues gestiona la promoción de uno de los nodos esclavos a maestro en caso de que el nodo maestro caiga.

La configuración de Sentinel se aplicará al fichero `avlsentinel.conf` que se ha creado previamente para evitar modificar el fichero original. Para agilizar la configuración, se utilizará el comando `sed` para modificar las líneas correspondientes.
Tambien se añaden las correspondientes comprobaciones para ver que se han realizado adecuadamente los cambios.

``` bash
cd /AVL/module/avlredis/software/redis/conf/

sed -i 's/daemonize no/daemonize yes/' avlsentinel.conf
sed -i 's|pidfile /var/run/redis-sentinel.pid|pidfile "/AVL/module/avlredis/software/redis/bin/redis-sentinel.pid"|' avlsentinel.conf
sed -i 's|logfile ""|logfile "/AVL/logs/avlredis/avlsentinel.log"|' avlsentinel.conf
sed -i 's|SENTINEL resolve-hostnames no|SENTINEL resolve-hostnames yes|' avlsentinel.conf
sed -i 's|SENTINEL announce-hostnames no|SENTINEL announce-hostnames yes|' avlsentinel.conf
sed -i 's|sentinel down-after-milliseconds mymaster 30000|sentinel down-after-milliseconds mymaster 3000|' avlsentinel.conf
sed -i 's|sentinel failover-timeout mymaster 180000|sentinel failover-timeout mymaster 60000|' avlsentinel.conf
sed -i 's|sentinel monitor mymaster 127.0.0.1 6379 2|sentinel monitor avlredis <IP REDIS MASTER> 6379 1|' archivo.conf

grep "daemonize yes" avlsentinel.conf 
grep 'pidfile "/AVL/module/avlredis/software/redis/bin/redis-sentinel.pid"' avlsentinel.conf 
grep 'logfile "/AVL/logs/avlredis/avlsentinel.log"' avlsentinel.conf 
grep "SENTINEL resolve-hostnames yes" avlsentinel.conf 
grep "SENTINEL announce-hostnames yes" avlsentinel.conf 
grep "sentinel down-after-milliseconds mymaster 3000" avlsentinel.conf 
grep "sentinel failover-timeout mymaster 60000" avlsentinel.conf
grep "sentinel monitor avlredis <IP REDIS MASTER> 6379 1" avlsentinel.conf 
```

## Creación de los servicios de Redis y Sentinel

La ejecución los servicios de Redis y Sentinel se va a delegar en el sistema operativo, por lo que se van a crear servicios de `systemctl` que faciliten la administración.

El primer servicio va a ser el que controla el Redis, asegurando que sea el usuario `avlmodule` quien levante el servicio:

``` bash
vi /etc/systemd/system/avlredis.service

```

``` text
[Unit]
Description=redis server
After=network-online.target

[Service]
Type=forking
WorkingDirectory=/AVL/module/avlredis/software/redis
ExecStart=/AVL/module/avlredis/software/redis/bin/redis-server /AVL/module/avlredis/software/redis/conf/avlredis.conf
ExecStop=/AVL/module/avlredis/software/redis/bin/redis-cli -h <IP REDIS ADDRESS> shutdown
PrivateTmp=true

User=avlmodule
Group=avlgroup

SuccessExitStatus=203

[Install]
WantedBy=multi-user.target
```

El servicio de Sentinel será necesario cuando se cree un cluster de Redis, siendo el usuario `avlmodule` el propietario del servicio.

``` bash
vi  /etc/systemd/system/avlsentinel.service
```

``` text
[Unit]
Description=redis sentinel
After=network-online.target

[Service]
Type=forking
WorkingDirectory=/AVL/module/avlredis/software/redis
ExecStart=/AVL/module/avlredis/software/redis/bin/redis-sentinel /AVL/module/avlredis/software/redis/conf/avlsentinel.conf
PrivateTmp=true

User=avlmodule
Group=avlgroup

[Install]
WantedBy=multi-user.target
```

## Configuración del rotado de logs

Para evitar que se sature la partición dedicada a los logs de los módulos, es necesario implementar un rotado de logs, lo que permite controlar el espacio ocupado y mantener cierto número de ficheros en casi de necesitar revisar logs antiguos

Cada día se realiza una comprobación del tamaño del log. Si es superior a 50MB realiza una copia y la comprime, truncando el fichero original. Se mantiene el fichero original y 5 ficheros rotados.

``` bash
vi /etc/logrotate.d/avlredis
```

``` text
/AVL/logs/avlredis/avlredis.log
{
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 50M
}

/AVL/logs/avlredis/avlsentinel.log
{
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 50M
}
```
