# Gobuster & ffuf — Guía de Referencia
### Fuzzing de directorios, subdominios y parámetros · CTF & lab

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

---

## Qué es el fuzzing web

El fuzzing web consiste en probar automáticamente listas de palabras (wordlists) contra un objetivo para descubrir recursos ocultos — directorios, ficheros, subdominios, parámetros o endpoints que no están enlazados públicamente pero existen en el servidor.

```
Tu máquina                         Servidor objetivo
    |                                      |
    |── GET /admin ──────────────────────→ | 404
    |── GET /login ──────────────────────→ | 200 ← encontrado
    |── GET /backup ─────────────────────→ | 301 ← encontrado
    |── GET /uploads ────────────────────→ | 403 ← existe pero restringido
    |── GET /xxxxxxx ────────────────────→ | 404
    ...
```

**Gobuster** — herramienta en Go, rápida, directa, muy usada en CTF.
**ffuf** — más flexible y configurable, ideal para fuzzing avanzado y parámetros.

---

## Wordlists — dónde están

En Kali Linux y Parrot las wordlists vienen en `/usr/share/wordlists/`.

```bash
# Ver wordlists disponibles
ls /usr/share/wordlists/
ls /usr/share/wordlists/dirb/
ls /usr/share/wordlists/dirbuster/

# Las más usadas en CTF
/usr/share/wordlists/dirb/common.txt              # directorios comunes — rápida
/usr/share/wordlists/dirb/big.txt                 # más completa
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  # la más usada en CTF
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt   # versión pequeña
/usr/share/seclists/Discovery/Web-Content/common.txt          # SecLists
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt  # subdominios

# Instalar SecLists si no está
sudo apt install seclists
```

**¿Qué wordlist usar?**

| Wordlist | Tamaño | Cuándo |
|---|---|---|
| `dirb/common.txt` | ~4.600 | Primera pasada rápida |
| `dirb/big.txt` | ~20.000 | Segunda pasada si common no da resultados |
| `dirbuster/directory-list-2.3-medium.txt` | ~220.000 | CTF — cobertura amplia |
| `seclists/common.txt` | ~4.700 | Buena calidad, bien mantenida |
| `seclists/raft-medium-words.txt` | ~63.000 | Equilibrio velocidad/cobertura |

---

## GOBUSTER

### Instalación

```bash
sudo apt install gobuster
# o
go install github.com/OJ/gobuster/v3@latest
```

### Modos disponibles

| Modo | Descripción |
|---|---|
| `dir` | Fuzzing de directorios y ficheros |
| `dns` | Enumeración de subdominios |
| `vhost` | Fuzzing de virtual hosts |
| `fuzz` | Fuzzing genérico (parámetros, headers…) |
| `s3` | Buckets S3 públicos |

---

### 01 · Gobuster — modo dir

```bash
# Sintaxis base
gobuster dir -u http://<target> -w <wordlist>

# Ejemplo básico
gobuster dir -u http://10.10.10.1 -w /usr/share/wordlists/dirb/common.txt

# Con extensiones de fichero
gobuster dir -u http://10.10.10.1 \
    -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
    -x php,html,txt,bak,zip

# Ignorar códigos de respuesta (ocultar 404, 403…)
gobuster dir -u http://10.10.10.1 -w wordlist.txt -b 404,403

# Mostrar solo códigos específicos
gobuster dir -u http://10.10.10.1 -w wordlist.txt --status-codes 200,301,302

# Con cookies de sesión (área autenticada)
gobuster dir -u http://10.10.10.1 -w wordlist.txt \
    -c "PHPSESSID=abc123; token=xyz"

# Con headers personalizados
gobuster dir -u http://10.10.10.1 -w wordlist.txt \
    -H "Authorization: Bearer TOKEN" \
    -H "X-Custom-Header: valor"

# Aumentar hilos (más rápido)
gobuster dir -u http://10.10.10.1 -w wordlist.txt -t 50

# Ignorar certificado SSL
gobuster dir -u https://10.10.10.1 -w wordlist.txt -k

# Guardar output
gobuster dir -u http://10.10.10.1 -w wordlist.txt -o resultados.txt

# Timeout por petición
gobuster dir -u http://10.10.10.1 -w wordlist.txt --timeout 10s

# Seguir redirecciones
gobuster dir -u http://10.10.10.1 -w wordlist.txt -r
```

