# Hydra — Guía de Referencia
### Fuerza bruta de servicios de red · CTF & lab

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

---

## Qué es Hydra

Hydra es una herramienta de fuerza bruta para servicios de red. Prueba combinaciones de usuario/contraseña contra un servicio hasta encontrar credenciales válidas.

```
Tu máquina                         Servicio objetivo
    |                                      |
    |── SSH admin:password1 ─────────────→ | Failed
    |── SSH admin:password2 ─────────────→ | Failed
    |── SSH admin:password3 ─────────────→ | Success ← credenciales válidas
    ...
```

**¿Cuándo usarlo?** — Cuando has identificado un servicio de autenticación (SSH, FTP, HTTP login, RDP…) y tienes un usuario o lista de usuarios probable. Hydra no sirve para descubrir usuarios — para eso primero enumeras.

---

## Instalación

```bash
sudo apt install hydra

# Verificar instalación
hydra -h
```

---

## Sintaxis base

```bash
hydra [opciones] <target> <servicio>

# Con usuario y wordlist de contraseñas
hydra -l <usuario> -P <wordlist> <target> <servicio>

# Con wordlist de usuarios y wordlist de contraseñas
hydra -L <usuarios.txt> -P <passwords.txt> <target> <servicio>

# Con usuario y contraseña fijos (verificar si funcionan)
hydra -l <usuario> -p <password> <target> <servicio>
```

---

## Flags principales

| Flag | Descripción |
|---|---|
| `-l` | Usuario único |
| `-L` | Wordlist de usuarios |
| `-p` | Contraseña única |
| `-P` | Wordlist de contraseñas |
| `-t` | Hilos paralelos (default 16) |
| `-s` | Puerto no estándar |
| `-v` | Verbose — muestra intentos |
| `-V` | Muy verbose — muestra cada intento |
| `-f` | Parar al encontrar la primera credencial válida |
| `-F` | Parar al encontrar credencial válida en cualquier host |
| `-o` | Guardar resultados en fichero |
| `-e nsr` | Probar: n=nulo, s=usuario como pass, r=usuario invertido |
| `-u` | Iterar por usuarios antes que por contraseñas |
| `-w` | Timeout de espera (segundos) |
| `-W` | Tiempo entre intentos (segundos) — evadir rate limiting |
| `-I` | Ignorar restore file y empezar desde cero |
| `-R` | Reanudar sesión anterior |

---

## Wordlists recomendadas

```bash
# Contraseñas — por orden de uso en CTF
/usr/share/wordlists/rockyou.txt                          # la más usada
/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt
/usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt
/usr/share/wordlists/dirb/others/names.txt                # nombres comunes

# Usuarios
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
/usr/share/seclists/Usernames/Names/names.txt
/usr/share/wordlists/dirb/others/names.txt

# Descomprimir rockyou si es necesario
gunzip /usr/share/wordlists/rockyou.txt.gz
```

---

## 01 · SSH

```bash
# Usuario conocido, fuerza bruta de contraseña
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<target>

# Lista de usuarios y contraseñas
hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt ssh://<target>

# Puerto no estándar
hydra -l admin -P rockyou.txt -s 2222 ssh://<target>

# Más rápido con más hilos
hydra -l admin -P rockyou.txt -t 4 ssh://<target>
# Nota: SSH limita conexiones paralelas — t4 suele ser el máximo útil

# Parar al encontrar credenciales válidas
hydra -l admin -P rockyou.txt -f ssh://<target>

# Con verbose para ver progreso
hydra -l admin -P rockyou.txt -V ssh://<target>

# Probar usuario como contraseña y contraseña vacía
hydra -l admin -P rockyou.txt -e nsr ssh://<target>

# Guardar resultados
hydra -l admin -P rockyou.txt -o resultados.txt ssh://<target>
```

**¿Por qué `-t 4` en SSH?** — SSH tiene protección contra conexiones paralelas excesivas. Más de 4 hilos suele generar errores de conexión y resultados incorrectos.

---

## 02 · FTP

```bash
# Básico
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://<target>

# Puerto no estándar
hydra -l admin -P rockyou.txt -s 2121 ftp://<target>

# Lista de usuarios
hydra -L usuarios.txt -P rockyou.txt ftp://<target>

# Probar anonymous (usuario vacío o "anonymous")
hydra -l anonymous -p anonymous ftp://<target>
hydra -l "" -p "" ftp://<target>
```

---

## 03 · HTTP — formularios web

El más complejo de configurar. Necesitas identificar el método (GET/POST), los campos del formulario y cómo detectar un login fallido.

### HTTP POST form

