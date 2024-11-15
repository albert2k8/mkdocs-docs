# Guía de instalación de TOMCAT

> [!WARNING]
> Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.
> * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
> * La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
> * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
> * La instalación es compatible con SELinux en modo `enforcing`.
> * La instalación es compatible con el uso del servicio `firewalld`.

## Rutas de instalación para módulos AFC

El paso previo a la instalación del Tomcat es crear el punto de montaje base para instalación de los diferentes módulos que vayan a convivir y el almacenamiento de logs.

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
# Puerto asignado para el servicio Tomcat - Puerto seguro
firewall-cmd --add-port=8443/tcp --permanent
# Reinicio del servicio para que actualice los cambios
firewall-cmd --reload
# Verificacion de que los puertos se han añadido
firewall-cmd --list-all
```

## Instalación del software TOMCAT

Tomcat requiere de la instalación de JAVA. Debido a temas de licenciamiento, se empleará la versión software libre de [openJDK](https://jdk.java.net/archive/). La versión `24.0.1.Final` de Wildfly es compatible con versiones de java iguales o superiores a la versión 11, por lo que descargaremos [openjdk-11.0.2](https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AFC/installers`.

``` bash
# Descarga de la version 11.0.2
wget https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz -P /AFC/installers
```

Iniciamos la instalación de ActiveMQ con la instalación de la java. Se descomprime en la ruta `/AFC/sw_base/` y se creará un link simbólico, lo que permite separar las configuraciones de la versión de java, facilitando el proceso de actualizar a una nueva versión.

``` bash
tar xvzf /AFC/installers/openjdk-11.0.2_linux-x64_bin.tar.gz -C /AFC/sw_base/
ln -s /AFC/sw_base/jdk-11.0.2 /AFC/sw_base/java
chown -R afcmodule: /AFC/sw_base/jdk-11.0.2
/AFC/sw_base/openjdk-11/bin/java -version
```

