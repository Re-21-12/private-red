# Universidad Mariano Galvez

## Facultad de Ingeniería en Sistemas Computacionales

---

### Documento Técnico

**Diseño e implementación de red de laboratorio con Docker, WireGuard y BIND9**

**Autor:**  
Victor Alfredo Macario Enriquez  
**Carrera:** Ingeniería en Sistemas  
**Matrícula:** 7690-22-5042

**Fecha:** 29 de octubre de 2025

---

## Resumen

El presente documento describe el diseño, implementación y pruebas de una red de laboratorio que integra resolución DNS interna mediante BIND9, servicios web (sitio institucional y privado) y acceso remoto seguro mediante VPN WireGuard. Se documentan todos los comandos utilizados, explicando su función y por qué no deben eliminarse, para garantizar la reproducibilidad y seguridad del entorno.

---

## Objetivos

- Implementar un servidor DNS autoritativo con zonas forward y reverse usando BIND9.
- Desplegar un servidor web (Nginx o Apache) con al menos dos virtual hosts (público y privado).
- Configurar una VPN (WireGuard) que permita acceso remoto y resolución de nombres internos vía DNS.
- Aplicar reglas de firewall básicas y documentar políticas de seguridad.
- Realizar pruebas funcionales y entregar documentación técnica y guía de instalación.

---

## Alcance

- Red de laboratorio (física o virtual) con mínimo 3 máquinas/VMs:
  - Controlador/servidor DNS + VPN (pueden ir en la misma VM)
  - Servidor Web (virtual host público y privado)
  - Equipo cliente (conectándose vía VPN)
- Plan de direccionamiento:
  - Red servidores: 192.168.10.0/24
    - DNS/VPN: 192.168.10.10
    - Web: 192.168.10.20
  - Red interna/administrativa: 192.168.20.0/24
  - Red VPN (WireGuard): 10.8.0.0/24
  - Zonas DNS internas: lab.local

---

## Explicación del uso de firewall en Fedora

En este laboratorio, el host principal utiliza Fedora Linux. Fedora emplea `firewalld` como sistema de gestión de firewall, que permite definir reglas de acceso y NAT de manera dinámica y sencilla.  
**¿Por qué usar firewalld?**

- Es el sistema recomendado y soportado en Fedora.
- Permite administrar zonas, puertos y reglas de NAT sin editar manualmente archivos de configuración.
- Facilita la integración con servicios como Docker y WireGuard.

**Comandos de firewalld utilizados:**

```bash
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --add-forward-port=port=51820:proto=udp:toport=51820
sudo firewall-cmd --reload
```

_No eliminar_: Estas reglas permiten el enmascaramiento (NAT) y el reenvío de puertos para la VPN, asegurando que los clientes remotos puedan acceder a la red interna a través del túnel WireGuard.

---

## Explicación de NAT y flujo de red

**¿Qué es NAT?**  
La Traducción de Direcciones de Red (NAT) permite que los paquetes provenientes de la red interna (por ejemplo, la VPN) salgan a otras redes (como Internet o la red del host) usando la IP del host. Esto es esencial para que los clientes VPN puedan comunicarse con servicios internos y externos.

