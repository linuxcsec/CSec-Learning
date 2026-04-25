# nmap — Guía de Referencia

> Uso siempre en redes y sistemas sobre los que tengas autorización explícita.

**Leyendas:**
- 🔴 `root` — requiere sudo/root
- 🔵 `CTF` — especialmente útil en CTF/pentesting
- 🟡 `lento` — puede tardar mucho
- 🟢 `recomendado` — uso habitual

---

## 01 · Descubrimiento de hosts

| Comando | Descripción |
|---|---|
| `nmap -sn 192.168.1.0/24` | Ping sweep — descubre hosts activos sin escanear puertos |
| `nmap -Pn 192.168.1.1` | Omite el descubrimiento de host y fuerza el escaneo de puertos |
| `nmap -PR 192.168.1.0/24` | Descubrimiento ARP (solo LAN local) |

**`-sn`** — Es el primer paso en reconocimiento: mapear qué IPs responden antes de gastar tiempo en escaneos de puertos. Más rápido y menos ruidoso que un escaneo completo. 🟢

**`-Pn`** — Muchos hosts bloquean pings ICMP pero tienen puertos abiertos. Con `-Pn` nmap los trata como activos y escanea igualmente. Imprescindible en redes con firewalls que filtran ICMP. 🔵

**`-PR`** — ARP no puede ser filtrado por firewalls en la misma red local, por lo que es el método más fiable en LAN. Evita falsos negativos que ocurren con ICMP bloqueado. 🔴

---

## 02 · Escaneo de puertos

| Comando | Descripción |
|---|---|
| `nmap 192.168.1.1` | Escaneo básico de los 1000 puertos más comunes |
| `nmap -p- 192.168.1.1` | Escaneo de los 65535 puertos |
| `nmap -p 22,80,443,3306 192.168.1.1` | Puertos específicos de interés |
| `nmap --top-ports 200 192.168.1.1` | Los N puertos estadísticamente más comunes |

**Básico** — El punto de partida. Cubre la mayoría de servicios habituales (SSH, HTTP, RDP, SMB…) en segundos. Suficiente para una primera impresión.

**`-p-`** — Los administradores mueven servicios críticos a puertos no estándar para reducir ruido. En CTF y auditorías reales, siempre hay servicios ocultos en puertos altos. Combínalo con `-T4` para acelerar. 🟢 🟡

**Puertos específicos** — Cuando ya sabes qué servicios buscas (e.g., verificar si MySQL está expuesto públicamente), esto es mucho más rápido que escanear todo.

**`--top-ports`** — Equilibrio entre velocidad y cobertura. Nmap prioriza puertos por frecuencia real en internet, por lo que 200 puertos cubren >90% de servicios típicos.

---

## 03 · Tipos de escaneo TCP

| Comando | Descripción |
|---|---|
| `nmap -sS 192.168.1.1` | SYN scan (half-open) — el más usado |
| `nmap -sT 192.168.1.1` | TCP connect scan — completa el handshake |
| `nmap -sU 192.168.1.1` | UDP scan |
| `nmap -sA 192.168.1.1` | ACK scan — mapea reglas de firewall |
| `nmap -sN / -sF / -sX 192.168.1.1` | NULL, FIN y Xmas scans |

**`-sS`** — Envía un SYN y analiza la respuesta sin completar el handshake TCP. Más rápido, menos registrado en logs de aplicación (el OS cierra con RST antes de que la app vea la conexión). El default cuando tienes root. 🔴 🟢

**`-sT`** — No requiere privilegios. Usa la syscall `connect()` del sistema. Más detectable porque deja conexiones en logs, pero es la única opción sin root.

**`-sU`** — DNS (53), SNMP (161), TFTP, NTP corren sobre UDP. Se pasan por alto frecuentemente. El escaneo UDP es lento porque los puertos cerrados responden con ICMP Port Unreachable (con rate limiting). Úsalo apuntando a puertos concretos. 🔴 🟡

**`-sA`** — No detecta puertos abiertos, sino si están filtrados o no filtrados. Un puerto "unfiltered" significa que el paquete llega al host. Ideal como paso previo a un escaneo más preciso. 🔴 🔵

