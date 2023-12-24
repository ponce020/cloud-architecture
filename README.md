# Infraestructura completa
![Diagrama completo de la inraestructura](Adjuntos/InfraestructuraBackground.png)

# Configuraciones previas

## Listado de las redes a utilizar

![Listado de Redes creadas en Virt-Manager](Adjuntos/Redes.png)

Cada red se ha configurado como NAT para tener acceso a internet.

| Nombre de Red | Direccion IPv4 | Descripción |
| - | - | - |
| 01-private-mariadb | 192.168.11.0/24 | Red para sincronización de los datos entre los servidores de mariadb |
| 02-private-maxscale | 192.168.12.0/24 | Red de comunicación entre el balanceador y los nodos de base de datos | 
| 03-database-service | 192.168.13.0/24 | Red pública por la cual habrá comunicación entre los clientes y el clúster | 

## Listado de maquinas virtuales con sistema opertaivo ubuntu 22.04
![Listado de maquians virtuales](Adjuntos/MaquinasVirtuales.png)

Caracteristicas de las maquinas virtuales

| Nombre | Requisitos Minimos |
| - | - |
| Maxscale | 1GB RAM, 2CPU, 15GB Storage |
| Maria-1 | 1GB RAM, 2CPU, 15GB Storage |
| Maria-2 | 1GB RAM, 2CPU, 15GB Storage |
| Maria-3 | 1GB RAM, 2CPU, 15GB Storage |

# Cluster de Base de Datos

![Cluster de Base de Datos](Adjuntos/DatabaseCluster.png)

Asignación de direcciones IP de los nodos:

| Nodo | Interfaz | IP |
| - | - | - |
| maxscale | enp1s0 | 192.168.13.5 |
| maxscale | enp2s0 | 192.168.11.5 |
| maria-1 | enp1s0 | 192.168.12.2 |
| maria-1 | enp2s0 | 192.168.11.2 |
| maria-2 | enp1s0 | 192.168.12.3 |
| maria-2 | enp2s0 | 192.168.11.3 |
| maria-3 | enp1s0 | 192.168.12.4 |
| maria-3 | enp2s0 | 192.168.11.4 |


## Configuracion previa

### Interfaces

Para configurar las direcciones IP de las maquinas hay que editar el archivo de netplan.
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Modificar el archivo para que quede de la siguiente manera teniendo en cuenta la indentación ya que es un archivo yaml. Modificar la direccion IP para cada nodo.
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      addresses:
        - 192.168.11.3/24
      routes:
        - to: default
          via: 192.168.11.1
      nameservers:
        addresses: [192.168.11.1]
    enp7s0:
      addresses:
        - 192.168.12.3/24
      routes:
        - to: 192.168.12.0/24
          via: 192.168.12.1
```

### Hostname
Ya que vamos a estar trabajando con varias maquias virtuales, es recomendado cambiar el hostname a todas para saber en todo momento, en que nodo estamos trabajando.
```bash
#Para el nodo maria-1
sudo hostnamectl set-hostname maria-1 && exec bash

#Para el nodo maria-2
sudo hostnamectl set-hostname maria-2 && exec bash

#Para el nodo maria-3
sudo hostnamectl set-hostname maria-3 && exec bash

#Para el nodo maxscale
sudo hostnamectl set-hostname maxscale && exec bash
```

### Archivo hosts
Para utilizar nombres de los nodos en lugar de direcciones IP en las configuraciones a realizar, los configuramos en el archivo hosts.
```bash
sudo nano /etc/hosts
```
Agregar las siguientes líneas en el archivo hosts de los nodos de mariadb:
```lua
192.168.11.2 maria-1
192.168.11.3 maria-2
192.168.11.4 maria-3
```

Agregar las siguientes líneas en el archivo hosts de maxscale:
```lua
192.168.13.5 maxscale
192.168.12.2 maria-1
192.168.12.3 maria-2
192.168.12.4 maria-3
```

### Configuración del repositorio
Agregar los repositorios de mariadb a la lista de repositorios de Ubuntu en las **4 máquinas virtuales** ya que los paquetes de mariadb y maxscale están en los repositorios de mariadb:

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash
```

