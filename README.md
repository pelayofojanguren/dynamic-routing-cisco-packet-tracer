# Protocolo de enrutamiento dinámico
Tarea 11: Implementación de VLSM/CIDR, VLANs, Router-on-a-Stick, migración de RIPv1 a RIPv2 y despliegue de OSPF en Cisco Packet Tracer.

* **Alumno:** Pelayo Fernández Ojanguren
* **Módulo:** Planificación y Administración de Redes (PAR)
* **Memoria en PDF:** [`Protocolo de enrutamiento dinamico.pdf`](./Protocolo%20de%20enrutamiento%20dinamico.pdf)
* **Archivo de Packet Tracer:** [`tarea11_PAR.pkt`](./tarea11_PAR.pkt)

---

## VLSM y CIDR

* **Red principal asignada:** `192.168.100.0/24`
* **ADMIN (50 hosts):** `/26` → $2^6 - 2 = 62$ hosts disponibles.
* **VENTAS (25 hosts):** `/27` → $2^5 - 2 = 30$ hosts disponibles.
* **INVITADOS (10 hosts):** `/28` → $2^4 - 2 = 14$ hosts disponibles.

### Tabla de direccionamiento

| Subred | Red | Rango de Hosts | Broadcast | Máscara |
| :--- | :--- | :--- | :--- | :---: |
| **ADMIN** | `192.168.100.0` | 192.168.100.1 - 192.168.100.62 | `192.168.100.63` | `/26` |
| **VENTAS** | `192.168.100.64` | 192.168.100.65 - 192.168.100.94 | `192.168.100.95` | `/27` |
| **INVITADOS** | `192.168.100.96` | 192.168.100.97 - 192.168.100.110 | `192.168.100.111` | `/28` |
| **R-A ↔ R-B** | `192.168.100.112` | 192.168.100.113 - 192.168.100.114 | `192.168.100.115` | `/30` |
| **R-B ↔ R-Edge** | `192.168.100.116` | 192.168.100.117 - 192.168.100.118 | `192.168.100.119` | `/30` |

* **Necesidad de VLSM:** Con una `/24` clásica daríamos 254 hosts a cada VLAN, desperdiciando cientos de IPs. VLSM permite asignar exactamente el tamaño necesario a cada segmento, optimizando el espacio de direccionamiento.
* **CIDR:** Permite agrupar múltiples redes continuas bajo un único prefijo. En caso de que la infraestructura crezca, sirve para resumir todas las subredes internas en una sola línea de enrutamiento hacia el exterior (`192.168.100.0/24`), reduciendo la carga y el tamaño de las tablas de los routers externos.

---

## 1. Configuración de VLANs y Trunks

En el switch principal (`SW-CORE`) y los de acceso (`SW-A`, `SW-B`), creamos las VLANs correspondientes y configuramos los enlaces troncales:

```cisco
SW-CORE# show vlan brief
# VLAN 10 (ADMIN), VLAN 20 (VENTAS), VLAN 30 (INVITADOS) activas

SW-CORE# show interfaces trunk
# Enlaces troncales dot1q activos en Fa0/1, Fa0/2 y Fa0/3 permitiendo el paso de las VLANs 1, 10, 20 y 30.
```

---

## 2. Configuración con RIP v1

### Router A (Subinterfaces y Router-on-a-Stick)
Configuración de subinterfaces con encapsulación `dot1Q` para actuar como puerta de enlace de cada VLAN:
```cisco
Router(config)# interface g0/1.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.100.1 255.255.255.192

Router(config)# interface g0/1.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.100.65 255.255.255.224

Router(config)# interface g0/1.30
Router(config-subif)# encapsulation dot1Q 30
Router(config-subif)# ip address 192.168.100.97 255.255.255.240

Router(config)# interface g0/0
Router(config-if)# ip address 192.168.100.113 255.255.255.252
Router(config-if)# no shutdown

Router(config)# router rip
Router(config-router)# network 192.168.100.0
```

### Router B
```cisco
Router(config)# interface g0/1
Router(config-if)# ip address 192.168.100.114 255.255.255.252
Router(config-if)# no shutdown

Router(config)# interface g0/0
Router(config-if)# ip address 192.168.100.117 255.255.255.252
Router(config-if)# no shutdown

Router(config)# router rip
Router(config-router)# network 192.168.100.0
```

---

## 3. Diagnóstico de errores en RIP v1