**Regla NAT utilizada:**

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlo1 -j MASQUERADE
```

_No eliminar_: Esta regla permite que los paquetes de la red VPN (10.8.0.0/24) salgan por la interfaz física del host (`wlo1`), ocultando las IPs internas y permitiendo la comunicación.

---

## Flujo de tráfico: Host → VPN → Contenedor VPN → DNS → Web → Retorno

1. **Cliente remoto** se conecta a la VPN WireGuard, estableciendo un túnel seguro hacia el host Fedora.
2. El **host Fedora** recibe el tráfico VPN en el puerto UDP 51820 y lo redirige al contenedor WireGuard (por NAT y reglas de firewall).
3. El **contenedor WireGuard** valida el acceso y permite que el tráfico de la VPN acceda a la red interna de Docker.
4. El cliente VPN realiza una consulta DNS (por ejemplo, para `web.lab.local`).
   - El tráfico pasa del contenedor VPN al contenedor DNS (BIND9).
   - El contenedor DNS responde con la IP interna del servidor web.
5. El cliente accede al **servidor web** usando la IP proporcionada por DNS.
   - El tráfico fluye del cliente VPN → túnel WireGuard → contenedor VPN → contenedor web.
6. Las respuestas del servidor web siguen el mismo camino de regreso:
   - Servidor web → contenedor VPN → túnel WireGuard → cliente remoto.

**Importante:**

- El firewall y NAT en Fedora aseguran que solo el tráfico autorizado fluya entre los componentes.
- El túnel VPN protege la comunicación y permite acceso seguro a servicios internos.
- Las reglas de firewall y NAT no deben eliminarse, ya que son esenciales para la conectividad y seguridad del entorno.

---

## Tipos de redes en Docker

- **bridge (Puente):** Red virtual interna (NAT) que aísla los contenedores del Host.  
  _No eliminar_: Es la red por defecto y permite comunicación entre contenedores y el host.
- **custom bridge:** Permite definir subredes y asignar IPs estáticas.  
  _No eliminar_: Útil para laboratorios con IPs fijas.
- **macvlan:** Asigna MAC e IP de la red física al contenedor.  
  _No eliminar_: Permite integración directa en la red física.
- **host:** El contenedor comparte la pila de red del Host.  
  _No eliminar_: Necesario para servicios que requieren acceso total.

---

## Tipos de zonas en bind9

- **Zona de Búsqueda Directa:** Traduce nombres a direcciones IP.  
  _No eliminar_: Esencial para la resolución de nombres en la red.
- **Zona de Búsqueda Inversa:** Traduce direcciones IP a nombres.  
  _No eliminar_: Necesaria para búsquedas inversas y validaciones.

---

## Tipos de registros DNS

| Tipo  | Nombre             | Propósito                                 | Uso en el proyecto                  |
| ----- | ------------------ | ----------------------------------------- | ----------------------------------- |
| A     | Address            | Mapea nombre a IPv4                       | `dns.lab.local`, `web.lab.local`    |
| NS    | Name Server        | Indica servidor autoritativo              | Necesario en `db.lab.local`         |
| SOA   | Start of Authority | Info administrativa (email, serial, etc.) | Obligatorio en todo archivo de zona |
| PTR   | Pointer            | IP → nombre (inversa)                     | Esencial para la zona inversa       |
| CNAME | Canonical Name     | Alias de un nombre a otro                 | Útil para nombres alternativos      |

_No eliminar_: Todos los tipos son requeridos para una configuración DNS completa y funcional.

---

## Ejemplos y notas de la topología

- Servidor DNS: `192.168.10.10`
- Servidor web: `192.168.10.20`
- El sitio privado solo es accesible desde la VPN.
- ACLs en la VPN para acceso a `web.lab.local`.

![Diagrama de red](image.png)

---

## Comandos de prueba y verificación

Todos los comandos siguientes son necesarios para instalar, probar y verificar los servicios.  
_No eliminar_: Permiten comprobar la funcionalidad y solucionar problemas.

### Probar DNS con Alpine

```bash
docker run --rm --network server_dns_net alpine \
  sh -c "apk add --no-cache bind-tools && dig @192.168.10.10 web.lab.local"
```

### Probar Nginx

```bash
docker run --rm --network server_dns_net appropriate/curl \
  --resolve web.lab.local:80:192.168.10.20 http://web.lab.local/
```

---

## Comandos rápidos para VPN WireGuard

- Encender la VPN:
  ```bash
  sudo wg-quick up wg0
  ```
- Apagar la VPN:
  ```bash
  sudo wg-quick down wg0
  ```

---

## Buenas prácticas rápidas

- Mantener un serial incremental en el SOA cada vez que cambies las zonas.
- Usar IPs estáticas en redes de laboratorio para reproducibilidad.
- Configurar registros PTR para todas las IPs importantes.
- Documentar subredes y reglas de firewall/VPN.

---

## Comandos de administración y verificación

### Verificar interface wg0

```bash
docker exec -it wg-easy wg show
```

### Verificar desde el servidor

```bash
docker exec -it wg-easy wg show
```

---

## Reglas de firewall y NAT

_No eliminar_: Protegen el entorno y permiten el acceso solo a servicios autorizados.

### Regla de enmascaramiento con firewalld

```bash
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --add-forward-port=port=51820:proto=udp:toport=51820
sudo firewall-cmd --reload
```

### Regla NAT enmascarado

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlo1 -j MASQUERADE
```

### Ver IP de la interfaz

```bash
ip addr show wlo1
```

---

## Configuración de DNS en Alpine

_No eliminar_: Permite instalar herramientas de diagnóstico DNS.

```bash
apk update
apk add bind-tools
```

---

## Debug y pruebas de conectividad

_No eliminar_: Permiten analizar tráfico y verificar conectividad.

```bash
# WireGuard
docker exec -it wg-easy tcpdump -i wg0

# bind9
docker exec -it bind tcpdump -i eth0
```

### Probar conectividad en la VPN

```bash
ping 10.8.0.2   # wg-easy
ping 10.8.0.254 # bind
```

### Probar resolución DNS

```bash
nslookup web.lab.local 10.8.0.254
```

---

## Desactivar systemd-resolved (si es necesario)

_No eliminar_: Evita conflictos de resolución DNS en sistemas Linux modernos.

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

---

## Referencias

- Documentación oficial de Docker
- Manual de WireGuard
- Guía de bind9 en Debian/Alpine

---

**Fin del documento**