**NULL / FIN / Xmas** — Explotan el RFC TCP: los puertos cerrados responden RST, los abiertos no responden. Evaden firewalls que solo filtran SYN. No funcionan contra Windows. Útiles en sistemas UNIX/Linux. 🔴 🔵

---

## 04 · Detección de servicios y OS

| Comando | Descripción |
|---|---|
| `nmap -sV 192.168.1.1` | Detecta versiones de servicios |
| `nmap -sV --version-intensity 9 192.168.1.1` | Máxima intensidad en detección de versiones (0–9) |
| `nmap -O 192.168.1.1` | Detecta el sistema operativo |
| `nmap -A 192.168.1.1` | Agresivo: OS + versiones + scripts default + traceroute |

**`-sV`** — Saber que el puerto 22 está abierto no basta — necesitas "OpenSSH 7.4" para buscar CVEs específicos. Indispensable antes de buscar exploits. 🟢

**`--version-intensity 9`** — La intensidad controla cuántas sondas envía nmap. Mayor intensidad = más precisión pero más ruido y tiempo. Úsalo en servicios que no identifica con la configuración por defecto (5).

**`-O`** — El OS determina qué vulnerabilidades aplican. Windows XP, Linux 2.6, FreeBSD tienen superficies de ataque muy distintas. Necesita al menos un puerto abierto y otro cerrado para funcionar bien. 🔴

**`-A`** — El combo completo de reconocimiento en un solo flag. Muy útil en CTF o cuando quieres toda la información posible. Es ruidoso — no apto para evasión. 🔵 🔴

---

## 05 · Nmap Scripting Engine (NSE)

| Comando | Descripción |
|---|---|
| `nmap -sC 192.168.1.1` | Scripts de la categoría "default" |
| `nmap --script=vuln 192.168.1.1` | Scripts de detección de vulnerabilidades |
| `nmap --script=smb-vuln* 192.168.1.1` | Vulnerabilidades SMB (EternalBlue, MS08-067…) |
| `nmap --script=http-enum -p80,443 192.168.1.1` | Enumeración de rutas y directorios web |
| `nmap --script=ftp-anon -p21 192.168.1.1` | Comprobar si FTP permite login anónimo |
| `nmap --script=banner 192.168.1.1` | Captura banners de servicios |

**`-sC`** — Los scripts default son seguros, rápidos y muy informativos: detectan configuraciones inseguras, enumeran shares SMB, extraen certificados SSL, etc. Siempre inclúyelo junto a `-sV`. 🟢

**`vuln`** — Prueba CVEs comunes como EternalBlue (MS17-010), Heartbleed, etc. Puede ser intrusivo. En CTF es clave para identificar el vector de ataque rápidamente. 🔵 🟡

**`smb-vuln*`** — SMB es históricamente uno de los vectores más explotados en Windows. Este wildcard lanza todos los scripts de vuln relacionados con SMB en un solo comando. 🔵

**`http-enum`** — Detecta `/admin`, `/phpmyadmin`, `/backup`, etc. sin herramientas extra. Útil como paso rápido de reconocimiento web antes de usar gobuster o ffuf.

**`ftp-anon`** — El acceso anónimo FTP es un misconfiguration clásico que da lectura (y a veces escritura) de archivos sin credenciales. Muy frecuente en CTFs de nivel inicial. 🔵

---

## 06 · Velocidad — Timing Templates (T0–T5)

| Plantilla | Nombre | Descripción |
|---|---|---|
| `-T0` | Paranoid | Extremadamente lento, evasión IDS. 5 min entre sondas |
| `-T1` | Sneaky | Muy lento, evasión IDS |
| `-T2` | Polite | Lento, reduce carga en la red |
| `-T3` | Normal | Default — equilibrio velocidad/fiabilidad |
| `-T4` | Aggressive | Rápido en redes de baja latencia 🟢 🔵 |
| `-T5` | Insane | Máxima velocidad, muchos falsos negativos |

**T0/T1** — Diseñados para evadir IDS/IPS basados en correlación temporal. Solo en contextos donde la evasión es crítica. Puede tardar horas. 🟡

