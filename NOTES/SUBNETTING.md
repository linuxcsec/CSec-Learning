# Subnetting — Guía de Referencia
### Tabla de máscaras · CIDR · VLSM · IPv4

---

## Fórmulas esenciales

| Concepto | Fórmula | Nota |
|---|---|---|
| Hosts utilizables | `2ⁿ − 2` | n = bits de host. Se restan red y broadcast |
| Número de subredes | `2ˢ` | s = bits prestados a la parte de red |
| Tamaño de bloque | `256 − octeto de máscara` | Incremento entre subredes en el octeto variable |
| Broadcast | `siguiente red − 1` | Última IP del bloque. No asignable |
| Primera IP usable | `dirección de red + 1` | Inmediatamente después de la dirección de red |
| Última IP usable | `broadcast − 1` | Inmediatamente antes del broadcast |

---

## Tabla completa de subnetting IPv4

| Prefijo | Máscara | Hosts útiles | Bloque | Subredes (/24) | Uso típico |
|---|---|---|---|---|---|
| /8 | 255.0.0.0 | 16.777.214 | — | — | Clase A · privada `10.x.x.x` |
| /9 | 255.128.0.0 | 8.388.606 | 128 | — | A subdividida |
| /10 | 255.192.0.0 | 4.194.302 | 64 | — | A subdividida |
| /11 | 255.224.0.0 | 2.097.150 | 32 | — | A subdividida |
| /12 | 255.240.0.0 | 1.048.574 | 16 | — | A/B · privada `172.16–172.31` |
| /13 | 255.248.0.0 | 524.286 | 8 | — | B subdividida |
| /14 | 255.252.0.0 | 262.142 | 4 | — | B subdividida |
| /15 | 255.254.0.0 | 131.070 | 2 | — | B subdividida |
| /16 | 255.255.0.0 | 65.534 | — | — | Clase B · privada `172.16.x.x` |
| /17 | 255.255.128.0 | 32.766 | 128 | 2 | B subdividida |
| /18 | 255.255.192.0 | 16.382 | 64 | 4 | B subdividida |
| /19 | 255.255.224.0 | 8.190 | 32 | 8 | B subdividida |
| /20 | 255.255.240.0 | 4.094 | 16 | 16 | B subdividida |
| /21 | 255.255.248.0 | 2.046 | 8 | 32 | B subdividida |
| /22 | 255.255.252.0 | 1.022 | 4 | 64 | Empresas medianas |
| /23 | 255.255.254.0 | 510 | 2 | 128 | Empresas medianas |
| /24 | 255.255.255.0 | 254 | — | — | Clase C · privada `192.168.x.x` |
| /25 | 255.255.255.128 | 126 | 128 | 2 | LAN mediana |
| /26 | 255.255.255.192 | 62 | 64 | 4 | LAN pequeña |
| /27 | 255.255.255.224 | 30 | 32 | 8 | LAN pequeña |
| /28 | 255.255.255.240 | 14 | 16 | 16 | LAN muy pequeña |
| /29 | 255.255.255.248 | 6 | 8 | 32 | Segmento pequeño |
| /30 | 255.255.255.252 | 2 | 4 | 64 | **Enlace punto a punto** |
| /31 | 255.255.255.254 | 2* | 2 | 128 | P2P — RFC 3021 |
| /32 | 255.255.255.255 | 1 | 1 | 256 | Host único / loopback |

> \* `/31`: sin red ni broadcast por RFC 3021. Ambas IPs usables en enlaces P2P.

---

## Rangos privados — RFC 1918

| Rango | CIDR | Hosts totales | Clase |
|---|---|---|---|
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | 16.777.216 | A |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | 1.048.576 | B |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | 65.536 | C |

---

## CIDR — Classless Inter-Domain Routing

CIDR expresa la máscara como número de bits a 1, eliminando la dependencia de las clases A/B/C. Permite bloques de cualquier tamaño.

### Notación

```
192.168.10.0/24
└──────────┘ └┘
  dirección   prefijo = 24 bits de red, 8 bits de host
              → 2⁸ − 2 = 254 hosts
```

### Espectro de tamaños

| Prefijo | Hosts aprox. | Uso típico |
|---|---|---|
| /8 | 16,7M | ISP / clase A |
| /12 | 1M | 172.16–172.31 |
| /16 | 65.534 | Empresa / clase B |
| /20 | 4.094 | Campus / edificio |
| /24 | 254 | LAN típica / clase C |
| /26 | 62 | Departamento |
| /28 | 14 | Segmento pequeño |
| /30 | 2 | Enlace P2P |
| /32 | 1 | Host único / ruta estática |

### Supernetting — agregación de rutas

CIDR también permite agregar múltiples redes contiguas en un prefijo más corto para reducir tablas de enrutamiento.

**Ejemplo: agregar 4 redes /24 en un /22**

```
Redes:   192.168.0.0/24
         192.168.1.0/24
         192.168.2.0/24
         192.168.3.0/24

3er octeto en binario:
  0000 00|00  → .0
  0000 00|01  → .1
  0000 00|10  → .2
  0000 00|11  → .3
         ↑↑ bits variables

Bits comunes: 22 → Supernet: 192.168.0.0/22  (1.022 hosts)
```

> Regla: las redes a agregar deben ser contiguas y el bloque resultante debe estar alineado a su tamaño.

---

## VLSM — Variable Length Subnet Mask

