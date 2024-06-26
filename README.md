# Hosting-Linux

### Proyecto
El objetivo del proyecto es proporcionar una experiencia completa de aprendizaje en administración de servidores y servicios web, a través de una serie de módulos estructurados.
Cada módulo se centra en aspectos específicos, desde la configuración de servidores DNS y web, hasta la implementación de medidas de seguridad, servidores de correo electrónico, herramientas anti-spam y plataformas de colaboración en línea.
El proyecto busca ayudar a adquirir habilidades prácticas y conocimientos sólidos en administración de sistemas, a medida que avanzan a través de los módulos y completan las prácticas recomendadas.

### Diagrama de arquitectura

### Servidores DNS

###### Server DNS Master

Entorno:
- SO: Ubuntu
- Network: 192.168.0.0/24
- IP: 192.168.0.150
- Hostname: ns1.hostingavanzado.intranet

- Configuración de red

Configuramos IP estática

Abrimos archivo de configuración netplan

```
# nano /etc/netplan/00-installer-config.yaml
```

Luego lo editamos dejandolo de la siguiente manera:

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.150/24
      nameservers:
       addresses: [8.8.8.8, 8.8.4.4]
      routes:
          - to: default
            via: 192.168.0.1
```

Aplicamos cambios
```
# netplan apply
```

Nombramos hostname

```
# hostnamectl set-hostname ns1.nameserver.intranet
```
Aplicamos cambios
```
# reboot
```

Instalación de BIND
```
# apt update && apt install bind9 bind9utils
```

Comando `rndc` para mostrar el estado del servidor BIND

```
# rndc status
```

`Output`:
```
version: BIND 9.18.18-0ubuntu0.22.04.2-Ubuntu (Extended Support Version) <id:>
running on localhost: Linux x86_64 5.15.0-101-generic #111-Ubuntu SMP Tue Mar 5 20:16:58 UTC 2024
boot time: Sun, 31 Mar 2024 21:50:12 GMT
last configured: Sun, 31 Mar 2024 21:50:12 GMT
configuration file: /etc/bind/named.conf
CPUs found: 2
worker threads: 2
UDP listeners per interface: 2
number of zones: 102 (97 automatic)
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is OFF
recursive clients: 0/900/1000
tcp clients: 0/150
TCP high-water: 0
server is up and running
```

Configuración BIND

Nos situamos en el fichero de configuración de `bind` que se encuentra en `/etc/bind/named.conf.options` y lo dejamos de la siguiente forma:

```
# nano named.conf.options
```
```
acl allowed {
        192.168.0.0/24
        localhost;
};

options {
        directory "/var/cache/bind";
        recursion no;                   # Disable recursive searchs
        allow-query { allowed; };       # Only hosts in sub-net by ACL
        listen-on { 192.168.0.150; };    # ns1 private ip address -
        allow-transfer { 192.168.0.155 }; # Slave IP address
        forwarders {                            # Forwarders servers
            8.8.8.8;
            8.8.4.4;
        };
        forward only;                   # Disable cache
        dnssec-validation auto;         # If present
        auth-nxdomain no;               # conform to RCF1035


 };
```

Explicación de configuración de parámetros:

1. ACL (Access Control List):
   
  - Se define un ACL llamado "allowed" que contiene dos entradas:
    
      - `192.168.0.0/24`: Una subred que abarca todas las direcciones IP dentro del rango 192.168.0.0 a 192.168.0.255.
    
      - `localhost`: Se refiere a la dirección IP local del servidor.
        
2. Options:

    - `directory "/var/cache/bind"`;`: Especifica el directorio donde BIND almacenará los archivos de cache y otros datos.

    - `recursion no;`: Desahabilita las búsquedas recursivas en este servidor DNS.Esto significa que este servidor no buscará información de otros servidores DNS en nombre de los clientes que lo consulten.
  
    - `allow-query { allowed; }`: Solo se permiten consultas desde hosts que están dentro de la subred especificada en el ACL "allowed".
  
    - `listen-on { 192.168.0.150; };`: Indica que el servidor BIND escuchará en la dirección IP `192.168.0.150`, lo que implica que actuará como servidor DNS primario en esa dirección.
  
    - `allow-transfer { 192.168.0.155 };`: Especifica la direccion IP del servidor esclavo al que se le permitirá transferir zonas de este servidor primario.
  
    - `forwarders { 8.8.8.8; 8.8.4.4; };`: Se especifican los servidores DNS hacia los cuales este servidor BIND reenviará las consultas que no pueda resolver localmente. En este caso, se utilizan los servidores DNS públicos de Gooogle.
  
    - `forward only;`: Configura BIND para reenviar todas las consultas y no almacenar en caché las respuestas.
  
    - `dnssec-validation auto;`: Habilita la validación de DNSSEC, si está presente,permitiendo que BIND determine si debe validar las firmas DNS automáticamente.
  
    - `auth-nxdomain no;`: Configura BIND para no emitir respuestas de tipo "nxdomain"(nombre de dominio no existente) como respuestas autenticadas, conforme a la RFC 1035.