Una vez configurado el repositorio actualizar los paquetes e instalar los paquetes necesarios **en los nodos de mariadb**.
```bash
sudo apt update
sudo apt install mariadb-server -y
```
## Configuración del cluster de Mariadb
Este paso se realiza en **los nodos de MariaDB** para configurarlos como un clúster.
Creamos el archivo de configuración del clúster.
```bash
sudo nano /etc/mysql/conf.d/mariadb.cnf
```

El archivo debe tener el siguiente contenido:
```bash
[mysqld]
query_cache_size=0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
query_cache_type=0
bind-address=0.0.0.0
wsrep_on=ON 
wsrep_provider=/usr/lib/galera/libgalera_smm.so

wsrep_cluster_name="ClusterMaria"
wsrep_cluster_address="gcomm://192.168.11.2,192.168.11.3,192.168.11.4"
wsrep_sst_method=rsync
wsrep_node_address="192.168.11.2"
wsrep_node_name="maria-1"
```

Modificar las lineas `wsrep_node_name` y `wsrep_node_address` dependiendo el nodo.

Modificar el archivo del servidor de mariadb para escuchar en la interfaz correspondiente.
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Modificar la line 27 con la IP donde escucha el servicio **en cada uno de los nodos mariadb**:
```php
bind-address            = 192.168.12.2
```

### Inicialización del Cluster
Iniciarlizar el cluster desde **unicamente un nodo del cluster**.
```bash
sudo galera_new_cluster
```
El comando no da salida por consola.

Si no hay ningún error, revisamos el estado del servicio.
```bash
sudo service mariadb status
```
Si le muestra que MariaDB está corriendo, esto quiere decir que arrancó con éxito el clúster, entonces podrá revisar el estado del clúster en cuanto a los nodos con la siguiente orden:  
(solicita la clave de root)

```bash
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

Si todo esta correcto debe mostrar la siguiente salida:
```bash
ponce020@maria-1:~$ sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
Enter password: 
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+
```

Lo anterior indica que solo hay un servidor de mariadb en el clúster el cual es el mismo donde se acaba de iniciaizar. 

Una vez el cluster funcionando reiniciar el servicio **en los otros nodos** para unirse al cluster.
```bash
sudo service mariadb restart
```

### Reinicio del clúster cuando se apagan todas las maquinas
Cuando se apagan todas las máquinas virtuales y se vuelven a encenderlas, el clúster está detenido porque hay que revisar que nodo tiene los últimos cambios o registros. Para eso hay dos formas:
1. Revisa el archivo /var/lib/mysql/grastate.dat
2. Usar el comando galera_recovery

En el primer caso, se debe revisar el archivo grastate.dat en todos los nodos y el cual tenga la línea `safe_to_bootstrap: 1` es el nodo en el cual debemos reiniciar el clúster. En la imagens e puede ver que el nodo maria-2 es el que tiene en 1 el valor de safe_to_bootstrap por lo que es en ese nodo donde se debe levantar el cluster.

![Archivo grastate en los 3 nodos](Adjuntos/Grastate.png)

En el segundo caso, se debe ejecutar el comando `sudo galera_recovery` en todos los nodos y revisar la salida. El nodo que devuelva el mayor número entero es el nodo que debemos usar para iniciar el clúster.

![Comando galera_recovery](Adjuntos/Galera_Recovery.png)

Conociendo el nodo correcto , se debe correr el siguiente comando para levantar el clúster teniendo en cuenta que el clúster no esté en operación.
```bash
sudo galera_new_cluster
```

Una vez se ejecute el comando **en el nodo correcto**, resta reiniciar el servicio de mariadb en **todos los otros nodos** para que se integren de nuevo al clúster.
```bash
sudo service mariadb restart
```

## Configuración de Balanceador
### Instalación de paquetes
Ya habiendo agregado el repositorio de mariadb instalamos maxscale:
```bash
sudo apt install maxscale -y
```

### Configuración del usuario para maxscale
Debe crearse un usuario en la base de datos y configurarlo para que haga el monitoreo y las peticiones al clúster.
Si ya está en funcionamiento el clúster, solo es necesario realizar los siguientes pasos en **un nodo del clúster** y se repicará a los demás:

Ingresamos a la CLI de mariadb/mysql
```bash
sudo mariadb
```

Cree el usuario MaxScale y asigne los permisos sobre las diferentes acciones.
```sql
CREATE USER 'maxscale'@'%' IDENTIFIED BY 'ultrasecreta';
GRANT SELECT ON mysql.user TO 'maxscale'@'%';
GRANT SELECT ON mysql.db TO 'maxscale'@'%';
GRANT SELECT ON mysql.tables_priv TO 'maxscale'@'%';
GRANT SELECT ON mysql.columns_priv TO 'maxscale'@'%';
GRANT SELECT ON mysql.proxies_priv TO 'maxscale'@'%';
GRANT SELECT ON mysql.procs_priv TO 'maxscale'@'%';
GRANT SELECT ON mysql.roles_mapping TO 'maxscale'@'%';
GRANT SHOW DATABASES ON *.* TO 'maxscale'@'%';
GRANT REPLICATION CLIENT on *.* to 'maxscale'@'%';
GRANT ALL ON infinidb_vtable.* TO 'maxscale'@'%';
```

### Configuración del servicio
Editar el archivo `/etc/maxscale.cnf`:

```bash
sudo nano /etc/maxscale.cnf
```

El contenido debe ser el siguiente:

```ini
[maxscale]
threads=auto
log_augmentation = 1
ms_timestamp = 1
syslog = 1
admin_host=0.0.0.0  
admin_secure_gui=false