```bash
hydra -l admin -P rockyou.txt <target> http-post-form \
    "/login:username=^USER^&password=^PASS^:F=Invalid credentials"

# Estructura del módulo http-post-form:
# "/ruta:campos_del_form:condicion_de_fallo"
#
# ^USER^ → hydra sustituye aquí el usuario
# ^PASS^ → hydra sustituye aquí la contraseña
# F=     → string que aparece cuando el login FALLA
# S=     → string que aparece cuando el login tiene ÉXITO (alternativa a F=)
```

### Ejemplos reales

```bash
# Login con mensaje de error "Invalid password"
hydra -l admin -P rockyou.txt 10.10.10.1 http-post-form \
    "/login.php:user=^USER^&pass=^PASS^:F=Invalid password"

# Login que redirige al fallar (302)
hydra -l admin -P rockyou.txt 10.10.10.1 http-post-form \
    "/login:username=^USER^&password=^PASS^:S=dashboard"
# S= → string que aparece SOLO en login exitoso

# Con cookie de sesión (CSRF token estático)
hydra -l admin -P rockyou.txt 10.10.10.1 http-post-form \
    "/login:user=^USER^&pass=^PASS^&token=abc123:F=Wrong"

# Con header adicional
hydra -l admin -P rockyou.txt 10.10.10.1 http-post-form \
    "/login:user=^USER^&pass=^PASS^:F=error:H=Cookie: session=xyz"
```

### HTTP GET form

```bash
hydra -l admin -P rockyou.txt <target> http-get-form \
    "/login?user=^USER^&pass=^PASS^:F=Invalid"
```

### HTTP Basic Auth

```bash
# Autenticación HTTP básica (ventana emergente del navegador)
hydra -l admin -P rockyou.txt http-get://<target>/ruta-protegida

# Con puerto
hydra -l admin -P rockyou.txt -s 8080 http-get://<target>/admin
```

### HTTPS

```bash
# SSL — usar https-post-form o https-get-form
hydra -l admin -P rockyou.txt <target> https-post-form \
    "/login:user=^USER^&pass=^PASS^:F=Invalid"

# HTTPS con Basic Auth
hydra -l admin -P rockyou.txt https-get://<target>/admin
```

**Cómo encontrar los campos del formulario:**

```bash
# Ver el código fuente del formulario de login
curl http://<target>/login | grep -i "form\|input\|action"

# O inspeccionar con Burp Suite — interceptar el POST y copiar los parámetros
```

---

## 04 · RDP

```bash
# Básico
hydra -l administrator -P rockyou.txt rdp://<target>

# Con lista de usuarios
hydra -L usuarios.txt -P rockyou.txt rdp://<target>

# Puerto no estándar
hydra -l administrator -P rockyou.txt -s 3390 rdp://<target>

# Reducir hilos — RDP es sensible a conexiones paralelas
hydra -l administrator -P rockyou.txt -t 4 rdp://<target>
```

---

## 05 · SMB

```bash
# Básico
hydra -l administrator -P rockyou.txt smb://<target>

# Lista de usuarios
hydra -L usuarios.txt -P rockyou.txt smb://<target>

# Dominio Windows
hydra -l "DOMINIO\\usuario" -P rockyou.txt smb://<target>
```

---

## 06 · MySQL / MariaDB

```bash
# Básico
hydra -l root -P rockyou.txt mysql://<target>

# Puerto no estándar
hydra -l root -P rockyou.txt -s 3307 mysql://<target>

# Lista de usuarios
hydra -L usuarios.txt -P rockyou.txt mysql://<target>
```

---

## 07 · PostgreSQL

```bash
hydra -l postgres -P rockyou.txt postgres://<target>

hydra -L usuarios.txt -P rockyou.txt postgres://<target>
```

---

## 08 · Telnet

```bash
hydra -l admin -P rockyou.txt telnet://<target>

# Con verbose para ver el prompt de login
hydra -l admin -P rockyou.txt -V telnet://<target>
```

---

## 09 · SMTP

```bash
# Fuerza bruta de autenticación SMTP
hydra -l usuario@dominio.com -P rockyou.txt smtp://<target>

# SMTPS (puerto 465)
hydra -l usuario@dominio.com -P rockyou.txt -s 465 smtp://<target>

# Con STARTTLS (puerto 587)
hydra -l usuario@dominio.com -P rockyou.txt smtp-starttls://<target>
```

---

## 10 · POP3 / IMAP

```bash
# POP3
hydra -l usuario -P rockyou.txt pop3://<target>
hydra -l usuario -P rockyou.txt -s 995 pop3s://<target>

# IMAP
hydra -l usuario -P rockyou.txt imap://<target>
hydra -l usuario -P rockyou.txt -s 993 imaps://<target>
```

---

