# Guía de instalación de SFTP Independiente

!!! warning "Consideraciones previas"
    Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.

    * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
    * La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
    * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
    * La instalación es compatible con SELinux en modo `enforcing`.
    * La instalación es compatible con el uso del servicio `firewalld`.
    * Se busca separar el servicio SFTP del servicio SSH, lo cual no implica que sea la única forma de montar un servicio de SFTP.
  
## Rutas de instalación para módulos AFC

El paso previo a la instalación del SFTP es crear el punto de montaje base para instalación de los diferentes módulos que vayan a convivir y el almacenamiento de logs.

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

# Creacion del grupo afcsftp para el enjaulado del SFTP
groupadd -g 4000 afcsftp
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

Dada la importancia que tiene la seguridad en el entorno, no se debe deshabilitar el firewall, por lo que será necesario habilitar el tráfico para el puerto donde se va a ejecutar el SFTP. Si al servicio se le asigna una IP diferente a la de la máquina virtual, no será necesario abrir el puerto 22, pues ya esta abierto por defecto. En caso contrario, no podrá utilizarse el puerto 22, por lo que habrá que abrir el puerto asignado al servicio:

``` bash
# Puerto asignado para el servicio SFTP
firewall-cmd --add-port=2222/tcp --permanent
# Reinicio del servicio para que actualice los cambios
firewall-cmd --reload
# Verificacion de que los puertos se han añadido
firewall-cmd --list-all
```

## Instalación del software ACTIVEMQ

!!! warning 
    En este manual, el disco añadido es `/dev/sdd` en otros entornos no tiene por que existir ni ser el mismo.

Para la creación del servicio SFTP se ha optado por emplear un disco adicional, lo que permite crear un módulo con un almacenamiento independiente, facilitando realizar modificaciones sobre el tamaño.
Tambien protege el sistema de ficheros, pues en caso de que se dispare el uso del disco, nunca podrá ocuparse más espacio del asignado, protegiendo el sistema operativo.

``` bash
# Creación de la particion sdd1
echo -e "n\np\n\n\n\nt\n8e\nw" | fdisk /dev/sdd

# Comprobacíon de que la nueva partición se ha creado
fdisk -l | grep sdd

# Creación del Physical Volume
pvcreate /dev/sdd1
pvs | grep sdd1
# Creación del Volume Group
vgcreate vg_afcsftp /dev/sdd1
vgs | grep vg_afcsftp
# Creación del Logical Volume
lvcreate -l 100%FREE -n afcsftp vg_afcsftp
lvs | grep vg_afcsftp
# Creación del filesystem con formato XFS
mkfs.xfs /dev/mapper/vg_afcsftp-afcsftp
# Adición de los puntos de monje a /etc/fstab para que los discos se monten tras levantar la VM
echo "/dev/mapper/vg_afcsftp-afcsftp /AFC/Files                   xfs     defaults        0 0" >>/etc/fstab
systemctl daemon-reload
# Creación del punto de montaje
mkdir /AFC/Files
mount /dev/mapper/vg_afcsftp-afcsftp /AFC/Files
df -h | grep vg_afcsftp
```

Para montar el servicio SFTP independiente, vamos a copiar una serie de ficheros para poder montar el servicio, además de crear los certificados correspondientes.

