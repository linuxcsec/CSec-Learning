# TryHackMe — Net Sec Challenge
> **Ruta:** Pre-Security / Jr Penetration Tester  
> **Dificultad:** Fácil  
> **Fecha:** Abril 2026

---

## Índice
1. [Reconocimiento de puertos](#1-reconocimiento-de-puertos)
2. [Banner Grabbing — SSH](#2-banner-grabbing--ssh)
3. [Enumeración de FTP en puerto no estándar](#3-enumeración-de-ftp-en-puerto-no-estándar)
4. [Fuerza bruta FTP con Hydra](#4-fuerza-bruta-ftp-con-hydra)
5. [Escaneo sigiloso — IDS Evasion](#5-escaneo-sigiloso--ids-evasion)
6. [Conceptos teóricos clave](#6-conceptos-teóricos-clave)

---

## 1. Reconocimiento de puertos

### Objetivo
Identificar todos los puertos abiertos en la máquina objetivo, incluyendo los que se encuentran fuera del rango estándar (por encima de 1000 y por debajo de 10.000).

### Comando utilizado
```bash
nmap -n -T5 -p 1000-9999 <IP>
```

### Explicación de flags
| Flag | Descripción |
|------|-------------|
| `-n` | Desactiva la resolución DNS, reduciendo ruido y tiempo de escaneo |
| `-T5` | Timing agresivo (escala T0–T5). Adecuado en entornos CTF donde la estabilidad no es crítica |
| `-p 1000-9999` | Rango de puertos objetivo. Se omite el 1–999 (well-known ports) ya que la pregunta apunta a puertos menos comunes |

### Concepto: Well-known vs Registered ports
- **Puertos 0–1023:** Well-known ports (HTTP:80, SSH:22, FTP:21, SMTP:25...)
- **Puertos 1024–49151:** Registered/dynamic ports — aquí suelen aparecer servicios en configuraciones no estándar
- **Puertos 49152–65535:** Ephemeral ports

---

## 2. Banner Grabbing — SSH

### Objetivo
Obtener el header/banner del servicio SSH para identificar la versión del servidor y encontrar información oculta.

### Comando utilizado
```bash
nmap -sV -p 22 <IP>
```

### Explicación de flags
| Flag | Descripción |
|------|-------------|
| `-sV` | Service Version Detection — fuerza al servicio a revelar su banner e información de versión |
| `-p 22` | Limita el escaneo al puerto SSH estándar |

### Concepto: Banner Grabbing
Cuando un servicio TCP recibe una conexión, frecuentemente devuelve un **banner** con información sobre el software y versión en uso. Esto es especialmente común en SSH, FTP y SMTP. El banner grabbing es una técnica de reconocimiento pasivo que no requiere autenticación.

---

## 3. Enumeración de FTP en puerto no estándar

### Objetivo
Identificar la versión del servicio FTP que corre en un puerto no estándar (distinto al puerto 21).

### Comando utilizado
```bash
nmap -sV -p 1000-9999 <IP>
```

### Concepto: Servicios en puertos no estándar
Los administradores a veces mueven servicios comunes (FTP, SSH) a puertos no estándar como medida de seguridad por oscuridad (*security through obscurity*). Aunque reduce el ruido de bots automatizados, no es una medida de seguridad robusta ya que un escaneo de puertos lo detecta igualmente.

---

## 4. Fuerza bruta FTP con Hydra

### Objetivo
Obtener credenciales válidas para los usuarios `eddie` y `quinn` en el servicio FTP, y acceder a la flag almacenada en sus archivos.

### Comando utilizado
```bash
hydra -l eddie -P /usr/share/wordlists/rockyou.txt -s <PUERTO> <IP> ftp
hydra -l quinn -P /usr/share/wordlists/rockyou.txt -s <PUERTO> <IP> ftp
```

### Explicación de flags
| Flag | Descripción |
|------|-------------|
| `-l` | Usuario único (minúscula). Usar `-L` para lista de usuarios desde fichero |
| `-P` | Wordlist de contraseñas (mayúscula). Usar `-p` para contraseña única |
| `-s` | Puerto personalizado. Necesario cuando el servicio no corre en el puerto estándar |

### Concepto: Diferencia -l vs -L / -p vs -P en Hydra
```
-l usuario    →  un único usuario
-L lista.txt  →  lista de usuarios desde fichero

-p password   →  una única contraseña
-P rockyou    →  diccionario de contraseñas
```

### Wordlist: rockyou.txt
Una de las wordlists más utilizadas en CTFs y pentesting real. Contiene millones de contraseñas reales filtradas en brechas de datos. Se encuentra en `/usr/share/wordlists/rockyou.txt` en Kali Linux y distribuciones similares.

---

## 5. Escaneo sigiloso — IDS Evasion

### Objetivo
Escanear la máquina objetivo de la forma más sigilosa posible, evitando ser detectado por el IDS. El servidor web en el puerto 8080 muestra en tiempo real el porcentaje de probabilidad de detección.

### Comando que funcionó
```bash
nmap -sN <IP>
```

### Comandos probados y análisis
| Comando | Resultado | Motivo |
|--------|-----------|--------|
| `sudo nmap -n -sS -T0 <IP>` | Detectado parcialmente | SYN scan reconocible por IDS |
| `sudo nmap -sS -n -f <IP>` | Puertos filtrados | Fragmentación no suficiente |
| `nmap -sN <IP>` | **Éxito** | NULL scan no detectado por el IDS |

### Concepto: NULL Scan (-sN)
El NULL scan envía paquetes TCP con **cero flags activados**. Esto no corresponde a ningún comportamiento legítimo del protocolo TCP, pero tampoco coincide con los patrones de escaneo que los IDS tienen en sus reglas de detección.

**Comportamiento según RFC 793:**
- Puerto **abierto** → el sistema descarta el paquete silenciosamente (sin respuesta)
- Puerto **cerrado** → el sistema responde con **RST**

Nmap interpreta el silencio como `open|filtered` y el RST como `closed`.

> ⚠️ **Limitación importante:** El NULL scan funciona correctamente en sistemas **Linux/Unix** que siguen el RFC 793 de forma estricta. En sistemas **Windows**, ambos casos (abierto y cerrado) responden con RST, haciendo el NULL scan inútil.

### Comparativa de técnicas de evasión

| Técnica | Flag | Mecanismo | Detección |
|--------|------|-----------|-----------|
| SYN Scan | `-sS` | Half-open, no completa handshake | Media |
| NULL Scan | `-sN` | Sin flags TCP | Baja |
| FIN Scan | `-sF` | Solo flag FIN | Baja |
| Xmas Scan | `-sX` | Flags FIN+PSH+URG | Baja |
| Fragmentación | `-f` | Fragmenta paquetes IP | Variable |
| Timing lento | `-T0` | Reduce velocidad de envío | Baja |

---

## 6. Conceptos teóricos clave

### Three-way handshake TCP
```
Cliente  →  SYN        →  Servidor
Cliente  ←  SYN-ACK    ←  Servidor
Cliente  →  ACK        →  Servidor
```
El SYN scan (-sS) interrumpe este proceso tras el SYN-ACK, enviando un RST en lugar del ACK final. Evita completar la conexión pero sigue siendo detectable.

### Flags TCP
| Flag | Descripción |
|------|-------------|
| SYN | Iniciar conexión |
| ACK | Confirmación |
| FIN | Cerrar conexión |
| RST | Reset/abortar conexión |
| PSH | Push de datos |
| URG | Datos urgentes |

### IDS vs IPS
- **IDS (Intrusion Detection System):** Monitoriza y alerta sobre tráfico sospechoso. No bloquea.
- **IPS (Intrusion Prevention System):** Monitoriza y **bloquea** tráfico sospechoso activamente.

---

## Herramientas utilizadas
- **Nmap** — Escaneo de puertos, detección de versiones, evasión de IDS
- **Hydra** — Fuerza bruta de credenciales
- **Navegador web** — Acceso al challenge del puerto 8080

---

*Writeup elaborado como parte del estudio autodidacta de ciberseguridad — TryHackMe*
