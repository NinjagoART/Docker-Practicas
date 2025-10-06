
Nota: Olvide darle permisos al usuario a docker, se resuelve haciendo esto:
```bash
su -
usermod -aG docker sadmin
exit
```
# Instalacion de un servidor DNS:

**Importante: Se requiere la instalacion de Docker en el servidor!!!**

Para instalar el servicio de DNS requeriremos:
- Un contendor http (nginx)
- 2 Contenedores ubuntu con ssh (Imagen custom)
- El contenedor de bind9 de ubuntu (ubuntu/bind9:lastest)

| Componente | Imagen          | IP         | Puerto              |
| ---------- | --------------- | ---------- | ------------------- |
| DNS        | ubuntu/bind9    | 10.10.0.2  | 53:53/tcp 53:53/udp |
| Http       | nginx           | 10.10.0.3  | 80:80               |
| PC1        | Ubuntu (custom) | 10.10.0.10 | 2021:22             |
| PC2        | Ubuntu (custom) | 10.10.0.20 | 2022:22             |

La topología de red es la siguiente:

![[Untitled Diagram.drawio(3).png]]

## Instalación de las imagenes:

### Imagen DNS y Http

```bash
docker pull nginx:latest
docker pull ubuntu/bind9
```
![[Pasted image 20251005123648.png]]

### Imagen de PCX

Esta imagen es custom basada en la de ubuntu, primero descargamos la imagen de ubuntu

```bash
docker pull ubuntu:22.04
```

Después de descargarla, construiremos nuestra propia version de esta imagen con ssh habilitado, usando docker build

```bash
# Creamos una carpeta para la imagen:
mkdir PC-ssh 
cd PC-ssh

# Creamos nuestro archivo `Dockerfile`
touch Dockerfile
nano Dockerfile
```

En el dockerfile colocamos lo siguiente:

```dockerfile
FROM ubuntu:22.04

# Actualiza e instala OpenSSH y utilidades básicas
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      openssh-server curl iputils-ping vim ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Configura SSH
RUN mkdir -p /var/run/sshd && \
    echo 'root:root' | chpasswd && \
    sed -ri 's/^#?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -ri 's/^#?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    echo "AcceptEnv LANG LC_*" >> /etc/ssh/sshd_config

# Expone el puerto SSH
EXPOSE 22

# Ejecuta SSHD en foreground (para mantener el contenedor vivo)
CMD ["/usr/sbin/sshd", "-D"]

```

Construimos la imagen llamandola ubuntu:custom

```shell
docker build -t ubuntu:custom .
```

Creación de las imagenes listo!!

## Montando el laboratorio

### Archivos necesarios:

En ambos metodos se requiere los siguientes archivos de configuracion para el dns:

1. Creamos una carpeta llamada config y los archivos de configuracion de zona:

```bash
mkdir config 
touch config/{named.conf,internal.TU_NOMBRE.osvp,external.TU_NOMBRE.osvp}
```

2. Editamos los archivos usando nano con lo siguiente (Cambia TU_NOMBRE por tu nombre xd)
   
`named.conf:`
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
    match-clients { 10.10.0.0/24; localhost; localnets; };

    zone "TU_NOMBRE.osvp" IN {
        type master;
        file "/etc/bind/internal.TU_NOMBRE.osvp";
    };
};

view "external" {
    match-clients { any; };

    zone "TU_NOMBRE.osvp" IN {
        type master;
        file "/etc/bind/external.TU_NOMBRE.osvp";
    };
};
```

`internal.TU_NOMBRE.osvp`
```dns-zone-file
$TTL 1h
@   IN SOA ns1.TU_NOMBRE.osvp. admin.TU_NOMBRE.osvp. (
        2025092401 ; serial
        1h         ; refresh
        15m        ; retry
        1w         ; expire
        1h )       ; minimum

    IN NS ns1.TU_NOMBRE.osvp.

ns1 IN A 10.10.0.2

; APEX y alias web
@    IN A 10.10.0.3
www  IN A 10.10.0.3

; Hosts
pc1  IN A 10.10.0.10
pc2  IN A 10.10.0.20
```

`external.TU_NOMBRE.osvp`
```dns-zone-file
$TTL 1h
@   IN SOA ns1.TU_NOMBRE.osvp. admin.TU_NOMBRE.osvp. (
        2025092501 ; serial
        1h
        15m
        1w
        1h )

    IN NS ns1.TU_NOMBRE.osvp.

; El DNS “externo” debe ser alcanzable desde fuera
ns1 IN A IP_MAQUINA_VIRTUAL

; Nombres de servicio resuelven al la maquina virtual
@    IN A IP_MAQUINA_VIRTUAL
www  IN A IP_MAQUINA_VIRTUAL