Comando para verificar la configuración general del servicio de ´BIND´.

```
# named-checkconf
```


Creación de la zona `nameserver.intranet` en el servidor primario.

Editamos el fichero `named.conf.local` y lo dejamos de la siguiente forma:

```
zone "nameserver.intranet" {
        type master;
        file "/etc/bind/db.nameserver.intranet";
};
```

Creamos el fichero de registros para la zona nameserver.intranet en el servidor primario, este debe poseer el registro `SOA` y los registros `ns` correspondientes.

```
# touch db.nameserver.intranet
```

Configuración de parámetros en el fichero `db.nameserver.intranet`:

```
@ IN SOA nameserver.intranet. admin.nameserver.intranet. (
                2 ; Serial
                604800 ; Refresh
                86400 ; Retry
                2419200 ; Expire
                604800 ) ; Negative Cache TTL

@ IN NS ns1.nameserver.intranet.
@ IN NS ns2.nameserver.intranet.
; Registros NS
ns1 IN A 192.168.0.150
ns2 IN A 192.168.0.155
```

Explicación de configuración de parámetros:

1. `@ IN SOA nameserver.intranet. admin.namesever.intranet.`:
   
   - `@`: Este símbolo represneta el nombre de la zona en sí misma, en este caso `nameserver.intranet`.
     
   - `IN`: Indica la clase de RR, en este caso, Start of Authority.ESte registro define los parámetros de la zona.
     
   - `nameserver.intranet.`: El nombre del servidor de nombres primario para esta zona.
     
   - `admin.nameserver.intranet.`: La dirección de correo electrónico del administrador de esta zona.
     
   - `( ; Serial`: El número de serie de la zona, que se utiliza para rastrearlas actualizaciones de la zona. Cada vez que se realiza un cambio en la zona, se debe incrementar este número.
     
   - `604800 ; Refresh`: Es el tiempo en segundos que el servidor secundario esperará antes de intentar actualizar la zona desde el servidor primario.
     
   - `86400 ; Retry`: Es el tiempo en segundos que el servidor seuncdario esperará antes de intentar una actualización fallida.
     
   - `2419200 ; Expire`: Es el tiempo en segundos después del cual los servidores secundarios dejarán de responder a las consultas para la zona si no pueden actualizarla desde el servidor primario.
     
   - `604800 ) ; Negative Cache TTL`: Es el tiempo de vida en caché para los registros negativos en esta zona.
  
2. `@ IN NS ns1.nameserver.intranet. y @ IN NS ns1.nameserver.intranet`:

   - Estos son registros de servidor de nombres (NS) que identifican los servidores de nombres autorizados para la zona.
     
   - `@`: De nuevo, representa el nombre de la zona.
     
   - `IN`: La clase de RR.

   - `NS`: El tipo de RR, en este caso, servidor de nombres.
  
   - `ns1.nameserver.intranet.` y `ns2.nameserver.intranet.`: Son los nombres de los servidores de nombres autorizados para esta zona.
  
3. `ns1 IN A 192.168.0.150` y `ns2 IN A 192.168.0.155`:

   - Estos son registros de direcciones de recursos (A) que mapean los nombres de host a direcciones IP.
  
   - `ns1` y `ns2`: Son los nombres de los servidores de nombres.
  
   - `IN`: La clase de RR.
  
   - `A`: El tipo de RR, en este caso, dirección IPv4.
  
   - `192.168.0.150` y `192.168.0.155`: Son las direcciones IP correspondientes a los servidores de nombres `ns1` y `ns2`, respectivamente.

