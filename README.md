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

- Creación de un **túnel GRE** entre R1 y R2 con `tunnel mode gre ip`, que permite transportar tráfico multiprotocolo y aplicar rutas estáticas sobre la interfaz lógica.
- Protección criptográfica del túnel GRE mediante **IPSec IKEv2** aplicado directamente con `tunnel protection ipsec profile` sobre la interfaz `Tunnel0`, usando el Transform Set en **modo transport** (ya que GRE se encarga de la encapsulación).
- Configuración del **IKEv2 Keyring** con `pre-shared-key local` y `pre-shared-key remote` por separado, requerido por IOSv para que la negociación funcione correctamente.
- Validación del túnel con comandos `show crypto ikev2` y `show interface tunnel`, y pruebas de conectividad entre `202.50.73.128/25` (Site A) y `202.50.73.0/25` (Site B).

### Diferencia clave respecto a otros labs GRE/IPSec

| Aspecto | GRE + Crypto Map (IKEv1) | GRE + tunnel protection (IKEv2) |
|---|---|---|
| Aplicación del cifrado | Crypto Map en `e0/0` | `tunnel protection ipsec profile` en `Tunnel0` |
| ACL de tráfico interesante | Requerida (captura proto 47) | No necesaria |
| Transform Set modo | Tunnel | **Transport** (GRE ya encapsula) |
| PSK en keyring | `crypto isakmp key` | `pre-shared-key local` + `pre-shared-key remote` |

```
[Paquete original PC1→PC2]
        ↓ GRE encapsula (tunnel mode gre ip)
[GRE Header | Paquete original]
        ↓ IPSec IKEv2 cifra en modo Transport (tunnel protection)
[IP WAN | ESP | GRE cifrado] → Cloud NAT → R2 → descifra → desencapsula → PC2
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
          │                │   IKEv2 + transport   │               │
          └───────┬────────┘   tunnel protection   └────────┬──────┘
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

> El túnel GRE se levanta entre `192.168.19.5` (R1) y `192.168.19.6` (R2), con `10.0.0.0/30` como red interna.  
> El cifrado IKEv2 se aplica directamente sobre `Tunnel0` via `tunnel protection` — sin Crypto Map ni ACL.

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
| Nombre     | `PROP_IKEv2_GRE`    | Identificador del proposal                              |
| Cifrado    | AES-CBC-256         | Cifrado simétrico del canal IKEv2                       |
| Integridad | SHA-256             | Verificación de integridad de mensajes IKEv2            |
| Grupo DH   | Group 14 (2048-bit) | Intercambio Diffie-Hellman para derivar clave de sesión |

> **Nota:** No se incluye `prf sha256` — no soportado en IOSv.

### IKEv2 Keyring

| Parámetro              | Valor            | Descripción                                                    |
|------------------------|------------------|----------------------------------------------------------------|
| Keyring                | `KR_GRE`         | Almacén de claves pre-compartidas                              |
| PSK local              | `ITLA2025Arlene` | Clave que este router envía al peer                            |
| PSK remote             | `ITLA2025Arlene` | Clave que este router espera recibir del peer                  |

> En IOSv es obligatorio definir `pre-shared-key local` y `pre-shared-key remote` por separado. Si solo se usa `pre-shared-key`, la negociación IKEv2 falla.

### IKEv2 Profile

| Parámetro      | Valor            | Descripción                                    |
|----------------|------------------|------------------------------------------------|
| Nombre         | `PROF_IKEv2_GRE` | Agrupa keyring, identidad y autenticación       |
| Match Identity | address (IP WAN) | Identifica al peer por su dirección IP pública |
| Authentication | pre-share        | Método de autenticación local y remota         |

### IPSec Transform Set y Profile

| Parámetro      | Valor                   | Descripción                                                             |
|----------------|-------------------------|-------------------------------------------------------------------------|
| Transform Set  | `TS_GRE_V2`             | ESP-AES-256 + ESP-SHA256-HMAC                                           |
| Modo           | **Transport**           | GRE ya encapsula — IPSec solo cifra el payload GRE, no re-encapsula    |
| IPSec Profile  | `GRE_IPSEC_PROFILE_V2`  | Aplicado en `Tunnel0` con `tunnel protection`                           |
| Lifetime SA    | 3600 s (1 h)            | Duración del túnel de datos antes de renegociar                         |

### Túnel GRE

| Parámetro          | R1                      | R2                      |
|--------------------|-------------------------|-------------------------|
| IP Tunnel          | 10.0.0.1/30             | 10.0.0.2/30             |
| Tunnel Source      | Ethernet0/0             | Ethernet0/0             |
| Tunnel Destination | 192.168.19.6            | 192.168.19.5            |
| Tunnel Mode        | gre ip                  | gre ip                  |
| IP MTU             | 1400                    | 1400                    |
| TCP MSS            | 1360                    | 1360                    |
| Protection         | GRE_IPSEC_PROFILE_V2    | GRE_IPSEC_PROFILE_V2    |

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

! ── Paso 1: IKEv2 Proposal ──────────────────────────────────
! Nota: no incluir "prf sha256" — no soportado en IOSv
crypto ikev2 proposal PROP_IKEv2_GRE
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── Paso 2: IKEv2 Policy ────────────────────────────────────
crypto ikev2 policy POL_IKEv2_GRE
 proposal PROP_IKEv2_GRE

! ── Paso 3: IKEv2 Keyring ───────────────────────────────────
! En IOSv es obligatorio definir local y remote por separado
crypto ikev2 keyring KR_GRE
 peer R2
  address 192.168.19.6
  pre-shared-key local  ITLA2025Arlene
  pre-shared-key remote ITLA2025Arlene

! ── Paso 4: IKEv2 Profile ───────────────────────────────────
crypto ikev2 profile PROF_IKEv2_GRE
 match identity remote address 192.168.19.6 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_GRE

! ── Paso 5: Transform Set en modo Transport ──────────────────
! GRE ya encapsula — IPSec usa transport, no tunnel
crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport

! ── Paso 6: IPSec Profile ───────────────────────────────────
crypto ipsec profile GRE_IPSEC_PROFILE_V2
 set transform-set TS_GRE_V2
 set ikev2-profile PROF_IKEv2_GRE
 set security-association lifetime seconds 3600

! ── Paso 7: Interfaz Tunnel GRE protegida con IPSec IKEv2 ───
interface Tunnel0
 description GRE-over-IPSec-IKEv2-hacia-R2
 ip address 10.0.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.6
 tunnel mode gre ip
 tunnel protection ipsec profile GRE_IPSEC_PROFILE_V2
 no shutdown

! ── Paso 8: Ruta estática hacia LAN remota vía Tunnel0 ───────
ip route 202.50.73.0 255.255.255.128 Tunnel0
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

! ── Paso 1: IKEv2 Proposal ──────────────────────────────────
crypto ikev2 proposal PROP_IKEv2_GRE
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── Paso 2: IKEv2 Policy ────────────────────────────────────
crypto ikev2 policy POL_IKEv2_GRE
 proposal PROP_IKEv2_GRE

! ── Paso 3: IKEv2 Keyring ───────────────────────────────────
crypto ikev2 keyring KR_GRE
 peer R1
  address 192.168.19.5
  pre-shared-key local  ITLA2025Arlene
  pre-shared-key remote ITLA2025Arlene

! ── Paso 4: IKEv2 Profile ───────────────────────────────────
crypto ikev2 profile PROF_IKEv2_GRE
 match identity remote address 192.168.19.5 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_GRE

! ── Paso 5: Transform Set en modo Transport ──────────────────
crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport

! ── Paso 6: IPSec Profile ───────────────────────────────────
crypto ipsec profile GRE_IPSEC_PROFILE_V2
 set transform-set TS_GRE_V2
 set ikev2-profile PROF_IKEv2_GRE
 set security-association lifetime seconds 3600

! ── Paso 7: Interfaz Tunnel GRE protegida con IPSec IKEv2 ───
interface Tunnel0
 description GRE-over-IPSec-IKEv2-hacia-R1
 ip address 10.0.0.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.5
 tunnel mode gre ip
 tunnel protection ipsec profile GRE_IPSEC_PROFILE_V2
 no shutdown

! ── Paso 8: Ruta estática hacia LAN remota vía Tunnel0 ───────
ip route 202.50.73.128 255.255.255.128 Tunnel0
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

> Si `Tunnel0` aparece `down/down`, verificar primero conectividad WAN con `ping 192.168.19.6` desde R1.

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
      Life/Active Time: 86400/72 sec
```