## 11 · VNC

```bash
# VNC solo usa contraseña, sin usuario
hydra -P rockyou.txt vnc://<target>

# Con verbose
hydra -P rockyou.txt -V vnc://<target>
```

---

## 12 · Múltiples targets

```bash
# Lista de IPs en fichero
hydra -l admin -P rockyou.txt -M targets.txt ssh

# Con puerto en el fichero de targets (formato IP:puerto)
# targets.txt:
# 10.10.10.1:22
# 10.10.10.2:2222
hydra -l admin -P rockyou.txt -M targets.txt ssh
```

---

## Servicios soportados — referencia rápida

| Servicio | Módulo Hydra |
|---|---|
| SSH | `ssh` |
| FTP | `ftp` |
| HTTP Basic Auth | `http-get` |
| HTTP POST form | `http-post-form` |
| HTTPS POST form | `https-post-form` |
| RDP | `rdp` |
| SMB | `smb` |
| MySQL | `mysql` |
| PostgreSQL | `postgres` |
| Telnet | `telnet` |
| SMTP | `smtp` |
| POP3 | `pop3` |
| IMAP | `imap` |
| VNC | `vnc` |
| SNMP | `snmp` |
| LDAP | `ldap3` |
| Redis | `redis` |

```bash
# Ver todos los módulos disponibles
hydra -U http-post-form    # ayuda de un módulo específico
hydra --list-modules       # listar todos
```

---

## Consejos para CTF

### Identificar el string de fallo correctamente

```bash
# Antes de lanzar hydra — verificar manualmente el mensaje de error
curl -X POST http://<target>/login \
    -d "username=admin&password=wrongpass" \
    -v 2>&1 | grep -i "invalid\|error\|failed\|incorrect"
```

### Optimizar velocidad

```bash
# SSH / RDP / SMB — máximo t4 por limitaciones del protocolo
hydra -l admin -P rockyou.txt -t 4 ssh://<target>

# HTTP — se puede subir más
hydra -l admin -P rockyou.txt -t 64 http-post-form "/login:..."

# Con delay para evadir rate limiting o lockout
hydra -l admin -P rockyou.txt -W 2 ssh://<target>   # 2s entre intentos
```

### Probar primero contraseñas obvias

```bash
# -e nsr → prueba: (n) contraseña vacía, (s) usuario=contraseña, (r) usuario invertido
hydra -l admin -P rockyou.txt -e nsr ssh://<target>

# Crear wordlist pequeña con contraseñas típicas de CTF
cat << EOF > ctf-passwords.txt
admin
password
123456
password123
admin123
root
toor
letmein
welcome
qwerty
EOF

hydra -l admin -P ctf-passwords.txt ssh://<target>
```

### Reanudar sesión interrumpida

```bash
# Hydra guarda el estado automáticamente en hydra.restore
# Para reanudar:
hydra -R

# Para ignorar el restore e iniciar desde cero:
hydra -I -l admin -P rockyou.txt ssh://<target>
```

---

## Hydra vs otras herramientas

| | Hydra | Medusa | Ncrack | ffuf |
|---|---|---|---|---|
| Velocidad | Alta | Alta | Alta | Muy alta |
| Servicios | Muchos | Muchos | Pocos | Solo HTTP |
| HTTP forms | Sí | Limitado | No | Sí (más flexible) |
| Mantenimiento | Activo | Poco activo | Poco activo | Activo |
| CTF | ⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ (HTTP) |

> Para fuerza bruta de formularios HTTP complejos (con CSRF dinámico, JS…) ffuf o Burp Intruder son mejores opciones que Hydra.

---

## Flujo completo en CTF

```bash
# 1 — identificar servicios abiertos
nmap -sV -p 21,22,80,443,3389,3306 <target>

# 2 — enumerar usuarios posibles antes de atacar
# (via OSINT, enumeración SMB, SMTP VRFY, etc.)

# 3 — probar contraseñas obvias primero
hydra -l admin -P ctf-passwords.txt -e nsr -f ssh://<target>

# 4 — si no hay resultado, lanzar rockyou
hydra -l admin -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://<target>

# 5 — si hay formulario web, inspeccionar y lanzar
curl -X POST http://<target>/login -d "user=test&pass=test" -v 2>&1 | grep -i "invalid\|error"
hydra -l admin -P rockyou.txt <target> http-post-form \
    "/login:user=^USER^&pass=^PASS^:F=Invalid" -f -V

# 6 — con credenciales válidas → acceder al servicio
ssh admin@<target>
ftp <target>
```

---

*Hydra Reference — uso exclusivo en entornos autorizados · CTF / laboratorio*
*github.com/vanhauser-thc/thc-hydra*
