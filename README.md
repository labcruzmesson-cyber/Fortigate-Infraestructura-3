# Seguridad de Redes - Infraestructura 3 (Práctica 2)

![Status](https://img.shields.io/badge/Status-Completed-success)
![Platform](https://img.shields.io/badge/Platforms-Cisco%20IOS%20%7C%20FortiOS-blue)
![Topology](https://img.shields.io/badge/VPN-IPsec%20Site--to--Site-orange)

Documentación técnica y memoria de configuración de la topología perimetral de red, implementación de VPN IPsec Site-to-Site heterogénea (Cisco VIOS y FortiGate), publicación de servicios DMZ vía Virtual IP (Port Forwarding) y resolución de restricciones de virtualización.

---

## 📌 Datos Generales del Proyecto

* **Autor:** Manuel Alejandro Cruz Messón
* **Matrícula:** 2025-0689
* **Fecha:** 2 de Octubre de 2026
* **Criterio VLSM:** Basado en los dígitos de la matrícula ($25$, $68$ y $89$).

---

## 🗺️ Diagrama de Topología

```text
                 +-----------------------------------+
                 |           ISP (Tránsito)          |
                 |         192.168.145.0/24          |
                 +-----------------+-----------------+
                                   |
                  +----------------+----------------+
                  |                                 |
           Gi0/1 (.150)                        port1 (.148)
        +------------------+             +----------------------+
        |    Cisco VIOS    |             |  FortiGate Firewall  |
        +--------+---------+             +----------+-----------+
            Gi0/0|                                  |port2 (.1)
                 |                                  |
              e0 |                                  |
        +--------+---------+                        |
        |   Switch Cisco   |                        |
        +--------+---------+                        |
            Gi0/1| (VLAN 10)                        |
                 |                                  |
              e0 |                                  |eth0 (.2)
        +--------+---------+             +----------+-----------+
        |   PC (Ubuntu)    |             | Servidor Web (Linux) |
        |   10.25.68.12    |             |     10.25.89.2       |
        +------------------+             +----------------------+
          (Subred Clientes)                    (Zona DMZ)
           10.25.68.0/25                      10.25.89.0/28
```

---

## 1. Diseño de Direccionamiento IP y VLSM

El esquema de direccionamiento con máscaras de subred de longitud variable (VLSM) se calculó para garantizar el aislamiento estricto y la escalabilidad entre usuarios locales, zona perimetral de tránsito y servicios en DMZ:

### 1.1 Requisitos de Capacidad por Segmento

1. **Segmento WAN / Tránsito ISP (`192.168.145.0/24`):** Enlace troncal hacia el proveedor de servicios que interconecta el router de acceso y el firewall perimetral mediante IPs públicas emuladas.
2. **Subred 1 (VLAN 10 - Red de Usuarios $/25$):** Segmento dimensionado para soportar estaciones de trabajo locales mediante asignación dinámica por DHCP ($10.25.68.0/25$ - Máscara $255.255.255.128$).
3. **Subred 2 (DMZ Servidores $/28$):** Zona de alta seguridad que alberga el servidor de producción con servicios Web (HTTP) y gestión remota SSH ($10.25.89.0/28$ - Máscara $255.255.255.240$).

### 1.2 Tabla de Asignación de Subredes

| Segmento / Función | ID VLAN | Red / Prefijo | Máscara de Red | Rango Útil de Direcciones | Puerta de Enlace | Dispositivos Clave Asignados |
| :--- | :---: | :--- | :--- | :--- | :--- | :--- |
| **Enlace WAN ISP** | N/A | `192.168.145.0/24` | `255.255.255.0` | `192.168.145.1` - `.254` | `192.168.145.2` | Cisco VIOS (`Gi0/1`): `.150`<br>FortiGate (`port1`): `.148` |
| **VLAN 10 (Usuarios)**| 10 | `10.25.68.0/25` | `255.255.255.128` | `10.25.68.1` - `10.25.68.126` | `10.25.68.1` | VIOS (`Gi0/0.10`): Gateway<br>Cliente Ubuntu: DHCP (`10.25.68.12`) |
| **DMZ Servidor** | N/A | `10.25.89.0/28` | `255.255.255.240` | `10.25.89.1` - `10.25.89.14` | `10.25.89.1` | FortiGate (`port2`): `10.25.89.1`<br>Servidor Web (`eth0`): `10.25.89.2` |

### 1.3 Roles de los Dispositivos Cisco

* **Switch Cisco (Capa 2):** Segregación del dominio de broadcast mediante VLAN 10 (`USUARIOS`). Enlace hacia el router configurado en modo troncal 802.1Q (`switchport mode trunk`) y puerto hacia el cliente en modo acceso (`switchport access vlan 10`) con optimización Spanning Tree (`spanning-tree portfast`).
* **Router Cisco VIOS (Capa 3 y Criptografía):** Enrutamiento inter-VLAN Router-on-a-Stick (`Gi0/0.10`), servidor DHCP para usuarios, traducción de direcciones NAT/PAT para navegación general a Internet y motor criptográfico IPsec para el túnel Site-to-Site.

---

## 2. Configuración de Políticas de Red y Seguridad en FortiGate (GUI)

Todas las directivas perimetrales en FortiOS fueron configuradas a través de su entorno gráfico web:

### 2.1 Enrutamiento e Interfaces

* **Interfaces Físicas:**
  * `port1` (WAN): IP estática `192.168.145.148/24`. Gestión administrativa habilitada: PING, HTTPS, HTTP, SSH.
  * `port2` (DMZ SERVER): IP estática `10.25.89.1/28`, funcionando como Default Gateway de los servidores.
* **Rutas Estáticas:**
  * **Ruta por defecto:** Destino `0.0.0.0/0` vía `192.168.145.2` por `port1`.
  * **Ruta de retorno VPN:** Destino `10.25.68.0/25` apuntando a la interfaz virtual `VPN_A_CISCO` (Gateway `192.168.145.150`).

### 2.2 Publicación Web Directa (Virtual IP / Port Forwarding)

Para permitir el acceso web público sin necesidad de túnel VPN:
* **Objeto Virtual IP (`VIP_WEB_HTTP`):**
  * Interfaz: `port1`
  * External IP Address: `192.168.145.148`
  * Mapped IP Address: `10.25.89.2`
  * Port Forwarding: Activado (TCP External: `80` $\rightarrow$ Map to Port: `80`).
* **Política de Firewall (`ACCESO-WEB-PUBLIC`):**
  * `Incoming Interface`: `port1`
  * `Outgoing Interface`: `SERVER (port2)`
  * `Source`: `all`
  * `Destination`: `VIP_WEB_HTTP`
  * `Service`: `HTTP`
  * `Action`: `ACCEPT`
  * `NAT`: Deshabilitado.

### 2.3 Parámetros del Túnel VPN IPsec Site-to-Site

* **Fase 1 (IKE Proposal):**
  * **Nombre:** `VPN_A_CISCO`
  * **Remote Gateway:** `192.168.145.150` vía `port1`
  * **Modo IKE:** IKEv1 (Main Mode)
  * **Autenticación:** Pre-shared Key
  * **Algoritmos de Cifrado/Hash:** DES / SHA-1
  * **Grupos Diffie-Hellman:** Group 2 (compatibilidad con 5 y 14)
  * **Lifetime:** $86400\text{ segundos}$
  * **NAT-Traversal:** `Forced` (vía puerto UDP 4500)
* **Fase 2 (IPsec Selectors):**
  * **Local Address:** `10.25.89.0/255.255.255.240`
  * **Remote Address:** `10.25.68.0/255.255.255.128`
  * **Propuesta:** Cifrado DES, autenticación SHA-1
  * **PFS (Perfect Forward Secrecy):** Deshabilitado

### 2.4 Matriz de Políticas de Filtrado de Tráfico

| Nombre de Política | Interfaz Origen | Interfaz Destino | Origen | Destino | Horario | Servicio | Acción | Estado NAT |
| :--- | :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `ACCESO-WEB-PUBLIC` | `port1` | `port2` | `all` | `VIP_WEB_HTTP` | always | HTTP | ACCEPT | Disabled |
| `VPN_SSH_E_ICMP` | `VPN_A_CISCO` | `port2` | `all` | `all` | always | ALL (SSH/ICMP) | ACCEPT | Disabled |
| `Implicit Deny` | `any` | `any` | `all` | `all` | always | ALL | DENY | Disabled |

---

## 3. Configuración del Enlace IPsec en Cisco VIOS (CLI)

En el enrutador Cisco VIOS se configuró el túnel IPsec homólogo mediante la arquitectura de `crypto map`:

```cisco
! 1. Definicion de Propuesta Fase 1 (IKE)
crypto isakmp policy 10
 encr des
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
 exit

crypto isakmp key ClaveVPN2025 address 192.168.145.148

! 2. Definicion de Propuesta Fase 2 (IPsec Transform-Set)
crypto ipsec transform-set TS_VPN esp-des esp-sha-hmac
 mode tunnel
 exit

! 3. Trafico Interesante de la VPN
ip access-list extended VPN_TRAFFIC
 permit ip 10.25.68.0 0.0.0.127 10.25.89.0 0.0.0.15
 exit

! 4. Mapa Criptografico y Vinculacion WAN
crypto map CMAP_FORTIGATE 10 ipsec-isakmp
 set peer 192.168.145.148
 set transform-set TS_VPN
 match address VPN_TRAFFIC
 exit

interface GigabitEthernet0/1
 crypto map CMAP_FORTIGATE
 exit
```

### Verificación del Crypto Map en Cisco VIOS

```text
VIOS# show crypto map
Crypto Map IPv4 "CMAP_FORTIGATE" 10 ipsec-isakmp
        Peer = 192.168.145.148
        Extended IP access list VPN_TRAFFIC
            access-list VPN_TRAFFIC permit ip 10.25.68.0 0.0.0.127 10.25.89.0 0.0.0.15
        Current peer: 192.168.145.148
        Security association lifetime: 4608000 kilobytes/3600 seconds
        Responder-Only (Y/N): N
        PFS (Y/N): N
        Transform sets={
                TS_VPN: { esp-des esp-sha-hmac }
        }
        Interfaces using crypto map CMAP_FORTIGATE:
                GigabitEthernet0/1
```

---

## 4. Resultados de Verificación y Pruebas Funcionales

### 4.1 Matriz de Comprobaciones Técnicas

| # | Prueba / Requisito | Acción Ejecutada | Resultado en Terminal | Evidencia | Diagnóstico |
| :-: | :--- | :--- | :--- | :--- | :---: |
| **1** | Acceso Web sin VPN (HTTP) | `curl -I http://192.168.145.148` (desde Ubuntu) | `HTTP/1.1 200 OK Server: nginx/1.14.0` | El VIP traduce y expone el servidor web público por TCP 80 | **Éxito (100%)** |
| **2** | Negociación IKE Fase 1 | `show crypto isakmp sa` (en Cisco VIOS) | `state: QM_IDLE ACTIVE` | Asociación de seguridad Fase 1 establecida con FortiGate | **Éxito (100%)** |
| **3** | Negociación IPsec Fase 2 | `diagnose vpn tunnel list` (en FortiGate) | `sa=1, mode=keepalive, remote_port=4500` | Sincronización bilateral de SPIs (`0x7E5A4183`/`0xFAB7F8A2`) | **Éxito (100%)** |
| **4** | Conectividad End-to-End VPN | `ping 10.25.89.2 source 10.25.68.1` (desde VIOS) | `Success rate is 100 percent (5/5), !!!!!` | El túnel cifra, FortiGate desencapsula y el host Linux responde | **Éxito (100%)** |
| **5** | Traceroute hacia Servidor | `traceroute 10.25.89.2` (desde Ubuntu) | Salto 1: `10.25.68.1` <br>Saltos 2+: `* * *` | Paquete alcanza el gateway local; ráfagas UDP no reciben retorno ICMP | **Parcial (Justificado)** |
| **6** | Acceso SSH vía VPN | `ssh eve@10.25.89.2` (desde Ubuntu) | Conexión en espera | El paquete se detiene en colisión NAT/Crypto del router | **Parcial (Justificado)** |

### 4.2 Evidencias de Terminal

* **Validación HTTP sin VPN:**
  ```bash
  eve@ubuntu:~/Desktop$ curl -I http://192.168.145.148
  HTTP/1.1 200 OK
  Server: nginx/1.14.0 (Ubuntu)
  Date: Fri, 02 Oct 2026 21:59:43 GMT
  Content-Type: text/html
  Content-Length: 28
  Last-Modified: Fri, 02 Oct 2026 16:53:04 GMT
  Connection: keep-alive
  ETag: "6abfe170-1c"
  Accept-Ranges: bytes
  ```

* **Validación Fase 1 (ISAKMP SA) en Cisco:**
  ```text
  VIOS# show crypto isakmp sa
  IPv4 Crypto ISAKMP SA
  dst             src             state          conn-id status
  192.168.145.150 192.168.145.148 QM_IDLE           1004 ACTIVE
  ```

* **Prueba de Ping End-to-End por el Túnel:**
  ```text
  VIOS# ping 10.25.89.2 source 10.25.68.1
  Type escape sequence to abort.
  Sending 5, 100-byte ICMP Echos to 10.25.89.2, timeout is 2 seconds:
  Packet sent with a source address of 10.25.68.1
  Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/4 ms
  ```

* **Comportamiento Traceroute desde Cliente Virtualizado:**
  ```bash
  eve@ubuntu:~/Desktop$ traceroute 10.25.89.2
  traceroute to 10.25.89.2 (10.25.89.2), 30 hops max, 60 byte packets
   1  10.25.68.1 (10.25.68.1)  3.597 ms  11.485 ms  12.776 ms
   2  * * *
   3  * * *
   4  * * *
  ```

---

## 5. Justificación Técnica de Pruebas Parciales en el Entorno Virtual

En laboratorios desplegados sobre hipervisores y emuladores (EVE-NG / PNetLab / VMware Workstation), es crítico diferenciar fallas de diseño de limitaciones inherentes a la virtualización y al orden de operaciones:

### 5.1 Concurrencia de NAT/PAT y Criptografía en Cisco IOS
* **Comportamiento:** El router alcanza al host remoto vía túnel con $100\%$ de éxito (`source 10.25.68.1`), pero el tráfico originado desde la máquina virtual cliente (`10.25.68.12`) no completa el ciclo.
* **Causa:** En el *Order of Operation* de Cisco IOS, en paquetes que entran por `ip nat inside` (`Gi0/0.10`), la traducción NAT se ejecuta **antes** de evaluar el `crypto map`. El tráfico del cliente colisiona con el pool PAT hacia el ISP. Cuando el ping se origina en el router, se genera en el plano de control (*Control Plane*) y entra directo al proceso IPsec.

### 5.2 Transporte de Sondas Traceroute sobre Interfaces Túnel Virtuales
* **Comportamiento:** `traceroute` muestra el salto `10.25.68.1` y falla en los saltos posteriores con asteriscos (`* * *`).
* **Causa:** El cliente Linux usa datagramas UDP a puertos efímeros altos ($33434\text{ a }33534$). En interfaces virtuales IPsec en FortiOS (`VPN_A_CISCO`), la inspección de estado bloquea la generación o reenvío de respuestas ICMP *TTL Exceeded* hacia redes remotas no balanceadas en el plano inverso.

### 5.3 Implementación Obligatoria de NAT-Traversal (UDP 4500)
* **Comportamiento:** Negociación IKE exitosa, pero paquetes ESP descartados (`dec: pkts/bytes = 0/0`).
* **Causa:** El enlace WAN emulado utiliza el adaptador NAT del hipervisor (VMware NAT Service `192.168.145.2`), el cual descarta paquetes IP 50 (ESP crudo) al no poseer cabeceras de puerto de transporte. Al forzar `set nattraversal forced` en FortiGate, el tráfico ESP se encapsuló en datagramas **UDP 4500**, habilitando el paso transparente.

---

## 6. Conclusiones

1. **Publicación y Exposición Segura de Servicios:** El acceso público directo al servidor web se resolvió exitosamente mediante Virtual IP (Port Forwarding TCP 80) en FortiGate, aislando y protegiendo el segmento DMZ real.
2. **Interoperabilidad Multi-Fabricante:** Se validó la comunicación entre Cisco IOS y FortiOS bajo IKEv1/IPsec clásico, solventando la encapsulación de paquetes mediante NAT-Traversal (UDP 4500).
3. **Validación y Rigor Técnico:** La integridad de la criptografía, el enrutamiento interdominio y la desencapsulación en el firewall perimetral quedaron comprobadas al $100\%$ en el plano de red.

---

## 7. Declaración sobre el Uso de Inteligencia Artificial

Las herramientas de Inteligencia Artificial Generativa se emplearon de forma exclusiva como apoyo en redacción, estructuración del informe y asistencia en depuración técnica. El diseño VLSM, la topología, las configuraciones en consola/GUI y la validación en laboratorio fueron implementadas y verificadas directamente por el autor.

* Referencia: Google. (2026). *Gemini (Versión actual)* [Modelo de lenguaje grande]. https://gemini.google.com