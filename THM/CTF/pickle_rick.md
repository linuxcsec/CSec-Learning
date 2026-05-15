# Pickle Rick — TryHackMe Writeup

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Web / Linux  
**Objetivo:** Encontrar tres ingredientes (flags) ocultos en la máquina

---

## Reconocimiento

El primer paso fue lanzar un escaneo con Nmap para identificar los servicios expuestos:

```bash
nmap -sV <IP>
```

El resultado reveló dos puertos abiertos:

- `22/tcp` — OpenSSH 8.2p1 (Ubuntu)
- `80/tcp` — Apache 2.4.41 (Ubuntu)

Con SSH necesitaríamos credenciales, así que el vector inicial de ataque apuntaba al servidor web.

---

## Enumeración Web

### Inspección manual

Lo primero fue navegar a la web y revisar el **código fuente** (`Ctrl+U`). En los comentarios HTML se encontró un **nombre de usuario**.

A continuación se visitó `/robots.txt`, donde aparecía una cadena de texto poco convencional que resultó ser la **contraseña**:

```
Wubbalubbadubdub
```

### Enumeración de directorios

Se lanzó Gobuster para descubrir rutas, incluyendo archivos `.php`:

```bash
gobuster dir -u http://<IP> -w <wordlist> -x php
```

Entre los resultados apareció un panel de login en una ruta `.php`. Usando las credenciales obtenidas (usuario del código fuente + contraseña del robots.txt) se accedió correctamente.

---

## Explotación — RCE

El panel autenticado contenía un campo para ejecutar comandos en el sistema, lo que supone una **Remote Code Execution (RCE)**.

Para verificar el contexto de ejecución:

```bash
whoami
# www-data
```

Se listó el contenido del directorio actual con `ls`, revelando varios archivos. El comando `cat` estaba deshabilitado, por lo que se utilizaron alternativas para leer archivos:

- Acceso directo vía URL (los archivos estaban en el webroot de Apache)
- `less <archivo>`
- `grep . <archivo>`

### Primera flag

En el directorio web se encontró un archivo `.txt` con el **primer ingrediente**:

```
FLAG{REDACTED}
```

---

## Post-explotación — Escalada de privilegios

### Segunda flag

Explorando `/home` se encontraron dos usuarios: `rick` y `ubuntu`. Dentro de `/home/rick` había un archivo llamado `second ingredients` (con espacio en el nombre).

Para leerlo sin `cat`, tratando el espacio correctamente:

```bash
less "/home/rick/second ingredients"
```

**Segundo ingrediente:**

```
FLAG{REDACTED}
```

### Tercera flag — Root

Se comprobaron los permisos sudo del usuario actual:

```bash
sudo -l
```

El output reveló que `www-data` podía ejecutar **cualquier comando como root sin contraseña**:

```
(ALL) NOPASSWD: ALL
```

Esto permitió acceder directamente al directorio `/root` y leer la flag final:

```bash
sudo less /root/3rd.txt
```

**Tercer ingrediente:**

```
FLAG{REDACTED}
```

---

## Resumen de vectores

| Fase | Técnica | Herramienta |
|---|---|---|
| Reconocimiento | Escaneo de puertos y versiones | Nmap |
| Enumeración web | Revisión de código fuente y robots.txt | Navegador |
| Enumeración de rutas | Fuerza bruta de directorios y archivos `.php` | Gobuster |
| Acceso inicial | Login con credenciales + RCE vía panel web | Navegador |
| Escalada de privilegios | `sudo` sin contraseña para todos los comandos | Bash |

---

## Lecciones aprendidas

- El código fuente HTML y `robots.txt` son los primeros sitios a revisar en cualquier reto web.
- Cuando `cat` no está disponible existen múltiples alternativas: `less`, `more`, `tac`, `strings`, `grep`.
- Los nombres de archivo con espacios requieren comillas o escape con `\`.
- `sudo -l` debe ser siempre uno de los primeros comandos tras obtener acceso a una shell.
