# Biohazard — CTF Writeup

**Plataforma:** TryHackMe  
**Dificultad:** Media  
**OS:** Linux (Ubuntu 18.04)  
**IP:** 10.67.128.169

---

## Índice

1. [Reconocimiento](#reconocimiento)
2. [Enumeración Web](#enumeración-web)
3. [Recolección de Crests](#recolección-de-crests)
4. [FTP y Esteganografía](#ftp-y-esteganografía)
5. [Acceso SSH y Escalada](#acceso-ssh-y-escalada)
6. [Flags capturadas](#flags-capturadas)

---

## Reconocimiento

### Escaneo de puertos

```bash
nmap -sV -p- 10.67.128.169
```

**Resultados:**

| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 21/tcp | open | ftp | vsftpd 3.0.3 |
| 22/tcp | open | ssh | OpenSSH 7.6p1 Ubuntu |
| 80/tcp | open | http | Apache httpd 2.4.29 |

Tres servicios expuestos. El vector de entrada inicial es la web (HTTP).

**Información adicional:**
- Nombre del equipo: *The Stars Alpha Team*
- Posibles usuarios: Chris, Jill, Barry, Weasker, Joseph
- Equipo Bravo: Raccoon City

---

## Enumeración Web

### /mansionmain/

Revisando el código fuente (`view-source:http://10.67.128.169/mansionmain/`) se descubre una ruta oculta:

```
/diningRoom/
```

---

### /diningRoom/

En el código fuente se encuentra un string en Base64:

```
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
```

Decodificado revela: `/teaRoom/`

Accediendo a `emblem.php` se obtiene la primera flag:

> **Flag:** `emblem{fec832623ea498e20bf4fe1821d58727}`

---

### /teaRoom/

En `master_of_unlock.html` se obtiene la segunda flag y se descubre la ruta `/artRoom/`:

> **Flag:** `lock_pick{037b35e2ff90916a9abf99129c8e1837}`

---

### /artRoom/

`MansionMap.html` revela el mapa completo con todas las ubicaciones:

```
/diningRoom/
/teaRoom/
/artRoom/
/barRoom/
/diningRoom2F/
/tigerStatusRoom/
/galleryRoom/
/studyRoom/
/armorRoom/
/attic/
```

---

### /barRoom/

Se usa `lock_pick{037b35e2ff90916a9abf99129c8e1837}` para desbloquear la puerta.

Se obtiene una nota musical (*Moonlight Sonata*) con el siguiente string en **Base32**:

```
NV2XG2LDL5ZWQZLFOR5TGNRSMQ3TEZDFMFTDMNLGGVRGIYZWGNSGCZLDMU3GCMLGGY3TMZL5
```

Decodificado:

> **Flag:** `music_sheet{362d72deaf65f5bdc63daece6a1f676e}`

Ingresando la flag en `gold_emblem.php`:

> **Flag:** `gold_emblem{58a8c41a9d08b8a4e38d02a4d7ff4843}`

Ingresando el `emblem` original en el bar → se obtiene la palabra **`rebecca`**.

---

### /diningRoom/ (revisita con gold emblem)

Ingresando la gold emblem aparece texto cifrado:

```
klfvg ks r wimgnd biz mpuiui ulg fiemok tqod. Xii jvmc tbkg ks tempgf tyi_hvgct_jljinf_kvc
```

Intentando César a -8 no produce resultado útil. Se usa **Vigenère** con clave `rebecca`:

```
there is a shield key inside the dining room. The html page is called the_great_shield_key
```

Accediendo a `http://10.67.128.169/diningRoom/the_great_shield_key.html`:

> **Flag:** `shield_key{48a7a9227cd7eb89f0a062590798cbac}`

---

### /diningRoom2F/

Texto cifrado en **ROT13**:

```
Lbh trg gur oyhr trz ol chfuvat gur fgnghf gb gur ybjre sybbe...
```

Decodificado:

```
You get the blue gem by pushing the status to the lower floor.
The gem is on the diningRoom first floor. Visit sapphire.html
```

Accediendo a `sapphire.html`:

> **Flag:** `blue_jewel{e1d457e96cac640f863ec7bc475d48aa}`

---

## Recolección de Crests

Se deben recolectar 4 crests, combinarlos y decodificar para obtener las credenciales FTP.

### Crest 1 — /tigerStatusRoom/

Requiere ingresar el `blue_jewel`. Resultado en Base64 → Base32:

```
S0pXRkVVS0pKQkxIVVdTWUpFM0VTUlk9
→ (Base64) → RlRQIHVzZXI6IG...
```

- 2 capas de codificación, 14 letras

### Crest 2 — /galleryRoom/

```
GVFWK5KHK5WTGTCILE4DKY3DNN4GQQRTM5AVCTKE
→ (Base32) → (Base58) → 5KeuGWm3LHY85cckxhB3gAQMD
```

- 2 capas de codificación, 18 letras

### Crest 3 — /armorRoom/

Requiere `shield_key{48a7a9227cd7eb89f0a062590798cbac}`.

```
MDAxMTAxMTAg... (Base64 → Binario → Hex)
→ c3M6IHlvdV9jYW50X2h...
```

- 3 capas de codificación, 19 letras

### Crest 4 — /attic/

Requiere `shield_key`.

```
gSUERauVpvKzRpyPpuYz66JDmRTbJubaoArM6CAQsnVwte6zF9J4GGYyun3k5qM9ma4s
→ (Base58) → (Hex) → pZGVfZm9yZXZlcg==
```

- 2 capas de codificación, 17 caracteres

### Combinación final

Concatenando los 4 crests y decodificando en Base64:

```
RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==
```

```
FTP user: hunter
FTP pass: you_cant_hide_forever
```

---

## FTP y Esteganografía

### Conexión FTP

```bash
ftp 10.67.128.169
# Usuario: hunter
# Contraseña: you_cant_hide_forever
```

Archivos disponibles:

```
001-key.jpg  (7994 bytes)
002-key.jpg  (2210 bytes)
003-key.jpg  (2146 bytes)
helmet_key.txt.gpg
important.txt
```

`important.txt` contiene una nota de Barry a Jill indicando que:
- El helmet key está en el archivo GPG
- Existe una puerta en `/hidden_closet/`

### Extracción de datos ocultos

**001-key.jpg → steghide**

```bash
steghide extract -sf 001-key.jpg
# passphrase: (vacía)
cat key-001.txt
# cGxhbnQ0Ml9jYW
```

**002-key.jpg → exiftool**

```bash
exiftool 002-key.jpg
# Comment: 5fYmVfZGVzdHJveV9
```

**003-key.jpg → binwalk**

```bash
binwalk -e 003-key.jpg
cat _003-key.jpg.extracted/key-003.txt
# 3aXRoX3Zqb2x0
```

### Passphrase GPG

Concatenando los 3 fragmentos y decodificando Base64:

```
cGxhbnQ0Ml9jYW + 5fYmVfZGVzdHJveV9 + 3aXRoX3Zqb2x0
→ plant42_can_be_destroy_with_vjolt
```

Descifrando el archivo GPG:

```bash
gpg -d helmet_key.txt.gpg
# passphrase: plant42_can_be_destroy_with_vjolt
```

> **Flag:** `helmet_key{458493193501d2b94bbab2e727f8db4b}`

---

## Acceso SSH y Escalada

### /studyRoom/

Requiere `helmet_key`. Se descarga `doom.tar.gz` que contiene `eagle-medal.txt`:

```
SSH user: umbrella_guest
```

### /hidden_closet/

Requiere `helmet_key`. Se encuentran:
- **MO Disk 1**: texto cifrado con Vigenère
- **Wolf Medal**: contraseña SSH

Descifrando el MO Disk 1 con Vigenère (clave: `albert` del MO Disk 2):

```
weasker login password: stars_members_are_my_guinea_pig
```

Del `wolf_medal`:

```
SSH password: T_virus_rules
```

### Primer acceso SSH

```bash
ssh umbrella_guest@10.67.128.169
# Password: T_virus_rules
```

Navegando a `/home/weasker/` se lee `weasker_note.txt` (diálogo narrativo del juego).

### Pivote a weasker

```bash
ssh weasker@10.67.128.169
# Password: stars_members_are_my_guinea_pig
```

Se lee `chris.txt`, que menciona el **MO Disk 2** con clave `albert`.

### Escalada a root

```bash
sudo -l
# (verificar permisos)
sudo su -
# root obtenido
```

---

## Flags capturadas

| Flag | Ubicación |
|------|-----------|
| `emblem{fec832623ea498e20bf4fe1821d58727}` | /diningRoom/emblem.php |
| `lock_pick{037b35e2ff90916a9abf99129c8e1837}` | /teaRoom/master_of_unlock.html |
| `music_sheet{362d72deaf65f5bdc63daece6a1f676e}` | /barRoom/ (Base32) |
| `gold_emblem{58a8c41a9d08b8a4e38d02a4d7ff4843}` | /barRoom/gold_emblem.php |
| `shield_key{48a7a9227cd7eb89f0a062590798cbac}` | /diningRoom/the_great_shield_key.html |
| `blue_jewel{e1d457e96cac640f863ec7bc475d48aa}` | /diningRoom2F/sapphire.html |
| `helmet_key{458493193501d2b94bbab2e727f8db4b}` | FTP → GPG |
| `plant42_can_be_destroy_with_vjolt` | Esteganografía (passphrase GPG) |
| `stars_members_are_my_guinea_pig` | /hidden_closet/ MO Disk 1 |

---

## Técnicas utilizadas

- Enumeración web manual (código fuente, rutas ocultas)
- Decodificación: Base64, Base32, Base58, ROT13, Binario, Hex
- Cifrado Vigenère (clave: `rebecca`, `albert`)
- Esteganografía: `steghide`, `exiftool`, `binwalk`
- Descifrado GPG con passphrase
- Pivote de usuarios vía SSH
- Escalada de privilegios con `sudo`
