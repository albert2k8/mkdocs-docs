# Guía de instalación de WILDFLY

> [!WARNING]
> "Consideraciones previas"
> Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.
> * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
> * La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
> * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
> * * La instalación es compatible con SELinux en modo `enforcing`.
>   * * La instalación es compatible con el uso del servicio `firewalld`.

## Rutas de instalación para módulos AFC

El paso previo a la instalación del Wildfly es crear el punto de montaje base para instalación de los diferentes módulos que vayan a convivir y el almacenamiento de logs.

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

Dada la importancia que tiene la seguridad en el entorno, no se debe deshabilitar el firewall, por lo que será necesario habilitar el tráfico para el puerto donde se va a ejecutar Wildfly:

``` bash
# Puerto asignado para el servicio SFTP
firewall-cmd --add-port=2222/tcp --permanent
# Reinicio del servicio para que actualice los cambios
firewall-cmd --reload
# Verificacion de que los puertos se han añadido
firewall-cmd --list-all
```

## Instalación del software WILDFLY

!!! warning 
    En este manual, el disco añadido es `/dev/sdb` en otros entornos no tiene por que existir ni ser el mismo.

Para la creación del servicio SFTP se ha optado por emplear un disco adicional, lo que permite crear un módulo con un almacenamiento independiente, facilitando realizar modificaciones sobre el tamaño.
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
vgcreate vg_afcweb /dev/sdb1
vgs | grep vg_afcweb
# Creación del Logical Volume
lvcreate -l 100%FREE -n afcweb vg_afcweb
lvs | grep vg_afcsftp
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_afcweb-afcweb
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_afcweb-afcweb /AFC/module/afcweb                   xfs     defaults        0 0" >>/etc/fstab
systemctl daemon-reload
# Creación del punto de montaje
mkdir /AFC/module/afcweb
mount /dev/mapper/vg_afcweb-afcweb /AFC/module/afcweb
df -h | grep vg_afcweb
```

Para instalar Wildfly, descargaremos la versión utilizada en proyecto, [wildfly-24.0.1.Final](https://download.jboss.org/wildfly/24.0.1.Final/wildfly-24.0.1.Final.tar.gz). En el caso de ser necesario, es posible descargar otras versiones de [wildfly](https://www.wildfly.org/downloads/). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AFC/installers`.

``` bash
# Descarga de la version 5.18.6
wget https://download.jboss.org/wildfly/24.0.1.Final/wildfly-24.0.1.Final.tar.gz -P /AFC/installers
```

Comenzaremos creando el árbol de directorios necesarios para el correcto funcionamiento del servicio y mantener organizado el versionado de los despliegues:

``` bash
# Creacion de directorios bases
mkdir -p /AFC/module/afcweb/software
mkdir -p /AFC/module/afcweb/configuration/certificates
mkdir -p /AFC/module/afcweb/configuration/environment
mkdir -p /AFC/module/afcweb/configuration/etc
mkdir -p /AFC/module/afcweb/current

# creación del directorio ejemplo de versiones de despliegues
mkdir -p /AFC/module/afcweb/releases/release-X.X.X/deployments

# Link simbólico al directorio con la version actual deplegada
ln -s /AFC/module/afcweb/releases/release-X.X.X/deployments /AFC/module/afcweb/current/deployments
```

El siguiente paso es descomprimir los archivos de Wildfly. Crearemos un link simbólico que permita independizar configuraciones de la versión de Wildfly, facilitando la actualización.
Tambien crearemos otro link simbólico que apunte a la ubicación donde se almacenan los despliegues.

``` bash
# Descompresion del software
tar -xzvf /AFC/installers/wildfly-24.0.1.Final.tar.gz -C /AFC/module/afcweb/software
# Link simbolico para separar configuraciones de version
ln -s /AFC/module/afcweb/software/wildfly-24.0.1.Final /AFC/module/afcweb/software/wildfly

# Renombrado del directorio donde se almacenan los despliegues
mv /AFC/module/afcweb/software/wildfly/standalone/deployments /AFC/module/afcweb/software/wildfly/standalone/deployments.orig
# Link simbolico a donde se almacenan los despliegues
ln -s /AFC/module/afcweb/current/deployments/ /AFC/module/afcweb/software/wildfly/standalone/deployments
```



``` bash
cd /AFC/module/afcweb/software/wildfly/standalone/configuration/

sed -i '/<interfaces>/,/<\/interfaces>/c\
    <interfaces>\
        <interface name="management">\
            <inet-address value="${jboss.bind.address.management:afcwebadmin.saepre.logrono.es}"/>\
        </interface>\
        <interface name="public">\
            <inet-address value="${jboss.bind.address:afcwebadmin.saepre.logrono.es}"/>\
        </interface>\
        <interface name="unsecure">\
            <inet-address value="${jboss.bind.address.unsecure:afcwebadmin.saepre.logrono.es}"/>\
        </interface>\
    </interfaces>' standalone.xml
```

A continuación, hay que crear la carpeta donde se almacenarán los logs, además de asignar la propiedad del contenido `/AFC/module/afcweb` y `/AFC/logs/afcweb` al usuario y grupo `afcmodule:afcgroup`.

``` bash
mkdir -p /AFC/logs/afcweb
chown -R afcmodule:afcgroup /AFC/module/afcweb
chown -R afcmodule:afcgroup /AFC/logs/afcweb
```

El último paso de la configuración será permitir que el servicio pueda levantar con SELinux activo en modo `enforcing` lo que mejora la seguridad del sistema.

``` bash
# Cambio de contexto para ser compatible con SELinux
chcon -Rv -u system_u -t bin_t /AFC/module/afcweb/software/wildfly/bin/standalone.sh

ls -lZ /AFC/module/afcweb/software/wildfly/bin/standalone.sh
```

## Creación del servicios de Wildfly

Tras terminar la configuración, ya se puede crear el servicio de `systemctl` que levante el servicio:

``` bash
vi /etc/systemd/system/afcweb.service
```

``` text
[Unit]
Description=afcweb service
After=network.target

[Service]
#Type=forking
Type=simple
EnvironmentFile=-/AFC/module/afcweb/Configuration/environment/environment.file
Environment=JAVA_HOME=/AFC/sw_base/openjdk-11
Environment=JBOSS_LOG_DIR=/AFC/logs/afcweb/
ExecStart=/AFC/module/afcweb/software/wildfly/bin/standalone.sh -c standalone.xml

User=afcmodule
Group=afcgroup

#Restart=always
#RestartSec=9

LimitNOFILE=16384:16384

[Install]
WantedBy=multi-user.target
```
