# Docker --> squid --> privoxy --> TOR --> https://hub.docker.com/
 
## Como opción número uno podemos instalar en nuestro mismo servidor (en el cual estamos corriendo Docker) **squid + privoxy + tor** y luego configurar docker via proxy.

### 1. Desde Ubuntu o Debian:

	$ sudo apt-get install squid3 privoxy tor -y 

### 2. Desde Fedora:
	$ sudo dnf install squid3 privoxy tor -y

... al concluir el proceso de instalacion pasamos a configurar estos tres amigos que nos acabamos de instalar.

### Configuracion de squid:
	$ cat /etc/squid3/squid.conf
	acl manager proto cache_object
	acl localhost src 127.0.0.1/32 ::1
	acl to_localhost dst 127.0.0.0/8 0.0.0.0/32 ::1
	acl ftp proto FTP
	
	acl SSL_ports port 443
	acl Safe_ports port 80          # http
	acl Safe_ports port 21          # ftp
	acl Safe_ports port 443         # https
	acl Safe_ports port 70          # gopher
	acl Safe_ports port 210         # wais
	acl Safe_ports port 1025-65535  # unregistered ports
	acl Safe_ports port 280         # http-mgmt
	acl Safe_ports port 488         # gss-http
	acl Safe_ports port 591         # filemaker
	acl Safe_ports port 777         # multiling http
	acl Safe_ports port 3128
	acl CONNECT method CONNECT

	http_access allow manager localhost
	http_access deny manager

	http_access deny !Safe_ports

	http_access deny CONNECT !SSL_ports
	
	http_access allow localhost

	http_port 3128

	hierarchy_stoplist cgi-bin ?

	cache_peer 127.0.0.1 parent 8118 7 no-query no-digest
	
	coredump_dir /var/spool/squid

	refresh_pattern ^ftp:           1440    20%     10080
	refresh_pattern ^gopher:        1440    0%      1440
	refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
	refresh_pattern .               0       20%     4320

	httpd_suppress_version_string on
	forwarded_for off
	always_direct allow ftp
	never_direct allow all

### Configuracion de Privoxy:
	$ sudo cat /etc/privoxy/conf
	forward-socks4a / 127.0.0.1:9050 .
	confdir /etc/privoxy
	logdir /var/log/privoxy
	actionsfile default.action   
	actionsfile user.action      
	filterfile default.filter

	logfile logfile

	debug   4096 
	debug   8192 

	user-manual /usr/share/doc/privoxy/user-manual
	listen-address  127.0.0.1:8118
	toggle  1
	enable-remote-toggle 0
	enable-edit-actions 0
	enable-remote-http-toggle 0
	buffer-limit 4096

### Configuracion de TOR:
	$ sudo cat /etc/tor/torrc
	SocksPort 9050 # what port to open for local application connections
	SocksBindAddress 127.0.0.1 # accept connections only from localhost
	AllowUnverifiedNodes middle,rendezvous
	Log notice syslog
	# En caso de tener que configurar tor para que salga por un proxy descomentar y modificar las siguientes lineas
	# Proxy settings
	#HTTPProxy 192.168.1.1:3128
	#HTTPProxyAuthenticator username:password
	#HTTPSProxy 192.168.1.1:3128
	#HTTPSProxyAuthenticator username:password

### Iniciamos los servicios:
	$ sudo systemctl start squid3
	$ sudo systemctl start privoxy
	$ sudo systemctl start tor

### Luego de esto pasamos a configurar las opciones de proxy de Docker:
	$ sudo mkdir -p /etc/systemd/system/docker.service.d
	$ sudo touch /etc/systemd/system/docker.service.d/http-proxy.conf	# esto habilita la variable HTTP_PROXY para HTTPS_PROXY creariamos el archivo https-proxy.conf

*...luego agregamos las siguientes líneas en etc/systemd/system/docker.service.d/http-proxy.conf*

	$ sudo cat etc/systemd/system/docker.service.d/http-proxy.conf
	[Service]
	Environment="HTTP_PROXY=http://127.0.0.1:3128/"

*Reiniciamos los procesos:*

	$ sudo systemctl daemon-reload && sudo systemctl restart docker

*Verificamos que la variable HTTP_PROXY está configurada correctamente:*

	$ systemctl show --property=Environment docker
	Environment=HTTP_PROXY=http://127.0.0.1:3128/

*Con esto ya tenemos Docker proxyficado a traves de Squid+Privoxy+TOR y estamos listos para hacer un docker pull sin problemas.*
