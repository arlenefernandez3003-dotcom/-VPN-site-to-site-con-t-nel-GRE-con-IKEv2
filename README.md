# 🔐 IPSec IKEv2 — VPN Site-to-Site con Túnel GRE

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv2-blue?style=for-the-badge&logo=cisco)
![Type](https://img.shields.io/badge/Tipo-Site--to--Site-brightgreen?style=for-the-badge)
![Mode](https://img.shields.io/badge/Modo-GRE%20over%20IPSec-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación del Túnel](#6-verificación-del-túnel)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Site-to-Site con túnel GRE protegido por IPSec IKEv2** en routers Cisco IOS dentro de PNetLab. La práctica cubre:

- Creación de un **túnel GRE** entre R1 y R2 como capa de encapsulamiento multiprotocolo sobre la red pública.
- Protección criptográfica del tráfico GRE mediante **IPSec IKEv2** usando Crypto Map, cifrando los paquetes GRE con la negociación moderna de 4 mensajes de IKEv2.
- Uso de una **ACL de tráfico interesante** que captura el protocolo GRE (protocolo IP 47) entre las IPs WAN de los peers para activar el cifrado IPSec.
- Validación del túnel con comandos `show crypto ikev2` y `show interface tunnel`, y pruebas de conectividad entre `202.50.73.128/25` (Site A) y `202.50.73.0/25` (Site B).

### ¿Qué aporta IKEv2 sobre GRE?

Respecto al Lab 03 (GRE over IKEv1), la arquitectura de encapsulamiento es idéntica. El cambio está en la negociación del canal seguro: IKEv2 usa solo 4 mensajes (vs 9 de IKEv1), tiene resistencia nativa a ataques de DoS, y su sintaxis en Cisco IOS es más estructurada y clara. GRE sigue aportando el soporte multiprotocolo y la posibilidad de correr protocolos de enrutamiento dinámico sobre el túnel.

```
[Paquete original PC1→PC2]
        ↓ GRE encapsula (tunnel mode gre ip)
[GRE Header | Paquete original]
        ↓ IKEv2 + IPSec cifra (ESP, Crypto Map en e0/0)
[IP WAN | ESP | GRE cifrado] → Cloud NAT → R2 descifra → desencapsula → PC2
```

---

## 2. Topología

```
                              [ INTERNET / ISP ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                                      │
                  ┌───────────────────┴───────────────────┐
                  │ e0/0: 192.168.19.5                     │ e0/0: 192.168.19.6
          ┌───────┴────────┐                      ┌────────┴───────┐
          │   R1 (Peer A)  │◄══ GRE over IPSec ═══►  R2 (Peer B)  │
          │                │     IKEv2 + Crypto    │               │
          └───────┬────────┘         Map           └────────┬──────┘
                  │ e0/1: 202.50.73.129/25                  │ e0/1: 202.50.73.1/25
                  │                                         │
          ┌───────┴────────┐                      ┌─────────┴──────┐
          │     SW1        │                      │      SW2       │
          └───────┬────────┘                      └────────┬───────┘
                  │                                        │
          ┌───────┴────────┐                      ┌────────┴───────┐
          │      PC1       │                      │      PC2       │
          │ 202.50.73.130  │                      │  202.50.73.2   │
          └────────────────┘                      └────────────────┘
           ◄── SITE A ──►                          ◄── SITE B ──►
           202.50.73.128/25                        202.50.73.0/25
```

> El túnel GRE se establece entre `192.168.19.5` (R1 e0/0) y `192.168.19.6` (R2 e0/0).  
> La red interna del túnel es `10.0.0.0/30`. El Crypto Map con IKEv2 se aplica en `e0/0` para cifrar todo el tráfico GRE.

**Flujo de encapsulamiento:**
1. PC1 envía tráfico a PC2 → R1 lo encamina por `Tunnel0` (GRE).
2. GRE encapsula el paquete: nuevo header IP src `192.168.19.5` → dst `192.168.19.6`.
3. La ACL detecta el paquete GRE (proto 47) → Crypto Map activa IPSec IKEv2.
4. IKEv2 negocia el canal en 4 mensajes → ESP cifra el paquete GRE completo.
5. El paquete cifrado viaja por Cloud NAT hasta R2 → descifra → desencapsula GRE → entrega a PC2.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo | Interfaz    | Dirección IP      | Máscara | Gateway        | Rol                         |
|-------------|-------------|-------------------|---------|----------------|-----------------------------|
| Cloud (NAT) | —           | 192.168.19.2      | /24     | —              | Gateway Cloud NAT (PNetLab) |
| **R1**      | **e0/0**    | **192.168.19.5**  | **/24** | 192.168.19.2   | WAN Peer A → Cloud          |
| **R1**      | **e0/1**    | **202.50.73.129** | **/25** | —              | Gateway LAN Site A          |
| **R1**      | **Tunnel0** | **10.0.0.1**      | **/30** | —              | Túnel GRE hacia R2          |
| **R2**      | **e0/0**    | **192.168.19.6**  | **/24** | 192.168.19.2   | WAN Peer B → Cloud          |
| **R2**      | **e0/1**    | **202.50.73.1**   | **/25** | —              | Gateway LAN Site B          |
| **R2**      | **Tunnel0** | **10.0.0.2**      | **/30** | —              | Túnel GRE hacia R1          |
| SW1         | —           | —                 | —       | —              | Capa 2 Site A               |
| SW2         | —           | —                 | —       | —              | Capa 2 Site B               |
| PC1         | eth0        | 202.50.73.130     | /25     | 202.50.73.129  | Host Site A                 |
| PC2         | eth0        | 202.50.73.2       | /25     | 202.50.73.1    | Host Site B                 |

### Tabla de Subredes

| Subred             | Rango Utilizable              | Broadcast       | Uso                    |
|--------------------|-------------------------------|-----------------|------------------------|
| `192.168.19.0/24`  | 192.168.19.1 – 192.168.19.254 | 192.168.19.255  | Segmento WAN/Cloud     |
| `202.50.73.0/25`   | 202.50.73.1 – 202.50.73.126   | 202.50.73.127   | LAN Site B             |
| `202.50.73.128/25` | 202.50.73.129 – 202.50.73.254 | 202.50.73.255   | LAN Site A             |
| `10.0.0.0/30`      | 10.0.0.1 – 10.0.0.2           | 10.0.0.3        | Interfaz GRE (Tunnel0) |

---

## 4. Parámetros Configurados

### IKEv2 Proposal

| Parámetro  | Valor               | Descripción                                             |
|------------|---------------------|---------------------------------------------------------|
| Nombre     | `IKEv2_PROP`        | Identificador del proposal                              |
| Cifrado    | AES-CBC-256         | Cifrado simétrico del canal IKEv2                       |
| Integridad | SHA-256             | Verificación de integridad de mensajes IKEv2            |
| Grupo DH   | Group 14 (2048-bit) | Intercambio Diffie-Hellman para derivar clave de sesión |

### IKEv2 Policy

| Parámetro | Valor        | Descripción                                  |
|-----------|--------------|----------------------------------------------|
| Nombre    | `IKEv2_POL`  | Política que asocia el proposal al peer       |
| Proposal  | `IKEv2_PROP` | Vincula la política al conjunto de algoritmos |

### IKEv2 Keyring

| Parámetro      | Valor            | Descripción                        |
|----------------|------------------|------------------------------------|
| Keyring        | `IKEv2_KEYRING`  | Almacén de claves pre-compartidas  |
| Peer R1→R2     | 192.168.19.6     | IP del peer remoto desde R1        |
| Peer R2→R1     | 192.168.19.5     | IP del peer remoto desde R2        |
| Pre-Shared Key | `ITLA2025Arlene` | Clave idéntica en ambos routers    |

### IKEv2 Profile

| Parámetro      | Valor            | Descripción                                    |
|----------------|------------------|------------------------------------------------|
| Nombre         | `IKEv2_PROFILE`  | Agrupa keyring, identidad y autenticación       |
| Match Identity | address (IP WAN) | Identifica al peer por su dirección IP pública |
| Authentication | pre-share        | Método de autenticación local y remota         |
| Keyring        | `IKEv2_KEYRING`  | Referencia al keyring con la PSK               |

### IPSec Transform Set y Crypto Map

| Parámetro      | Valor              | Descripción                                        |
|----------------|--------------------|----------------------------------------------------|
| Transform Set  | `TS_AES256_SHA256` | ESP-AES-256 + ESP-SHA256-HMAC, modo Tunnel         |
| Crypto Map     | `CMAP_GRE_IKEv2`   | Aplicado en `e0/0`, referencia el IKEv2 Profile    |
| Lifetime SA    | 3600 s (1 h)       | Duración del túnel de datos antes de renegociar    |

### Tráfico Interesante — ACL para cifrar GRE

| Router | ACL              | Fuente        | Destino       | Protocolo |
|--------|------------------|---------------|---------------|-----------|
| R1     | `ACL_GRE_IKEv2`  | 192.168.19.5  | 192.168.19.6  | GRE (47)  |
| R2     | `ACL_GRE_IKEv2`  | 192.168.19.6  | 192.168.19.5  | GRE (47)  |

> La ACL captura el tráfico **GRE entre IPs WAN** — no las LANs directamente. Todo lo que GRE encapsula queda protegido automáticamente por IPSec.

### Túnel GRE

| Parámetro          | R1            | R2            |
|--------------------|---------------|---------------|
| IP Tunnel          | 10.0.0.1/30   | 10.0.0.2/30   |
| Tunnel Source      | Ethernet0/0   | Ethernet0/0   |
| Tunnel Destination | 192.168.19.6  | 192.168.19.5  |
| Tunnel Mode        | gre ip        | gre ip        |

---

## 5. Scripts de Configuración

### R1 — Site A

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Site A | GRE over IPSec IKEv2
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.5 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 202.50.73.129 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: Túnel GRE ────────────────────────────────────────
interface Tunnel0
 description GRE-Tunnel-hacia-R2
 ip address 10.0.0.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.6
 tunnel mode gre ip

! ── Paso 2: Ruta hacia LAN remota vía Tunnel0 ────────────────
ip route 202.50.73.0 255.255.255.128 Tunnel0

! ── Paso 3: IKEv2 Proposal ──────────────────────────────────
crypto ikev2 proposal IKEv2_PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── Paso 4: IKEv2 Policy ────────────────────────────────────
crypto ikev2 policy IKEv2_POL
 proposal IKEv2_PROP

! ── Paso 5: IKEv2 Keyring (PSK) ─────────────────────────────
crypto ikev2 keyring IKEv2_KEYRING
 peer R2
  address 192.168.19.6
  pre-shared-key ITLA2025Arlene

! ── Paso 6: IKEv2 Profile ───────────────────────────────────
crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 192.168.19.6 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2_KEYRING

! ── Paso 7: Transform Set ───────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 8: ACL — cifrar tráfico GRE entre peers WAN ────────
ip access-list extended ACL_GRE_IKEv2
 permit gre host 192.168.19.5 host 192.168.19.6

! ── Paso 9: Crypto Map ──────────────────────────────────────
crypto map CMAP_GRE_IKEv2 10 ipsec-isakmp
 set peer 192.168.19.6
 set transform-set TS_AES256_SHA256
 set ikev2-profile IKEv2_PROFILE
 match address ACL_GRE_IKEv2
 set security-association lifetime seconds 3600

! ── Paso 10: Aplicar Crypto Map en interfaz WAN ─────────────
interface Ethernet0/0
 crypto map CMAP_GRE_IKEv2
```

### R2 — Site B

```cisco
! ══════════════════════════════════════════════════════════════
!  R2 — Site B | GRE over IPSec IKEv2
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R2

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.6 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 202.50.73.1 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: Túnel GRE ────────────────────────────────────────
interface Tunnel0
 description GRE-Tunnel-hacia-R1
 ip address 10.0.0.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.5
 tunnel mode gre ip

! ── Paso 2: Ruta hacia LAN remota vía Tunnel0 ────────────────
ip route 202.50.73.128 255.255.255.128 Tunnel0

! ── Paso 3: IKEv2 Proposal ──────────────────────────────────
crypto ikev2 proposal IKEv2_PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── Paso 4: IKEv2 Policy ────────────────────────────────────
crypto ikev2 policy IKEv2_POL
 proposal IKEv2_PROP

! ── Paso 5: IKEv2 Keyring (PSK) ─────────────────────────────
crypto ikev2 keyring IKEv2_KEYRING
 peer R1
  address 192.168.19.5
  pre-shared-key ITLA2025Arlene

! ── Paso 6: IKEv2 Profile ───────────────────────────────────
crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 192.168.19.5 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2_KEYRING

! ── Paso 7: Transform Set ───────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 8: ACL — cifrar tráfico GRE entre peers WAN ────────
ip access-list extended ACL_GRE_IKEv2
 permit gre host 192.168.19.6 host 192.168.19.5

! ── Paso 9: Crypto Map ──────────────────────────────────────
crypto map CMAP_GRE_IKEv2 10 ipsec-isakmp
 set peer 192.168.19.5
 set transform-set TS_AES256_SHA256
 set ikev2-profile IKEv2_PROFILE
 match address ACL_GRE_IKEv2
 set security-association lifetime seconds 3600

! ── Paso 10: Aplicar Crypto Map en interfaz WAN ─────────────
interface Ethernet0/0
 crypto map CMAP_GRE_IKEv2
```

### Hosts (VPCs en PNetLab)

```bash
# PC1 — Site A
ip 202.50.73.130 255.255.255.128 202.50.73.129

# PC2 — Site B
ip 202.50.73.2 255.255.255.128 202.50.73.1
```

---

## 6. Verificación del Túnel

### Estado de la interfaz GRE

```cisco
R1# show interface tunnel 0
```

Salida esperada:

```
Tunnel0 is up, line protocol is up
  Internet address is 10.0.0.1/30
  Tunnel source 192.168.19.5, destination 192.168.19.6
  Tunnel protocol/transport GRE/IP
```

> Si `Tunnel0` aparece `down`, verificar primero la conectividad WAN entre los peers (`ping 192.168.19.6`) antes de revisar IPSec.

---

### Estado de la sesión IKEv2

```cisco
R1# show crypto ikev2 sa
```

Salida esperada:

```
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         192.168.19.5/500      192.168.19.6/500      none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/65 sec
```

---

### Estado IPSec SA

```cisco
R1# show crypto ipsec sa
```

Salida esperada (fragmento):

```
interface: Ethernet0/0
    Crypto map tag: CMAP_GRE_IKEv2, local addr 192.168.19.5

   local  ident: (192.168.19.5/255.255.255.255/47/0)
   remote ident: (192.168.19.6/255.255.255.255/47/0)

    #pkts encaps: 25, #pkts encrypt: 25, #pkts digest: 25
    #pkts decaps: 25, #pkts decrypt: 25, #pkts verify: 25
```

> El protocolo `47` en los identifiers confirma que IPSec IKEv2 está cifrando tráfico GRE.

---

### Conectividad extremo a extremo

```bash
# Desde PC1 hacia PC2
PC1> ping 202.50.73.2

# Ping al extremo del túnel GRE desde R1
R1# ping 10.0.0.2 source 10.0.0.1
```

Resultado esperado:

```
!!!!!!!!!!
Success rate is 100 percent (10/10)
```

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show interface tunnel 0` | Estado del túnel GRE (debe ser `up/up`). |
| `show crypto ikev2 sa` | Estado de la sesión IKEv2. Debe mostrar `READY`. |
| `show crypto ikev2 sa detailed` | Algoritmos negociados, lifetime y autenticación del peer. |
| `show crypto ipsec sa` | SAs IPSec activas con protocolo 47 (GRE) en los identifiers. |
| `show ip access-lists ACL_GRE_IKEv2` | Hits en la ACL que captura tráfico GRE. |
| `show crypto map` | Confirma el Crypto Map y el IKEv2 Profile asociado en `e0/0`. |

---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles, todos los nodos encendidos. |
| 2 | [Config R1 – GRE + IKEv2](evidencias/2.png) | Consola R1: `Tunnel0` GRE configurado e IKEv2 Profile + Crypto Map aplicados en `e0/0`. |
| 3 | [IKEv2 SA READY + Tunnel up](evidencias/3.png) | Salida de `show crypto ikev2 sa` (`READY`) y `show interface tunnel 0` (`up/up`). |
| 4 | [Ping exitoso](evidencias/4.png) | Ping exitoso de PC1 (`202.50.73.130`) a PC2 (`202.50.73.2`) con GRE over IKEv2 activo. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**

**Duración estimada:** < 8 minutos

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 06: IPSec IKEv2 GRE over IPSec*

</div>