``` bash
# Creacion del directorio para almacenar los ficheros de configuracion
mkdir -p /AFC/module/afcsftp/conf/

# Copia de los ficheros requeridos para el funcionamiento del SFTP
cp -a /etc/ssh/sshd_config /AFC/module/afcsftp/conf/sshd_afcsftp_config
cp -a /etc/sysconfig/sshd /AFC/module/afcsftp/conf/sshd
cp -a /etc/crypto-policies/back-ends/opensshserver.config /AFC/module/afcsftp/conf/opensshserver.config

# Creacion de certificados para el servicio SFTP
cd /AFC/module/afcsftp/conf/
ssh-keygen -t dsa -f /AFC/module/afcsftp/conf/ssh_host_dsa_key -N ""
ssh-keygen -t ecdsa -f /AFC/module/afcsftp/conf/ssh_host_ecdsa_key -N ""
ssh-keygen -t rsa -f /AFC/module/afcsftp/conf/ssh_host_rsa_key -N ""
ssh-keygen -t ed25519 -f /AFC/module/afcsftp/conf/ssh_host_ed25519_key -N ""

# Cambio de propietario para edirecotrio del modulo afcsftp
chown -R afcmodule: /AFC/module/afcsftp

# Cambio de propietario para el direcotrio del SFTP
chmod -R 750 /AFC/Files
chown  root:afcsftp /AFC
chown -R root:afcsftp /AFC/Files

# Cambio de contexto de seguridad para ser compatible con SELinux
# ----- Fichero de configuracion del SFTP
chcon -Rv -u system_u -t bin_t /AFC/module/afcsftp/conf/sshd_afcsftp_config
# ----- Ficheros de clave privada
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_ecdsa_key
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_ed25519_key
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_rsa_key
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_dsa_key
# ----- Ficheros de clave publica
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_ecdsa_key.pub
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_ed25519_key.pub
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_rsa_key.pub
chcon -Rv -u system_u -t sshd_key_t /AFC/module/afcsftp/conf/ssh_host_dsa_key.pub
```

Una vez tenemos todo lo necesario, podemos pasar a configurar el servicio de SFTP:

