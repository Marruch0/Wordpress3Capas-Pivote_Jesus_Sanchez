# Índice
- [Introducción](#introducción)
  - [Arquitectura](#arquitectura)
  - [Requisitos de la Infraestructura](#requisitos-de-la-infraestructura)
  - [Servicios](#servicios)
  - [Detalles de Configuración](#detalles-de-configuración)
- [Estructura AWS](#estructura-aws)
  - [VPC](#vpc)
  - [Subredes](#subredes)
    - [Subred para el Balanceador](#subred-para-el-balanceador)
    - [Subred para los Servidores Web y NFS](#subred-para-los-servidores-web-y-nfs)
    - [Subred para la Base de Datos](#subred-para-la-base-de-datos)
    - [Subred para el Pivote](#subred-para-el-pivote)
  - [Tablas de Enrutamiento](#tablas-de-enrutamiento)
    - [Tabla de Enrutamiento Privada](#tabla-de-enrutamiento-privada)
    - [Tabla de Enrutamiento Pública](#tabla-de-enrutamiento-pública)
    - [Tabla de Enrutamiento para el Pivote](#tabla-de-enrutamiento-para-el-pivote)
  - [Gateway](#gateway)
  - [Grupos de Seguridad](#grupos-de-seguridad)
  - [ACL](#acl)
  - [Instancias](#instancias)
- [Balanceador](#balanceador)
- [Servidor NFS](#servidor-nfs)
- [Servidores Webs](#servidores-webs)
- [Servidor Base de Datos](#servidor-base-de-datos)
- [Comprobación del Funcionamiento](#comprobación-del-funcionamiento)
# Introducción

Esta actividad consiste en **desplegar un CMS Wordpress en AWS**, configurándolo para que sea **escalable y tenga alta disponibilidad**. Se usará una **arquitectura en tres capas** para organizar los componentes.

### Arquitectura

1. **Capa 1 (pública):** Balanceador de carga con un servidor Apache.
2. **Capa 2 (privada):**
    - Dos servidores Apache para el Backend.
    - Un servidor NFS que almacenará los archivos del CMS y los compartirá con los servidores Backend.
3. **Capa 3 (privada):** Servidor de base de datos (MySQL o MariaDB).

### Requisitos de la Infraestructura

- **Acceso desde el exterior** solo permitido a la capa pública (balanceador de carga).
- **Restricción de conectividad directa** entre las capas 1 y 3.
- Uso de **grupos de seguridad** para proteger las máquinas.

### Servicios

- **Balanceador de carga (Capa 1):** Servidor web Apache.
- **Backend (Capa 2):** Dos servidores web Apache conectados al servidor NFS.
- **BBDD (Capa 3):** Base de datos MySQL o MariaDB.

### Detalles de Configuración

- Cada máquina tendrá un **nombre (hostname)** personalizado con el nombre del alumno, como "BalanceadorCarlosGlez".
- **Wordpress debe estar personalizado**, mostrando el nombre del alumno.
- **Seguridad:** Se recomienda usar un sitio web HTTPS asociado a un dominio público que apunte a una IP elástica de AWS.

---

# Estructura AWS

Lo primero que tenemos que hacer es crear la infraestructura en AWS.
El esquema incluye nombres que solo se ven en modo claro de github
![EsquemaAWspng](https://github.com/user-attachments/assets/9b9c1230-f835-42b3-9937-235cece4ee91)

Una vez sabemos que estructura vamos a seguir pasaremos a crearla en vagrant.

—Es importante que al crear todo lo necesario se le de un nombre descriptivo—

## VPC

Para ello, primero nos dirigimos al apartado de VPC en AWS.

Aquí crearemos la VPC que utilizaremos para este proyecto.

En mi caso, la configuración será la siguiente:

- **IPv4:** 192.168.30.0/24
- **Nombre:** Wordpress-VPC

---

## Subredes

Una vez creada la VPC, procederemos a la creación de subredes.

En este caso, se crearán **4 subredes**.

### Subred para el Balanceador

Primero crearemos una subred para el balanceador. La configuración es:

- **IPv4:** 192.168.30.0/26
- **Nombre:** public-Wordpress-SUB

### Subred para los Servidores Web y NFS

A continuación, crearemos la subred que se utilizará para los dos servidores web y el servidor NFS. La configuración es:

- **IPv4:** 192.168.30.64/26
- **Nombre:** private-Wordpress-SUB

### Subred para la Base de Datos

Ahora crearemos una subred para el servidor de base de datos. Su configuración será:

- **IPv4:** 192.168.30.128/26
- **Nombre:** private2-Wordpress-SUB

### Subred para el Pivote

Un pivote es un nodo intermediario que permite acceder a redes internas o máquinas restringidas en una arquitectura de red.

Configuración de su subred:

- **IPv4:** 192.168.30.192/26
- **Nombre:** pivote-Wordpress-SUB

## Tablas de Enrutamiento

Ahora procederemos a crear las tablas de enrutamiento.

### Tabla de Enrutamiento Privada

La primera tabla será la privada, en la cual asignaremos las siguientes subredes:

- **private-Wordpress-SUB**
- **private2-Wordpress-SUB**

El nombre asignado será: **private-tablaenrutamiento**

### Tabla de Enrutamiento Pública

La siguiente tabla será la pública, destinada al balanceador. Se asignará la subred:

- **public-Wordpress-SUB**

El nombre asignado será: **public-tablaenrutamiento**

### Tabla de Enrutamiento para el Pivote

Finalmente, crearemos la tabla de enrutamiento para el pivote, a la que asignaremos la subred:

- **pivote-Wordpress-SUB**

El nombre asignado será: **pivote-tablaenrutamiento**

## Gateway

Ahora crearemos una gateway, que será asignada a las tablas de enrutamiento **public-tablaenrutamiento** y **pivote-tablaenrutamiento**.

Antes de asignarla, le daremos el nombre: **gateway-Wordpress**

## Grupos de Seguridad

Antes de crear las instancias, configuraremos los grupos de seguridad. Cada grupo tendrá sus propias reglas de entrada.

Los grupos de seguridad a crear son:

- **ProxyBalanceador-Wordpress**
    
    
    | **Tipo** | **Protocolo** | **Intervalo de puertos** | **Origen** | **Descripción** |
    | --- | --- | --- | --- | --- |
    | HTTP | TCP | 80 | 0.0.0.0/0 | Permitir tráfico HTTP desde cualquier lugar |
    | SSH | TCP | 22 | 192.168.30.192/26 | Permitir SSH desde la máquina pivote |
    | HTTPS | TCP | 443 | 0.0.0.0/0 | Permitir tráfico HTTPS desde cualquier lugar |
- **ServidorWeb-Wordpress**
    
    
    | **Tipo** | **Protocolo** | **Intervalo de puertos** | **Origen** | **Descripción** |
    | --- | --- | --- | --- | --- |
    | HTTP | TCP | 80 | 192.168.30.0/26 | Permitir tráfico HTTP al balanceador |
    | NFS | TCP | 2049 | 192.168.30.64/26 | Permitir NFS a la máquina NFS |
    | SSH | TCP | 22 | 192.168.30.192/26 | Permitir SSH mediante la máquina pivote |
    | MSSQL | TCP | 1433 | 192.168.30.128/26 | Permitir puerto 1433 para la base de datos |
- **NFS-Wordpress**
    
    
    | **Tipo** | **Protocolo** | **Intervalo de puertos** | **Origen** | **Descripción** |
    | --- | --- | --- | --- | --- |
    | SSH | TCP | 22 | 192.168.30.192/26 | Permitir SSH mediante la máquina |
    | NFS | TCP | 2049 | 192.168.30.64/26 | Permitir tráfico NFS a todo el rango de máquinas web |
- **BaseDatos-Wordpress**
    
    
    | **Tipo** | **Protocolo** | **Intervalo de puertos** | **Origen** | **Descripción** |
    | --- | --- | --- | --- | --- |
    | MYSQL/Aurora | TCP | 3306 | 192.168.30.64/26 | Permitir tráfico del puerto 3306 al rango de los servidores web |
    | SSH | TCP | 22 | 192.168.30.192/26 | Permitir SSH desde máquina pivote |
- **Pivote-Wordpress**
    
    
    | **Tipo** | **Protocolo** | **Intervalo de puertos** | **Origen** | **Descripción** |
    | --- | --- | --- | --- | --- |
    | SSH | TCP | 22 | 0.0.0.0/0 | Permitir SSH de pivote a todas las máquinas del rango 30.0 |

## ACL

Ahora configuraremos las **ACLs de salida**, ya que en la entrada hemos definido las reglas mediante los **grupos de seguridad**. Sin embargo, en las **ACLs de entrada** se replicarán las mismas reglas que en los grupos de seguridad de entrada, con algunas excepciones, como los **puertos efímeros** y los **puertos utilizados por la base de datos**.

Las **ACL** son:

- **ProxyBalanceador**
    
    Entrada:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Origen** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | SSH (22) | TCP (6) | 22 | 192.168.30.192/26 | Permitir |
    | 2 | HTTP (80) | TCP (6) | 80 | 0.0.0.0/0 | Permitir |
    | 3 | HTTPS (443) | TCP (6) | 443 | 0.0.0.0/0 | Permitir |
    | 4 | TCP personalizado | TCP (6) | 32768 - 60999 | 0.0.0.0/0 | Permitir |
    - Los puertos de 32768 - 60999 es un rango de  puertos efímeros que tendremos que permitir.
    
    Salida:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Destino** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | TCP personalizado | TCP (6) | 32768 - 65535 | 0.0.0.0/0 | Permitir |
    | 2 | HTTP (80) | TCP (6) | 80 | 0.0.0.0/0 | Permitir |
    | 3 | HTTPS (443) | TCP (6) | 443 | 0.0.0.0/0 | Permitir |
    | 4 | DNS (TCP) (53) | TCP (6) | 53 | 0.0.0.0/0 | Permitir |
    | 5 | TCP personalizado | TCP (6) | 5222 - 32768 | 0.0.0.0/0 | Permitir |
    
    El rango de puertos **5222 - 32768** es algo que he tenido que habilitar porque, sin ellos, la conexión en mi casa no funciona. Sin embargo, en casa de mi pareja, de Víctor y en clase sí funciona sin problemas. De todas formas, he decidido mencionarlo por si acaso en tu caso tampoco puedes acceder como me ocurre a mí.
    
- **ServidoresWebsyNfs**
    
    Entrada:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Origen** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | SSH (22) | TCP (6) | 22 | 192.168.30.192/26 | Permitir |
    | 2 | HTTP (80) | TCP (6) | 80 | 192.168.30.0/26 | Permitir |
    | 3 | MS SQL (1433) | TCP (6) | 1433 | 192.168.30.128/26 | Permitir |
    | 4 | TCP personalizado | TCP (6) | 32768 - 60999 | 192.168.30.128/26 | Permitir |
    
    Salida:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Destino** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | TCP personalizado | TCP (6) | 32768 - 60999 | 0.0.0.0/0 | Permitir |
    | 2 | HTTP (80) | TCP (6) | 80 | 192.168.30.0/26 | Permitir |
    | 3 | MySQL/Aurora (3306) | TCP (6) | 3306 | 192.168.30.128/26 | Permitir |
- **BasedeDatos**
    
    Entrada:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Origen** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | SSH (22) | TCP (6) | 22 | 192.168.30.192/26 | Permitir |
    | 2 | MySQL/Aurora (3306) | TCP (6) | 3306 | 192.168.30.64/26 | Permitir |
    | 3 | TCP personalizado | TCP (6) | 32768 - 60999 | 192.168.30.64/26 | Permitir |
    
    Salida:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Destino** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | MySQL/Aurora (3306) | TCP (6) | 3306 | 192.168.30.64/26 | Permitir |
    | 2 | TCP personalizado | TCP (6) | 32768 - 60999 | 0.0.0.0/0 | Permitir |
    | 3 | SSH (22) | TCP (6) | 22 | 192.168.30.192/26 | Permitir |
- **Pivote**
    
    Entrada:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Origen** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | SSH (22) | TCP (6) | 22 | 0.0.0.0/0 | Permitir |
    | 2 | TCP personalizado | TCP (6) | 32768 - 60999 | 0.0.0.0/0 | Permitir |
    
    Salida:
    
    | **Número de regla** | **Tipo** | **Protocolo** | **Rango de puertos** | **Destino** | **Permitir/Denegar** |
    | --- | --- | --- | --- | --- | --- |
    | 1 | SSH (22) | TCP (6) | 22 | 0.0.0.0/0 | Permitir |
    | 2 | TCP personalizado | TCP (6) | 32768 - 60999 | 0.0.0.0/0 | Permitir |
    | 3 | TCP personalizado | TCP (6) | 5222 - 32768 | 0.0.0.0/0 | Permitir |

## Instancias

Una vez que tenemos la estructura de red creada, procederemos a crear las instancias y a asociarlas con sus respectivas subredes y grupos de seguridad, previamente definidos con nombres descriptivos.

Yo tengo las siguientes máquinas configuradas:

- **ProxyBalanceador-Wordpress_JesusSanchez**
- **ServidorWeb1-Wordpress_JesusSanchez**
- **ServidorWeb2-Wordpress_JesusSanchez**
- **NFS-Wordpress_JesusSanchez**
- **BaseDatos-Wordpress_JesusSanchez**
- **Pivote-Wordpress_JesusSanchez**

---

Una vez completada toda la infraestructura, procederemos a instalar todo lo necesario:

# Balanceador

Este es el script de aprovisionamiento y configuración para el balanceador:

```bash
#!/bin/bash

# Actualizar paquetes
sudo apt update

# Instalar Apache
sudo apt install apache2 -y

# Habilitar módulos de Apache necesarios para el balanceo
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests

# Reiniciar Apache para cargar los módulos
sudo systemctl restart apache2

# Crear el archivo de configuración del balanceador (HTTP)
# Partimos del 000-default.conf para mantener algunas directivas, luego lo sobreescribimos
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/load-balancer.conf

# Sobrescribimos el contenido de load-balancer.conf
sudo bash -c 'cat <<EOF > /etc/apache2/sites-available/load-balancer.conf
<VirtualHost *:80>
    ServerName wordpressjesusasir.ddns.net
    <Proxy "balancer://mycluster">
        BalancerMember "http://192.168.30.70"
        BalancerMember "http://192.168.30.73"
    </Proxy>

    ProxyPass "/" "balancer://mycluster/"
    ProxyPassReverse "/" "balancer://mycluster/"
</VirtualHost>
EOF'

# Habilitar el nuevo sitio y deshabilitar el default
sudo a2ensite load-balancer.conf
sudo a2dissite 000-default.conf
sudo systemctl restart apache2

# Habilitar módulos SSL, rewrite y headers
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2enmod headers
sudo systemctl restart apache2

# Instalar Certbot
sudo apt install certbot python3-certbot-apache -y

# Generar certificado (interactivo, ajustar si se quiere automático)
sudo certbot --apache -d wordpressjesusasir.ddns.net

# Crear archivo de configuración SSL (HTTPS)
sudo bash -c 'cat <<EOF > /etc/apache2/sites-available/load-balancer-le-ssl.conf
<VirtualHost *:80>
    ServerName wordpressjesusasir.ddns.net
    ErrorLog /var/log/apache2/error.log
    CustomLog /var/log/apache2/access.log combined
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName wordpressjesusasir.ddns.net
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/wordpressjesusasir.ddns.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/wordpressjesusasir.ddns.net/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf

    <Proxy "balancer://mycluster">
        BalancerMember "http://192.168.30.70"
        BalancerMember "http://192.168.30.73"
    </Proxy>

    ProxyPass "/" "balancer://mycluster/"
    ProxyPassReverse "/" "balancer://mycluster/"
    ProxyPreserveHost On

    RequestHeader set X-Forwarded-Proto "https" env=HTTPS
    RequestHeader set X-Forwarded-Host "wordpressjesusasir.ddns.net"

    ErrorLog /var/log/apache2/error.log
    CustomLog /var/log/apache2/access.log combined
</VirtualHost>
</IfModule>
EOF'

# Deshabilitar el sitio HTTP sin SSL y activar el sitio con SSL
sudo a2dissite load-balancer.conf
sudo a2ensite load-balancer-le-ssl.conf
sudo systemctl reload apache2

# Cambiar el hostname
sudo hostnamectl set-hostname BalanceadorJesusSanchez

echo "Aprovisionamiento completado."
```

Este script:

1. Actualiza el sistema, instala Apache y habilita los módulos necesarios.
2. Crea la configuración para el balanceador en HTTP y luego para HTTPS con Let’s Encrypt.
3. Ejecuta `certbot` para obtener los certificados SSL (requiere interacción).
4. Ajusta la configuración SSL y redirige el tráfico a HTTPS.
5. Cambia el hostname del sistema.

---

# Servidor NFS

Este es el script de aprovisionamiento y configuración del servidor nfs:

```bash
#!/bin/bash
# Actualizar paquetes
sudo apt update -y

# Instalar el servidor NFS
sudo apt install nfs-kernel-server -y

# Crear el directorio NFS para WordPress
sudo mkdir -p /var/nfs/wordpress
sudo chown nobody:nogroup /var/nfs/wordpress/

# Añadir la línea de exportación en /etc/exports
echo "/var/nfs/wordpress      192.168.30.64/26(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports

# Aplicar la configuración de NFS
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

# Descargar la última versión nocturna de WordPress
sudo wget https://wordpress.org/nightly-builds/wordpress-latest.zip -O /tmp/wordpress-latest.zip

# Instalar unzip si no está instalado
sudo apt install unzip -y

# Descomprimir WordPress en el directorio NFS
sudo unzip /tmp/wordpress-latest.zip -d /var/nfs/

# Ajustar permisos en el directorio WordPress
sudo chown nobody:nogroup /var/nfs/wordpress/*
sudo chmod 755 /var/nfs/wordpress/*

# Crear el archivo wp-config.php sin usar nano
sudo bash -c 'cat <<EOF > /var/nfs/wordpress/wp-config.php
<?php
/** The database collate type. Don\'t change this if in doubt. */
define( \'DB_COLLATE\', \'\' );

/**#@+
 * Authentication unique keys and salts.
 */
define( \'AUTH_KEY\',         \'`mx/7y3H:[1y/5ME6&2`u401_&FN4Q}@v7S>0RZCbg747zI57r8x5w|;q;qmj.B6\' );
define( \'SECURE_AUTH_KEY\',  \'fxojfx^MNK=UI&p+^~0Z 3-BW,iW[3lT=KVYzUZw+bI#6<@}eR aO,)MvFP/*`Xb\' );
define( \'LOGGED_IN_KEY\',    \'Q^2vnz]c\$_[U:{G0&.l(kr AO3TgZ7^I$2e4|rr{;>NIsc1RtQwG_%{$wufF^M@X\' );
define( \'NONCE_KEY\',        \'2SCkinOuKDXv?LS3+UVY!+/@h^q<JB]MfU4rzew.F,y!?URY7M,lv1mL~Elb.p:Z\' );
define( \'AUTH_SALT\',        \'vsI}qeae yS`Qj((Xg~|+f~C_jNuQ@V0v4uo(k6f.01BJ)ID3%#Qwd>i-Qe4?m/A\' );
define( \'SECURE_AUTH_SALT\', \']00%Y8:nM21ysf9[X7=2]5=;3YZb=M@-=w!#}m{zZQQ/=`Qq/r`J|-oQC,4WAH+p\' );
define( \'LOGGED_IN_SALT\',   \'Ef#3_zxHX0nn+shplY0g8; ?_=WQKWbj4phLoOhr-<Sya_u]Ej/n?j!XtVea_nL9\' );
define( \'NONCE_SALT\',       \'q<;}eMyL099#T(2o}=[Jn[vGEkiOnn}voJ;7kqZN/&g>xO3U<<0=sqUd76^Vjs7l\' );

/**#@-*/

/**
 * WordPress database table prefix.
 */
$table_prefix = \'wp_\';

/**
 * Para desarrolladores: modo de depuración de WordPress.
 */
define( \'WP_DEBUG\', false );
define( \'WP_HOME\', \'https://wordpressjesusasir.ddns.net/\' );
define( \'WP_SITEURL\', \'https://wordpressjesusasir.ddns.net/\' );

/* Añadir valores personalizados aquí */

/* ¡Eso es todo! Deja de editar. ¡Feliz blogging! */

/** Ruta absoluta al directorio de WordPress. */
if ( ! defined( \'ABSPATH\' ) ) {
    define( \'ABSPATH\', __DIR__ . \'/\' );
}

/** Configura las variables de WordPress e incluye los ficheros. */
require_once ABSPATH . \'wp-settings.php\';

if (isset(\$_SERVER[\'HTTP_X_FORWARDED_PROTO\']) && \$_SERVER[\'HTTP_X_FORWARDED_PROTO\'] === \'https\') {
    \$_SERVER[\'HTTPS\'] = \'on\';
}

if (isset(\$_SERVER[\'HTTP_X_FORWARDED_HOST\'])) {
    \$_SERVER[\'HTTP_HOST\'] = \$_SERVER[\'HTTP_X_FORWARDED_HOST\'];
}
EOF'
# Cambiar el hostname
sudo hostnamectl set-hostname NFSJesusSanchez

echo "Aprovisionamiento completado."
```

Este script:

- Actualiza el sistema e instala el servidor NFS.
- Crea el directorio compartido para WordPress, lo exporta a la red y reinicia el servidor NFS.
- Descarga y descomprime WordPress en `/var/nfs/wordpress`.
- Establece los permisos adecuados y crea el fichero `wp-config.php` usando heredoc en lugar de `nano`.

---

# Servidores Webs

Es es el script de aprovisionamiento y de configuración de los dos servidores webs:

```bash
#!/bin/bash

# Actualizar paquetes
sudo apt update -y

# Instalar NFS y PHP con las extensiones necesarias
sudo apt install nfs-common -y
sudo apt install php8.3 -y
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-xml php-mbstring php-xmlrpc php-zip php-soap php-intl -y

# Crear el directorio para el montaje NFS
sudo mkdir -p /nfs/compartido

# Montar el recurso NFS remoto
sudo mount 192.168.30.74:/var/nfs/wordpress /nfs/compartido

# Añadir el montaje a /etc/fstab
echo "192.168.30.74:/var/nfs/wordpress    /nfs/compartido   nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" | sudo tee -a /etc/fstab

# Copiar el archivo de sitio por defecto de Apache a wordpress.conf
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/wordpress.conf

# Crear el archivo wordpress.conf con la configuración adecuada
sudo bash -c 'cat <<EOF > /etc/apache2/sites-available/wordpress.conf
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /nfs/compartido/wordpress
        ErrorLog \${APACHE_LOG_DIR}/error.log
        CustomLog \${APACHE_LOG_DIR}/access.log combined

        <Directory /nfs/compartido/wordpress>
                Options FollowSymlinks
                AllowOverride All
                Require all granted
        </Directory>

        SetEnvIf X-Forwarded-Proto "https" HTTPS=on
</VirtualHost>
EOF'

# Deshabilitar el sitio por defecto y habilitar el sitio de WordPress
sudo a2dissite 000-default.conf
sudo a2ensite wordpress.conf

# Reiniciar Apache
sudo systemctl restart apache2

# Cambiar el hostname
sudo hostnamectl set-hostname ServidorWEBJesusSanchez

echo "Aprovisionamiento completado."
```

Este script:

1. Instala los paquetes necesarios (NFS, PHP y extensiones).
2. Monta el recurso NFS y actualiza `/etc/fstab` para el montaje permanente.
3. Crea la configuración del sitio de WordPress en Apache sin usar `nano`.
4. Habilita el nuevo sitio y reinicia Apache.

---

# Servidor Base de Datos

Este es el script de aprovisionamiento y de configuración para el servidor de base de datos

```bash
#!/bin/bash

# Actualizar repositorios e instalar MySQL Server
sudo apt update -y
sudo apt install mysql-server -y

# Modificar bind-address a 0.0.0.0 en mysqld.cnf
# Primero comprobamos si existe la directiva bind-address. Si existe, la sustituimos; si no, la añadimos.
if grep -q "^bind-address" /etc/mysql/mysql.conf.d/mysqld.cnf; then
    sudo sed -i 's/^bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
else
    echo "bind-address = 0.0.0.0" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
fi

# Reiniciar el servicio MySQL para aplicar cambios
sudo systemctl restart mysql

# Crear la base de datos y el usuario de WordPress
sudo mysql -u root -e "CREATE DATABASE wordpress_db; FLUSH PRIVILEGES;"
sudo mysql -u root -e "CREATE USER 'wordpress_user'@'192.168.30.%' IDENTIFIED BY 'nueva_contraseña';"
sudo mysql -u root -e "GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'192.168.30.%'; FLUSH PRIVILEGES;"

# Añadir la opción skip-name-resolve
echo "skip-name-resolve" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf

# Reiniciar MySQL nuevamente
sudo systemctl restart mysql

# Cambiar el hostname
sudo hostnamectl set-hostname BaseDeDatosJesusSanchez

echo "Aprovisionamiento completado."
```

Este script:

1. Instala el servidor MySQL.
2. Cambia la dirección de enlace del servidor MySQL a 0.0.0.0 para permitir conexiones remotas.
3. Crea la base de datos `wordpress_db` y el usuario `wordpress_user` con privilegios desde la subred indicada.
4. Añade la directiva `skip-name-resolve` para mejorar el rendimiento evitando la resolución DNS inversa.
5. Reinicia MySQL para aplicar todos los cambios.

---

# Comprobación del funcionamiento

![imagen.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/13978038-732b-4391-bc17-bf7e39c061f7/cc1e7ac3-90f8-4f02-9804-e699619de66b/imagen.png)
 
