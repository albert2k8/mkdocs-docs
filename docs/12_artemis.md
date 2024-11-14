# Guía de instalación de ARTEMIS

!!! warning
    Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.

    * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
    * La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
    * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
    * La instalación es compatible con SELinux en modo `enforcing`.
    * La instalación es compatible con el uso del servicio `firewalld`.

## Rutas de instalación para módulos AVL

El paso previo a la instalación de Artemis es crear el punto de montaje base para instalación de los diferentes módulos que vayan a convivir y el almacenamiento de logs.

Se va a comenzar por crear una nueva partición ya que el disco del sistema operativo dispone de `100GB` de los cuales solo se aprovisionan `56GB`.

``` bash
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

Dada la importancia que tiene la seguridad en el entorno, no se debe deshabilitar el firewall, por lo que será necesario habilitar el tráfico para los puertos en los que van a levantar los diferentes bróker de mensajería:

``` bash
# Puerto asignado para consola web
firewall-cmd --add-port=8161/tcp --permanent
# Puerto asignado para broker AMQP
firewall-cmd --add-port=5672/tcp --permanent
# Puerto asignado para broker STOMP
firewall-cmd --add-port=61613/tcp --permanent
# Puerto asignado para broker OPENWIRE
firewall-cmd --add-port=61616/tcp --permanent
# Reinicio del servicio para que actualice los cambios
firewall-cmd --reload
# Verificacion de que los puertos se han añadido
firewall-cmd --list-all
```

## Instalación del software ARTEMIS

!!! warning 
    En este manual, el disco añadido es `/dev/sdc` en otros entornos no tiene por que existir ni ser el mismo.

Para la instalación de Artemis se ha optado por emplear un disco adicional, lo que permite crear un módulo con un almacenamiento independiente, facilitando realizar modificaciones sobre el tamaño.
Tambien protege el sistema de ficheros, pues en caso de que se dispare el uso del disco, nunca podrá ocuparse más espacio del asignado, protegiendo el sistema operativo.

``` bash
# Creación de la particion sdc1
echo -e "n\np\n\n\n\nt\n8e\nw" | fdisk /dev/sdc

# Comprobacíon de que la nueva partición se ha creado
fdisk -l | grep sdc

# Creación del Physical Volume
pvcreate /dev/sdc1
pvs | grep sdc1
# Creación del Volume Group
vgcreate vg_avlbroker /dev/sdc1
vgs | grep vg_avlbroker
# Creación del Logical Volume
lvcreate -l 100%FREE -n avlbroker vg_avlbroker
lvs | grep vg_avlbroker
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_avlbroker-avlbroker
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_avlbroker-avlbroker /AVL/module/avlbroker                   xfs     defaults        0 0" >>/etc/fstab
systemctl daemon-reload
# Creación del punto de montaje
mkdir /AVL/module/avlbroker
mount /dev/mapper/vg_avlbroker-avlbroker /AVL/module/avlbroker
df -h | grep vg_avlbroker
```

Para instalar Artemis, descargaremos la versión más reciente disponible, [apache-artemis-2.38.0](https://archive.apache.org/dist/activemq/activemq-artemis/2.38.0/apache-artemis-2.38.0-bin.tar.gz). En el caso de ser necesario, es posible descargar versiones anteriores de [artemis](https://archive.apache.org/dist/activemq/activemq-artemis/). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AVL/installers`.

``` bash
# Descarga de la version 2.38.0
wget https://archive.apache.org/dist/activemq/activemq-artemis/2.38.0/apache-artemis-2.38.0-bin.tar.gz -P /AVL/installers
```

