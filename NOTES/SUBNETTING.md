# Subnetting — Resumen

## Conceptos base

Una dirección IPv4 son **32 bits** divididos en 4 octetos (ej: `192.168.1.0`).

Toda dirección tiene dos partes:
- **Network portion** → identifica la red
- **Host portion** → identifica el dispositivo dentro de esa red

La **máscara de subred** define dónde termina una y empieza la otra.

---

## Notación CIDR

`/24` significa que los primeros 24 bits son red → quedan 8 bits para hosts.

| CIDR | Máscara | Hosts útiles |
|------|---------|-------------|
| /8   | 255.0.0.0 | 16.777.214 |
| /16  | 255.255.0.0 | 65.534 |
| /24  | 255.255.255.0 | 254 |
| /25  | 255.255.255.128 | 126 |
| /26  | 255.255.255.192 | 62 |
| /27  | 255.255.255.224 | 30 |
| /28  | 255.255.255.240 | 14 |
| /30  | 255.255.255.252 | 2 |

> **Fórmula:** `2ⁿ - 2` donde `n` = bits de host.  
> Se restan 2 por la dirección de red y broadcast.

---

## Las 3 direcciones clave de cualquier subred

Dada `192.168.1.0/24`:

| | Dirección |
|--|--|
| **Network address** | `192.168.1.0` → identifica la red, no asignable |
| **Broadcast** | `192.168.1.255` → llega a todos los hosts |
| **Rango útil** | `192.168.1.1` – `192.168.1.254` |

---

## Cómo calcular una subred manualmente

**Ejemplo:** `192.168.10.130/26`

1. `/26` → máscara `255.255.255.192` → bloque de **64** direcciones
2. Bloques: `0–63`, `64–127`, `128–191`, `192–255`
3. El host `.130` cae en el bloque `128–191`
4. Por tanto:
   - **Network:** `192.168.10.128`
   - **Broadcast:** `192.168.10.191`
   - **Rango útil:** `.129` – `.190`
   - **Hosts útiles:** 62

---

## Clases (contexto histórico)

| Clase | Rango primer octeto | Uso |
|-------|-------------------|-----|
| A | 1–126 | Redes enormes |
| B | 128–191 | Redes medianas |
| C | 192–223 | Redes pequeñas |

---

## IPs privadas (RFC 1918)

Estas no se enrutan en internet:

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

Son las que aparecen constantemente en entornos de laboratorio y CTFs.

---

## Relevancia en pentesting

- **Reconocimiento:** definir el rango de escaneo → `nmap 192.168.1.0/24`
- **Pivoting:** identificar subredes accesibles desde un host comprometido
- **Herramientas:** nmap, gobuster, metasploit aceptan notación CIDR directamente

---

## Referencia rápida — bits de host

| Bits de host | Hosts útiles | CIDR típico |
|---|---|---|
| 1 | 0 | /31 (punto a punto RFC 3021) |
| 2 | 2 | /30 |
| 3 | 6 | /29 |
| 4 | 14 | /28 |
| 5 | 30 | /27 |
| 6 | 62 | /26 |
| 7 | 126 | /25 |
| 8 | 254 | /24 |