``` bash
cd /AFC/module/afcsftp/conf/

# Modificaciones de la configuracion de entradas ya existentes
sed -i '/^Include \/etc\/ssh\/sshd_config.d\/\*.conf/s/^/#/' sshd_afcsftp_config
sed -i 's/#AddressFamily any/AddressFamily inet/' sshd_afcsftp_config
sed -i 's/ListenAddress 0.0.0.0/ListenAddress afcsftp.saepre.logrono.es/' sshd_afcsftp_config
sed -i 's|#HostKey /etc/ssh/ssh_host_rsa_key|HostKey /AFC/module/afcsftp/conf/ssh_host_rsa_key|' sshd_afcsftp_config
sed -i 's|#HostKey /etc/ssh/ssh_host_ecdsa_key|HostKey /AFC/module/afcsftp/conf/ssh_host_ecdsa_key|' sshd_afcsftp_config
sed -i 's|#HostKey /etc/ssh/ssh_host_ed25519_key|HostKey /AFC/module/afcsftp/conf/ssh_host_ed25519_key|' sshd_afcsftp_config
sed -i 's/#SyslogFacility AUTH/SyslogFacility LOCAL3/' sshd_afcsftp_config
sed -i 's/#LogLevel INFO/LogLevel INFO/' sshd_afcsftp_config
sed -i 's/LoginGraceTime 60/LoginGraceTime 30s/' sshd_afcsftp_config
sed -i 's/^MaxAuthTries 4/#MaxAuthTries 4/' sshd_afcsftp_config
sed -i 's/^DisableForwarding yes/#DisableForwarding yes/' sshd_afcsftp_config
sed -i 's/^MaxSessions 4/#MaxSessions 4/' sshd_afcsftp_config
sed -i 's/^HostbasedAuthentication no/#HostbasedAuthentication no/' sshd_afcsftp_config
sed -i 's/^IgnoreRhosts yes/#IgnoreRhosts yes/' sshd_afcsftp_config
sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication yes/' sshd_afcsftp_config
sed -i 's/#GSSAPIAuthentication no/GSSAPIAuthentication no/' sshd_afcsftp_config
sed -i 's/#GSSAPICleanupCredentials yes/GSSAPICleanupCredentials no/' sshd_afcsftp_config
sed -i 's/#PrintMotd yes/PrintMotd no/' sshd_afcsftp_config
sed -i 's/ClientAliveInterval 15/ClientAliveInterval 30/' sshd_afcsftp_config
sed -i 's/ClientAliveCountMax 3/ClientAliveCountMax 0/' sshd_afcsftp_config
sed -i 's/#UseDNS no/UseDNS yes/' sshd_afcsftp_config
sed -i 's|#PidFile /var/run/sshd.pid|PidFile /var/run/afcsftp.pid|' sshd_afcsftp_config
sed -i 's/MaxStartups 10:30:60/MaxStartups 300:10:400/' sshd_afcsftp_config
sed -i 's/#PermitTunnel no/PermitTunnel no/' sshd_afcsftp_config
sed -i 's|Banner /etc/issue.net|#Banner /etc/issue.net|' sshd_afcsftp_config
sed -i 's/^\(\t*Subsystem\tsftp\t\/usr\/libexec\/openssh\/sftp-server\)/#\1/' sshd_afcsftp_config
sed -i '/^DenyUsers nobody$/d' sshd_afcsftp_config

# Adicion de nuevas entradas de configuracion
sed -i '124 i\Subsystem\tsftp\tinternal-sftp' sshd_afcsftp_config
sed -i '68 i\ChallengeResponseAuthentication no' sshd_afcsftp_config
sed -i '122 i\\' sshd_afcsftp_config
sed -i '123 i\# Accept locale-related environment variables' sshd_afcsftp_config
sed -i '124 i\AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES' sshd_afcsftp_config
sed -i '125 i\AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT' sshd_afcsftp_config
sed -i '126 i\AcceptEnv LC_IDENTIFICATION LC_ALL LANGUAGE' sshd_afcsftp_config
sed -i '127 i\AcceptEnv XMODIFIERS' sshd_afcsftp_config

# Comprobacion de que los cambios se han aplicado correctamente
grep "#Include /etc/ssh/sshd_config.d/" sshd_afcsftp_config
grep "AddressFamily inet" sshd_afcsftp_config
grep "ListenAddress afcsftp.saepre.logrono.es" sshd_afcsftp_config
grep "HostKey /AFC/module/afcsftp/conf/ssh_host_rsa_key" sshd_afcsftp_config
grep "HostKey /AFC/module/afcsftp/conf/ssh_host_ecdsa_key" sshd_afcsftp_config
grep "HostKey /AFC/module/afcsftp/conf/ssh_host_ed25519_key" sshd_afcsftp_config
grep "SyslogFacility LOCAL3" sshd_afcsftp_config
grep "LogLevel INFO" sshd_afcsftp_config
grep "LoginGraceTime 30s" sshd_afcsftp_config
grep "#MaxAuthTries 4" sshd_afcsftp_config
grep "#DisableForwarding yes" sshd_afcsftp_config
grep "#MaxSessions 4" sshd_afcsftp_config
grep "AuthorizedPrincipalsFile yes" sshd_afcsftp_config
grep "#HostbasedAuthentication no" sshd_afcsftp_config
grep "#IgnoreRhosts yes" sshd_afcsftp_config
grep "GSSAPIAuthentication no" sshd_afcsftp_config
grep "GSSAPICleanupCredentials no" sshd_afcsftp_config
grep "PrintMotd no" sshd_afcsftp_config
grep "ClientAliveInterval 30" sshd_afcsftp_config
grep "ClientAliveCountMax 0" sshd_afcsftp_config
grep "UseDNS yes" sshd_afcsftp_config
grep "PidFile /var/run/afcsftp.pid" sshd_afcsftp_config
grep "MaxStartups 300:10:400" sshd_afcsftp_config
grep "PermitTunnel no" sshd_afcsftp_config
grep "#Banner /etc/issue.net" sshd_afcsftp_config
grep -P '#Subsystem\tsftp\t/usr/libexec/openssh/sftp-server' sshd_afcsftp_config
grep -P 'Subsystem\tsftp\tinternal-sftp' sshd_afcsftp_config
grep "ChallengeResponseAuthentication no" sshd_afcsftp_config
grep "AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES" sshd_afcsftp_config
grep "AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT" sshd_afcsftp_config
grep "AcceptEnv LC_IDENTIFICATION LC_ALL LANGUAGE" sshd_afcsftp_config
grep "AcceptEnv XMODIFIERS" sshd_afcsftp_config
```

### Creación de la jaula SFTP

!!! warning "Enjaulado SFTP"
    El enjaulado SFTP permite que los usuarios no puedan acceder a los árboles de directorios de los demás usuarios.
    Es importante que el directorio raíz de cada usuario tenga permisos `750` y propiedad `root:afcsftp` o dará error al acceder.
    El resto de directorios y ficheros dentro de cada rama SFTP puede tener permisos diferentes.
    El usuario `sftp_test_user` se crea para probar la conexión SFTP, no es necesario crearlo y se puede borrar cuando no sea necesario.

