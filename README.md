# 🔐 VPN Client-to-Site — L2TP over IPSec IKEv1

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![L2TP](https://img.shields.io/badge/Protocolo-L2TP%20over%20IPSec-blue?style=for-the-badge&logo=cisco)
![IKEv1](https://img.shields.io/badge/IKE-IKEv1-orange?style=for-the-badge)
![Client](https://img.shields.io/badge/Cliente-Windows%207-lightgrey?style=for-the-badge&logo=windows)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Configuración del Cliente Windows 7](#6-configuración-del-cliente-windows-7)
7. [Verificación del Túnel](#7-verificación-del-túnel)
8. [Capturas de Pantalla](#8-capturas-de-pantalla)
9. [Video Demostrativo](#9-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Client-to-Site punto a multipunto con L2TP sobre IPSec IKEv1** en un router Cisco IOS dentro de PNetLab. La práctica cubre:

- Configuración del router **R1** como servidor L2TP/IPSec con un pool de direcciones IP para los clientes VPN.
- Establecimiento del canal seguro **IPSec IKEv1** con Dynamic Crypto Map, que cifra el tráfico L2TP (UDP 1701) entre el cliente y el servidor.
- Creación del servidor **VPDN (Virtual Private Dial-up Network)** con `Virtual-Template` para asignar IPs dinámicamente a los clientes.
- Autenticación de usuarios VPN con **AAA local** usando MS-CHAPv2/CHAP sobre PPP.
- Conexión desde un cliente **Windows 7** mediante el cliente VPN nativo del sistema operativo y verificación de conectividad hacia la LAN interna.

### ¿Cómo funciona L2TP over IPSec?

L2TP aporta el túnel de capa 2 (PPP) que permite asignar una IP al cliente como si estuviera en la LAN. IPSec aporta el cifrado y la autenticación del canal. Juntos forman el estándar de VPN cliente más compatible con sistemas Windows sin software adicional.

```
[Cliente Win7]
    ↓ Negocia IPSec IKEv1 con PSK (puerto UDP 500)
[Canal IPSec cifrado establecido]
    ↓ L2TP sobre IPSec (UDP 1701 cifrado)
[PPP negocia MS-CHAPv2 → asigna IP del pool]
    ↓ Cliente obtiene IP 202.50.73.194–.206
[Acceso a LAN interna 202.50.73.192/26]
```

---

## 2. Topología

```
                    [ ISP ]
                   /       \
             e0/0 /         \ e0/1
      202.50.73.1/26    192.168.19.2/24
               │                │
              e0                Gi1: 192.168.19.5
        ┌─────┴──────┐    ┌─────┴──────────────┐
        │  Cliente   │    │        R1           │
        │  Win7      │    │  (Servidor L2TP)    │
        │ (DHCP ISP) │    └─────────┬───────────┘
        └────────────┘           Gi2: 202.50.73.193/26
                                     │
                                  ┌──┴──┐
                                  │ SW  │ (e0/0 ↔ e0/2)
                                  └──┬──┘
                                   eth0
                                  ┌──┴──┐
                                  │ PC1 │
                                  │202.50.73.194... pool
                                  └─────┘

  Túnel L2TP over IPSec:
  Cliente (e0 → ISP e0/0) ══════IPSec══════ R1 Gi1
                            UDP 1701 cifrado
  IP asignada al cliente: 202.50.73.194–202.50.73.206 (pool VPN)
```

> El **ISP** actúa como red pública de tránsito. El cliente Windows 7 recibe conectividad hacia Internet a través del ISP y establece el túnel L2TP/IPSec hacia `192.168.19.5` (R1 Gi1).

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo   | Interfaz | Dirección IP      | Máscara | Gateway        | Rol                            |
|---------------|----------|-------------------|---------|----------------|--------------------------------|
| **R-ISP**     | e0/0     | 202.50.73.1       | /26     | —              | Hacia cliente Windows 7        |
| **R-ISP**     | e0/1     | 192.168.19.2      | /24     | —              | Hacia R1 (gateway WAN)         |
| **R1**        | **Gi1**  | **192.168.19.5**  | **/24** | 192.168.19.2   | WAN → ISP (crypto map)         |
| **R1**        | **Gi2**  | **202.50.73.193** | **/26** | —              | LAN interna → SW               |
| SW            | e0/0     | —                 | —       | —              | Uplink → R1 Gi2                |
| SW            | e0/2     | —                 | —       | —              | Downlink → PC1                 |
| PC1           | eth0     | 202.50.73.194     | /26     | 202.50.73.193  | Host LAN interna               |
| **Cliente**   | e0       | (vía ISP/DHCP)    | /26     | 202.50.73.1    | Cliente VPN Windows 7          |
| **VPN Pool**  | Virtual  | 202.50.73.194–.206| /26     | —              | IPs asignadas a clientes VPN   |

### Tabla de Subredes

| Subred             | Rango Utilizable              | Broadcast       | Uso                        |
|--------------------|-------------------------------|-----------------|----------------------------|
| `192.168.19.0/24`  | 192.168.19.1 – 192.168.19.254 | 192.168.19.255  | Segmento WAN R1 ↔ ISP      |
| `202.50.73.0/26`   | 202.50.73.1 – 202.50.73.62    | 202.50.73.63    | Red ISP → Cliente          |
| `202.50.73.192/26` | 202.50.73.193 – 202.50.73.254 | 202.50.73.255   | LAN interna + Pool VPN     |

> El Pool VPN (`202.50.73.194–202.50.73.206`) está dentro de la misma subred `/26` de la LAN interna, por lo que los clientes VPN quedan lógicamente en la misma red que PC1.

### Credenciales VPN

| Parámetro       | Valor            |
|-----------------|------------------|
| Usuario         | `cliente1`       |
| Contraseña      | `cisco123`       |
| Pre-Shared Key  | `ITLA2025Arlene` |
| Servidor VPN    | `192.168.19.5`   |

---

## 4. Parámetros Configurados

### IPSec IKEv1 — ISAKMP Policy

| Parámetro          | Valor           | Descripción                                             |
|--------------------|-----------------|----------------------------------------------------------|
| Número de política | `10`            | Prioridad de la política ISAKMP                          |
| Cifrado            | 3DES            | Cifrado del canal IKE                                    |
| Hash / Integridad  | SHA             | Verificación de integridad                               |
| Autenticación      | Pre-Shared Key  | Clave compartida con el cliente                          |
| Grupo DH           | Group 2         | Intercambio Diffie-Hellman 1024-bit                      |
| Lifetime SA        | 86400 s (24 h)  | Duración del canal IKE                                   |
| Pre-Shared Key     | `ITLA2025Arlene`| Wildcard `0.0.0.0 0.0.0.0` — acepta cualquier cliente   |
| NAT Keepalive      | 20 s            | Mantiene el canal activo a través de NAT                 |

### IPSec — Transform Set y Dynamic Crypto Map

| Parámetro       | Valor          | Descripción                                                  |
|-----------------|----------------|--------------------------------------------------------------|
| Transform Set   | `TS-L2TP`      | ESP-3DES + ESP-SHA-HMAC                                      |
| Modo            | Transport      | L2TP ya encapsula — IPSec cifra solo el payload UDP 1701     |
| ACL             | `permit udp any any eq 1701` | Captura tráfico L2TP (UDP 1701)              |
| Dynamic Map     | `DMAP seq 10`  | Permite conexiones de clientes con IP dinámica               |
| Crypto Map      | `CMAP`         | Referencia al dynamic map — aplicado en Gi1 (WAN)            |

### L2TP / VPDN

| Parámetro            | Valor               | Descripción                                       |
|----------------------|---------------------|---------------------------------------------------|
| VPDN Group           | `L2TP-CLIENTES`     | Grupo que acepta conexiones L2TP entrantes         |
| Protocol             | l2tp                | Protocolo de tunneling de capa 2                  |
| Virtual Template     | `1`                 | Plantilla de interfaz PPP para clientes           |
| L2TP Auth            | Deshabilitada       | `no l2tp tunnel authentication` — compatibilidad  |
| PPP Authentication   | MS-CHAPv2 + CHAP    | Autenticación del usuario VPN                     |
| IP Pool              | `POOL-VPN`          | 202.50.73.194 – 202.50.73.206                     |
| IP Unnumbered        | GigabitEthernet2    | El VT comparte la IP de la interfaz LAN           |

### AAA

| Parámetro    | Valor   | Descripción                               |
|--------------|---------|-------------------------------------------|
| Modelo       | local   | Base de datos local del router            |
| Usuario      | `cliente1` | Nombre de usuario VPN                  |
| Contraseña   | `cisco123` | Contraseña del usuario VPN             |

---

## 5. Scripts de Configuración

### R1 — Servidor L2TP/IPSec

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Servidor L2TP/IPSec IKEv1 Client-to-Site
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1

! ── Interfaces ──────────────────────────────────────────────
interface GigabitEthernet1
 description WAN-hacia-ISP
 ip address 192.168.19.5 255.255.255.0
 crypto map CMAP
 no shutdown

interface GigabitEthernet2
 description LAN-Servidor
 ip address 202.50.73.193 255.255.255.192
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── AAA — Autenticación local PPP ───────────────────────────
aaa new-model
aaa authentication ppp default local
username cliente1 password cisco123

! ── IPSec IKEv1 — Fase 1 ────────────────────────────────────
crypto isakmp policy 10
 encryption 3des
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

! Wildcard: acepta PSK de cualquier cliente
crypto isakmp key ITLA2025Arlene address 0.0.0.0 0.0.0.0
crypto isakmp nat keepalive 20

! ── IPSec — Transform Set (modo transport para L2TP) ────────
crypto ipsec transform-set TS-L2TP esp-3des esp-sha-hmac
 mode transport

! ── ACL — captura tráfico L2TP (UDP 1701) ───────────────────
access-list 100 permit udp any any eq 1701

! ── Dynamic Crypto Map + Crypto Map ─────────────────────────
crypto dynamic-map DMAP 10
 set transform-set TS-L2TP
 match address 100

crypto map CMAP 10 ipsec-isakmp dynamic DMAP

! ── Servidor L2TP (VPDN) ────────────────────────────────────
vpdn enable

vpdn-group L2TP-CLIENTES
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

! ── Pool de IPs para clientes VPN ───────────────────────────
ip local pool POOL-VPN 202.50.73.194 202.50.73.206

! ── Virtual Template (interfaz PPP para cada cliente) ────────
interface Virtual-Template1
 ip unnumbered GigabitEthernet2
 peer default ip address pool POOL-VPN
 ppp authentication ms-chap-v2 chap
```

### R-ISP

```cisco
! ══════════════════════════════════════════════════════════════
!  R-ISP — Router de tránsito público
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R-ISP

interface Ethernet0/0
 description Hacia-Cliente-Windows7
 ip address 202.50.73.1 255.255.255.192
 no shutdown

interface Ethernet0/1
 description Hacia-R1-L2TP
 ip address 192.168.19.2 255.255.255.0
 no shutdown
```

### PC1 — Host LAN interna

```bash
ip 202.50.73.194 255.255.255.192 202.50.73.193
```

---

## 6. Configuración del Cliente Windows 7

El cliente Windows 7 usa el cliente VPN nativo del sistema operativo para conectarse al servidor L2TP/IPSec.

### Paso 1 — Agregar la clave pre-compartida en el registro

Antes de crear la conexión VPN, es necesario habilitar L2TP con PSK en Windows 7 editando el registro:

```
Ruta: HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\RasMan\Parameters
Valor DWORD: ProhibitIpSec = 0
```

O alternativamente, agregar la clave PSK vía regedit:

```
Ruta: HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\RasMan\Parameters
Nuevo valor String: AllowL2TPWeakCrypto = 1
```

> En Windows 7 también se puede configurar la PSK directamente en las propiedades de la conexión VPN (ver Paso 3).

---

### Paso 2 — Crear la conexión VPN

1. Ir a **Panel de Control → Centro de redes y recursos compartidos**.
2. Clic en **"Configurar una nueva conexión o red"**.
3. Seleccionar **"Conectarse a un área de trabajo"** → Siguiente.
4. Seleccionar **"Usar mi conexión a Internet (VPN)"**.
5. Ingresar los datos:

| Campo | Valor |
|---|---|
| Dirección de Internet | `192.168.19.5` |
| Nombre de destino | `VPN-ITLA-L2TP` |
| ☑ No conectar ahora | Marcar para configurar primero |

6. Clic en **Siguiente**.
7. Ingresar credenciales:

| Campo | Valor |
|---|---|
| Nombre de usuario | `cliente1` |
| Contraseña | `cisco123` |

8. Clic en **Crear**.

---

### Paso 3 — Configurar tipo L2TP y PSK

1. En **Centro de redes** → clic derecho en `VPN-ITLA-L2TP` → **Propiedades**.
2. Pestaña **Seguridad**:
   - Tipo de VPN: **Protocolo de túnel de capa 2 (L2TP/IPSec)**
   - Clic en **"Configuración avanzada"**
   - Seleccionar **"Usar clave precompartida para autenticación"**
   - Ingresar clave: `ITLA2025Arlene`
3. Cifrado de datos: **Cifrado máximo de nivel**
4. Permitir protocolos: marcar **MS-CHAP v2** y **CHAP**
5. Clic en **Aceptar**.

---

### Paso 4 — Conectar

1. Clic en el ícono de red en la barra de tareas.
2. Seleccionar `VPN-ITLA-L2TP` → clic en **Conectar**.
3. Verificar que aparece **Conectado** con una IP del pool `202.50.73.194–.206`.

---

## 7. Verificación del Túnel

### Estado de sesiones L2TP activas en R1

```cisco
R1# show vpdn session
```

Salida esperada:

```
%PPP L2TP Active Tunnels/Sessions 1/1
LocID RemID Remote Name   State  Last Chg Uniq ID
 1     1    192.168.19.x  est    00:00:45  1
```

---

### Verificar interfaz Virtual-Access asignada al cliente

```cisco
R1# show interface virtual-access 1
```

Salida esperada:

```
Virtual-Access1 is up, line protocol is up
  Internet address is 202.50.73.193/26 (unnumbered from GigabitEthernet2)
  Peer IP address: 202.50.73.194
```

---

### Estado IPSec IKEv1

```cisco
R1# show crypto isakmp sa
```

Salida esperada:

```
dst             src             state          conn-id status
192.168.19.5    <IP-Cliente>    QM_IDLE        1001    ACTIVE
```

---

### Estado IPSec SA con contadores

```cisco
R1# show crypto ipsec sa
```

Confirmar que los contadores `#pkts encaps` y `#pkts decaps` incrementan con tráfico del cliente.

---

### Conectividad desde el cliente hacia PC1

Desde el cliente Windows 7 (con VPN conectada), abrir **CMD** y ejecutar:

```cmd
tracert 202.50.73.194
```

Resultado esperado:

```
Trazando ruta a 202.50.73.194 con un máximo de 30 saltos

  1    <1 ms    <1 ms    <1 ms   202.50.73.193   [R1 Virtual-Template]
  2    <1 ms    <1 ms    <1 ms   202.50.73.194   [PC1 LAN interna]

Traza completa.
```

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show vpdn session` | Sesiones L2TP activas con estado y tunnel ID. |
| `show vpdn tunnel` | Túneles L2TP establecidos con el cliente. |
| `show interface virtual-access 1` | Interfaz PPP asignada al cliente con IP del pool. |
| `show crypto isakmp sa` | SA IKEv1 activa con el cliente. Debe mostrar `QM_IDLE`. |
| `show crypto ipsec sa` | Contadores ESP — confirma que el L2TP viaja cifrado. |
| `show ip local pool POOL-VPN` | IPs del pool disponibles y asignadas. |

---
---

## 8. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.jpeg) | Topología en PNetLab con nombre y matrícula visibles: Cliente Win7, ISP, R1, SW y PC1 encendidos. |
| 2 | [Config VPN Windows 7](evidencias/2.jpeg) | Propiedades de la conexión VPN en Win7 mostrando tipo L2TP/IPSec y clave precompartida configurada. |
| 3 | [Sesión L2TP activa en R1](evidencias/3.jpeg) | Salida de `show vpdn session` y `show crypto isakmp sa` con sesión activa y estado `QM_IDLE`. |
| 4 | [Traceroute exitoso](evidencias/4.jpeg) | Resultado de `tracert 202.50.73.194` desde el cliente Windows 7 con VPN conectada, mostrando ruta hacia PC1. |

---

## 9. Video Demostrativo

🎥  **[Ver en YouTube — enlace pendiente](https://youtu.be/X_LxLzhN3gk)**
---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 09: L2TP over IPSec IKEv1 Client-to-Site*

</div>