[maria-1]
type=server
address=192.168.12.2
port=3306
protocol=MariaDBBackend

[maria-2]
type=server
address=192.168.12.3
port=3306
protocol=MariaDBBackend

[maria-3]
type=server
address=192.168.12.4
port=3306
protocol=MariaDBBackend

[Galera-Monitor]
type=monitor
module=galeramon
servers=maria-1,maria-2,maria-3
user=maxscale
password=ultrasecreta
monitor_interval=2000ms

[Read-Con-Route-Galera-Service]
type=service
router=readconnroute
servers=maria-1,maria-2,maria-3
user=maxscale
password=ultrasecreta

[Read-Con-Route-Galera-Listener]
type=listener
service=Read-Con-Route-Galera-Service
protocol=MariaDBClient
port=3306
address=192.168.13.5
```

Inicializar el servicio
```bash
sudo service maxscale start
```

Debe mostrar la siguiente salida de consola:
```bash
ponce020@maxscale:~$ sudo maxctrl list servers
┌─────────┬──────────────┬──────┬─────────────┬─────────────────────────┬──────┬────────────────┐
│ Server  │ Address      │ Port │ Connections │ State                   │ GTID │ Monitor        │
├─────────┼──────────────┼──────┼─────────────┼─────────────────────────┼──────┼────────────────┤
│ maria-1 │ 192.168.12.2 │ 3306 │ 0           │ Slave, Synced, Running  │      │ Galera-Monitor │
├─────────┼──────────────┼──────┼─────────────┼─────────────────────────┼──────┼────────────────┤
│ maria-2 │ 192.168.12.3 │ 3306 │ 0           │ Master, Synced, Running │      │ Galera-Monitor │
├─────────┼──────────────┼──────┼─────────────┼─────────────────────────┼──────┼────────────────┤
│ maria-3 │ 192.168.12.4 │ 3306 │ 0           │ Slave, Synced, Running  │      │ Galera-Monitor │
└─────────┴──────────────┴──────┴─────────────┴─────────────────────────┴──────┴────────────────┘
```

La configuracion realizada cuenta con un panel de administracion web por el puerto 8989. El usuario por defecto es `admin` y la contraseña es `mariadb`.

![Panel Web de Maxscale](Adjuntos/MaxscaleWebPanel.png)

![Visualizacion de la arquitectura del cluster](Adjuntos/MaxscalePanelVisualization.png)

