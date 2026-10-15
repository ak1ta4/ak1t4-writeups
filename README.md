# ak1t4 // Writeups

Writeups de máquinas de **DockerLabs** resueltas en laboratorio, documentadas paso a paso:
reconocimiento, explotación, escalada de privilegios e informe de vulnerabilidades
(CWE / OWASP · impacto · mitigación).

Acompañan a mi portfolio: **https://ak1ta4.github.io/ak1t4-portfolio/**

## 📂 Máquinas (18)

### Fácil
- [**Walking Dead**](writeups/walking-dead.md) — El código fuente de la web filtra una web shell oculta; con ffuf descubro su parámetro vulnerable (cmd), consigo ejecución de comandos y una reverse shell como www-data, y escalo a root abusando del bit SUID en python3.8.
- [**File**](writeups/file.md) — FTP anónimo que filtra un hash, subida de archivos con bypass de extensión (.phar) para una web shell como www-data, fuerza bruta local con su y una escalada encadenada mario → julen → iker → root abusando de sudo (awk, env) y de un Python Library Hijacking.
- [**Allien**](writeups/allien.md) — Enumeración SMB anónima que filtra usuarios y un JWT, fuerza bruta de satriani7 por SMB, un fichero con la contraseña del administrador, acceso por SSH, una reverse shell PHP en el webroot para pasar a www-data y escalada a root abusando de un sudo NOPASSWD sobre service.
- [**Flynn**](writeups/flynn.md) — La web expone el nombre de usuario en el propio HTML; con hydra descubro que su contraseña es trivial (igual al usuario) y escalo a root abusando de un sudo NOPASSWD sobre /usr/bin/env.
- [**Analyst**](writeups/analyst.md) — Análisis forense de red al estilo SOC: a partir de una captura PCAP reconstruyo toda la cadena de ataque —reconocimiento, bypass de subida por doble extensión, web shell, reverse shell y fuga de credenciales— con tshark, respondo las 9 preguntas del caso y termino escalando a root con la credencial filtrada.
- [**Duque**](writeups/duque.md) — Enumeración web con gobuster hasta un panel de facturación vulnerable a SQL injection, volcado de credenciales con sqlmap, fuerza bruta de un ID de factura con crunch + ffuf reutilizando la cookie de sesión, y escalada a root abusando del bit SUID en /usr/bin/env.
- [**Pequeñas Mentirosas**](writeups/pequenas-mentirosas.md) — Una pista en la web señala al usuario 'a', al que le saco la contraseña por fuerza bruta SSH con hydra. Desde dentro pivoto al usuario spencer con otra fuerza bruta y escalo a root abusando de sudo sobre python3.
- [**Ejotapete**](writeups/ejotapete.md) — Drupal 8.5.0 vulnerable a Drupalgeddon2 explotado con Metasploit para shell de www-data, y escalada a root abusando del bit SUID en find.
- [**Injection**](writeups/injection.md) — Bypass de autenticación por SQL injection para filtrar credenciales, acceso por SSH y escalada a root abusando del bit SUID en /usr/bin/env.

### Muy fácil
- [**Acme**](writeups/acme.md) — El banner del SSH filtra unas credenciales temporales en texto plano; con ellas accedo como 'usuario' y escalo a root abusando del bit SUID en /usr/bin/bash.
- [**TpRoot**](writeups/tproot.md) — El puerto FTP corre vsftpd 2.3.4, una versión con una puerta trasera conocida (CVE-2011-2523). La exploto con un exploit público de GitHub que da shell de root directa.
- [**Trust**](writeups/trust.md) — Un fichero oculto (secret.php) descubierto con gobuster filtra un usuario; con hydra saco su contraseña y escalo a root abusando de un permiso sudo sobre vim.
- [**Vacaciones**](writeups/vacaciones.md) — Un comentario en la web filtra dos usuarios; con hydra saco la contraseña de camilo, un correo interno revela la de juan y escalo a root abusando de un sudo NOPASSWD sobre ruby.
- [**Obsession**](writeups/obsession.md) — Directorio /backup expuesto con un usuario, fuerza bruta SSH con Hydra y escalada a root abusando de un sudo sobre Vim.
- [**BorazuwarahCTF**](writeups/borazuwarahctf.md) — Usuario oculto en los metadatos de una imagen (Exiftool), fuerza bruta SSH con Hydra y escalada a root por un sudo mal configurado.
- [**BreakMySSH**](writeups/breakmyssh.md) — Único puerto SSH; fuerza bruta con Hydra sobre root revela la contraseña y da acceso directo como root.
- [**FirstHacking**](writeups/firsthacking.md) — Único puerto FTP con vsftpd 2.3.4 vulnerable; su backdoor da shell de root directa mediante un exploit público.
- [**HedgeHog**](writeups/hedgehog.md) — Usuario filtrado en la web, fuerza bruta SSH con un rockyou invertido y escalada encadenada tails → sonic → root vía sudo.

## ⚠️ Uso responsable

Todo el contenido técnico se ha realizado en **entornos de laboratorio controlados y autorizados**
(DockerLabs) con fines educativos. No utilices estas técnicas contra sistemas sobre los que no tengas
autorización expresa. La IP del equipo atacante aparece como `<IP_ATACANTE>` para que la sustituyas por la tuya.

---

© 2026 Adrián Sepúlveda Ramos · ak1t4