VLSM permite dividir un bloque en subredes de **distinto tamaño**, asignando a cada segmento exactamente el espacio que necesita. Elimina el desperdicio del subnetting clásico de tamaño fijo.

### Reglas

| Regla | Descripción |
|---|---|
| **Orden** | Asignar siempre de mayor a menor requerimiento de hosts |
| **Alineación** | Cada subred debe estar alineada al tamaño de su bloque |
| **Sin solapamiento** | Los rangos no pueden superponerse |
| **Protocolo** | El protocolo de enrutamiento debe soportar CIDR (OSPF, EIGRP, BGP…) |

### Proceso

1. Ordenar segmentos por número de hosts requeridos (descendente)
2. Para cada segmento: elegir el prefijo mínimo que cubra los hosts (`2ⁿ − 2 ≥ hosts`)
3. Asignar desde el inicio del bloque disponible
4. La siguiente subred empieza en `broadcast + 1`

---

### Ejemplo completo — 192.168.1.0/24

**Requerimientos:** Ventas (60 hosts) · IT (28 hosts) · RRHH (12 hosts) · Dirección (4 hosts) · WAN (2 hosts)

| Subred | Hosts req. | Prefijo | Hosts disp. | Red | Primera IP | Última IP | Broadcast |
|---|---|---|---|---|---|---|---|
| Ventas | 60 | /26 | 62 | 192.168.1.0 | 192.168.1.1 | 192.168.1.62 | 192.168.1.63 |
| IT | 28 | /27 | 30 | 192.168.1.64 | 192.168.1.65 | 192.168.1.94 | 192.168.1.95 |
| RRHH | 12 | /28 | 14 | 192.168.1.96 | 192.168.1.97 | 192.168.1.110 | 192.168.1.111 |
| Dirección | 4 | /29 | 6 | 192.168.1.112 | 192.168.1.113 | 192.168.1.118 | 192.168.1.119 |
| WAN | 2 | /30 | 2 | 192.168.1.120 | 192.168.1.121 | 192.168.1.122 | 192.168.1.123 |
| **Libre** | — | — | 132 | 192.168.1.124 | — | — | 192.168.1.255 |

**Uso del espacio:** 124 IPs de 256 → ~48% de eficiencia vs. subnetting fijo que desperdiciaría ~60%.

---

## Cálculo paso a paso

### Método del bloque

El truco más rápido para calcular la subred de cualquier IP sin hacer binario completo:

```
1. Bloque = 256 − octeto_de_máscara
2. Divide la IP objetivo entre el bloque
3. Redondea hacia abajo al múltiplo → esa es la dirección de red
```

**Ejemplo: 192.168.1.77/26**

```
Máscara /26 → 255.255.255.192
Octeto variable: 192

Bloque = 256 − 192 = 64

77 / 64 = 1.2 → múltiplo = 1 × 64 = 64

Red:       192.168.1.64/26
Rango:     192.168.1.64 – 192.168.1.127
Broadcast: 192.168.1.127
```

---

### Ejemplo completo: 172.16.45.14/20

```
Dirección:   172.16.45.14
Prefijo:     /20 → máscara 255.255.240.0

Bits de host: 32 − 20 = 12 bits
Hosts útiles: 2¹² − 2 = 4.094 hosts

Bloque: 256 − 240 = 16  (octeto variable es el 3º)

Múltiplos de 16 en el 3er octeto: ... 32, 48 ...
45 está entre 32 y 48 → bloque es .32 – .47

Red:        172.16.32.0/20
Primera IP: 172.16.32.1
Última IP:  172.16.47.254
Broadcast:  172.16.47.255
```

---

### Ejemplo: dividir 10.0.0.0/8 en subredes /11

```
Bits prestados: 11 − 8 = 3 bits
Subredes:       2³ = 8 subredes
Hosts/subred:   2²¹ − 2 = 2.097.150 hosts
Bloque:         256 − 224 = 32  (2º octeto)

Subredes resultantes:
  10.0.0.0/11    →  10.0.0.0   – 10.31.255.255
  10.32.0.0/11   →  10.32.0.0  – 10.63.255.255
  10.64.0.0/11   →  10.64.0.0  – 10.95.255.255
  10.96.0.0/11   →  ...
  ...hasta 10.224.0.0/11
```

---

## Tabla de potencias de 2

| 2ⁿ | Valor | 2ⁿ | Valor |
|---|---|---|---|
| 2¹ | 2 | 2⁹ | 512 |
| 2² | 4 | 2¹⁰ | 1.024 |
| 2³ | 8 | 2¹² | 4.096 |
| 2⁴ | 16 | 2¹⁴ | 16.384 |
| 2⁵ | 32 | 2¹⁶ | 65.536 |
| 2⁶ | 64 | 2²⁰ | 1.048.576 |
| 2⁷ | 128 | 2²⁴ | 16.777.216 |
| 2⁸ | 256 | 2³² | 4.294.967.296 |

## Bloques de máscara — referencia rápida

| Octeto máscara | Prefijo | Bloque | Hosts útiles |
|---|---|---|---|
| 128 | /25 | 128 | 126 |
| 192 | /26 | 64 | 62 |
| 224 | /27 | 32 | 30 |
| 240 | /28 | 16 | 14 |
| 248 | /29 | 8 | 6 |
| 252 | /30 | 4 | 2 |
| 254 | /31 | 2 | 2* |
| 255 | /32 | 1 | 1 |

---

*Subnetting Reference · RFC 1918 · RFC 4632 (CIDR) · RFC 1009 (VLSM)*
