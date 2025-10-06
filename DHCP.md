# Práctica: Servidor DHCP con Docker

> **Objetivo:** Configurar un servidor DHCP que asigne direcciones IP y DNS automáticamente a los contenedores de una red Docker.

---

## Antes de comenzar

### Requisitos previos
- Haber completado y probado la **práctica de DNS** (el laboratorio anterior).
- Tener instalado y funcionando **Docker**.
- Haber detenido el laboratorio anterior si sigue en ejecución:
  ```bash
  docker compose -f ~/lab-dns/docker-compose.yml down 2>/dev/null || true
  ```

---

## Estructura del laboratorio DHCP

| Dispositivo | Imagen                      | IP           | Descripción                           |
| ----------- | --------------------------- | ------------ | ------------------------------------- |
| DNS         | ubuntu/bind9                | 192.168.30.2 | Servidor DNS interno                  |
| DHCP        | networkboot/dhcpd o dnsmasq | 192.168.30.3 | Asignará direcciones IP dinámicamente |
| HTTP        | nginx                       | 192.168.30.4 | Servidor web de prueba                |
| PC1         | ubuntu:customdh             | DHCP         | Cliente DHCP 1                        |
| PC2         | ubuntu:customdh             | DHCP         | Cliente DHCP 2                        |

> Solo los servidores (DNS, DHCP y HTTP) tendrán IP fija. Las PCs (pc1, pc2) recibirán IP automática vía DHCP.

---

## Paso 1: Crear carpeta de trabajo

Ejecuta esto en tu **máquina virtual Linux (no dentro de un contenedor):**
```bash
mkdir ~/lab-dhcp && cd ~/lab-dhcp
```

---

## Paso 2: Crear imagen personalizada con DHCP Client

Esta imagen permitirá que las PCs soliciten una IP automáticamente al servidor DHCP.

Crea el Dockerfile:
```bash
mkdir PC-ssh && cd PC-ssh
touch Dockerfile
nano Dockerfile
```

Pega lo siguiente:
```dockerfile
FROM ubuntu:22.04

# OpenSSH + DHCP Client + utilidades
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      openssh-server isc-dhcp-client curl iputils-ping vim ca-certificates net-tools iproute2 && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /var/run/sshd && \
    echo 'root:root' | chpasswd && \
    sed -ri 's/^#?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -ri 's/^#?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config

# Script de inicio: solicita IP por DHCP y lanza SSHD
RUN printf '%s\n' \
  '#!/bin/bash' \
  'set -e' \
  'ip addr flush dev eth0 || true' \
  'dhclient -v eth0 || true' \
  'exec /usr/sbin/sshd -D' > /usr/local/bin/start && \
  chmod +x /usr/local/bin/start

EXPOSE 22
CMD ["/usr/local/bin/start"]
```

Compila la imagen:
```bash
docker build -t ubuntu:customdh .
```

---

## Paso 3: Ajustar la configuración del DNS

Ya que la red cambia a `192.168.30.0/24`, debes editar tu archivo **config/internal.TU_NOMBRE.osvp** del laboratorio anterior.

Edita con:
```bash
nano ~/lab-dns/config/internal.TU_NOMBRE.osvp
```

Cambia el contenido a:
```dns-zone-file
$TTL 1h
@   IN SOA ns1.TU_NOMBRE.osvp. admin.TU_NOMBRE.osvp. (
        2025092401 ; serial
        1h ; refresh
        15m ; retry
        1w ; expire
        1h ) ; minimum

    IN NS ns1.TU_NOMBRE.osvp.

ns1 IN A 192.168.30.2
@    IN A 192.168.30.4
www  IN A 192.168.30.4
```
>  Ya no es necesario definir `pc1` ni `pc2`, pues ahora obtendrán IP vía DHCP.

Guarda y cierra el archivo.

---

## Paso 4: Crear red Docker para DHCP

```bash
docker network create netlab_dhcp \
  --driver bridge \
  --subnet 192.168.30.0/24 \
  --gateway 192.168.30.1
```

Verifica:
```bash
docker network inspect netlab_dhcp | grep Subnet
```

---

## Paso 5: Elige un servidor DHCP

Puedes usar **una de las dos opciones siguientes (elige solo una):**

### Opción 1 — Usar `dnsmasq`

#### a) Crear configuración de DHCP
```bash
mkdir dnsmasq
nano dnsmasq/dhcp.conf
```

Contenido:
```bash
interface=eth0
bind-interfaces

dhcp-range=192.168.30.100,192.168.30.150,255.255.255.0,12h
dhcp-option=option:router,192.168.30.1
dhcp-option=option:dns-server,192.168.30.2

log-dhcp
log-queries
```

#### b) Crear archivo `docker-compose.yml`
```bash
nano docker-compose.yml
```