; (Opcional) si quieres mantener nombres, que apunten al borde
pc1  IN A IP_MAQUINA_VIRTUAL
pc2  IN A IP_MAQUINA_VIRTUAL
```
**Nota**: Reemplaza "`IP_MAQUINA_VIRTUAL`" con la ip de la maquina virtual

2. Archivos de configuracion del servidor web (Puedes poner otro index)
```bash
mkdir web
touch web/index.html
nano web/index.html
```

`web/index.html`
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hola soy Arturo</title>
    <style>
        body {
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background: linear-gradient(135deg, #6a11cb, #2575fc);
            font-family: "Poppins", sans-serif;
            color: #fff;
            text-align: center;
        }

        h1 {
            font-size: 3em;
            margin: 0;
            animation: fadeIn 2s ease-in-out;
        }

        p {
            font-size: 1.2em;
            opacity: 0.8;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(-20px); }
            to { opacity: 1; transform: translateY(0); }
        }

        footer {
            position: absolute;
            bottom: 10px;
            font-size: 0.9em;
            opacity: 0.7;
        }
    </style>
</head>
<body>
    <h1>¡Hola, soy <span style="color:#ffea00;">TU NOMBRE</span>! 👋</h1>
    <p>Bienvenido a mi servidor web.</p>

    <footer>Hecho con ❤️ y HTML</footer>
</body>
</html>
```

---
### Método Manual 
#### 1. Crear la red bridge con IP fija

```bash
docker network create netlab \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1
```

Verifica:

```bash
docker network inspect netlab | grep Subnet
```

---

#### 2. Crear el contenedor **DNS (bind9)**

```bash
docker run -d \
  --name dns \
  --hostname bind-dns \
  --network netlab \
  --ip 10.10.0.2 \
  -e BIND9_USER=root \
  -e TZ=America/New_York \
  -v ./config:/etc/bind \
  -p 53:53/tcp \
  -p 53:53/udp \
  --restart unless-stopped \
  ubuntu/bind9:latest
```


---

#### 3. Crear el contenedor **PC1** y PC2

```bash
docker run -d \
  --name pc1 \
  --hostname pc1 \
  --network netlab \
  --ip 10.10.0.10 \
  --dns 10.10.0.2 \
  -p 2021:22 \
  --restart unless-stopped \
  ubuntu:custom
```

```bash
docker run -d \
  --name pc2 \
  --hostname pc2 \
  --network netlab \
  --ip 10.10.0.20 \
  --dns 10.10.0.2 \
  -p 2022:22 \
  --restart unless-stopped \
  ubuntu:custom
```

---

#### 4. Crear el contenedor **HTTP (nginx)**

```bash
docker run -d \
  --name httpd \
  --hostname httpd \
  --network netlab \
  --ip 10.10.0.3 \
  -v ./web:/usr/share/nginx/html:ro \
  -p 80:80 \
  --restart unless-stopped \
  nginx:latest
```

---

### Método Automatico con docker compose (Recomendado):

En este metodo creamos todo el lab usando compose (esperando a que funcione)

```bash
touch compose.yml
```

`compose.yml`
```yaml
services: 
  dns:
    container_name: dns
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
      netlab:
        ipv4_address: 10.10.0.2
    restart: unless-stopped

  pc1:
    image: ubuntu:custom
    container_name: pc1
    hostname: pc1
    networks:
      netlab:
        ipv4_address: 10.10.0.10
    dns:
      - 10.10.0.2
    ports:
      - "2021:22"  
    restart: unless-stopped
    depends_on:
      - dns

  pc2:
    image: ubuntu:custom
    container_name: pc2
    hostname: pc2
    networks:
      netlab:
        ipv4_address: 10.10.0.20
    dns:
      - 10.10.0.2
    ports:
      - "2022:22" 
    restart: unless-stopped
    depends_on:
      - dns

  http: 
    image: nginx:latest
    container_name: httpd
    networks:
      netlab:
        ipv4_address: 10.10.0.3
    volumes:
      - ./web/:/usr/share/nginx/html:ro  
    ports:
      - "80:80"        # HTTP externo real
    restart: unless-stopped
    depends_on:
      - dns

networks:
  netlab:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.0.0/24
          gateway: 10.10.0.1
```

Y lo ejecutamos con `docker compose up -d`

---
## Añadir DNS a maquina host:

### En linux: 

Añade la siguiente línea en el archivo `/etc/resolve.conf` :

```bash
# Maquina virtual:
nameserver 10.10.0.2
#Maquina real (Si usas linux):
nameserver IP_MAQUINA_VIRTUAL
```

### En Windows:

**Nota**: Modifica la red en la que te encuentras

![[Pasted image 20251005140027.png]]
![[Pasted image 20251005140102.png]]

---
## Verificación

Lista de contenedores:

```bash
docker ps
```

Comprobación de red:

```bash
docker network inspect netlab | grep IPv4Address
```

---
## Capturas del proceso:

![[Pasted image 20251005133603.png]]
![[Pasted image 20251005134754.png]]
![[Pasted image 20251005133839.png]]

#### Método manual:

![[Pasted image 20251005134932.png]]
![[Pasted image 20251005135024.png]]
![[Pasted image 20251005141139.png]]
![[Pasted image 20251005141315.png]]
