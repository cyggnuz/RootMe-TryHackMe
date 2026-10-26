
# RootMe — TryHackMe

## 1. Objetivo del laboratorio

El objetivo de este laboratorio fue resolver la máquina RootMe de TryHackMe, identificar las vulnerabilidades que permitían avanzar en el desafío y obtener acceso con privilegios de administrador.

Para esto, se realizaron distintas pruebas que permitieron revisar los servicios disponibles, buscar directorios en el servidor web e identificar errores de configuración que podían aprovecharse. Además, se analizaron las vulnerabilidades encontradas y se propusieron medidas para evitar que estos problemas ocurrieran en un entorno real.

- **Sistema operativo:** Linux (Ubuntu).
- **Dificultad:** Fácil.
- **Plataforma:** [TryHackMe - RootMe](https://tryhackme.com/room/rrootme).

## 2. Herramientas utilizadas

| Herramienta | Uso |
|---|---|
| Nmap | Identificar los puertos abiertos y los servicios disponibles. |
| Gobuster | Buscar directorios y rutas del servidor web. |
| cURL | Probar el formulario de subida de archivos y acceder a los archivos almacenados. |
| Netcat (nc) | Recibir la conexión de la reverse shell. |
| PHP reverse shell | Obtener una conexión remota mediante un archivo PHP. |
| Python 2.7 | Comprobar la escalada de privilegios mediante un binario con permisos SUID. |

## 3. Desarrollo del laboratorio

### 3.1. Reconocimiento

Primero se realizó un escaneo con Nmap para identificar los puertos abiertos y conocer los servicios que estaban funcionando en la máquina.

```bash
nmap -sC -sV -v -oN nmap1.txt IP_DE_TRYHACKME
```

Se debe reemplazar `IP_DE_TRYHACKME` por la dirección IP asignada a la máquina en TryHackMe.

En los resultados se identificaron dos puertos abiertos:

- **Puerto 22/tcp:** SSH, con OpenSSH 8.2p1.
- **Puerto 80/tcp:** HTTP, con Apache 2.4.41.

También se detectó la cookie `PHPSESSID` en las cabeceras HTTP, lo que indicó que el sitio probablemente utilizaba PHP.

**Evidencia:** `evidencias/01-scaneo-nmap.png`

### 3.2. Enumeración web

Después se utilizó Gobuster para buscar directorios disponibles en el servidor web.

```bash
gobuster dir -u http://IP_DE_TRYHACKME/ -w /usr/share/wordlists/dirb/common.txt -t 50
```

Durante la búsqueda se identificaron dos rutas importantes:

- `/panel`: contenía un formulario para subir archivos.
- `/uploads`: permitía acceder a los archivos que se habían subido.

Estas rutas permitieron continuar con las pruebas, ya que se podía revisar cómo funcionaba el formulario y qué ocurría con los archivos almacenados en el servidor.

**Evidencia:** `evidencias/02-gobuster-scan.png`

### 3.3. Subida de archivos

Primero se probó la subida de un archivo de texto para comprobar si el formulario lo aceptaba.

```bash
curl -F "fileUpload=@test.txt" -F "submit=Upload" http://IP_DE_TRYHACKME/panel/
```

El archivo quedó disponible en `/uploads`, por lo que se confirmó que el servidor permitía subir archivos y almacenarlos.

Después se intentó subir un archivo PHP llamado `shell.php`, pero fue rechazado con el mensaje *"PHP não é permitido!"*. Esto permitió identificar que existía un filtro que bloqueaba esa extensión.

Para comprobar si el filtro era suficiente, se cambió la extensión del archivo a `.phtml`, que también puede utilizarse para ejecutar código PHP dependiendo de la configuración del servidor.

```bash
cp shell.php shell.phtml
curl -F "fileUpload=@shell.phtml" -F "submit=Upload" http://IP_DE_TRYHACKME/panel/
```

Esta vez el archivo fue aceptado. Luego se accedió al archivo desde `/uploads` y se utilizó Netcat para recibir la conexión remota.

En una terminal se ejecutó:

```bash
nc -lvnp 4444
```

Después, desde otra terminal, se accedió al archivo subido:

```bash
curl http://IP_DE_TRYHACKME/uploads/shell.phtml
```

Como resultado, se obtuvo una reverse shell con el usuario `www-data`, utilizado por el servicio web.

Con esta prueba se comprobó que el filtro de extensiones no era suficiente para impedir la ejecución de código PHP, ya que se aceptaba una extensión alternativa que el servidor interpretaba como PHP.

**Evidencias:**

- `evidencias/03-filtro-rechazado.png`
- `evidencias/04-bypass.png`
- `evidencias/05-reverse-shell.png`

### 3.4. Obtención de la flag de usuario

Después de obtener acceso a la máquina, se buscó el archivo `user.txt`, utilizado para comprobar el avance del desafío.

Para localizarlo se ejecutó el siguiente comando:

```bash
find / -type f -name user.txt 2>/dev/null
```

Una vez encontrada su ubicación, se consultó el archivo:

```bash
cat /var/www/user.txt
```

Con esto se obtuvo la flag de usuario y se completó esta parte del laboratorio. El valor completo no se incluye en el informe.

**Evidencia:** `evidencias/06-userflag.png`

### 3.5. Escalada de privilegios

Una vez obtenido el acceso como `www-data`, se revisó si existían binarios con permisos SUID que pudieran permitir una escalada de privilegios.

El bit SUID permite que un programa se ejecute con los permisos de su propietario. Por esta razón, una configuración incorrecta puede permitir que un usuario consiga privilegios superiores a los que debería tener.

Para buscar estos archivos se ejecutó:

```bash
find / -type f -user root -perm -u=s 2>/dev/null | grep -v snap
```

Entre los resultados se identificó `/usr/bin/python2.7` con el bit SUID activado.

Esto representaba un riesgo, ya que Python permite ejecutar instrucciones del sistema y, con esa configuración, podía utilizarse para intentar abrir una shell con los privilegios de `root`.

Para comprobarlo se ejecutó:

```bash
python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Después se utilizó el siguiente comando para comprobar el usuario de la sesión:

```bash
whoami
```

El resultado fue `root`, por lo que se confirmó que la configuración del binario permitía escalar privilegios.

Finalmente, se buscó el archivo `root.txt`:

```bash
find / -type f -name root.txt 2>/dev/null
```

Una vez encontrada su ubicación, se consultó el archivo y se obtuvo la flag de root. Con esto se completó el objetivo principal del laboratorio. El valor completo no se incluye en el informe.

**Evidencias:**

- `evidencias/07-suid.png`
- `evidencias/08-privesc-root.png`
- `evidencias/09-root-flag.png`

## 4. Vulnerabilidades encontradas

### 4.1. Subida insegura de archivos

**¿Cómo se identificó?**

Al revisar el formulario de `/panel`, se comprobó que era posible subir archivos y acceder a ellos desde `/uploads`. El servidor rechazaba los archivos con extensión `.php`, pero aceptaba `.phtml` y ejecutaba su contenido al acceder al archivo.

**¿Qué permitió la explotación?**

El filtro de extensiones era incompleto y el directorio donde se almacenaban los archivos permitía ejecutar código PHP. Esto hizo posible obtener una reverse shell y acceder al sistema como `www-data`.

**¿Qué consecuencias podría tener?**

En un entorno real, esta vulnerabilidad podría permitir que una persona sin autorización ejecutara código en el servidor. Dependiendo de los permisos disponibles, podría acceder a información, modificar archivos o intentar comprometer otros recursos.

### 4.2. Binario Python con permisos SUID

**¿Cómo se identificó?**

Se realizó una búsqueda de archivos con el bit SUID activado y propiedad de `root`. Durante la revisión se encontró `/usr/bin/python2.7` con este permiso.

**¿Qué permitió la explotación?**

Python permite ejecutar instrucciones del sistema y, en este caso, su configuración permitió iniciar una shell con privilegios elevados. Esto hizo posible pasar de `www-data` a `root`.

**¿Qué consecuencias podría tener?**

En un sistema real, una persona que ya tuviera acceso limitado podría aprovechar una configuración como esta para obtener permisos de administrador. Esto le permitiría acceder o modificar recursos que normalmente estarían protegidos.

## 5. Medidas de remediación

### 5.1. Subida insegura de archivos

Para evitar este problema, se recomienda aplicar las siguientes medidas:

- **Validar el contenido de los archivos:** no depender solamente de la extensión, sino comprobar también el tipo de archivo.
- **Permitir solo los formatos necesarios:** utilizar una lista de extensiones autorizadas en lugar de bloquear únicamente algunas.
- **Desactivar la ejecución de scripts en `/uploads`:** configurar el servidor para que los archivos almacenados en ese directorio no puedan ejecutar código PHP.
- **Cambiar los nombres de los archivos:** asignar nombres controlados por la aplicación para reducir los riesgos relacionados con nombres manipulados.
- **Limitar el tamaño de los archivos:** establecer un tamaño máximo permitido para las subidas.
- **Revisar los permisos:** impedir que personas sin autorización puedan modificar archivos o acceder a recursos protegidos.

### 5.2. Binario Python con permisos SUID

Para evitar que se aproveche esta configuración, se recomienda:

- **Retirar el permiso SUID de Python si no es necesario:** antes de hacerlo, comprobar que ningún proceso legítimo dependa de ese permiso.
- **Revisar los archivos con SUID y SGID:** comprobar periódicamente cuáles tienen estos permisos y si realmente los necesitan.
- **Aplicar el principio de mínimo privilegio:** cada usuario y programa debe tener solo los permisos necesarios para funcionar.
- **Mantener el sistema actualizado:** instalar las actualizaciones de seguridad disponibles.
- **Revisar los permisos de los servicios:** limitar los privilegios de las cuentas utilizadas por el servidor web para reducir el impacto de una posible intrusión.

## 6. Conclusiones

Este laboratorio permitió practicar el reconocimiento de servicios, la búsqueda de directorios y la revisión de permisos en Linux. Durante el proceso se identificó una forma de ejecutar código mediante la subida de archivos y después se consiguió escalar privilegios aprovechando una mala configuración del permiso SUID.
Uno de los puntos más importantes fue comprobar que bloquear una extensión no es suficiente para evitar la subida de archivos peligrosos. También es necesario revisar dónde se almacenan los archivos y si el servidor puede ejecutar su contenido.
Además se pudo comprender la importancia de revisar los permisos SUID, ya que una configuración incorrecta puede permitir que un usuario con acceso limitado consiga privilegios de administrador.
En general, el desafío permitió relacionar las vulnerabilidades encontradas con sus posibles consecuencias y comprender qué medidas se pueden aplicar para prevenir problemas similares.

---