Contenido:
```yaml
services:
  dhcpd:
    image: 4km3/dnsmasq
    container_name: dhcpd
    restart: unless-stopped
    ports:
      - 67:67/udp
    volumes:
      - ./dnsmasq/dhcp.conf:/etc/dnsmasq.d/dhcp.conf:ro
    cap_add:
      - NET_ADMIN
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.3
    command: ["--log-facility=-"]

  dns_dhcp:
    image: ubuntu/bind9:latest
    container_name: dns_dhcp
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    volumes:
      - ./config:/etc/bind
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.2
    restart: unless-stopped

  http:
    image: nginx:latest
    container_name: httpd_dhcp
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.4
    volumes:
      - ./web:/usr/share/nginx/html:ro
    ports:
      - "80:80"
    restart: unless-stopped
    depends_on:
      - dns_dhcp

  pc1:
    image: ubuntu:customdh
    container_name: pc1_dhcp
    hostname: pc1
    dns: [192.168.30.2]
    cap_add: [NET_ADMIN]
    networks:
      - netlab_dhcp
    ports: ["2021:22"]
    restart: unless-stopped
    depends_on: [dhcpd, dns_dhcp]

  pc2:
    image: ubuntu:customdh
    container_name: pc2_dhcp
    hostname: pc2
    dns: [192.168.30.2]
    cap_add: [NET_ADMIN]
    networks:
      - netlab_dhcp
    ports: ["2022:22"]
    restart: unless-stopped
    depends_on: [dhcpd, dns_dhcp]

networks:
  netlab_dhcp:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.30.0/24
          gateway: 192.168.30.1
```

Levanta los servicios:
```bash
docker compose up -d
```

---

###  Opción 2 — Usar `isc-dhcp-server`

#### a) Crear configuración de DHCP
```bash
mkdir dhcp
nano dhcp/dhcpd.conf
```

Contenido:
```bash
default-lease-time 3600;
max-lease-time 7200;
authoritative;

subnet 192.168.30.0 netmask 255.255.255.0 {
  range 192.168.30.100 192.168.30.150;
  option routers 192.168.30.1;
  option domain-name-servers 192.168.30.2;
  option subnet-mask 255.255.255.0;
}
```

#### b) Crear archivo `docker-compose.yml`
```bash
nano docker-compose.yml
```

Contenido:
```yaml
services:
  dhcpd:
    image: networkboot/dhcpd
    container_name: dhcpd
    ports: ["67:67/udp"]
    command: ["eth0"]
    cap_add: [NET_ADMIN]
    volumes:
      - ./dhcp:/data
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.3
    restart: unless-stopped
    depends_on: [dns_dhcp]

  dns_dhcp:
    image: ubuntu/bind9:latest
    container_name: dns_dhcp
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    volumes:
      - ./config:/etc/bind
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.2
    restart: unless-stopped

  http:
    image: nginx:latest
    container_name: httpd_dhcp
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.4
    volumes:
      - ./web:/usr/share/nginx/html:ro
    ports: ["80:80"]
    restart: unless-stopped
    depends_on: [dns_dhcp]

  pc1:
    image: ubuntu:customdh
    container_name: pc1_dhcp
    hostname: pc1
    dns: [192.168.30.2]
    cap_add: [NET_ADMIN]
    networks:
      - netlab_dhcp
    ports: ["2021:22"]
    restart: unless-stopped
    depends_on: [dhcpd, dns_dhcp]

  pc2:
    image: ubuntu:customdh
    container_name: pc2_dhcp
    hostname: pc2
    dns: [192.168.30.2]
    cap_add: [NET_ADMIN]
    networks:
      - netlab_dhcp
    ports: ["2022:22"]
    restart: unless-stopped
    depends_on: [dhcpd, dns_dhcp]

networks:
  netlab_dhcp:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.30.0/24
          gateway: 192.168.30.1
```

Levanta los servicios:
```bash
docker compose up -d
```

---

## Paso 6: Verificar funcionamiento

Lista de contenedores:
```bash
docker ps
```

Ver logs del DHCP:
```bash
docker logs -f dhcpd
```
>   Deberías ver mensajes como:
> ```
> DHCPDISCOVER from 02:42:c0:a8:1e:65 via eth0
> DHCPOFFER on 192.168.30.101
> DHCPACK on 192.168.30.101
> ```

Comprobar IP asignada y DNS:
```bash
docker exec -it pc1_dhcp bash -lc 'ip -br a; cat /etc/resolv.conf'
docker exec -it pc2_dhcp bash -lc 'ip -br a; cat /etc/resolv.conf'
```

Prueba con `ping`:
```bash
ping www.TU_NOMBRE.osvp
```

---

##  ¡Laboratorio DHCP funcionando!

Si todo está correcto:
- PC1 y PC2 reciben IP automáticamente.
- El DNS resuelve nombres internos.
- El servidor web responde desde su IP o dominio.

>  **Consejo:** Si algo falla, ejecuta `docker logs <nombre_contenedor>` para diagnosticar el problema.

