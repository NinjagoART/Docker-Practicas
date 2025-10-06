# Instalacion de un servidor linux en virtualbox

## Requisitos:
- VirtualBox
- [Debian 13 NetInstall](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.1.0-amd64-netinst.iso)

| Caracteristicas  | Valor  |
| ---------------- | ------ |
| Ram              | 2GB    |
| CPU              | 2      |
| Espacio en disco | 30 GB  |
| Interfaz de Red  | Bridge |


Caracteristicas en VirtualBox:

![](20251005110405.png)
![](20251005110454.png)

-----------------------------
## Instalacion de Debian 13:

La instalación de debian será con las siguientes caracteristicas (Cambialas si deseas)

Nota: Si alguna de las opciones no aparece en esta guia, su valor serán por defecto, osea dar "siguiente siguiente aceptar"

| Parametro                | Valor                             |
| ------------------------ | --------------------------------- |
| Nombre de usuario Normal | `Sadmin`                          |
| Contraseña de sadmin     | `Docker`                          |
| Contraseña de Root       | `root`                            |
| Tipo de instalacion      | `Graphical Install`               |
| Nombre de la maquina     | `Servidor`                        |
| Particionado de disco    | `Guiado - utilizar todo el disco` |
| GRUB                     | Si                                |
| Unidad de grub           | `/dev/sda`                        |
#### Programas a instalar
* `SSH server`
* `Utilidades estándar del sistema`
* **Remover todas las demás**
### Contraseña del Usuario sadmin: 

El usuario Sadmin con nombre sadmin:
![image](20251005113658.png)

### Particionado del Disco:

![](20251005113821.png)
![](20251005113838.png)
![](20251005113851.png)
![](20251005113903.png)
![](20251005113915.png)

### Instalación de Paquetes:

La instalacion de los paquetes debe de ser la siguiente:
![](20251005114407.png)

### Grub:
Para Grub, instalalo en la unidad principal, y selecionas el disco `/dev/sda`

![](20251005114552.png) ![](20251005114607.png)


![](20251005114908.png)

# Instalacion de Docker por medio de ssh

### 1. Consultamos la ip de la maquina virtual: 
	ip a

![](20251005115109.png)
### 2. Nos conectamos por medio de ssh al servidor:
	ssh sadmin@x.x.x.x
	
![](20251005115340.png)
### 3. Conseder permisos de sudo al usuario sadmin

[Sudo](https://www.sudo.ws/sudo/) permite al administrador del sistema delegar autoridad para otorgar a ciertos usuarios (o grupos de usuarios) la capacidad de ejecutar órdenes como superusuario _(root)_ u otro mientras proporciona un registro de auditoría de las órdenes y sus argumentos.

```bash 
su -
apt update && apt install -y sudo
usermod -aG sudo sadmin
exit
```

![](20251005115945.png)

Nos desconectamos de ssh y nos volvemos a conectar para que surta efecto los cambios al usuario, una vez heco, usamos `sudo su` para comprobar el uso de sudo:

![](20251005120441.png)

### 4. Instalación de docker

 * [Guia Oficial de docker](https://docs.docker.com/engine/install/debian/)
#### Comandos a usar:

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

![](20251005120522.png)
![](20251005120605.png)
![](20251005120638.png)
![](20251005120710.png)
Para comprobar la instalacion de Docker usamos `docker --version`

![](20251005120820.png)

¡Instalación del Servidor Hecha :D!