Además, Artemis requiere de la instalación de JAVA. Debido a temas de licenciamiento, se empleará la versión software libre de [openJDK](https://jdk.java.net/archive/). La versión más reciente de Artemis es compatible con versiones de java iguales o superiores a la versión 11, por lo que descargaremos [openjdk-11.0.2](https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AVL/installers`.

``` bash
# Descarga de la version 11.0.2
wget https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz -P /AVL/installers
```

Iniciamos la instalación de Artemis con la instalación de la java. Se descomprime en la ruta `/AVL/sw_base/` y se creará un link simbólico, lo que permite separar las configuraciones de la versión de java, facilitando el proceso de actualizar a una nueva versión.

``` bash
tar xvzf /AVL/installers/openjdk-11.0.2_linux-x64_bin.tar.gz -C /AVL/sw_base/
ln -s /AVL/sw_base/jdk-11.0.2 /AVL/sw_base/openjdk-11
chown -R avlmodule: /AVL/sw_base/jdk-11.0.2
/AVL/sw_base/openjdk-11/bin/java -version
```

El siguiente paso es descomprimir Artemis en la ruta `/AVL/installers` para poder comenzar con la instalación. Se creará tambien un link simbólico para que el servicio que gestione artemis y otros elementos pueda configurarse de forma independiente a la versión instalada de artemis.

``` bash
mkdir -p /AVL/module/avlbroker/software

tar -xvzf /AVL/installers/apache-artemis-2.38.0-bin.tar.gz -C /AVL/module/avlbroker/software
ln -s /AVL/module/avlbroker/software/apache-artemis-2.38.0 /AVL/module/avlbroker/software/artemis
```

Una vez descomprimido, podemos crear el bróker de mensajería con cierta configuración ya implementada, además de crear una serie de usuarios adicionales con roles concretos para tengan funcionalidades limitadas en el servicio.

``` bash
# Variable de entorno con la ruta de la java
export JAVA_HOME=/AVL/sw_base/openjdk-11

# Creacion del broker "avlbroker"
/AVL/module/avlbroker/software/artemis/bin/artemis create --force --user admin --password < ADMIN PASSWORD > --allow-anonymous --no-hornetq-acceptor --no-mqtt-acceptor --name=avlbroker --default-port=61616 --http-port=8161 --host=< BROKER FQDN > /AVL/module/avlbroker/broker/
```

Una vez terminada la instalación básica, se aplicarán una serie de configuraciones adicionales para completar las funcionalidades del bróker.

``` bash
cp -p /AVL/module/avlbroker/broker/etc/broker.xml /AVL/module/avlbroker/broker/etc/broker.xml.orig
cp -p /AVL/module/avlbroker/broker/etc/bootstrap.xml /AVL/module/avlbroker/broker/etc/bootstrap.xml.orig
cp -p /AVL/module/avlbroker/broker/etc/jolokia-access.xml /AVL/module/avlbroker/broker/etc/jolokia-access.xml.orig
cp -p /AVL/module/avlbroker/broker/etc/artemis.profile /AVL/module/avlbroker/broker/etc/artemis.profile.orig
cp -p /AVL/module/avlbroker/broker/etc/log4j2.properties /AVL/module/avlbroker/broker/etc/log4j2.properties.orig
cp -p /AVL/module/avlbroker/broker/etc/management.xml /AVL/module/avlbroker/broker/etc/management.xml.orig

cd /AVL/module/avlbroker/broker/etc/

# Edicion de bootstrap.xml
sed -i 's|http://localhost:8161|http://<BROKER FQDN>:8161|' /AVL/module/avlbroker/broker/etc/bootstrap.xml

# Edicion de jolokia-access.xml
sed -i 's|\*://localhost\*|*://<BROKER FQDN>*|' /AVL/module/avlbroker/broker/etc/jolokia-access.xml

# Edicion de log4j2.properties
sed -i 's|${sys:artemis.instance}/log/artemis.log|/AVL/logs/avlbroker/artemis.log|' log4j2.properties
sed -i 's|${sys:artemis.instance}/log/audit.log|/AVL/logs/avlbroker/audit.log|' log4j2.properties

# Edicion de management.xml
sed -i 's|<access method="list\*" roles="amq"/>|<access method="list*" roles="amq,view"/>|' management.xml
sed -i 's|<access method="get\*" roles="amq"/>|<access method="get*" roles="amq,view"/>|' management.xml
sed -i 's|<access method="is\*" roles="amq"/>|<access method="is*" roles="amq,view"/>|' management.xml
sed -i 's|<access method="browse\*" roles="amq"/>|<access method="browse*" roles="amq,view"/>|' management.xml
sed -i 's|<access method="count\*" roles="amq"/>|<access method="count*" roles="amq,view"/>|' management.xml
```

A continuación, hay que crear la carpeta donde se almacenarán los logs, además de asignar la propiedad del contenido `/AVL/module/avlbroker` y `/AVL/logs/avlbroker` al usuario y grupo `avlmodule:avlgroup`.

``` bash
mkdir -p /AVL/logs/avlbroker
chown -R avlmodule:avlgroup /AVL/module/avlbroker
chown -R avlmodule:avlgroup /AVL/logs/avlbroker
```

El último paso de la configuración será permitir que el servicio pueda levantar con SELinux activo en modo `enforcing` lo que mejora la seguridad del sistema.

``` bash
# Cambio de contexto para ser compatible con SELinux
chcon -Rv -u system_u -t bin_t /AVL/module/avlbroker/broker/bin/artemis-service

ls -lZ /AVL/module/avlbroker/broker/bin/artemis-service
```

## Creación del servicios de Artemis

Tras terminar de configurar el bróker, ya se puede crear el servicio de `systemctl` que levante el bróker:

``` bash
vi /etc/systemd/system/avlbroker.service
```

``` text
[Unit]
Description=Apache ActiveMQ Artemis
After=network-online.target

[Service]
Type=forking

Environment=JAVA_HOME=/AVL/sw_base/openjdk-11
PIDFile=/AVL/module/avlbroker/broker/data/artemis.pid
ExecStart=/AVL/module/avlbroker/broker/bin/artemis-service start
ExecStop=/AVL/module/avlbroker/broker/bin/artemis-service stop
ExecReload=/AVL/module/avlbroker/broker/bin/artemis-service restart

User=avlmodule
Group=avlgroup

SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

## Creación de usuarios con roles

Una vez creado el bróker y verificado que el servicio levanta sin problemas, podemos pasar a crear diferentes usuarios con distintos roles, evitando asi que el usuario `admin` con sus credenciales estén disponibles para todos los usuarios del servicio.

```bash
# Creación de los usuarios con sus roles
/AVL/module/avlbroker/software/artemis/bin/artemis user add --user-command-user avladmin --user-command-password < PASSWORD > --role amq
/AVL/module/avlbroker/software/artemis/bin/artemis user add --user-command-user operador --user-command-password < PASSWORD > --role view
/AVL/module/avlbroker/software/artemis/bin/artemis user add --user-command-user monitor --user-command-password < PASSWORD > --role view
/AVL/module/avlbroker/software/artemis/bin/artemis user add --user-command-user user --user-command-password < PASSWORD > --role view
```
