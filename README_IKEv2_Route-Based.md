# VPN Site-to-Site IPSec IKEv2 — Basada en Enrutamiento (VTI)

---
Link de video: https://youtu.be/sBzwy2DMWMk  

## Objetivo

Implementar una VPN Site-to-Site usando **IPSec IKEv2** con **Virtual Tunnel Interface (VTI)**. La diferencia respecto a IKEv1 route-based es que el `crypto ipsec profile` ahora incluye `set ikev2-profile`, referenciando el profile IKEv2 que contiene el keyring y los parámetros de autenticación.

---

## Topología


<img width="785" height="644" alt="image" src="https://github.com/user-attachments/assets/7c4ae039-6959-40ea-8301-8581fa074666" />



## Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP     | Descripción          |
|-------------|----------|------------------|----------------------|
| R1          | e0/0     | 10.11.85.65/26   | WAN → ISP            |
| R1          | e0/1     | 10.11.85.1/26    | LAN → SW1            |
| R1          | Tunnel0  | 10.10.10.1/30    | VTI hacia R2         |
| ISP         | e0/0     | 10.11.85.66/26   | Hacia R1             |
| ISP         | e0/1     | 10.11.85.129/26  | Hacia R2             |
| R2          | e0/0     | 10.11.85.130/26  | WAN → ISP            |
| R2          | e0/1     | 10.11.85.193/26  | LAN → SW2            |
| R2          | Tunnel0  | 10.10.10.2/30    | VTI hacia R1         |
| PC1         | e0       | 10.11.85.2/26    | GW: 10.11.85.1       |
| PC2         | e0       | 10.11.85.194/26  | GW: 10.11.85.193     |

---

## Parámetros

### IKEv2

| Parámetro              | Valor             |
|------------------------|-------------------|
| Proposal               | PROP-IKEV2        |
| Cifrado                | AES-CBC-256       |
| Integridad             | SHA-256           |
| Grupo DH               | 14 (2048-bit)     |
| Keyring                | KR-IKEV2          |
| PSK                    | ITLA2024          |
| Profile                | PROF-IKEV2        |

### IPSec + VTI

| Parámetro              | Valor                  |
|------------------------|------------------------|
| Transform Set          | TS-IKEV2-VTI           |
| Modo                   | Tunnel                 |
| IPSec Profile          | PROF-VTI-IKEV2         |
| IKEv2 Profile en TS    | PROF-IKEV2 (set ikev2-profile) |
| Tunnel mode            | ipsec ipv4             |

---

## Configuración

### R1
```
hostname R1
!
interface Ethernet0/0
 ip address 10.11.85.65 255.255.255.192
 no shutdown
interface Ethernet0/1
 ip address 10.11.85.1 255.255.255.192
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.66
!
crypto ikev2 proposal PROP-IKEV2
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy POL-IKEV2
 proposal PROP-IKEV2
!
crypto ikev2 keyring KR-IKEV2
 peer R2
  address 10.11.85.130
  pre-shared-key ITLA2024
!
crypto ikev2 profile PROF-IKEV2
 match identity remote address 10.11.85.130 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEV2
!
crypto ipsec transform-set TS-IKEV2-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile PROF-VTI-IKEV2
 set transform-set TS-IKEV2-VTI
 set ikev2-profile PROF-IKEV2
!
interface Tunnel0
 ip address 10.10.10.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.130
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI-IKEV2
 no shutdown
!
ip route 10.11.85.192 255.255.255.192 Tunnel0
```

### R2
```
hostname R2
!
interface Ethernet0/0
 ip address 10.11.85.130 255.255.255.192
 no shutdown
interface Ethernet0/1
 ip address 10.11.85.193 255.255.255.192
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.129
!
crypto ikev2 proposal PROP-IKEV2
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy POL-IKEV2
 proposal PROP-IKEV2
!
crypto ikev2 keyring KR-IKEV2
 peer R1
  address 10.11.85.65
  pre-shared-key ITLA2024
!
crypto ikev2 profile PROF-IKEV2
 match identity remote address 10.11.85.65 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEV2
!
crypto ipsec transform-set TS-IKEV2-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile PROF-VTI-IKEV2
 set transform-set TS-IKEV2-VTI
 set ikev2-profile PROF-IKEV2
!
interface Tunnel0
 ip address 10.10.10.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.65
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI-IKEV2
 no shutdown
!
ip route 10.11.85.0 255.255.255.192 Tunnel0
```

---

## Verificación

```
show crypto ikev2 sa
show crypto ikev2 sa detailed
show crypto ipsec sa
show interface Tunnel0
show ip route
show crypto session
```

---

