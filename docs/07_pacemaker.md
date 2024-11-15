# Guía de instalación de PaceMaker

> [!WARNING]
> Este manual se ha diseñado empleando una forma de estructurar los directorios y el particionado muy concretas, no implica que sea la única forma de proceder.    
> * La instalación se realiza sobre un sistema operativo Oracle Linux 9.
> * La máquina virtual consta de un particionado base de aproximadamente `56GB`, quedando libres `44GB`.
> * La máquina virtual cumple con al menos el 80% de [CIS Oracle Linux 9 Benchmark v2.0.0](https://learn.cisecurity.org/l/799323/2024-06-12/4tq143)
> * La instalación es compatible con SELinux en modo `enforcing`.
> * La instalación es compatible con el uso del servicio `firewalld`.
> * Esta guía presenta la forma de configurar pacemaker para los siguientes casos:
>   + **Clúster de PaceMaker**: uso habitual con al menos dos nodos y quorum.
>   + **Monoclúster de PaceMaker**: funciona como un `systemctl` potenciado.

## Instalación de un monoclúster

Esta es la forma más simple de realizar la instalación de PaceMaker, permitiendo monitorizar el estado de los servicios y reiniciarlos automáticamente si detecta que han fallado.

```bash
# Adicion de los repositorios necesarios.
dnf config-manager --enable ol8_appstream ol8_baseos_latest ol8_addons
# Instalación de los paquetes
dnf -y install pcs pacemaker resource-agents fence-agents-all
```

Una vez instalados los servicios, habilitaremos los servicios de `systemctl` para que levanten junto a la máquina virtual en caso:

``` bash
# Habilitacion de servicios
systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker
```

Tambien configuraremos el servicio `pcsd` para que escuche por cualquier IP en el puerto `2224`. Una vez aplicados los cambios, podemos levantar el servicio:

``` bash
sed -i 's/^#PCSD_BIND_ADDR=.*/PCSD_BIND_ADDR='"'"'0.0.0.0'"'"'/' /etc/sysconfig/pcsd
grep PCSD_BIND_ADDR /etc/sysconfig/pcsd

sed -i 's/^#PCSD_PORT=2224/PCSD_PORT=2224/' /etc/sysconfig/pcsd
grep PCSD_PORT /etc/sysconfig/pcsd

# Inicio del servicio pcsd
systemctl start pcsd
```

Junto a la instalación de servicios, se habrá creado el usuario `hacluster`. Para que pacemaker funcione adecuadamente, es necesario asignar una password segura.

``` bash
passwd hacluster
```

Para crear el clúster de forma cómoda, almacenaremos en la infomración en las siguientes variable de entorno 
* `NODO1` la IP o el hostname de la máquina virtual donde se quiere montar el monoclúster 
* `CLUSTER_NAME` el nombre que queremos asignar al monoclúster

``` bash
export NODO1=<IP/FQDN NODO 1>
export CLUSTER_NAME=<CLUSTER NAME>
```

El siguiente paso es crear el monoclúster, para ello utilizaremos el siguiente comando para autenticar el usuario **hacluster** mediante la password creada anteriormente:

``` bash
pcs host auth $NODO1 -u hacluster
```

Una vez autenticado, se ejecutarán los siguientes comandos para terminar de configurar el monoclúster:

``` bash
pcs cluster setup $CLUSTER_NAME $NODO1
pcs cluster start --all
pcs cluster enable --all
pcs property set stonith-enabled=false
```

Si todo ha salido correctamente, se podrá visualizar el estado del clúster con el comando:

``` bash
pcs status cluster
```


## Instalación de un clúster

### Configuración del nodo de quorum

El nodo de quorum es importante en un clúster para que la pareja de dos nodos sea capaz de determinar cual tomará los recursos compartidos.

``` bash
# Adicion de repositorios
dnf config-manager --enable ol9_appstream ol9_baseos_latest ol9_addons
# Instalación de paquetes
dnf -y install pcs corosync-qnetd
```

Una vez instalados los servicios, habilitaremos los servicios de `systemctl` para que levanten junto a la máquina virtual en caso:

``` bash
# Habilitacion de servicios
systemctl enable pcsd
```

Tambien configuraremos el servicio `pcsd` para que escuche por cualquier IP en el puerto `2224`. Una vez aplicados los cambios, podemos levantar el servicio:

``` bash
sed -i 's/^#PCSD_BIND_ADDR=.*/PCSD_BIND_ADDR='"'"'0.0.0.0'"'"'/' /etc/sysconfig/pcsd
grep PCSD_BIND_ADDR /etc/sysconfig/pcsd

sed -i 's/^#PCSD_PORT=2224/PCSD_PORT=2224/' /etc/sysconfig/pcsd
grep PCSD_PORT /etc/sysconfig/pcsd

# Inicio del servicio pcsd
systemctl start pcsd
```

Junto a la instalación de servicios, se habrá creado el usuario `hacluster`. Para que pacemaker funcione adecuadamente, es necesario asignar una password segura.

``` bash
passwd hacluster
```

Como el quorum va a conectar con diferentes máquinas virtuales, es necesario habilitar los correspondientes puertos en el firewall:

```bash
# Puerto para la autenticacion del usuario hacluster mediante corosync
firewall-cmd --add-port=5403/udp --permanent
# Puertos para la comunicacion del cluter
firewall-cmd --add-service=high-availability --permanent
# Reinicio del firewall para que actualice los cambios aplicados
firewall-cmd --reload
```


### Configuración de los nodos en clúster




## Configuraciones de recursos de clúster

### Configuración del grupo de recursos


### Restricciones para los recursos