Al comprobar las rutas en `R-B` mediante `show ip route`:
* **Problema:** En `R-B` no aparece ninguna ruta aprendida por el protocolo.
* **Causa técnica:** RIPv1 es un protocolo que envía las actualizaciones del enrutamiento sin incluir la máscara de subred. Al utilizar VLSM, con las diferentes máscaras `/26`, `/27`, `/28` y `/30`, `R-B` no puede interpretar la segmentación real que le anuncia `R-A` y descarta las rutas o las asume bajo la máscara con clase por defecto (`/24`). Además, la sumarización automática (*automatic network summarization is in effect*) agrava el problema.

---

## 4. Migración a RIP v2

Activamos la versión 2 y desactivamos la sumarización automática en ambos routers:

```cisco
Router(config)# router rip
Router(config-router)# version 2
Router(config-router)# no auto-summary
```

### Verificación
Comprobamos en `R-B` con `show ip route` y `show ip protocols`:
* Las rutas aparecen ahora marcadas con `R` con sus máscaras exactas:
  * `192.168.100.0/26` vía `192.168.100.113`
  * `192.168.100.64/27` vía `192.168.100.113`
  * `192.168.100.96/28` vía `192.168.100.113`
* Se confirma que `Automatic network summarization is not in effect`.

---

## 5. Configuración de OSPF

Eliminamos RIP e implementamos OSPF Área 0 con sus *wildcards* correspondientes en toda la topología.

### Router A
```cisco
Router(config)# no router rip
Router(config)# router ospf 1
Router(config-router)# network 192.168.100.0 0.0.0.63 area 0
Router(config-router)# network 192.168.100.64 0.0.0.31 area 0
Router(config-router)# network 192.168.100.96 0.0.0.15 area 0
Router(config-router)# network 192.168.100.112 0.0.0.3 area 0
```

### Router B
```cisco
Router(config)# no router rip
Router(config)# router ospf 1
Router(config-router)# network 192.168.100.112 0.0.0.3 area 0
Router(config-router)# network 192.168.100.116 0.0.0.3 area 0
```

### Router Edge (Salida a ISP y propagación por defecto)
```cisco
Router(config)# interface g0/1
Router(config-if)# ip address 192.168.100.118 255.255.255.252
Router(config-if)# no shutdown

Router(config)# interface g0/0
Router(config-if)# ip address 192.168.200.1 255.255.255.252
Router(config-if)# no shutdown

# Ruta estática por defecto hacia el ISP simulado
Router(config)# ip route 0.0.0.0 0.0.0.0 192.168.200.2

# Inyección de la ruta por defecto al proceso OSPF
Router(config)# router ospf 1
Router(config-router)# network 192.168.100.116 0.0.0.3 area 0
Router(config-router)# default-information originate
```

---

## 6. Verificación final y conectividad extremo a extremo

### Comprobación en Router A
* `show ip ospf neighbor`: Adyacencia en estado **`FULL/DR`** con el vecino `192.168.100.117`.
* `show ip route`: Aprende la red interna remota vía `O` y la ruta por defecto inyectada por el router de borde como `O*E2 0.0.0.0/0 [110/1]`.

### Prueba desde PC-A1 (Extremo a extremo)
Desde el terminal de comandos del equipo `PC-A1`, realizamos `ping` y `tracert` hacia la IP del ISP simulado (`192.168.200.2`):
```text
C:\> ping 192.168.200.2
Reply from 192.168.200.2: bytes=32 time=1ms TTL=125
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

C:\> tracert 192.168.200.2
1   <1 ms   192.168.100.1   (Gateway VLAN ADMIN)
2   <1 ms   192.168.100.114 (R-B)
3    3 ms   192.168.100.118 (R-Edge)
4    4 ms   192.168.200.2   (ISP Simulado)
```
> Confirmamos conectividad bidireccional completa atravesando los 4 saltos de la infraestructura.

---

## Conclusión
* **Diseño VLSM y CIDR:** Permite evitar el desperdicio de direcciones IP asignando a cada subred el tamaño exacto necesario, facilitando además la escalabilidad y agregación de rutas hacia el exterior.
* **Comparativa RIPv1 vs RIPv2:** Se evidencian de forma práctica las limitaciones de los protocolos *classful* (RIPv1) al no enviar máscaras en sus actualizaciones, impidiendo el funcionamiento correcto con VLSM hasta migrar a RIPv2 con `no auto-summary`.
* **Despliegue de OSPF:** Ofrece una convergencia mucho más rápida y la capacidad de propagar automáticamente la ruta por defecto mediante `default-information originate`, simplificando la administración global de la red.