**Flags más importantes:**

| Flag | Descripción |
|---|---|
| `-u` | URL objetivo |
| `-w` | Wordlist |
| `-x` | Extensiones a probar |
| `-t` | Hilos (default 10) |
| `-b` | Códigos a ignorar |
| `-c` | Cookies |
| `-H` | Headers personalizados |
| `-k` | Ignorar SSL |
| `-o` | Fichero de output |
| `-r` | Seguir redirecciones |
| `-q` | Modo silencioso (solo resultados) |

---

### 02 · Gobuster — modo dns

Enumera subdominios de un dominio objetivo.

```bash
# Básico
gobuster dns -d ejemplo.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Con resolución de IPs
gobuster dns -d ejemplo.com \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -i

# Usando un resolver DNS específico
gobuster dns -d ejemplo.com \
    -w wordlist.txt \
    --resolver 8.8.8.8

# Wildcard detection — si el dominio resuelve todo
gobuster dns -d ejemplo.com -w wordlist.txt --wildcard
```

---

### 03 · Gobuster — modo vhost

Fuzzing de virtual hosts — útil cuando varios dominios apuntan a la misma IP y quieres encontrar los que no conoces.

```bash
# Básico
gobuster vhost -u http://10.10.10.1 \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Añadir dominio base
gobuster vhost -u http://10.10.10.1 \
    -w wordlist.txt \
    --domain ejemplo.com

# Excluir longitudes de respuesta para reducir ruido
gobuster vhost -u http://10.10.10.1 \
    -w wordlist.txt \
    --exclude-length 302
```

**Vhost vs DNS:**
- `dns` resuelve subdominios reales en DNS
- `vhost` prueba el header `Host:` — descubre sitios en el mismo servidor aunque no tengan registro DNS

---

## FFUF

### Instalación

```bash
sudo apt install ffuf
# o
go install github.com/ffuf/ffuf/v2@latest
```

### Concepto clave — FUZZ

ffuf usa la palabra `FUZZ` como marcador de posición. Donde escribas `FUZZ`, ffuf sustituirá cada entrada de la wordlist.

```
http://target/FUZZ          → fuzzing de directorios
http://target/?id=FUZZ      → fuzzing de parámetros
http://FUZZ.target.com      → fuzzing de subdominios
```

---

### 04 · ffuf — fuzzing de directorios

```bash
# Básico
ffuf -u http://<target>/FUZZ -w wordlist.txt

# Con extensiones
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -e .php,.html,.txt,.bak,.zip

# Filtrar por código de respuesta (mostrar solo 200,301,302)
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -mc 200,301,302

# Filtrar por tamaño de respuesta (ocultar respuestas de N bytes)
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -fs 4242

# Filtrar por número de palabras en respuesta
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -fw 12

# Filtrar por número de líneas
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -fl 10

# Aumentar hilos
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -t 100

# Con cookies
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -b "PHPSESSID=abc123"

# Con headers
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -H "Authorization: Bearer TOKEN"

# Ignorar SSL
ffuf -u https://<target>/FUZZ -w wordlist.txt \
    -k

# Guardar output
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -o resultados.json -of json

# Output en markdown
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -o resultados.md -of md

# Modo silencioso
ffuf -u http://<target>/FUZZ -w wordlist.txt -s

# Delay entre peticiones (evadir rate limiting)
ffuf -u http://<target>/FUZZ -w wordlist.txt \
    -p 0.1          # 0.1 segundos entre peticiones
```