Para instalar Wildfly, descargaremos la versión utilizada en proyecto, [apache-tomcat-9.0.97](https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.97/bin/apache-tomcat-9.0.97.tar.gz). En el caso de ser necesario, es posible descargar otras versiones de [apache-tomcat](https://tomcat.apache.org/download-90.cgi). El software puede descargarse directamente en la máquina virtual siempre que haya salida a internet con el comando `wget` y lo guardaremos en la ruta `/AFC/installers`.

``` bash
# Descarga de la version 5.18.6
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.97/bin/apache-tomcat-9.0.97.tar.gz -P /AFC/installers
```

Comenzaremos creando el árbol de directorios necesarios para el correcto funcionamiento del servicio y mantener organizado el versionado de los despliegues:

``` bash
# Creacion de directorios bases
mkdir -p /AFC/module/afcreport/software/env
mkdir -p /AFC/logs/afcreport/tomcat
```

El siguiente paso es descomprimir los archivos de Tomcat. Crearemos un link simbólico que permita independizar configuraciones de la versión de Tomcat, facilitando la actualización.

``` bash
# Descompresion del software
tar -xvzf /AFC/installers/apache-tomcat-9.0.97.tar.gz -C /AFC/module/afcreport/software/
# Link simbolico para separar configuraciones de version
ln -s /AFC/module/afcreport/software/apache-tomcat-9.0.97 /AFC/module/afcreport/software/tomcat
```

Continuamos con el proceso de instalación de Tomcat creando un certificado autofirmado para que sirva  de *placeholder* para cuando esté disponible el certificado válido.

``` bash
cd /AFC/security

cp -p /AFC/sw_base/java/lib/security/cacerts /AFC/security/

/AFC/sw_base/java/bin/keytool -genkeypair -keyalg RSA -keysize 2048 -keystore $(hostname -d).jks -alias *.$(hostname -d) -validity 365
#	Enter keystore password: 5Drmsu4MZnU0ycIeZ4ql
#	Re-enter new password: 5Drmsu4MZnU0ycIeZ4ql
#	What is your first and last name?
#	  [Unknown]:  
#	What is the name of your organizational unit?
#	  [Unknown]:  
#	What is the name of your organization?
#	  [Unknown]:  
#	What is the name of your City or Locality?
#	  [Unknown]:  
#	What is the name of your State or Province?
#	  [Unknown]:  
#	What is the two-letter country code for this unit?
#	  [Unknown]:  
#	Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
#	  [no]:  yes

/AFC/sw_base/java/bin/keytool -selfcert -alias *.$(hostname -d) -keystore $(hostname -d).jks
# Enter keystore password: 5Drmsu4MZnU0ycIeZ4ql

# Modificación de propietarios de los certificados
chown afcmodule: /AFC/security/*
```

A continuación, se va a configurar Tomcat. El primer paso será crear copias de seguridad de los ficheros que se van a modificar.

``` bash
cp -p /AFC/module/afcreport/software/tomcat/conf/logging.properties /AFC/module/afcreport/software/tomcat/conf/logging.properties.orig
cp -p /AFC/module/afcreport/software/tomcat/conf/server.xml /AFC/module/afcreport/software/tomcat/conf/server.xml.orig
```

Seguidamente, vamos a crear el fichero de variables de entorno que utilizará el servicio `systemctl` para ejecutar el software

``` bash
cd /AFC/module/afcreport/software/env/

echo "JAVA_HOME='/AFC/sw_base/java'" > environment.config
echo "CATALINA_PID='/AFC/module/afcreport/software/tomcat/temp/tomcat.pid'" >> environment.config
echo "CATALINA_HOME='/AFC/module/afcreport/software/tomcat'" >> environment.config
echo "CATALINA_BASE='/AFC/module/afcreport/software/tomcat'" >> environment.config
echo "CATALINA_OPTS='-Xms2048m -Xmx4096m -Xss2m -server -XX:+UseG1GC -Djava.locale.providers=COMPAT'" >> environment.config
echo "JAVA_OPTS='-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom -Djava.net.preferIPv4Stack=true -Dlog4j2.formatMsgNoLookups=true -Djavax.net.ssl.trustStore=/AFC/security/cacerts'" >> environment.config
echo "CATALINA_OUT='/AFC/logs/afcreport/tomcat/catalina.out'" >> environment.config

cat /AFC/module/afcreport/software/env/environment.config
```

El siguiente paso es configurar la ubicación de los logs de Tomcat. Para ello se realizarán los siguientes cambios:

``` bash
cd /AFC/module/afcreport/software/tomcat/conf/

sed -i 's|^1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs|1catalina.org.apache.juli.AsyncFileHandler.directory = /AFC/logs/afcreport/tomcat|' logging.properties
sed -i 's|^2localhost.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs|2localhost.org.apache.juli.AsyncFileHandler.directory = /AFC/logs/afcreport/tomcat|' logging.properties
sed -i 's|^3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs|3manager.org.apache.juli.AsyncFileHandler.directory = /AFC/logs/afcreport/tomcat|' logging.properties
sed -i 's|^4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs|4host-manager.org.apache.juli.AsyncFileHandler.directory = /AFC/logs/afcreport/tomcat|' logging.properties
```

El último paso de la configuración es editar el fichero `server.xml'. Para ello usaremos una version simplifiacda del contenido en la que se han eliminado las secciones comentadas.
Para facilitar la configuración, se utilizarán variables de entorno que reemplazarán sus valores al reescribir el fichero.

``` bash
export HOSTNAME=afcreport1-vip.saepre.logrono.es
echo $HOSTNAME
export KEYSTOREFILE=/AFC/security/$(hostname -d).jks
echo $KEYSTOREFILE
export KEYSTOREPASS=5Drmsu4MZnU0ycIeZ4ql
echo $KEYSTOREPASS
export KEYALIAS=*.$(hostname -d)
echo $KEYALIAS

cat <<EOF > server.xml
<?xml version="1.0" encoding="UTF-8"?>

<Server port="8005" shutdown="hajsdkh%6_u">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.security.SecurityListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">

    <Connector address="$HOSTNAME" port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxParameterCount="1000"
               />

    <Connector address="$HOSTNAME" port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               keystoreFile="$KEYSTOREFILE" keystoreType="jks" keystorePass="$KEYSTOREPASS" keyAlias="$KEYALIAS"
               sslEnabledProtocols="TLSv1.2,TLSv1.3" connectionTimeout="20000" server="TRANTOR_ANX" xpoweredBy="false"/>

    <Engine name="Catalina" defaultHost="localhost">


      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

      <Valve className="org.apache.catalina.valves.AccessLogValve" directory="/AFC/logs/afcreport/tomcat"
               prefix="localhost_access_log" suffix=".txt" rotatable="false"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

      </Host>
    </Engine>
  </Service>
</Server>
EOF

```

A continuación, hay que crear la carpeta donde se almacenarán los logs, además de asignar la propiedad del contenido /AVL/module/avlbroker y /AVL/logs/avlbroker al usuario y grupo avlmodule:avlgroup.

``` bash
mkdir -p /AFC/logs/afcreport/tomcat
chown -R afcmodule:afcgroup /AFC/module/afcreport
chown -R afcmodule:afcgroup /AFC/logs/afcreport
```

El último paso de la configuración será permitir que el servicio pueda levantar con SELinux activo en modo enforcing lo que mejora la seguridad del sistema.

``` bash
# Cambio de contexto para ser compatible con SELinux
chcon -Rv -u system_u -t bin_t /AFC/module/afcreport/software/tomcat/bin/startup.sh
chcon -Rv -u system_u -t bin_t /AFC/module/afcreport/software/tomcat/bin/shutdown.sh

ls -lZ /AFC/module/afcreport/software/tomcat/bin/startup.sh
ls -lZ /AFC/module/afcreport/software/tomcat/bin/shutdown.sh
```

## Creación del servicios de TOMCAT

Tras terminar de configurar el Tomcat, ya se puede crear el servicio de `systemctl` que levante el bróker:

``` bash
vi /etc/systemd/system/afcreport.service
```

``` text
[Unit]
Description=Apache Tomcat Web Application Container
After=syslog.target network-online.target

[Service]
Type=forking

EnvironmentFile=-/AFC/module/afcreport/software/env/environment.config
ExecStart=/AFC/module/afcreport/software/tomcat/bin/startup.sh
ExecStop=/AFC/module/afcreport/software/tomcat/bin/shutdown.sh

User=afcmodule
Group=afcgroup

SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

## Configuración del rotado de logs

Para evitar que se sature la partición dedicada a los logs de los módulos, es necesario implementar un rotado de logs, lo que permite controlar el espacio ocupado y mantener cierto número de ficheros en casi de necesitar revisar logs antiguos

Cada día se realiza una comprobación del tamaño del log. Si es superior a 10MB realiza una copia y la comprime, truncando el fichero original. Se mantiene el fichero original y 5 ficheros rotados.

``` bash
vi /etc/logrotate.d/afcreport
```

``` text
/AFC/logs/afcreport/tomcat/catalina.out
{
   su afcmodule afcgroup
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 10M
}

/AFC/logs/afcreport/tomcat/catalina.log
{
   su afcmodule afcgroup
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 10M
}

/AFC/logs/afcreport/tomcat/host-manager.log
{
   su afcmodule afcgroup
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 10M
}

/AFC/logs/afcreport/tomcat/manager.log
{
   su afcmodule afcgroup
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 10M
}

/AFC/logs/afcreport/tomcat/localhost.log
{
   su afcmodule afcgroup
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 10M
}

/AFC/logs/afcreport/tomcat/localhost_access_log.txt
{
   su afcmodule afcgroup
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 10M
}
```





