Comando para verificar la configuración de las zonas del servicio `BIND`.

```
# named-checkzone nameserver.intranet db.nameserver.intranet
```


###### Server DNS Slave

Entorno:
- SO: Ubuntu
- Network: 192.168.0.0/24
- IP: 192.168.0.155
- Hostname: ns2.nameserver.intranet

- Configuración de red

Configuración de IP estática

Abrimos archivo de configuración netplan

```
# nano /etc/netplan/00-installer-config.yaml
```

Luego lo editamos dejandolo de la siguiente manera:

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.155/24
      nameserver:
       addresses: [8.8.8.8, 8.8.4.4]
      routes:
          - to: default
            via: 192.168.0.1
```

Aplicamos cambios
```
# netplan apply
```

Nombramos hostname

```
# hostnamectl set-hostname ns2.hostingavanzado.intranet
```

Reiniciamos sistema

```
# reboot
```

Instalación de BIND

```
# apt update && apt install bind9 bind9utils
```

Configuración BIND

Nos situamos en el fichero de configuración de `bind` que se encuentra en `/etc/bind/named.conf.options` y lo dejamos de la siguiente forma:

```
acl allowed {
        192.168.0.0/24;
        localhost;
};

options {
        directory "/var/cache/bind";
        recursion yes;
        allow-query { allowed; };
        listen-on { 192.168.0.155; };
        allow-transfer { none; };
        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
        forward only;
        // };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;

        auth-nxdomain no;


};
```
Creación de la zona `nameserver.intranet` en el servidor secundario.

Editamos el fichero `named.conf.local` y lo dejamos de la siguiente forma:

```
zone "nameserver.intranet" {
        type slave;
        file "db.nameserver.intranet";
        masters { 192.168.0.150; };
};
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
```
Verificamos transferencia de fichero de registro

```
# ls /var/cache/bind
```

`output`:
```
db.nameserver.intranet  managed-keys.bind  managed-keys.bind.jnl
```

### Stack LAMP

Entorno:
- SO: Ubuntu
- Network: 192.168.0.0/24
- IP: 192.168.0.160
- Hostname: apache-01ns1.stacklamp.intranet

- Configuración de red.

Configuramos IP estática.

Abrimos archivo de configuración netplan

```
# nano /etc/netplan/00-installer-config.yaml
```

Luego lo editamos dejandolo de la siguiente manera:

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.160/24
      nameservers:
       addresses: [8.8.8.8, 8.8.4.4]
      routes:
          - to: default
            via: 192.168.0.1
```

Aplicamos cambios.

```
# netplan apply
```

Nombramos hostname.

```
# hostnamectl set-hostname apache-01.stacklamp.intranet
```

Aplicamos cambios.

```
# reboot
```

Instalación de STACK LAMP

```
# apt install apache2 mysql-server php libapache2-mod-php php-mysql
```

Comprobamos el estado del servicio `apache2`

```
# systemctl status apache2
```

`Output:`
```
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor prese>
     Active: active (running) since Mon 2024-04-08 00:27:34 UTC; 2min 0s ago
       Docs: https://httpd.apache.org/docs/2.4/
    Process: 10885 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/S>
   Main PID: 10890 (apache2)
      Tasks: 6 (limit: 2220)
     Memory: 10.6M
        CPU: 36ms
     CGroup: /system.slice/apache2.service
             ├─10890 /usr/sbin/apache2 -k start
             ├─10900 /usr/sbin/apache2 -k start
             ├─10901 /usr/sbin/apache2 -k start
             ├─10902 /usr/sbin/apache2 -k start
             ├─10903 /usr/sbin/apache2 -k start
             └─10904 /usr/sbin/apache2 -k start
```

Comprobamos estado del servicio `MySQL`

```
# systemctl status mysql
```

`Output:`
```
● mysql.service - MySQL Community Server
     Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset:>
     Active: active (running) since Mon 2024-04-08 00:27:04 UTC; 6min ago
   Main PID: 3445 (mysqld)
     Status: "Server is operational"
      Tasks: 37 (limit: 2220)
     Memory: 365.5M
        CPU: 2.713s
     CGroup: /system.slice/mysql.service
             └─3445 /usr/sbin/mysqld
```

Comprobamos la instalación de `PHP`

```
# php -v
```