---

### Estado IPSec SA

```cisco
R1# show crypto ipsec sa
```

Salida esperada (fragmento):

```
interface: Tunnel0
   Crypto map tag: Tunnel0-head-0, local addr 192.168.19.5

   local  ident: (192.168.19.5/255.255.255.255/47/0)
   remote ident: (192.168.19.6/255.255.255.255/47/0)

    #pkts encaps: 20, #pkts encrypt: 20, #pkts digest: 20
    #pkts decaps: 20, #pkts decrypt: 20, #pkts verify: 20
```

> El protocolo `47` confirma que IPSec está cifrando tráfico GRE. Nótese que el Crypto Map se genera automáticamente asociado a `Tunnel0`.

---

### Conectividad extremo a extremo

```bash
# Desde PC1 hacia PC2
PC1> ping 202.50.73.2

# Ping al extremo del túnel GRE
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
| `show crypto ipsec sa` | SAs IPSec con protocolo 47 (GRE) en los identifiers. |
| `show ip route static` | Confirma rutas hacia LANs remotas apuntando a `Tunnel0`. |
| `show crypto session` | Resumen rápido del estado de la sesión IPSec/IKEv2. |

---

---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles, todos los nodos encendidos. |
| 2 | [Config R1 – GRE + IKEv2](evidencias/2.png) | Consola R1: `Tunnel0` GRE con `tunnel protection ipsec profile` y IKEv2 configurados. |
| 3 | [IKEv2 SA READY + Tunnel up](evidencias/3.png) | Salida de `show crypto ikev2 sa` (`READY`) y `show interface tunnel 0` (`up/up`). |
| 4 | [Ping exitoso](evidencias/4.png) | Ping exitoso de PC1 (`202.50.73.130`) a PC2 (`202.50.73.2`) con GRE over IKEv2 activo. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**


---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 06: IPSec IKEv2 GRE over IPSec*

</div>
