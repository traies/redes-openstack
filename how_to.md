# Instructivo para el deploy de una nube de OpenStack de tres Nodos

Esta guía sigue en mayor parte las [instrucciones en la documentación de OpenStack](https://docs.openstack.org/install-guide/)

## Requisitos previos:
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads),
* [Ubuntu Server 16.04](http://releases.ubuntu.com/16.04/)

## Configuracion inicial
1. Crear 2 maquinas virtuales en VirtualBox:
    * Controller:
		* instanceName: redesOpStackController
		* memory: 4GB
		* storage: 5GB
		* hostname: controller
		* user: traies
		* pass: traies123

    * Compute1:
		* instanceName: redesOpStackCompute1
		* memory: 2GB
		* storage: 10GB
		* hostname: Compute1
		* user: jcaracciolo
		* pass: jcaracciolo123

2 - Setear un 2 NAT Servers en VirtualBox
	nombre: openstack-management
	red: 10.0.0.0/24

	nombre: opnestack-provider
	red: 203.0.113.0/24

	Deshabilitar DHCP en ambas (vamos a usar IPs fijas)
4 - Setear las IPs fijas modificando con sudo vim /etc/network/interfaces
	# The provider network interface
	auto enp0s8
	iface enp0s8 inet manual
		up ip link set dev $IFACE up
		down ip link set dev $IFACE down

	auto enp0s8
	iface enp0s8 inet static
		address 10.0.0.2 // para compute1, 10.0.0.3
		netmask 255.255.255.0
		network 10.0.0..0
		gateway 10.0.0.1
		dns-nameserver 8.8.8.8

5 - Levantar las interfaces:
	ip link show
	sudo ifconfig enp0s8 up

6 - Setear la interfaz del NAT como default gateway
	route add -net 0.0.0.0 gw 10.0.0.1

7 - Probar la conexion entre los dos hosts 

8 - Setear la resolucion de nombres sudo vim /etc/hosts
	10.0.0.2 	controller
	10.0.0.3	compute1
	10.0.0.4	storage

9 - Instalar el servicio chrony para sincronizar los nodos
	En controller:
		sudo apt install chrony
	Luego, sudo vim /etc/chrony/chrony.conf y agregar:  
		server 1.ar.pool.ntp.org iburst
		allow 10.0.0.0/24
	En los demas:
		sudo apt install chrony
		sudo vim /etch/chrony/chrony.conf
			server controller iburst
			y comentar pool 2.debian.pool.ntp.org offline iburst

	chronyc sources para chequear la configuracion

10 - Agregar el repositorio de OpenStack Queebns en todos los nodos
	apt install software-properties-common
	add-apt-repository cloud-archive:queens

	sudo apt update && sudo apt dist-upgrade
	sudo apt install python-openstackclient

11 - En el controller, hay que setear una base de datos MySql
	apt install mariadb-server python-pymysql

	sudo vim /etc/mysql/mariadb.conf.d/99-openstack.cnf
		[mysqld]
		bind-address = 10.0.0.2

		default-storage-engine = innodb
		innodb_file_per_table = on
		max_connections = 4096
		collation-server = utf8_general_ci
		character-set-server = utf8

	sudo service mysql restart
	mysql_secure_installation

12 - Instalar RabitMQ en el controller
	sudo apt install rabbitmq-server
	sudo rabbitmqctl add_user openstack RABBIT_PASS
	Nuestra RABBIT_PASS es bugsbunny

	// Esto setea los permisos
	rabbitmqctl set_permissions openstack ".*" ".*" ".*"

13 - Instalar memcached en el controller
	sudo apt install memcached python-memcache
	sudo vim /etc/memcached.conf
		reemplazar -l 127.0.0.1 con -l 10.0.0.2

	sudo service memchached restart

14 - Instalar Etcd:
	crear usuario:
	sudo groupadd --system etcd
	sudo useradd --home-dir "/var/lib/etcd" --system --shell /bin/false -g etcd etcd

	Crear directorios:
		sudo mkdir -p /etc/etcd
		sudo chown etcd:etcd /etc/etcd
		sudo mkdir -p /var/lib/etcd
		sudo chown etcd:etcd /var/lib/etcd

	Descargar e instalar:
		ETCD_VER=v3.2.7
		sudo rm -rf /tmp/etcd && mkdir -p /tmp/etcd
		sudo curl -L https://github.com/coreos/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
		sudo tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1
		sudo cp /tmp/etcd/etcd /usr/bin/etcd
		sudo cp /tmp/etcd/etcdctl /usr/bin/etcdctl

	Configurar sudo vim /etc/etcd/etcd.conf.yml
		name: controller
		data-dir: /var/lib/etcd
		initial-cluster-state: 'new'
		initial-cluster-token: 'etcd-cluster-01'
		initial-cluster: controller=http://10.0.0.11:2380
		initial-advertise-peer-urls: http://10.0.0.11:2380
		advertise-client-urls: http://10.0.0.11:2379
		listen-peer-urls: http://0.0.0.0:2380
		listen-client-urls: http://10.0.0.11:2379

	Configurar sudo vim /lib/systemd/system/etcd.service
		[Unit]
		After=network.target
		Description=etcd - highly-available key value store

		[Service]
		LimitNOFILE=65536
		Restart=on-failure
		Type=notify
		ExecStart=/usr/bin/etcd --config-file /etc/etcd/etcd.conf.yml
		User=etcd

		[Install]
		WantedBy=multi-user.target

	Finalmente:
		sudo systemctl enable etcd
		sudo systemctl start etcd

## Instalacion de servicios
### Keystone, servicio de identidad.
1. Crear la base de datos para Keystone
	1. ``` sudo mysql ```
	2. ``` MariaDB [(none)]> CREATE DATABASE keystone; ```
	3. ``` MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS'; ```
	4. ```MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS'; ```
	* Nota: usamos ``piedraangular`` como ``` KEYSTONE_DBPASS ```
2. Instalar los paquetes necesarios
	* ``` sudo apt install keystone  apache2 libapache2-mod-wsgi ```
3. Configurar ``` sudo vim /etc/keystone/keystone.conf  ```
	```
	[database] 
	connection = mysql  +pymysql://keystone:piedraangular@controller/keystone 
	
	```
	```
	[token]
	provider = fernet
	```
4. Iniciar la base de datos
	* ``` sudo -i ```
	* ``` su -s /bin/sh -c "keystone-manage db_sync" keystone ```
	* Si hay algun error de permisos en algun archivo, ``` chown keystone:keystone <archivo> ```
5. Iniciar los repositorios de Fernet
	* ``` sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone ```
	* ``` sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone ```
6. Iniciar el servicio
```
sudo keystone-manage bootstrap --bootstrap-password traies123 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
7. Editar Apache ``` sudo vim  /etc/apache2/apache2.conf ```
	* ServerName controller
8. Reiniciar Apache ``` sudo service apache2 restart ```
9. Crear el siguiente script bash para setear las variables de entorno necesaria para registrar servicios ``` ~/.admin-openrc.sh ```
```
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

## Crear Dominios y usuarios en Keystone
1. Crear un dominio de ejemplo: 
	* ```openstack domain create --description "An Example Domain" example```
2. Crear el proyecto service, cada servicio tendra un usuario distinto.
	* ``` openstack project create --domain default --description "Service Project" service ```
3. Crear un proyecto demo para usuarios no privilegiados
	* ```  openstack project create --domain default --description "Demo Project" demo```
4. Crear un usuario demo
	* ```  openstack user create --domain default --password-prompt demo ```
	* Clave: demo
5. Crear el rol usuario 
	* ``` openstack role create user ```
6. Agregar el rol usuario al proyecto demo y al usuario demo
	* ``` openstack role add --project demo --user demo user ```

## Glance, servicio de imagenes
### Registrar el servicio
1. Crear la base de datos para Glance:
	* ``` sudo mysql ```
	* ``` CREATE DATABASE glance ```
	* ``` GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'mirada'; ```
	* ``` GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'mirada'; ```
2. Registrar el usuario, rol y servicio en Keystone
	* ``` openstack user create --domain default --password-prompt glance ```
	* clave: mirada
	* ``` openstack role add --project service --user glance admin ```
	* ``` openstack service create --name glance   --description "OpenStack Image" image ```
3. Registrar los endpoints por los que pueden acceder otros servicios
	* ``` openstack endpoint create --region RegionOne image public http://controller:9292 ```
	* ``` openstack endpoint create --region RegionOne   image internal http://controller:9292 ```
	* ``` openstack endpoint create --region RegionOne image admin http://controller:9292 ```

### Instalar Glance
1. ``` sudo apt install glance ```
2. Editar ```/etc/glance/glance-api.conf```
```
[database]
# ...
connection = mysql+pymysql://glance:mirada@controller/glance
...
[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS
...
[paste_deploy]
# ...
flavor = keystone
...
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
3. Editar ``` /etc/glance/glance-registry.conf ```
```
[database]
# ...
connection = mysql+pymysql://glance:mirada@controller/glance
...
[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
...
```

4. Popular la base de datos 
	* ```sudo -i ```
	* ``` su -s /bin/sh -c "glance-manage db_sync" glance ```
5. Reiniciar
	* ``` sudo service glance-registry restart ```
	* ``` sudo service glance-api restart```

### Agregar una imagen de prueba
1. Usar las credenciales de administrador con ``` . admin-openrc.sh ```
2. Descargar la imagen de cirros ``` wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img ```
3. Publicar la imagen
	* ```  openstack image create "cirros" --file cirros-0.3.5-x86_64-disk.img --disk-format qcow2 --container-format bare --public ```


## Nova, Servicio de Computo
### 