**Flags de filtrado — clave para reducir ruido:**

| Flag | Descripción |
|---|---|
| `-mc` | Mostrar solo estos códigos (match) |
| `-fc` | Ocultar estos códigos (filter) |
| `-ms` | Mostrar solo respuestas de N bytes |
| `-fs` | Ocultar respuestas de N bytes |
| `-mw` | Mostrar solo respuestas con N palabras |
| `-fw` | Ocultar respuestas con N palabras |
| `-ml` | Mostrar solo respuestas con N líneas |
| `-fl` | Ocultar respuestas con N líneas |

---

### 05 · ffuf — fuzzing de subdominios

```bash
# Fuzzing de subdominios via header Host
ffuf -u http://<target> \
    -H "Host: FUZZ.<dominio>" \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Filtrar por tamaño para eliminar respuestas genéricas
ffuf -u http://10.10.10.1 \
    -H "Host: FUZZ.ejemplo.com" \
    -w wordlist.txt \
    -fs 1234         # tamaño de la respuesta por defecto del servidor

# Ejemplo real
ffuf -u http://10.10.10.1 \
    -H "Host: FUZZ.permx.htb" \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -fc 302 -fs 4
```

**¿Por qué filtrar por tamaño (`-fs`)?** — Primero lanza ffuf sin filtros, mira el tamaño de las respuestas negativas (404 genérico del servidor) y luego filtra ese tamaño. Lo que queda son los subdominios reales.

---

### 06 · ffuf — fuzzing de parámetros GET

```bash
# Fuzzing del valor de un parámetro
ffuf -u "http://<target>/index.php?id=FUZZ" \
    -w /usr/share/seclists/Fuzzing/numbers.txt

# Fuzzing del nombre del parámetro
ffuf -u "http://<target>/index.php?FUZZ=valor" \
    -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# Fuzzing de LFI
ffuf -u "http://<target>/index.php?page=FUZZ" \
    -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
    -fs 1234

# Fuzzing de SQLi básico
ffuf -u "http://<target>/index.php?id=FUZZ" \
    -w /usr/share/seclists/Fuzzing/SQLi/Generic-SQLi.txt
```

---

### 07 · ffuf — fuzzing de parámetros POST

```bash
# Fuzzing en cuerpo POST
ffuf -u http://<target>/login \
    -X POST \
    -d "username=admin&password=FUZZ" \
    -w /usr/share/wordlists/rockyou.txt \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -fc 302         # ocultar redirecciones (login fallido)

# Fuzzing de JSON POST
ffuf -u http://<target>/api/login \
    -X POST \
    -d '{"username":"admin","password":"FUZZ"}' \
    -w wordlist.txt \
    -H "Content-Type: application/json" \
    -mc 200
```

---

### 08 · ffuf — fuzzing con múltiples wordlists

ffuf soporta múltiples marcadores (W1, W2…) para fuzzing combinado.

```bash
# Dos wordlists — fuzzing de usuario y contraseña simultáneo
ffuf -u http://<target>/login \
    -X POST \
    -d "username=W1&password=W2" \
    -w usuarios.txt:W1 \
    -w passwords.txt:W2 \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -fc 302

# Modo clusterbomb — todas las combinaciones
ffuf -u http://<target>/FUZZ1/FUZZ2 \
    -w wordlist1.txt:FUZZ1 \
    -w wordlist2.txt:FUZZ2
```

---

### 09 · ffuf — fuzzing de headers y cookies

```bash
# Fuzzing del valor de un header
ffuf -u http://<target>/ \
    -H "X-Forwarded-For: FUZZ" \
    -w /usr/share/seclists/Fuzzing/

# Fuzzing de User-Agent
ffuf -u http://<target>/ \
    -H "User-Agent: FUZZ" \
    -w /usr/share/seclists/Fuzzing/User-Agents.txt

# Fuzzing de cookie
ffuf -u http://<target>/ \
    -b "session=FUZZ" \
    -w wordlist.txt
```