``` bash
# Creación de un usuario de pruebas
useradd -u 4994 -M -d / -c "SFTP test user" -g 4000 -p 'UcGKQk9@iBIcMJnAQCuB' -s /sbin/nologin sftp_test_user

# Creacion del directorio raiz del usuario y ACL para los permisos de los ficheros y directorios que se creen
mkdir -p /AFC/Files/sftp_test_user
setfacl -PRdm u::rwx,g::rwx,o::- /AFC/Files/sftp_test_user

cd /AFC/module/afcsftp/conf/

# Enjaulado SFTP en rutas segun usuario
sed -i '$a\\' sshd_afcsftp_config
sed -i '$a Match User sftp_test_user' sshd_afcsftp_config
sed -i '$a\        X11Forwarding no' sshd_afcsftp_config
sed -i '$a\        AllowTcpForwarding no' sshd_afcsftp_config
sed -i '$a\        ChrootDirectory /AFC/Files/%u' sshd_afcsftp_config
sed -i '$a\        ForceCommand internal-sftp -u 007' sshd_afcsftp_config

# Enjaulado SFTP en rutas segun grupo
sed -i '$a\\' sshd_afcsftp_config
sed -i '$a Match Group afcsftp' sshd_afcsftp_config
sed -i '$a\        X11Forwarding no' sshd_afcsftp_config
sed -i '$a\        AllowTcpForwarding no' sshd_afcsftp_config
sed -i '$a\        ChrootDirectory /AFC/Files/%u' sshd_afcsftp_config
sed -i '$a\        ForceCommand internal-sftp -u 007' sshd_afcsftp_config
```

### Redirección de los logs

Continuamos la configuración con la redirección de logs al punto de montaje reservado para los logs.

``` bash
# Creacion del fichero vacio para almacenar logs
mkdir -p /AFC/logs/afcsftp
touch /AFC/logs/afcsftp/afcsftp.log

# Cambios de contexto para que rsyslog pueda redirigir logs y sea compatible con SELinux
chcon -t var_log_t /AFC
chcon -t var_log_t /AFC/logs
chcon -Rv -t var_log_t /AFC/logs/afcsftp/

# Cambio de propietario y permisos para el directorio de logs del modulo afcsftp
chown -R afcmodule: /AFC/logs/afcsftp/
chmod 750 /AFC/logs/afcsftp
chmod 660 /AFC/logs/afcsftp/*

# Comprobacion de propietario, permisos y contexto de seguridad
ls -lZ /AFC/logs/afcsftp/afcsftp.log

# Modificacion del rsyslog para redirigir logs
sed -i 's/local2,local3\.\*/local2.*/' /etc/rsyslog.conf
sed -i 's/\(*.info;mail.none;authpriv.none;cron.none\)/\1;local3.none/' /etc/rsyslog.conf
sed -i '102 i\local3.*                                                /AFC/logs/afcsftp/afcsftp.log' /etc/rsyslog.conf

# Reinicio del servicio de rsyslog
systemctl restart rsyslog
```

## Creación del servicio de SFTP

Tras terminar de configurar el SFTP, ya se puede crear el servicio de systemctl que levante el servicio:

``` bash
vi /etc/systemd/system/afcsftp.service
```

``` text
[Unit]
Description=afcsftp service
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target

[Service]
Type=notify
EnvironmentFile=-/AFC/module/afcsftp/conf/opensshserver.config
EnvironmentFile=-/AFC/module/afcsftp/conf/sshd
ExecStart=/usr/sbin/sshd -D -f /AFC/module/afcsftp/conf/sshd_afcsftp_config $OPTIONS $CRYPTO_POLICY
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s
User=root
#Group=afcgroup

[Install]
WantedBy=multi-user.target
```

## Configuración del rotado de logs

Para evitar que se sature la partición dedicada a los logs de los módulos, es necesario implementar un rotado de logs, lo que permite controlar el espacio ocupado y mantener cierto número de ficheros en casi de necesitar revisar logs antiguos

Cada día se realiza una comprobación del tamaño del log. Si es superior a 50MB realiza una copia y la comprime, truncando el fichero original. Se mantiene el fichero original y 5 ficheros rotados.

``` bash
vi /etc/logrotate.d/afcsftp
```

``` text
/AFC/logs/afcsftp/afcsftp.log
{
   copytruncate
   daily
   rotate 5
   compress
   missingok
   size 50M
}
```
