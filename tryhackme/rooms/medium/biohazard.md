# Biohazard — TryHackMe

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-blue?style=flat)

> **Fecha:** 01/05/2026
> **Autor:** [Josue Rendón](https://github.com/Slasho01)
> **Room:** [Biohazard](https://tryhackme.com/room/biohazard)

---

## 📋 Descripción

Room de dificultad media con temática de Resident Evil. Cubre enumeración web manual a través de rutas ocultas, múltiples capas de decodificación (Base64, Base32, Base58, ROT13, Vigenère), esteganografía en imágenes con tres técnicas distintas, descifrado GPG, acceso FTP con credenciales construidas combinando cuatro crests, y escalada de privilegios vía `sudo`.

---

## 🎯 Objetivos

* Recolectar todas las flags de la mansión
* Obtener credenciales FTP via combinación de crests
* Obtener acceso SSH
* Escalar privilegios a root

---

## 🔍 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sV -p- 10.67.128.169
```

**Resultado:**

```
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

**Puertos relevantes:**

* `21` — FTP (vsftpd 3.0.3) ← credenciales obtenidas más adelante
* `22` — SSH
* `80` — HTTP (Apache) ← vector de entrada principal

**Información del servidor:**

* Nombre del equipo: *The Stars Alpha Team*
* Posibles usuarios: `chris`, `jill`, `barry`, `weasker`, `joseph`

---

## 🌐 2. Enumeración Web

### /mansionmain/

Revisando el código fuente de `http://10.67.128.169/mansionmain/` se descubre una ruta oculta:

```
/diningRoom/
```

### /diningRoom/

El código fuente contiene un string en Base64:

```
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
```

Decodificando → `/teaRoom/`

Accediendo a `emblem.php` se obtiene la primera flag:

> 🚩 `emblem{fec832623ea498e20bf4fe1821d58727}`

### /teaRoom/

En `master_of_unlock.html` se obtiene la siguiente flag y se descubre la ruta `/artRoom/`:

> 🚩 `lock_pick{037b35e2ff90916a9abf99129c8e1837}`

### /artRoom/

`MansionMap.html` revela el mapa completo con todas las ubicaciones de la mansión:

```
/diningRoom/      /teaRoom/         /artRoom/
/barRoom/         /diningRoom2F/    /tigerStatusRoom/
/galleryRoom/     /studyRoom/       /armorRoom/
/attic/
```

### /barRoom/

Se usa `lock_pick{037b35e2ff90916a9abf99129c8e1837}` para desbloquear la puerta. Se obtiene una nota musical (*Moonlight Sonata*) con un string en **Base32**:

```
NV2XG2LDL5ZWQZLFOR5TGNRSMQ3TEZDFMFTDMNLGGVRGIYZWGNSGCZLDMU3GCMLGGY3TMZL5
```

Decodificando Base32:

> 🚩 `music_sheet{362d72deaf65f5bdc63daece6a1f676e}`

Ingresando esa flag en `gold_emblem.php`:

> 🚩 `gold_emblem{58a8c41a9d08b8a4e38d02a4d7ff4843}`

Ingresando el `emblem` original (primera flag) en el slot del bar se obtiene la respuesta **`rebecca`** — esta será la clave Vigenère más adelante.

### /diningRoom/ (revisita con gold emblem)

Ingresando la gold emblem aparece el siguiente texto cifrado:

```
klfvg ks r wimgnd biz mpuiui ulg fiemok tqod. Xii jvmc tbkg ks tempgf tyi_hvgct_jljinf_kvc
```

Aplicando **Vigenère** con clave `rebecca`:

```
there is a shield key inside the dining room. The html page is called the_great_shield_key
```

Accediendo a `http://10.67.128.169/diningRoom/the_great_shield_key.html`:

> 🚩 `shield_key{48a7a9227cd7eb89f0a062590798cbac}`

### /diningRoom2F/

Texto cifrado en **ROT13**:

```
Lbh trg gur oyhr trz ol chfuvat gur fgnghf gb gur ybjre sybbe.
Gur trz vf ba gur qvavatEbbz svefg sybbe. Ivfvg fnccuver.ugzy
```

Decodificando:

```
You get the blue gem by pushing the status to the lower floor.
The gem is on the diningRoom first floor. Visit sapphire.html
```

Accediendo a `sapphire.html`:

> 🚩 `blue_jewel{e1d457e96cac640f863ec7bc475d48aa}`

---

## 🧩 3. Recolección de Crests

Cada room entrega un fragmento codificado (crest). Deben combinarse en orden y el resultado final, decodificado en Base64, entrega las credenciales FTP.

> **Nota:** La combinación es `crest1 + crest2 + crest3 + crest4` y el resultado es un string Base64.

### Crest 1 — /tigerStatusRoom/

Requiere el `blue_jewel` para acceder.

Codificación: **Base64 → Base32** (2 capas, 14 letras en claro)

```
S0pXRkVVS0pKQkxIVVdTWUpFM0VTUlk9
→ (Base64) → RlRQIHVzZXI6IGh1...  (fragmento del resultado final)
```

### Crest 2 — /galleryRoom/

Codificación: **Base32 → Base58** (2 capas, 18 letras en claro)

```
GVFWK5KHK5WTGTCILE4DKY3DNN4GQQRTM5AVCTKE
→ (Base32) → (Base58) → fragmento del resultado final
```

### Crest 3 — /armorRoom/

Requiere `shield_key` para acceder.

Codificación: **Base64 → Binario → Hex** (3 capas, 19 letras en claro)

```
MDAxMTAxMTAg...  (string largo en Base64)
→ (Base64) → string binario
→ (Binario a texto) → string hex
→ (Hex a texto) → fragmento del resultado final
```

### Crest 4 — /attic/

Requiere `shield_key` para acceder.

Codificación: **Base58 → Hex** (2 capas, 17 caracteres en claro)

```
gSUERauVpvKzRpyPpuYz66JDmRTbJubaoArM6CAQsnVwte6zF9J4GGYyun3k5qM9ma4s
→ (Base58) → (Hex a texto) → fragmento del resultado final
```

### Combinación final

Concatenando los 4 fragmentos y decodificando en Base64:

```
RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==
```

```
FTP user: hunter
FTP pass: you_cant_hide_forever
```

---

## 📁 4. FTP y Esteganografía

### Conexión FTP

```bash
ftp 10.67.128.169
# Name: hunter
# Password: you_cant_hide_forever
```

**Archivos disponibles:**

```
001-key.jpg        (7994 bytes)
002-key.jpg        (2210 bytes)
003-key.jpg        (2146 bytes)
helmet_key.txt.gpg (121 bytes)
important.txt      (170 bytes)
```

```bash
ftp> get important.txt
ftp> get 001-key.jpg
ftp> get 002-key.jpg
ftp> get 003-key.jpg
ftp> get helmet_key.txt.gpg
```

`important.txt` es una nota de Barry a Jill que confirma dos cosas: el helmet key está cifrado en el `.gpg` y existe una puerta en `/hidden_closet/`.

### Extracción de datos ocultos

Las tres imágenes ocultan datos mediante técnicas distintas. Cada una aporta un fragmento del string final.

**001-key.jpg — steghide** (datos embebidos en la imagen)

```bash
steghide extract -sf 001-key.jpg
# Passphrase: (vacía, Enter)
cat key-001.txt
# cGxhbnQ0Ml9jYW
```

**002-key.jpg — exiftool** (datos en campo EXIF Comment)

```bash
exiftool 002-key.jpg
# Comment: 5fYmVfZGVzdHJveV9
```

**003-key.jpg — binwalk** (archivo ZIP embebido al final del JPG)

```bash
binwalk -e 003-key.jpg
cat _003-key.jpg.extracted/key-003.txt
# 3aXRoX3Zqb2x0
```

### Passphrase para el GPG

Concatenando los 3 fragmentos y decodificando en Base64:

```
cGxhbnQ0Ml9jYW + 5fYmVfZGVzdHJveV9 + 3aXRoX3Zqb2x0
→ plant42_can_be_destroy_with_vjolt
```

Descifrando el archivo GPG con esa passphrase:

```bash
gpg -d helmet_key.txt.gpg
# gpg: AES256 encrypted data
# Passphrase: plant42_can_be_destroy_with_vjolt
```

> 🚩 `helmet_key{458493193501d2b94bbab2e727f8db4b}`

---

## 🔑 5. Acceso SSH

### /studyRoom/

Requiere `helmet_key` para acceder. Se descarga el archivo `doom.tar.gz`:

```bash
tar -xvf doom.tar.gz
cat eagle-medal.txt
# SSH user: umbrella_guest
```

### /hidden_closet/

Requiere `helmet_key` para acceder. Contiene dos elementos:

**MO Disk 1** — texto cifrado por sustitución. Usando [dcode.fr](https://www.dcode.fr/cipher-identifier) se identifica el cifrado y se descifra:

```
weasker login password: stars_members_are_my_guinea_pig
```

**Wolf Medal** — contiene la contraseña del primer acceso SSH:

```
SSH password: T_virus_rules
```

### Primer acceso — umbrella_guest

```bash
ssh umbrella_guest@10.67.128.169
# Password: T_virus_rules
```

Navegando al home de `weasker` se encuentra `weasker_note.txt` (diálogo narrativo del juego) y `chris.txt`, donde Chris entrega el **MO Disk 2** con la clave `albert` — confirmando la clave que se usó para descifrar el MO Disk 1.

### Pivote a weasker

```bash
ssh weasker@10.67.128.169
# Password: stars_members_are_my_guinea_pig
```

---

## 🚩 6. Flag de usuario

```bash
cat ~/user.txt
```

```
flag{user_flag_aqui}
```

---

## ⬆️ 7. Escalada de privilegios

```bash
sudo -l
```

El usuario `weasker` tiene permisos completos de sudo. Se escala directamente a root:

```bash
sudo su -
whoami
# root
```

---

## 🚩 8. Flag de root

```bash
cat /root/root.txt
```

```
flag{root_flag_aqui}
```

---

## 📊 9. Resumen de flags

| Flag | Ubicación | Técnica |
|------|-----------|---------|
| `emblem{fec832623ea498e20bf4fe1821d58727}` | /diningRoom/emblem.php | Enumeración web (código fuente) |
| `lock_pick{037b35e2ff90916a9abf99129c8e1837}` | /teaRoom/master_of_unlock.html | Enumeración web |
| `music_sheet{362d72deaf65f5bdc63daece6a1f676e}` | /barRoom/ | Decodificación Base32 |
| `gold_emblem{58a8c41a9d08b8a4e38d02a4d7ff4843}` | /barRoom/gold_emblem.php | Encadenamiento de flags |
| `shield_key{48a7a9227cd7eb89f0a062590798cbac}` | /diningRoom/the_great_shield_key.html | Vigenère (clave: `rebecca`) |
| `blue_jewel{e1d457e96cac640f863ec7bc475d48aa}` | /diningRoom2F/sapphire.html | ROT13 |
| `helmet_key{458493193501d2b94bbab2e727f8db4b}` | FTP → GPG | Esteganografía + descifrado GPG |
| `plant42_can_be_destroy_with_vjolt` | Imágenes FTP | steghide + exiftool + binwalk |
| `stars_members_are_my_guinea_pig` | /hidden_closet/ MO Disk 1 | Cifrado por sustitución |

---

## 📝 10. Lecciones aprendidas

* Siempre revisar el **código fuente** de cada página — las rutas ocultas rara vez aparecen con fuzzing directo
* Cuando un string no tiene sentido con un solo decodificador, aplicar **capas múltiples** — la pista suele indicar cuántas
* La **esteganografía** puede manifestarse de formas distintas en una misma imagen: datos embebidos (`steghide`), metadatos EXIF (`exiftool`) o archivos comprimidos al final del binario (`binwalk`)
* El cifrado **Vigenère** sin clave es impráctico — en CTFs la clave siempre está escondida en el propio escenario
* Los archivos **GPG** simétricos se pueden descifrar si la passphrase fue obtenida por otro vector de la máquina
* `sudo -l` debe ser **siempre el primer comando** tras obtener una shell — los permisos mal configurados son el vector más común de escalada

---

## 🔗 Referencias

* [CyberChef — Decodificación multi-capa](https://gchq.github.io/CyberChef/)
* [dcode.fr — Identificador de cifrados](https://www.dcode.fr/cipher-identifier)
* [GTFOBins](https://gtfobins.github.io/)
* [HackTricks — Linux Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
* [Steghide](http://steghide.sourceforge.net/)

---

*Write-up por [Josue Rendón](https://github.com/Slasho01) · [TryHackMe Profile](https://tryhackme.com/p/Slasho)*
