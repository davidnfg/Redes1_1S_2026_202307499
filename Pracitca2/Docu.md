# Documentación Técnica — Red LAN Aeropuerto Internacional La Aurora
**Nombre:** David Norberto Fabro Guzmán  
**Carnet:** 202307499  


---

## Tabla de Contenidos

1. [Descripción del Proyecto](#1-descripción-del-proyecto)
2. [Áreas Operativas Seleccionadas](#2-áreas-operativas-seleccionadas)
3. [Topología de Red](#3-topología-de-red)
4. [Segmentación por VLANs](#4-segmentación-por-vlans)
5. [Tabla de Subnetting](#5-tabla-de-subnetting)
6. [Tabla de Asignación de Direcciones IP](#6-tabla-de-asignación-de-direcciones-ip)
7. [Configuración de Switches](#7-configuración-de-switches)
8. [Capturas de Pantalla](#8-capturas-de-pantalla)

---

## 1. Descripción del Proyecto

El presente documento describe el diseño e implementación de una red LAN segmentada para el Aeropuerto Internacional La Aurora, Guatemala. La red fue diseñada con los siguientes criterios:

- Segmentación por VLANs para separar el tráfico de cada área operativa.
- Alta disponibilidad mediante EtherChannel LACP en las áreas críticas.
- Subnetting eficiente usando VLSM (Variable Length Subnet Masking).
- Protocolo Rapid PVST+ para prevención de bucles.
- VTP para administración centralizada de VLANs por área.
- VLAN Nativa 99 para tráfico de gestión entre switches.
- VLAN Blackhole 999 para puertos sin uso.

---

## 2. Áreas Operativas Seleccionadas

| # | Área | Descripción |
|---|------|-------------|
| 1 | Terminal de Pasajeros | Área de check-in, salas de espera y puertas de abordaje. Incluye muelles Norte y Central. |
| 2 | Pista de Aterrizaje y Despegue (Pista 02/20) | Pista de asfalto de 2,987 metros con zonas de seguridad RESA. |
| 3 | Torre de Control | Servicios de navegación aérea. Gestión de despegues, aterrizajes y movimiento de aeronaves. |
| 4 | Plataforma de Estacionamiento (Rampa) | Área de parqueo de aeronaves, carga de combustible y servicios de catering. |
| 5 | Terminal de Carga (Combex-Im) | Gestión de importaciones y exportaciones, depósitos aduaneros y cámaras frías. |
| 6 | Área de Mantenimiento y Hangares | Talleres certificados para inspección y reparación de aeronaves. |
| 7 | Estación de Bomberos (SSEI) | Servicio de Salvamento y Extinción de Incendios, operativo 24/7. |
| 8 | Área de Migración y Aduanas (SAT) | Control migratorio y fiscalización de mercancías en llegadas internacionales. |

---

## 3. Topología de Red

### 3.1 Tipo de Topología

Se implementó una **topología jerárquica en estrella de tres capas**:

- **Capa Core:** SW-Core (2960-24TT) — punto central de la red, conecta todos los switches principales.
- **Capa de Distribución:** Un switch principal (2960-24TT) por cada área operativa — actúa como servidor VTP y Root Bridge de su VLAN.
- **Capa de Acceso:** Switches de acceso (2960-24TT) en áreas con mayor cantidad de hosts — actúan como clientes VTP y conectan los dispositivos finales.

### 3.2 Justificación de la Topología

La topología jerárquica en estrella fue seleccionada por las siguientes razones:

- **Orden y administración:** Cada área opera de forma independiente con su propio switch principal, facilitando la administración y el aislamiento de fallos.
- **Escalabilidad:** Agregar nuevas áreas o dispositivos no afecta el resto de la red.
- **Redundancia:** Las áreas críticas cuentan con EtherChannel (3 enlaces físicos agrupados) hacia el core, garantizando que un fallo en un enlace no interrumpa el servicio.
- **Rendimiento:** El tráfico de cada área se mantiene local dentro de su VLAN, reduciendo la carga en el core.

### 3.3 Tipo de Cableado

- **Cobre cruzado (Crossover):** Todos los enlaces switch ↔ switch.
- **Cobre directo (Straight-through):** Todos los enlaces switch ↔ dispositivo final (PCs, servidores, impresoras, etc.).

### 3.4 Alta Disponibilidad — Áreas Críticas

Las siguientes áreas fueron identificadas como críticas y cuentan con EtherChannel LACP (3 enlaces físicos) hacia el SW-Core:

| Área | Justificación |
|------|---------------|
| Área 1 — Terminal de Pasajeros | Mayor volumen de tráfico (74 hosts). Operación continua de check-in y abordaje. |
| Área 3 — Torre de Control | Tolerancia cero a fallos. Las operaciones ATC son críticas para la seguridad aérea. |
| Área 5 — Terminal de Carga | Operaciones aduaneras y comerciales que requieren disponibilidad constante. |

Los switches de Mantenimiento (Área 6) y Bomberos (Área 7) utilizan enlace troncal simple debido al límite hardware del modelo 2960-24TT, que soporta un máximo de 6 EtherChannels simultáneos.

### 3.5 Dominios de Colisión

Cada puerto de acceso activo en cada switch representa un dominio de colisión independiente. Los switches delimitan los dominios de colisión por puerto. El total de dominios de colisión es igual al total de puertos de acceso activos en toda la red. Las VLANs funcionan como dominios de broadcast separados, siendo el SW-Core quien separa el tráfico entre áreas.

### 3.6 Carga de Tráfico

Las áreas con mayor número de hosts (Terminal de Pasajeros con 74h, Mantenimiento con 47h) cuentan con switches de acceso adicionales para distribuir la carga. Los EtherChannel en las áreas críticas proporcionan hasta 300 Mbps de ancho de banda agregado (3 × 100 Mbps FastEthernet), evitando cuellos de botella.

---

## 4. Segmentación por VLANs

| Área | VLAN ID | Nombre | Tipo |
|------|---------|--------|------|
| Terminal de Pasajeros | 19 | Terminal-Pasajeros | Datos |
| Pista 02/20 | 29 | Pista | Datos |
| Torre de Control | 39 | Torre-Control | Datos |
| Plataforma Rampa | 49 | Plataforma-Rampa | Datos |
| Terminal de Carga | 59 | Terminal-Carga | Datos |
| Mantenimiento | 69 | Mantenimiento | Datos |
| Bomberos SSEI | 79 | Bomberos | Datos |
| Migración y Aduana | 89 | Migracion-Aduana | Datos |
| Nativa | 99 | Nativa | Gestión (trunk) |
| Blackhole | 999 | Blackhole | Seguridad (puertos sin uso) |

### 4.1 Configuración VTP

| Switch | Modo VTP | Dominio | Contraseña |
|--------|----------|---------|------------|
| SW-Core | Transparent | 202307499 | — |
| SW-Tpasarejos | Server | 202307499 | area1 |
| SW-Pista | Transparent | 202307499 | — |
| SW-TorreControl | Server | 202307499 | area3 |
| SW-Estacionamiento | Transparent | 202307499 | — |
| SW-Tcarga | Server | 202307499 | area5 |
| SW-Mantenimiento | Transparent | 202307499 | — |
| SW-Bomberos | Transparent | 202307499 | — |
| SW-Migracion | Transparent | 202307499 | — |
| SW-A1-1 | Client | 202307499 | area1 |
| SW-A1-2 | Client | 202307499 | area1 |
| SW-A3-1 | Transparent | 202307499 | — |
| SW-AS-1 | Client | 202307499 | area5 |

> **Nota:** Los switches configurados como Transparent crean sus VLANs localmente debido a limitaciones de sincronización VTP en Packet Tracer con el modelo 2960-24TT.

---

## 5. Tabla de Subnetting

**Red base:** 192.168.0.0/16  
**Técnica:** VLSM (Variable Length Subnet Masking) — asignación de mayor a menor cantidad de hosts para maximizar eficiencia.

| Orden | Área | VLAN | Hosts Req. | Subred | Prefijo | Máscara | Gateway | Rango Usable | Broadcast | Cap. Total |
|-------|------|------|-----------|--------|---------|---------|---------|--------------|-----------|-----------|
| 1 | Terminal Pasajeros | 19 | 74 | 192.168.0.0 | /25 | 255.255.255.128 | 192.168.0.1 | 192.168.0.2 – 192.168.0.126 | 192.168.0.127 | 126 |
| 2 | Mantenimiento | 69 | 47 | 192.168.0.128 | /26 | 255.255.255.192 | 192.168.0.129 | 192.168.0.130 – 192.168.0.190 | 192.168.0.191 | 62 |
| 3 | Torre de Control | 39 | 32 | 192.168.0.192 | /26 | 255.255.255.192 | 192.168.0.193 | 192.168.0.194 – 192.168.0.254 | 192.168.0.255 | 62 |
| 4 | Terminal de Carga | 59 | 26 | 192.168.1.0 | /27 | 255.255.255.224 | 192.168.1.1 | 192.168.1.2 – 192.168.1.30 | 192.168.1.31 | 30 |
| 5 | Pista 02/20 | 29 | 25 | 192.168.1.32 | /27 | 255.255.255.224 | 192.168.1.33 | 192.168.1.34 – 192.168.1.62 | 192.168.1.63 | 30 |
| 6 | Migración y Aduana | 89 | 20 | 192.168.1.64 | /27 | 255.255.255.224 | 192.168.1.65 | 192.168.1.66 – 192.168.1.94 | 192.168.1.95 | 30 |
| 7 | Bomberos SSEI | 79 | 8 | 192.168.1.96 | /28 | 255.255.255.240 | 192.168.1.97 | 192.168.1.98 – 192.168.1.110 | 192.168.1.111 | 14 |
| 8 | Plataforma Rampa | 49 | 5 | 192.168.1.112 | /28 | 255.255.255.240 | 192.168.1.113 | 192.168.1.114 – 192.168.1.126 | 192.168.1.127 | 14 |

### 5.1 Justificación del Subnetting

Se utilizó **VLSM** ordenando las subredes de mayor a menor cantidad de hosts requeridos. Esto garantiza:

- Subredes contiguas sin desperdicio de espacio de direccionamiento.
- Cada subred tiene exactamente la potencia de 2 mínima que supera los hosts requeridos (2ⁿ − 2 ≥ hosts).
- Uso eficiente del espacio de direcciones dentro del bloque 192.168.0.0/16.

---

## 6. Tabla de Asignación de Direcciones IP

### Área 1 — Terminal de Pasajeros (VLAN 19 | 192.168.0.0/25)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| Kiosco1 | 192.168.0.2 | 255.255.255.128 | 192.168.0.1 |
| Kiosco2 | 192.168.0.3 | 255.255.255.128 | 192.168.0.1 |
| Kiosco3 | 192.168.0.4 | 255.255.255.128 | 192.168.0.1 |
| K1-P (Impresora) | 192.168.0.5 | 255.255.255.128 | 192.168.0.1 |
| K2-P (Impresora) | 192.168.0.6 | 255.255.255.128 | 192.168.0.1 |
| K3-P (Impresora) | 192.168.0.7 | 255.255.255.128 | 192.168.0.1 |
| Pasajero1 (Laptop) | 192.168.0.8 | 255.255.255.128 | 192.168.0.1 |
| Pasajero2 (Laptop) | 192.168.0.9 | 255.255.255.128 | 192.168.0.1 |
| Pasajero3 (Laptop) | 192.168.0.10 | 255.255.255.128 | 192.168.0.1 |
| Pasajero4 (Laptop) | 192.168.0.11 | 255.255.255.128 | 192.168.0.1 |
| Pasajero5 (Laptop) | 192.168.0.12 | 255.255.255.128 | 192.168.0.1 |
| Smartphone1 | 192.168.0.13 | 255.255.255.128 | 192.168.0.1 |
| Smartphone2 | 192.168.0.14 | 255.255.255.128 | 192.168.0.1 |

### Área 3 — Torre de Control (VLAN 39 | 192.168.0.192/26)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| Server T1 | 192.168.0.194 | 255.255.255.192 | 192.168.0.193 |
| Server T2 | 192.168.0.195 | 255.255.255.192 | 192.168.0.193 |
| PC T1 | 192.168.0.196 | 255.255.255.192 | 192.168.0.193 |
| PC T2 | 192.168.0.197 | 255.255.255.192 | 192.168.0.193 |
| PC T3 | 192.168.0.198 | 255.255.255.192 | 192.168.0.193 |
| PC T4 | 192.168.0.199 | 255.255.255.192 | 192.168.0.193 |

### Área 5 — Terminal de Carga (VLAN 59 | 192.168.1.0/27)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| PC C1 | 192.168.1.2 | 255.255.255.224 | 192.168.1.1 |
| PC C2 | 192.168.1.3 | 255.255.255.224 | 192.168.1.1 |
| PC C3 | 192.168.1.4 | 255.255.255.224 | 192.168.1.1 |
| PC C4 | 192.168.1.5 | 255.255.255.224 | 192.168.1.1 |

### Área 7 — Bomberos SSEI (VLAN 79 | 192.168.1.96/28)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| PC B1 | 192.168.1.98 | 255.255.255.240 | 192.168.1.97 |
| PC B2 | 192.168.1.99 | 255.255.255.240 | 192.168.1.97 |
| PC B3 | 192.168.1.100 | 255.255.255.240 | 192.168.1.97 |

### Área 6 — Mantenimiento y Hangares (VLAN 69 | 192.168.0.128/26)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| PC M1 | 192.168.0.130 | 255.255.255.192 | 192.168.0.129 |
| PC M2 | 192.168.0.131 | 255.255.255.192 | 192.168.0.129 |

### Área 4 — Plataforma Rampa / Estacionamiento (VLAN 49 | 192.168.1.112/28)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| PC E1 | 192.168.1.114 | 255.255.255.240 | 192.168.1.113 |
| PC E3 | 192.168.1.115 | 255.255.255.240 | 192.168.1.113 |

### Área 2 — Pista 02/20 (VLAN 29 | 192.168.1.32/27)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| PC P1 | 192.168.1.34 | 255.255.255.224 | 192.168.1.33 |
| PC P2 | 192.168.1.35 | 255.255.255.224 | 192.168.1.33 |

### Área 8 — Migración y Aduana (VLAN 89 | 192.168.1.64/27)

| Dispositivo | IP | Máscara | Gateway |
|-------------|-----|---------|---------|
| Server Mi1 | 192.168.1.66 | 255.255.255.224 | 192.168.1.65 |
| Server Mi2 | 192.168.1.67 | 255.255.255.224 | 192.168.1.65 |
| IP Phone0 | 192.168.1.68 | 255.255.255.224 | 192.168.1.65 |
| PC Migra1 | 192.168.1.69 | 255.255.255.224 | 192.168.1.65 |
| PC Migra2 | 192.168.1.70 | 255.255.255.224 | 192.168.1.65 |
| PC Migra3 | 192.168.1.71 | 255.255.255.224 | 192.168.1.65 |

---

## 7. Configuración de Switches

### 7.1 SW-Core

```
enable
configure terminal
hostname SW-Core
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vlan 19
 name Terminal-Pasajeros
vlan 29
 name Pista
vlan 39
 name Torre-Control
vlan 49
 name Plataforma-Rampa
vlan 59
 name Terminal-Carga
vlan 69
 name Mantenimiento
vlan 79
 name Bomberos
vlan 89
 name Migracion-Aduana
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
interface range Fa0/1 - 3
 channel-group 1 mode active
 channel-protocol lacp
exit
interface range Fa0/4 - 6
 channel-group 2 mode active
 channel-protocol lacp
exit
interface range Fa0/7 - 9
 channel-group 3 mode active
 channel-protocol lacp
exit
interface range Fa0/14 - 16
 channel-group 4 mode active
 channel-protocol lacp
exit
interface range Fa0/13, Fa0/17 - 18
 channel-group 5 mode active
 channel-protocol lacp
exit
interface range Fa0/19 - 20
 channel-group 6 mode active
 channel-protocol lacp
exit
interface port-channel 1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 19,99,999
exit
interface port-channel 2
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 39,99,999
exit
interface port-channel 3
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 59,99,999
exit
interface port-channel 4
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 89,99,999
exit
interface port-channel 5
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 29,99,999
exit
interface port-channel 6
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 49,99,999
exit
interface Fa0/12
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 69,99,999
exit
interface Fa0/10
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 79,99,999
exit
interface range Fa0/11, Fa0/21 - 24, Gig0/1 - 2
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.2 SW-Tpasarejos (Área 1 — VLAN 19)

```
enable
configure terminal
hostname SW-Tpasarejos
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode server
vtp domain 202307499
vtp password area1
vlan 19
 name Terminal-Pasajeros
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 19 priority 4096
interface range Fa0/1 - 3
 channel-group 1 mode active
 channel-protocol lacp
exit
interface port-channel 1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 19,99,999
exit
interface Fa0/4
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 19,99,999
exit
interface Fa0/5
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 19,99,999
exit
interface range Fa0/6 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.3 SW-A1-1 (Acceso Área 1 — Sector Check-in)

```
enable
configure terminal
hostname SW-A1-1
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode client
vtp domain 202307499
vtp password area1
spanning-tree mode rapid-pvst
interface Fa0/1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 19,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 19
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 19
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 19
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 19
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 19
exit
interface Fa0/7
 switchport mode access
 switchport access vlan 19
exit
interface range Fa0/8 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.4 SW-A1-2 (Acceso Área 1 — Salas de Espera)

```
enable
configure terminal
hostname SW-A1-2
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode client
vtp domain 202307499
vtp password area1
spanning-tree mode rapid-pvst
interface Fa0/1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 19,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 19
exit
interface range Fa0/3 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.5 SW-TorreControl (Área 3 — VLAN 39)

```
enable
configure terminal
hostname SW-TorreControl
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode server
vtp domain 202307499
vtp password area3
vlan 39
 name Torre-Control
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 39 priority 4096
interface range Fa0/1 - 3
 channel-group 2 mode active
 channel-protocol lacp
exit
interface port-channel 2
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 39,99,999
exit
interface Fa0/10
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 39,99,999
exit
interface range Fa0/4 - 9, Fa0/11 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.6 SW-A3-1 (Acceso Área 3 — Torre de Control)

```
enable
configure terminal
hostname SW-A3-1
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vtp domain 202307499
vlan 39
 name Torre-Control
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
interface Fa0/1
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 39,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 39
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 39
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 39
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 39
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 39
exit
interface Fa0/7
 switchport mode access
 switchport access vlan 39
exit
interface range Fa0/8 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.7 SW-Tcarga (Área 5 — VLAN 59)

```
enable
configure terminal
hostname SW-Tcarga
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode server
vtp domain 202307499
vtp password area5
vlan 59
 name Terminal-Carga
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 59 priority 4096
interface range Fa0/1 - 3
 channel-group 3 mode active
 channel-protocol lacp
exit
interface port-channel 3
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 59,99,999
exit
interface Fa0/4
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 59,99,999
exit
interface range Fa0/5 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.8 SW-AS-1 (Acceso Área 5 — Terminal de Carga)

```
enable
configure terminal
hostname SW-AS-1
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode client
vtp domain 202307499
vtp password area5
spanning-tree mode rapid-pvst
interface Fa0/1
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 59,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 59
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 59
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 59
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 59
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 59
exit
interface range Fa0/7 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.9 SW-Bomberos (Área 7 — VLAN 79)

```
enable
configure terminal
hostname SW-Bomberos
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vlan 79
 name Bomberos
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 79 priority 4096
interface Fa0/1
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 79,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 79
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 79
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 79
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 79
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 79
exit
interface range Fa0/7 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.10 SW-Migracion (Área 8 — VLAN 89)

```
enable
configure terminal
hostname SW-Migracion
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vlan 89
 name Migracion-Aduana
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 89 priority 4096
interface range Fa0/1, Fa0/8 - 9
 channel-group 4 mode active
 channel-protocol lacp
exit
interface port-channel 4
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 89,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 89
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 89
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 89
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 89
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 89
exit
interface Fa0/7
 switchport mode access
 switchport access vlan 89
exit
interface range Fa0/10 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.11 SW-Mantenimiento (Área 6 — VLAN 69)

```
enable
configure terminal
hostname SW-Mantenimiento
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vlan 69
 name Mantenimiento
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 69 priority 4096
interface Fa0/1
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 69,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 69
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 69
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 69
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 69
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 69
exit
interface Fa0/7
 switchport mode access
 switchport access vlan 69
exit
interface range Fa0/8 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.12 SW-Pista (Área 2 — VLAN 29)

```
enable
configure terminal
hostname SW-Pista
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vlan 29
 name Pista
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 29 priority 4096
interface Fa0/1
 no shutdown
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 29,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 29
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 29
exit
interface Fa0/4
 switchport mode access
 switchport access vlan 29
exit
interface Fa0/5
 switchport mode access
 switchport access vlan 29
exit
interface range Fa0/6 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

### 7.13 SW-Estacionamiento (Área 4 — VLAN 49)

```
enable
configure terminal
hostname SW-Estacionamiento
enable secret 202307499
line console 0
password 202307499
login
exit
line vty 0 4
password 202307499
login
exit
vtp mode transparent
vlan 49
 name Plataforma-Rampa
vlan 99
 name Nativa
vlan 999
 name Blackhole
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 49 priority 4096
interface range Fa0/1, Fa0/4 - 5
 channel-group 6 mode active
 channel-protocol lacp
exit
interface port-channel 6
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 49,99,999
exit
interface Fa0/2
 switchport mode access
 switchport access vlan 49
exit
interface Fa0/3
 switchport mode access
 switchport access vlan 49
exit
interface Fa0/6
 switchport mode access
 switchport access vlan 49
exit
interface Fa0/7
 switchport mode access
 switchport access vlan 49
exit
interface range Fa0/8 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
exit
end
write memory
```

---

## 8. Capturas de Pantalla



