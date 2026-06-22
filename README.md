# VPN Site-to-Site IPSec IKEv2 — Túnel GRE (GRE over IPSec)


---
Link de Video: https://youtu.be/AWkiQB95K7o 
## Objetivo

Implementar **GRE over IPSec con IKEv2**. Combina la capacidad de GRE (soporte multicast, broadcast, enrutamiento dinámico) con la seguridad de IPSec IKEv2 (negociación más eficiente, DPD nativo, mejor soporte de movilidad). La diferencia respecto a IKEv1 GRE es únicamente la sección de configuración IKE: `crypto ikev2` en lugar de `crypto isakmp`, más `set ikev2-profile` en el crypto map.

---

## Topología

<img width="721" height="592" alt="image" src="https://github.com/user-attachments/assets/7581eb6d-fd19-481d-b505-d1e4e9cb24a4" />



## Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP     | Descripción          |
|-------------|----------|------------------|----------------------|
| R1          | e0/0     | 10.11.85.65/26   | WAN → ISP            |
| R1          | e0/1     | 10.11.85.1/26    | LAN → SW1            |
| R1          | Tunnel0  | 10.10.10.1/30    | GRE hacia R2         |
| ISP         | e0/0     | 10.11.85.66/26   | Hacia R1             |
| ISP         | e0/1     | 10.11.85.129/26  | Hacia R2             |
| R2          | e0/0     | 10.11.85.130/26  | WAN → ISP            |
| R2          | e0/1     | 10.11.85.193/26  | LAN → SW2            |
| R2          | Tunnel0  | 10.10.10.2/30    | GRE hacia R1         |
| PC1         | e0       | 10.11.85.2/26    | GW: 10.11.85.1       |
| PC2         | e0       | 10.11.85.194/26  | GW: 10.11.85.193     |

---

## Parámetros

| Parámetro              | Valor                          |
|------------------------|--------------------------------|
| IKEv2 Proposal         | PROP-IKEV2 (AES-256, SHA256, DH14) |
| IKEv2 Keyring          | KR-IKEV2 / PSK: ITLA2024       |
| IKEv2 Profile          | PROF-IKEV2                     |
| Transform Set          | TS-GRE-IKEV2                   |
| Modo IPSec             | **Transport** (GRE encapsula)  |
| ACL tráfico            | `permit gre host X host Y`     |
| Crypto Map             | CM-GRE-IKEV2 + set ikev2-profile |
| Tunnel mode            | gre ip                         |

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
interface Tunnel0
 ip address 10.10.10.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.130
 tunnel mode gre ip
 no shutdown
!
ip route 10.11.85.192 255.255.255.192 Tunnel0
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
crypto ipsec transform-set TS-GRE-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport
!
ip access-list extended ACL-GRE-VPN
 permit gre host 10.11.85.65 host 10.11.85.130
!
crypto map CM-GRE-IKEV2 10 ipsec-isakmp
 set peer 10.11.85.130
 set transform-set TS-GRE-IKEV2
 set ikev2-profile PROF-IKEV2
 match address ACL-GRE-VPN
!
interface Ethernet0/0
 crypto map CM-GRE-IKEV2
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
interface Tunnel0
 ip address 10.10.10.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.65
 tunnel mode gre ip
 no shutdown
!
ip route 10.11.85.0 255.255.255.192 Tunnel0
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
crypto ipsec transform-set TS-GRE-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport
!
ip access-list extended ACL-GRE-VPN
 permit gre host 10.11.85.130 host 10.11.85.65
!
crypto map CM-GRE-IKEV2 10 ipsec-isakmp
 set peer 10.11.85.65
 set transform-set TS-GRE-IKEV2
 set ikev2-profile PROF-IKEV2
 match address ACL-GRE-VPN
!
interface Ethernet0/0
 crypto map CM-GRE-IKEV2
```

---

## Verificación

```
show crypto ikev2 sa
show crypto ipsec sa
show interface Tunnel0
show ip access-lists ACL-GRE-VPN
```
