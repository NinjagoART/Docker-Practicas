# Instalación de un server DHCP 



<center> Importante: Tener creado el laboratorio de DNS </center>

**Objetivo.**  
Implementar y configurar un servicio DHCP que proporcione direcciones IP y configuración DNS automática.

_2. Configuración del servidor DHCP_  
Configurar el servicio para que proporcione direcciones en la red 192.168.30.0/24.  
Establecer el rango de direcciones IP disponibles (192.168.30.100 - 192.168.30.150)

#### Configuración de Dispositivos

| Dispositivo | Imagen                        | IP           |
| ----------- | ----------------------------- | ------------ |
| DNS         | ubuntu/bind9                  | 192.168.30.2 |
| DHCP        | networkboot/dhcpd (o dnsmasq) | 192.168.30.3 |
| Http        | nginx                         | 192.168.30.4 |
| PC1         | Ubuntu (customdh)             | DHCP         |
| PC2         | Ubuntu (customdh)             | DHCP         |


1. Haremos unas modificaciones al laboratorio, que incluye la red a una macvlan 
2. Usaremos docker compose apartir de este punto.

## Creamos nuestro estapcio de trabajo:
```bash
cd ~
mkdir DHCP
cd DHCP
```

### Modificaciones al Dockerfile

Modificaremos el dockerfile para hacer que la imagen del contenedor pueda pedir ip por medio de dhcp

`dockerfile`
```dockerfile
FROM ubuntu:22.04

# OpenSSH + utilidades + cliente DHCP
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      openssh-server isc-dhcp-client curl iputils-ping vim ca-certificates net-tools iproute2 && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /var/run/sshd && \
    echo 'root:root' | chpasswd && \
    sed -ri 's/^#?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -ri 's/^#?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    echo "AcceptEnv LANG LC_*" >> /etc/ssh/sshd_config

# Entrypoint: pide IP por DHCP y luego arranca sshd en foreground
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

Construimos usando: 
```bash
docker build -t ubuntu:customdh . 
```

### Modificaciones al dns

Al modificar el rango de direcciones, se requiere modificar el archivo de zona interna, y remover el acceso interno por nombre a los contenedores pc (si hay una solucion extra favor de buscarla).

`config/internal.TU_NOMBRE.osvp`
```$TTL 1h
@   IN SOA ns1.TU_NOMBRE.osvp. admin.TU_NOMBRE.osvp. (
        2025092401 ; serial
        1h         ; refresh
        15m        ; retry
        1w         ; expire
        1h )       ; minimum

    IN NS ns1.TU_NOMBRE.osvp.

ns1 IN A 192.168.30.2

; APEX y alias web
@    IN A 192.168.30.4
www  IN A 192.168.30.4

; Hosts
; pc1  IN A 10.10.0.10
; pc2  IN A 10.10.0.20
``` 

De igual manera se debera de modificar el archivo de configuracion de dhcp para ajustar al nuevo segmento de red.

`config/named.conf`
```bash
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 1.1.1.1; 8.8.8.8; };
    dnssec-validation no;
    listen-on { any; };
    listen-on-v6 { any; };
};

view "internal" {
    match-clients { 192.168.30.0/24; localhost; localnets; };

    zone "tunombre.osvp" IN {
        type master;
        file "/etc/bind/internal.TU_NOMBRE.osvp";
    };
};

view "external" {
    match-clients { any; };

    zone "tunombre.osvp" IN {
        type master;
        file "/etc/bind/external.TU_NOMBRE.osvp";
    };
};
``` 


---------------

## Usando dnsmaq:

`compose.yml`
```yaml
services: 
  dhcpd:
    container_name: dhcpd
    image: 4km3/dnsmasq
    restart: unless-stopped
    ports:
        - 67:67/udp
        - 67:67
    volumes:
      - ./dnsmasq/dhcp.conf:/etc/dnsmasq.d/dhcp.conf:ro
    cap_add:
      - NET_ADMIN
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.3
    command: [ "--log-facility=-"]

  dns_dhcp:
    container_name: dns_dhcp
    image: ubuntu/bind9:latest
    environment:
      - BIND9_USER=root
      - TZ=America/New_York
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
      - ./web/:/usr/share/nginx/html:ro  
    ports:
      - "80:80"        # HTTP externo real
    restart: unless-stopped
    depends_on:
      - dns_dhcp
  
  pc1:
    image: ubuntu:customdh
    container_name: pc1_dhcp
    hostname: pc1
    dns: 
      - 192.168.30.2
    cap_add:
      - NET_ADMIN
    networks:
        netlab_dhcp:
    ports:
      - "2021:22"  
    restart: unless-stopped
    depends_on:
      - dhcpd
      - dns_dhcp

  pc2:
    image: ubuntu:customdh
    container_name: pc2_dhcp
    hostname: pc2
    dns: 
      - 192.168.30.2
    cap_add:
      - NET_ADMIN
    networks:
        netlab_dhcp:
    ports:
      - "2022:22" 
    restart: unless-stopped
    depends_on:
      - dhcpd
      - dns_dhcp