---

## Gobuster vs ffuf

| | Gobuster | ffuf |
|---|---|---|
| Velocidad | Alta | Alta |
| Facilidad de uso | Simple, directo | Más opciones, más complejo |
| Fuzzing de parámetros | Limitado | Completo |
| Múltiples wordlists | No | Sí |
| Filtros | Básicos | Muy completos |
| Output formats | txt | json, csv, md, html |
| Modos | dir, dns, vhost, fuzz, s3 | Universal (FUZZ en cualquier parte) |
| Mejor para | Reconocimiento rápido | Fuzzing avanzado |

**Recomendación:** usar Gobuster para el reconocimiento inicial rápido y ffuf para fuzzing más específico (parámetros, subdominios, POST).

---

## Códigos de respuesta HTTP — referencia rápida

| Código | Significado | En fuzzing |
|---|---|---|
| 200 | OK | Recurso existe y es accesible |
| 301 | Redirect permanente | Directorio existe (añadir `/`) |
| 302 | Redirect temporal | Puede indicar login requerido |
| 403 | Forbidden | Existe pero sin acceso — interesante |
| 404 | Not Found | No existe |
| 500 | Server Error | El payload causó error — muy interesante |

> El **403** es especialmente interesante — el recurso existe pero está restringido. A veces se puede bypassear con técnicas de path traversal o headers.

---

## Combos habituales en CTF

### Reconocimiento inicial rápido

```bash
gobuster dir -u http://<target> \
    -w /usr/share/wordlists/dirb/common.txt \
    -x php,html,txt \
    -t 50 -q
```

### Pasada completa con ffuf

```bash
ffuf -u http://<target>/FUZZ \
    -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
    -e .php,.html,.txt,.bak,.zip,.conf \
    -mc 200,301,302,403 \
    -t 100 \
    -o resultados.json -of json
```

### Descubrir subdominios

```bash
# Con gobuster
gobuster dns -d <dominio> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -t 50

# Con ffuf — añadir al /etc/hosts los que encuentres
ffuf -u http://<target> \
    -H "Host: FUZZ.<dominio>" \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -fs <tamaño_respuesta_negativa>
```

### Fuzzing de parámetro con LFI

```bash
ffuf -u "http://<target>/index.php?page=FUZZ" \
    -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
    -mc 200 \
    -fs <tamaño_respuesta_normal>
```

### Fuerza bruta de login

```bash
ffuf -u http://<target>/login \
    -X POST \
    -d "username=admin&password=FUZZ" \
    -w /usr/share/wordlists/rockyou.txt \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -fc 200 \        # ocultar login fallido (200 con mensaje de error)
    -mc 302          # mostrar solo redirección (login exitoso)
```

---

## Flujo completo en CTF

```bash
# 1 — reconocimiento inicial
gobuster dir -u http://<target> \
    -w /usr/share/wordlists/dirb/common.txt \
    -x php,html,txt -t 50

# 2 — si hay subdominio sospechoso → añadir a /etc/hosts
echo "<ip> <subdominio>.<dominio>" >> /etc/hosts

# 3 — pasada más profunda en rutas interesantes
ffuf -u http://<target>/admin/FUZZ \
    -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
    -e .php,.txt,.bak \
    -mc 200,301,403

# 4 — si hay formulario de login → fuerza bruta
ffuf -u http://<target>/login \
    -X POST \
    -d "user=admin&pass=FUZZ" \
    -w /usr/share/wordlists/rockyou.txt \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -fc 200 -mc 302

# 5 — si hay parámetro → fuzzing de LFI/SQLi
ffuf -u "http://<target>/page.php?file=FUZZ" \
    -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
    -fs <tamaño_normal>
```

---

*Gobuster & ffuf Reference — uso exclusivo en entornos autorizados · CTF / laboratorio*
*github.com/OJ/gobuster · github.com/ffuf/ffuf · github.com/danielmiessler/SecLists*