**T4** — El setting más usado en CTFs y laboratorios locales. En redes lentas puede dar falsos negativos (puertos marcados como cerrados por timeout). 🟢 🔵

**T5** — Sacrifica precisión por velocidad. Útil solo en loopback o redes Gigabit de laboratorio.

---

## 07 · Evasión de firewalls e IDS

| Comando | Descripción |
|---|---|
| `nmap -f 192.168.1.1` | Fragmenta paquetes IP en trozos de 8 bytes |
| `nmap -D RND:10 192.168.1.1` | 10 IPs señuelo aleatorias junto a la tuya |
| `nmap --source-port 53 192.168.1.1` | Simula tráfico saliente desde puerto DNS (53) |
| `nmap --spoof-mac 0 192.168.1.1` | MAC address aleatoria en cada escaneo |

**`-f`** — Algunos IDS/firewalls tienen problemas para reensamblar paquetes muy fragmentados. Reduce la probabilidad de detección. Puede causar falsos negativos en redes con MTU bajo. 🔴

**`-D RND:10`** — El destino ve tráfico de múltiples IPs simultáneamente, dificultando identificar cuál es el atacante real. No oculta tu IP completamente, pero enturbia el análisis forense. 🔴 🔵

**`--source-port 53`** — Algunos firewalls permiten tráfico entrante desde el puerto 53 asumiendo que es respuesta DNS legítima. Esta técnica explota ese comportamiento. 🔴

**`--spoof-mac 0`** — Útil en redes donde el logging se hace por MAC (switches gestionados, NAC). El `0` genera una MAC aleatoria; también puedes poner un vendor específico (`"Apple"`). 🔴

---

## 08 · Formatos de salida

| Comando | Descripción |
|---|---|
| `nmap -oA escaneo` | Los tres formatos a la vez: .nmap, .xml, .gnmap 🟢 |
| `nmap -oN resultado.txt` | Output normal (legible) |
| `nmap -oX resultado.xml` | XML — importable en Metasploit, Faraday |
| `nmap -oG resultado.gnmap` | Grepable — una línea por host |
| `nmap -v / -vv` | Verbosidad — muestra resultados en tiempo real |

**`-oA`** — Regla de oro: siempre guarda resultados. El `-oA` es el más práctico — `.nmap` para lectura humana, `.xml` para importar en Metasploit, `.gnmap` para grep rápido. No repitas escaneos innecesariamente. 🟢

**`-oG`** — Permite extraer rápidamente IPs con puertos específicos:
```bash
grep "22/open" resultado.gnmap | awk '{print $2}'
```

**`-v`** — Con `-v` nmap imprime los puertos abiertos según los descubre, sin esperar al final. En escaneos largos (`-p-`) es muy útil para ir trabajando mientras sigue corriendo.

---

## Combos de referencia

### Reconocimiento inicial rápido (CTF / lab)
```bash
nmap -sS -sV -sC -O -p- -T4 -oA full_scan <target>
```
SYN scan completo + versiones + scripts default + OS detection. Requiere root. El combo estándar cuando quieres toda la información posible y la velocidad importa.

### Dos fases — rápido primero, profundo después
```bash
# Fase 1: descubrir todos los puertos abiertos
nmap -sS -p- -T4 --open -oG puertos.gnmap <target>

# Fase 2: analizar solo esos puertos en profundidad
nmap -sV -sC -p $(grep open puertos.gnmap | grep -oP '\d+(?=/open)' | tr '\n' ',') -oA detail <target>
```
Reduce el tiempo total considerablemente en rangos completos.

### Modo silencioso — evasión básica
```bash
nmap -sS -sV -p- -T2 -f -D RND:5 --source-port 53 -oA stealth <target>
```
Fragmentación + decoys + source port spoofing + timing lento. Más difícil de detectar pero mucho más lento.

### Búsqueda de vulnerabilidades en Windows
```bash
nmap -sS -sV --script=smb-vuln*,vuln -p 139,445,3389 -T4 <target>
```
Apunta directamente a SMB y RDP. Detecta EternalBlue, MS08-067 y otras vulns críticas.

---

*nmap 7.x — nmap.org — uso ético y autorizado únicamente*
