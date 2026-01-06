# DockerLabs-WalkingCMS
**Fuerza Bruta en Wordpress(CMS), Reverse Shell y SUID Abuse**

### Reconocimiento

Comenzamos comprobando la conectividad hacia el objetivo enviando un **ping ICMP**. Esto nos permite verificar si el host responde y obtener datos básicos como el **TTL** y el tiempo de respuesta.

Gracias al **TTL = 64**, podemos deducir que el objetivo probablemente es un sistema Linux.
<br>
<br>
![ping](https://github.com/user-attachments/assets/51923c7c-3f30-46b8-abd5-8d38ddbbaf34)
<br>
<br>

### Enumeración

Realizamos un escaneo con **Nmap** utilizando parámetros de reconocimiento y enumeración para identificar los puertos abiertos y los servicios que se ejecutan en el objetivo.

```bash
sudo nmap -p- -sS -sC -sV --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG Report
```


**Resultado:**

**Puertos abiertos:** Se identificó que el puerto **80/tcp** se encuentra **abierto**, lo que indica la presencia de un servicio web activo.

**Servicios detectados:** El servicio que responde en dicho puerto corresponde a **Apache httpd 2.4.57 (Debian)**. La página por defecto del servidor indica *“Apache2 Debian Default Page: It works”*, lo cual confirma que el servidor está operativo
<br>
<br>
![80](https://github.com/user-attachments/assets/e8448023-ad35-4473-b4cc-aa62512e9f3e)
<br>
<br>


### Página Web (Puerto 80)

Al identificar que el **puerto 80** se encuentra **abierto**, accedimos a la dirección IP desde el navegador.
El servidor respondió con la página por defecto de **Apache2 en Debian**, lo que confirma que el servicio web está activo.

Posteriormente, se revisó el **código fuente de la página**, sin encontrarse información ni elementos relevantes que pudieran aportar pistas adicionales.
<br>
<br>
![apache](https://github.com/user-attachments/assets/a676b2cd-4436-4448-b319-eb54d071e0d0)
<br>
<br>

### Fuzzing Web

Usamos la herramienta **gobuster** en busca de directorios/archivos relevantes donde podamos iniciar con la intrusión.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 200 -x php,bak,html,js,txt,json,xml
```

**Resultado:**

La enumeración devolvió múltiples entradas con código **403 Forbidden**, además de un `index.html` que muestra la página de Pache (irrelevante para el objetivo). Destaca la ruta **`/wordpress`**, la cual respondió con un **301 (redirección)** a `http://172.17.0.2/wordpress/`, confirmando la existencia de un sitio WordPress en esa ubicación.
<br>
<br>
![gobuster 1](https://github.com/user-attachments/assets/55d49f5f-d66c-412b-8252-e2ef6771969d)
<br>
<br>


### Enumeración del sitio WordPress

Se encontró la ruta `/wordpress`; al acceder muestra el título **"Web vulnerable"**, confirmando que es una instalación de WordPress. Inspeccionaremos la página en busca de pistas pero no encontramos nada relevante.
<br>
<br>
![enumeracion](https://github.com/user-attachments/assets/b8ee5b39-b98a-4611-aeec-3650a069a7cc)
<br>
<br>

### Fuzzing sobre `/wordpress`

Realizamos un fuzzing dirigido a la ruta `/wordpress` para descubrir directorios y rutas relevantes ocultas. Este proceso busca archivos, endpoints y recursos expuestos que no aparecen en la raíz y que puedan aportar vectores de ataque o información útil para la enumeración.

```bash
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 200
```

**Resultado:**

Hemos identificado 3 rutas interesantes bajo `/wordpress`: `/wp-content`, `/wp-includes` y `/wp-admin`. Todas responden con redirecciones (301) hacia sus respectivas ubicaciones dentro de `/wordpress/`.

La ruta más relevante es **`/wp-admin`**, ya que normalmente aloja el panel de administración de WordPress. Como siguiente paso accederemos a `/wordpress/wp-admin` (y a `/wordpress/wp-login.php`) para comprobar el formulario de login y continuar con la enumeración de usuarios y posibles vectores de acceso.
<br>
<br>
![gobuster2](https://github.com/user-attachments/assets/b6c6b227-0f21-4324-8f1e-841b0f49158c)
<br>
<br>

### Panel Inicio de sesion WordPress

Al acceder a `/wordpress/wp-admin` se muestra el formulario de login de WordPress (`wp-login.php`). Dado que aún no disponemos de usuarios válidos, procederemos a **enumerar usuarios** (por ejemplo con `wpscan`) para obtener posibles nombres de cuenta a probar.
<br>
<br>
![panelwp](https://github.com/user-attachments/assets/19865ebd-d134-4caf-a745-78e124244f31)
<br>
<br>


### Enumeración de usuario con WPScan

Vamos hacer una enumeración simple de unicamente usuarios.

```bash
wpscan --url http://172.17.0.2/wordpress -e u
```

**Resultado:**

Identificamos el usuario `mario`
<br>
<br>
![mario](https://github.com/user-attachments/assets/c62415fc-1769-4517-9a39-a4f07b504e50)
<br>
<br>

### Fuerza bruta con `wpscan`

Con el usuario identificado, ejecutamos un ataque de fuerza bruta para obtener su contraseña:

```bash
wpscan --url http://172.17.0.2/wordpress/ -U mario -P /usr/share/wordlists/rockyou.txt
```

**Resultado:**

Hemos obtenido la contraseña del usuario **mario** en WordPress gracias a la herramienta `wpscan`.
<br>
<br>
![love](https://github.com/user-attachments/assets/5c5921b8-903b-42ef-9d83-fc056a795770)
<br>
<br>

### Acceso al panel de WordPress

Ingresamos al panel de administración de WordPress utilizando las credenciales previamente obtenidas.
<br>
<br>
![panel](https://github.com/user-attachments/assets/ef1badab-ee6a-46fd-8ccc-ad82fe4b36d1)
<br>
<br>


Accedimos a **Apariencia** → **Theme Code Editor** y abrimos el archivo `index.php` del tema para su edición.
<br>
<br>
![theme](https://github.com/user-attachments/assets/ba2595c6-0cda-4400-83b6-22c9e295c471)
<br>
<br>


### Obtención de Reverse Shell

Generamos un *payload* en PHP usando Revshells ([revshells.com](https://www.revshells.com/)) para crear y ajustar la carga útil a nuestras necesidades; escogimos la plantilla PHP (PentestMonkey) como base. A continuación insertamos ese fragmento en `index.php` del tema desde el **Theme Code Editor** del panel.

Importante guardar el cambio con el boton **"Update File"**.
<br>
<br>
![upadate](https://github.com/user-attachments/assets/5cad9dde-550a-4b01-a94a-06eabb8b1922)
<br>
<br>


En una terminal configuramos un *listener* con **netcat** y dejamos la escucha a un puerto específico, en mi caso el **puerto 443**.
<br>
<br>
![nc](https://github.com/user-attachments/assets/d467c707-4c06-4ce5-87fa-e679df95e561)
<br>
<br>


Accedimos a la ruta del archivo modificado: `/wordpress/wp-content/themes/twentytwentytwo/index.php`, donde se aplicaron y guardaron los cambios.

La pestaña del navegador permanecerá en estado de carga mientras se ejecuta el payload.
<br>
<br>
![payload](https://github.com/user-attachments/assets/0094ef4c-722e-48d9-8007-350125fc8e67)
<br>
<br>


Desde nuestro lado, hemos recibido correctamente la reverse shell.
<br>
<br>
![shell](https://github.com/user-attachments/assets/765641bc-fea2-4d43-86f7-873b59ef4d9a)
<br>
<br>


### Establecer una nueva conexión + Tratamiento TTY

Debemos establecer una nueva conexión para evitar perder el acceso actual y obtener una terminal más estable y funcional.

1. Crea un entorno interactivo con Bash.

```bash
script /dev/null -c bash
```

Suspendemos la sesión temporalmente.

`Ctrl + Z`

Restaura la entrada y activa un TTY estable.

```
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash`
```

Sincronizamos las filas y columnas para que la shell se adapte al tamaño real de nuestra ventana.

```bash
stty rows <n> columns <n>
```

### Escalar Privilegios

Probamos con `sudo -l` para verificar si es posible ejecutar comandos con privilegios sin necesidad de contraseña, la utilidad **sudo** no está disponible para este usuario.
<br>
<br>
![sudo -l](https://github.com/user-attachments/assets/d66cc816-dcc6-4a38-8922-182da2729e21)
<br>
<br>


Buscamos desde la raíz binarios con el bit **SUID** que puedan ser susceptibles de explotación.

El binario **`/usr/bin/env`** con permisos **SUID** resulta inusual, por lo que intentaremos una **escalada de privilegios** aprovechando este binario.
<br>
<br>
![find](https://github.com/user-attachments/assets/ad35b629-a8a2-4048-be21-57ee7ef23530)
<br>
<br>


**Comando:**

`/usr/bin/env /bin/sh -p`

**Resultado**:

Si `env` tiene **SUID root**, esta instrucción lanza `/bin/sh` en modo privilegiado y puede devolver una shell con **EUID=root**.
<br>
<br>

![root](https://github.com/user-attachments/assets/50434583-70ab-43d6-b3dd-f2f898bb1057)




