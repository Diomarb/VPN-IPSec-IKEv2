# VPN Site-to-Site IPSec IKEv2 — Basada en Políticas

---

## Objetivo

Implementar una VPN Site-to-Site usando **IPSec con IKEv2** en modalidad **policy-based**. IKEv2 reemplaza el protocolo IKEv1 con un intercambio de mensajes más eficiente (4 mensajes vs 9), soporte nativo de movilidad, reauntenticación y detección de dead peers (DPD). El tráfico interesante sigue siendo definido por una ACL extendida.

---

## Topología

<img width="784" height="644" alt="image" src="https://github.com/user-attachments/assets/0074a7fa-9bf2-4771-a2fd-9590bc93b65b" />


## Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP     | Descripción          |
|-------------|----------|------------------|----------------------|
| R1          | e0/0     | 10.11.85.65/26   | WAN → ISP            |
| R1          | e0/1     | 10.11.85.1/26    | LAN → SW1            |
| ISP         | e0/0     | 10.11.85.66/26   | Hacia R1             |
| ISP         | e0/1     | 10.11.85.129/26  | Hacia R2             |
| R2          | e0/0     | 10.11.85.130/26  | WAN → ISP            |
| R2          | e0/1     | 10.11.85.193/26  | LAN → SW2            |
| PC1         | e0       | 10.11.85.2/26    | GW: 10.11.85.1       |
| PC2         | e0       | 10.11.85.194/26  | GW: 10.11.85.193     |

---

## Parámetros IKEv2

### IKEv2 Proposal / Policy

| Parámetro        | Valor             |
|------------------|-------------------|
| Proposal         | PROP-IKEV2        |
| Cifrado          | AES-CBC-256       |
| Integridad       | SHA-256           |
| Grupo DH         | 14 (2048-bit)     |
| Policy           | POL-IKEV2         |

### IKEv2 Keyring / Profile

| Parámetro              | Valor             |
|------------------------|-------------------|
| Keyring                | KR-IKEV2          |
| Clave PSK              | ITLA2024          |
| Profile                | PROF-IKEV2        |
| Auth local/remota      | Pre-share         |
| Match identity remota  | IP del peer       |

### IPSec Transform Set

| Parámetro    | Valor               |
|--------------|---------------------|
| Nombre       | TS-IKEV2            |
| Protocolo    | ESP                 |
| Cifrado      | AES-256             |
| Integridad   | HMAC-SHA-256        |
| Modo         | Tunnel              |
| Crypto Map   | CM-IKEV2            |

---

## IKEv1 vs IKEv2 — Diferencias en la configuración

| Elemento             | IKEv1                          | IKEv2                                   |
|----------------------|--------------------------------|-----------------------------------------|
| Parámetros IKE       | `crypto isakmp policy`         | `crypto ikev2 proposal` + `policy`      |
| Clave PSK            | `crypto isakmp key X addr Y`   | `crypto ikev2 keyring` → peer           |
| Profile              | No existe                      | `crypto ikev2 profile` (obligatorio)    |
| Vincular al crypto map | No necesario               | `set ikev2-profile` en crypto map       |
| Verificación SA      | `show crypto isakmp sa`        | `show crypto ikev2 sa`                  |
| Estado SA activa     | `QM_IDLE`                      | `READY`                                 |
| Mensajes negociación | 9 (Main + Quick Mode)          | 4 (IKE_SA_INIT + IKE_AUTH)             |

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
crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended ACL-VPN-R1
 permit ip 10.11.85.0 0.0.0.63 10.11.85.192 0.0.0.63
!
crypto map CM-IKEV2 10 ipsec-isakmp
 set peer 10.11.85.130
 set transform-set TS-IKEV2
 set ikev2-profile PROF-IKEV2
 match address ACL-VPN-R1
!
interface Ethernet0/0
 crypto map CM-IKEV2
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
crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended ACL-VPN-R2
 permit ip 10.11.85.192 0.0.0.63 10.11.85.0 0.0.0.63
!
crypto map CM-IKEV2 10 ipsec-isakmp
 set peer 10.11.85.65
 set transform-set TS-IKEV2
 set ikev2-profile PROF-IKEV2
 match address ACL-VPN-R2
!
interface Ethernet0/0
 crypto map CM-IKEV2
```
## ISP
```
enable
configure terminal
hostname ISP
 
interface Ethernet0/0
 ip address 10.11.85.66 255.255.255.192
 no shutdown
 
interface Ethernet0/1
 ip address 10.11.85.129 255.255.255.192
 no shutdown
 
end
write memory

---
```
## Verificación

```
! Estado SA IKEv2 (reemplaza show crypto isakmp sa)
show crypto ikev2 sa
! Esperado: estado READY

! Detalle completo de sesión IKEv2
show crypto ikev2 sa detailed

! SA IPSec + contadores
show crypto ipsec sa



```

---
