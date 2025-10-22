# README — Redes Docker y DNS (bind9)

## Resumen

Documento breve sobre tipos de redes en Docker, zonas y registros en bind9, y comandos útiles para pruebas en un laboratorio con VPN.

---

## Tipos de redes en Docker

- **bridge (Puente)**  
  Driver por defecto. Crea una red virtual interna (NAT) que aísla los contenedores del Host.  
  Ideal para: contenedores que necesitan comunicarse entre sí y exponer puertos al Host.

- **custom bridge**  
  Versión personalizada del bridge donde defines la subred y puedes asignar IPs estáticas.  
  Perfecto para: laboratorios que requieren IPs fijas y conocidas (ej. 192.168.10.20).

- **macvlan**  
  Asigna una MAC e IP de la red física al contenedor, haciéndolo parecer un dispositivo real.  
  Ideal para: contenedores que deben estar en la misma red L2 que el Host o evitar el firewall del Host.

- **host**  
  El contenedor comparte directamente la pila de red del Host. No hay aislamiento.  
  Uso limitado: útil para servicios que requieren acceso total al Host (p.ej. ciertos servidores VPN).

---

## Tipos de zonas en bind9

- **Zona de Búsqueda Directa (Forward Lookup Zone)**  
  Traduce nombres a direcciones IP.  
  Ejemplo: `web.lab.local → 192.168.10.20`.

- **Zona de Búsqueda Inversa (Reverse Lookup Zone)**  
  Traduce direcciones IP a nombres.  
  Ejemplo: `192.168.10.20 → web.lab.local`.

---

## Tipos de registros DNS (resumen)

| Tipo  | Nombre             |                                   Propósito |                         Uso en el proyecto |
| ----- | ------------------ | ------------------------------------------: | -----------------------------------------: |
| A     | Address            |                  Mapea un nombre a una IPv4 | Esencial: `dns.lab.local`, `web.lab.local` |
| NS    | Name Server        |                Indica servidor autoritativo |                Necesario en `db.lab.local` |
| SOA   | Start of Authority | Info administrativa (email, serial, timers) |        Obligatorio en todo archivo de zona |
| PTR   | Pointer            |                       IP → nombre (inversa) |              Esencial para la zona inversa |
| CNAME | Canonical Name     |                   Alias de un nombre a otro |             Útil para nombres alternativos |

---

## Ejemplos y notas de la topología

- Servidor DNS: `192.168.10.10`
- Servidor web: `192.168.10.20`
- El sitio privado solo es accesible desde la VPN (la VPN está en una subred separada).
- Asegurarse de otorgar permisos/ACLs necesarios en la VPN para acceder a `web.lab.local`.

![Diagrama de red](image.png)

---

## Comandos de prueba

Probar DNS con un contenedor Alpine (instala bind-tools y hace dig):

```bash
docker run --rm --network server_dns_net alpine \
  sh -c "apk add --no-cache bind-tools && dig @192.168.10.10 web.lab.local"
```

Probar Nginx (resuelve `web.lab.local` hacia la IP del servidor web):

```bash
docker run --rm --network server_dns_net appropriate/curl \
  --resolve web.lab.local:80:192.168.10.20 http://web.lab.local/
```

---

## Buenas prácticas rápidas

- Mantener un serial incremental en el SOA cada vez que cambies las zonas.
- Usar IPs estáticas en redes de laboratorio cuando necesites reproducibilidad.
- Configurar registros PTR para todas las IPs importantes si dependes de búsquedas inversas.
- Documentar subredes y reglas de firewall/VPN para evitar problemas de acceso.
# private-red
