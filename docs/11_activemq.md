# Guía de instalación de ACTIVEMQ

!!! warning "Consideraciones previas"
    Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.

    * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
    * La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
    * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
    * La instalación es compatible con SELinux en modo `enforcing`.
    * La instalación es compatible con el uso del servicio `firewalld`.

## Rutas de instalación para módulos AFC

El paso previo a la instalación de Artemis es crear el punto de montaje base para instalación de los diferentes módulos que vayan a convivir y el almacenamiento de logs.

Se va a comenzar por crear una nueva partición ya que el disco del sistema operativo dispone de `100GB` de los cuales solo se aprovisionan `56GB`.

``` bash
# Creación de la particion sda4
echo -e "n\n4\n\n\nt\n\nE6D6D379-F507-44C2-A23C-238F2A3DF928\nw" | fdisk /dev/sda

# Comprobacíon de que la nueva partición se ha creado
fdisk -l | grep sda
```

Una vez creada la partición, se puede comenzar el proceso para crear los puntos de montaje `/AFC` y `/AFC/logs`

``` bash
# Creación del Physical Volume
pvcreate /dev/sda4
pvs | grep sda4
# Creación del Volume Group
vgcreate vg_afc /dev/sda4 
vgs | grep vg_afc
# Creación del Logical Volume
lvcreate -L 30G -n afc vg_afc
lvcreate -L 10G -n logs vg_afc
lvs | grep vg_afc
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_afc-afc
mkfs.xfs /dev/mapper/vg_afc-logs
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_afc-afc /AFC                    xfs     defaults        0 0" >> /etc/fstab
echo "/dev/mapper/vg_afc-logs /AFC/logs                    xfs     defaults        0 0" >> /etc/fstab
# Inclusión de los cambios realizados en /etc/fstab
systemctl daemon-reload
# Creación del punto de montaje /AFC 
mkdir -p /AFC
mount /dev/mapper/vg_afc-afc /AFC
df -h | grep vg_afc
# Creación del punto de montaje
mkdir -p /AFC/logs
mount /dev/mapper/vg_afc-logs /AFC/logs
df -h | grep vg_afc
```

El siguiente paso, será crear el grupo `afcgroup` y los usuarios `afcmodule` y `afcadmin`. El primero será el usuario que levante los servicios, mientras que el segundo, será el usuario encargado de administrar las configuraciones.

``` bash
# Creación del grupo afcgruop
groupadd -g 2000 afcgroup
# Creación del usario afcadmin que gestiona los servicios
useradd -u 2000 -d /AFC -c "AFC Admin Account" -g afcgroup -p $(openssl passwd -1 MMNUTXvCIm3@DCpjsrVg) afcadmin
# Creación del usario afcmodule que ejecuta los servicios
useradd -u 2001 -d /AFC -c "AFC module runner" -g afcgroup -s /sbin/nologin afcmodule
```

El proceso finaliza creando los directorios necesarios para organizar los diferentes elementos y asignando los propietarios y permisos correspondientes. La ACL permitirá que a medida que se vayan creando ficheros, estos ya tengan los permisos adecuados.

``` bash
# Creación del árbol de directorios para el módulo AFC
mkdir -p /AFC/scripts
mkdir -p /AFC/sw_base
mkdir -p /AFC/module
mkdir -p /AFC/security
mkdir -p /AFC/installers

# Configuración de usuarios, permisos y ACL
chown -R afcmodule:afcgroup /AFC
chmod -R 750 /AFC
chmod -R 750 /AFC/logs
setfacl -PRdm u::rwx,g::rw,o::- /AFC
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

## Instalación del software ACTIVEMQ

!!! warning 
    En este manual, el disco añadido es `/dev/sdc` en otros entornos no tiene por que existir ni ser el mismo.

Para la instalación de ActiveMQ se ha optado por emplear un disco adicional, lo que permite crear un módulo con un almacenamiento independiente, facilitando realizar modificaciones sobre el tamaño.
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
vgcreate vg_afcbroker /dev/sdc1
vgs | grep vg_afcbroker
# Creación del Logical Volume
lvcreate -l 100%FREE -n afcbroker vg_afcbroker
lvs | grep vg_afcbroker
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_afcbroker-afcbroker
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_afcbroker-afcbroker /AFC/module/afcbroker                   xfs     defaults        0 0" >>/etc/fstab
systemctl daemon-reload
# Creación del punto de montaje
mkdir /AFC/module/afcbroker
mount /dev/mapper/vg_afcbroker-afcbroker /AFC/module/afcbroker
df -h | grep vg_afcbroker
```