networks:
  netlab_dhcp:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.30.0/24
          gateway: 192.168.30.1
```

##### Archivo de configuración de dhcp:

```bash 
mkdir dnsmasq
touch dnsmasq/dhcp.conf
nano dnsmasq/dhcp.conf
```

`dnsmasq/dhcp.conf`
```bash
interface=eth0
bind-interfaces

# Rango DHCP 192.168.30.100 - 192.168.30.150
dhcp-range=192.168.30.100,192.168.30.150,255.255.255.0,12h

# Router y DNS que se entregan a los clientes
dhcp-option=option:router,192.168.30.1
dhcp-option=option:dns-server,192.168.30.2

# Logs útiles
log-dhcp
log-queries

``` 

```bash 
docker compose up -d
``` 


---

## Usando isc-dhcp-server:

`compose.yml`
```yaml
services: 
  dhcpd:
    image: networkboot/dhcpd
    container_name: dhcpd
    dns:
      - 192.168.30.2
    ports:
        - 67:67/udp
    command: ["eth0"]          # interfaz dentro del contenedor macvlan
    cap_add:
      - NET_ADMIN
    volumes:
      - ./dhcp:/data
    networks:
      netlab_dhcp:
        ipv4_address: 192.168.30.3
    restart: unless-stopped
    depends_on:
      - dns_dhcp
        
  dns_dhcp:
    container_name: dns_dhcp
    image: ubuntu/bind9:latest
    environment:
      - BIND9_USER=root
      - TZ=America/New_York
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
      - ./web/:/usr/share/nginx/html:ro  
    ports:
      - "80:80"        # HTTP externo real
    restart: unless-stopped
    depends_on:
      - dns_dhcp
  
  pc1:
    image: ubuntu:custom
    container_name: pc1_dhcp
    hostname: pc1
    dns: 
      - 192.168.30.2
    cap_add:
      - NET_ADMIN
    networks:
        netlab_dhcp:
    ports:
      - "2021:22"  
    restart: unless-stopped
    depends_on:
      - dhcpd
      - dns_dhcp

  pc2:
    image: ubuntu:custom
    container_name: pc2_dhcp
    hostname: pc2
    dns: 
      - 192.168.30.2
    cap_add:
      - NET_ADMIN
    networks:
        netlab_dhcp:
    ports:
      - "2022:22" 
    restart: unless-stopped
    depends_on:
      - dhcpd
      - dns_dhcp

networks:
  netlab_dhcp:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.30.0/24
          gateway: 192.168.30.1
```

##### Archivo de configuración de dhcp:

```bash 
mkdir dhcp
touch dhcp/dhcpd.conf
nano dhcp/dhcpd.conf
```

`config/dhcpd.conf`
```bash
default-lease-time 3600;
max-lease-time 7200;
authoritative;

subnet 192.168.30.0 netmask 255.255.255.0 {
  range 192.168.30.100 192.168.30.150;
  option routers 192.168.30.1;           
  option domain-name-servers 192.168.30.2;  # tu BIND
  option subnet-mask 255.255.255.0;
}
``` 

```bash 
docker compose up -d
``` 

---

## Verificación

Lista de contenedores:

```bash
docker ps
```

Comprobación de red:

```bash
docker logs -f dhcpd      # Deberías ver DISCOVER/OFFER/ACK
docker exec -it pc1_dhcp bash -lc 'ip -br a; cat /etc/resolv.conf'
docker exec -it pc2_dhcp bash -lc 'ip -br a; cat /etc/resolv.conf'
```