Para instalar ActiveMQ, descargaremos una de las versiones más recientes disponibles, [apache-activemq-5.18.6](https://dlcdn.apache.org/activemq/5.18.6/apache-activemq-5.18.6-bin.tar.gz). En el caso de ser necesario, es posible descargar versiones anteriores de [activemq](https://activemq.apache.org/components/classic/documentation/download-archives). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AVL/installers`.

``` bash
# Descarga de la version 5.18.6
wget https://dlcdn.apache.org/activemq/5.18.6/apache-activemq-5.18.6-bin.tar.gz -P /AFC/installers
```

ActiveMQ requiere de la instalación de JAVA. Debido a temas de licenciamiento, se empleará la versión software libre de [openJDK](https://jdk.java.net/archive/). La versión `5.18.6` de ActiveMQ es compatible con versiones de java iguales o superiores a la versión 11, por lo que descargaremos [openjdk-11.0.2](https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AFC/installers`.

``` bash
# Descarga de la version 11.0.2
wget https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz -P /AFC/installers
```

Iniciamos la instalación de ActiveMQ con la instalación de la java. Se descomprime en la ruta `/AFC/sw_base/` y se creará un link simbólico, lo que permite separar las configuraciones de la versión de java, facilitando el proceso de actualizar a una nueva versión.

``` bash
tar xvzf /AFC/installers/openjdk-11.0.2_linux-x64_bin.tar.gz -C /AFC/sw_base/
ln -s /AFC/sw_base/jdk-11.0.2 /AFC/sw_base/openjdk-11
chown -R afcmodule: /AFC/sw_base/jdk-11.0.2
/AFC/sw_base/openjdk-11/bin/java -version
```

El siguiente paso es descomprimir ActiveMQ en la ruta `/AFC/installers` para poder comenzar con la instalación. Se creará tambien un link simbólico para que el servicio que gestione artemis y otros elementos pueda configurarse de forma independiente a la versión instalada de artemis.

``` bash
mkdir -p /AFC/module/afcbroker/software

tar -xvzf /AFC/installers/apache-activemq-5.18.6-bin.tar.gz -C /AFC/module/afcbroker/software/
ln -s /AFC/module/afcbroker/software/apache-activemq-5.18.6 /AFC/module/afcbroker/software/activemq
```

Una vez descomprimido, podemos pasar a crear el bróker de mensajería con cierta configuración ya implementada.

```bash
cd /AFC/module/afcbroker/software/activemq/bin/

# Edicion del fichero env
sed -i 's|ACTIVEMQ_USER=""|ACTIVEMQ_USER="afcmodule"|' env
sed -i 's|#JAVA_HOME=""|JAVA_HOME="/AFC/sw_base/openjdk-11"|' env
sed -i 's|^JAVACMD="auto"|#JAVACMD="auto"|' env

# Verificación de los cambios aplicados
grep 'ACTIVEMQ_USER="afcmodule"' env
grep 'JAVA_HOME="/AFC/sw_base/java"' env
grep '#JAVACMD="auto"' env
```

```bash
cd /AFC/module/afcbroker/software/activemq/conf/

# Modificación del fichero log4j2.properties
sed -i 's|${sys:activemq.data}|/AFC/logs/afcbroker|' log4j2.properties
sed -i 's|<property name="host" value="127.0.0.1"/>|<property name="host" value="afcbroker.saepre.logrono.es"/>|' jetty.xml
sed -i 's|brokerName="localhost"|brokerName="afcbroker.saepre.logrono.es"|' activemq.xml

# Verificación de los cambios aplicados
grep "/AFC/logs/afcbroker" log4j2.properties
grep "afcbroker.saepre.logrono.es" jetty.xml
grep "afcbroker.saepre.logrono.es" activemq.xml
```

A continuación, crearemos los usuarios con los diferentes roles para cada tipo de usuario

```bash
cd /AFC/module/afcbroker/software/activemq/bin/

sed -i ':a;N;$!ba;s/\n[^\n]*\n[^\n]*$//' jetty-realm.properties

# Creación del usuario admin
md5_hash=$(echo -n "password" | md5sum | awk '{print $1}')
echo "admin: MD5:$md5_hash,admin" | tee -a jetty-realm.properties

# Creación del usuario afcadmin
md5_hash=$(echo -n "password" | md5sum | awk '{print $1}')
echo "afcadmin: MD5:$md5_hash,admin" | tee -a jetty-realm.properties

# Creación del usuario operador
md5_hash=$(echo -n "password" | md5sum | awk '{print $1}')
echo "operador: MD5:$md5_hash,user" | tee -a jetty-realm.properties

# Creación del usuario monitor
md5_hash=$(echo -n "password" | md5sum | awk '{print $1}')
echo "monitor: MD5:$md5_hash,user" | tee -a jetty-realm.properties

# Creación del usuario user
md5_hash=$(echo -n "password" | md5sum | awk '{print $1}')
echo "user: MD5:$md5_hash,user" | tee -a jetty-realm.properties
```

A continuación, hay que crear la carpeta donde se almacenarán los logs, además de asignar la propiedad del contenido `/AFC/module/afcbroker` y `/AFC/logs/afcbroker` al usuario y grupo `afcmodule:afcgroup`.

``` bash
mkdir -p /AFC/logs/afcbroker
chown -R afcmodule:afcgroup /AFC/module/afcbroker
chown -R afcmodule:afcgroup /AFC/logs/afcbroker
```

El último paso de la configuración será permitir que el servicio pueda levantar con SELinux activo en modo `enforcing` lo que mejora la seguridad del sistema.

``` bash
# Cambio de contexto para ser compatible con SELinux
chcon -Rv -u system_u -t bin_t /AFC/module/afcbroker/software/activemq/bin/activemq

ls -lZ /AFC/module/afcbroker/software/activemq/bin/activemq
```

## Creación del servicio de ActiveMQ

Tras terminar de configurar el bróker, ya se puede crear el servicio de `systemctl` que levante el bróker:

``` bash
vi /etc/systemd/system/afcbroker.service
```

``` text
[Unit]
Description=afcbroker service
After=network.target

[Service]
Type=forking
ExecStart=/AFC/module/afcbroker/software/activemq/bin/activemq start
ExecStop=/AFC/module/afcbroker/software/activemq/bin/activemq stop
PIDFile=/AFC/module/afcbroker/software/activemq/data/activemq.pid

User=afcmodule
Group=afcgroup

Restart=always
RestartSec=9

LimitNOFILE=16384:16384

[Install]
WantedBy=multi-user.target
```
