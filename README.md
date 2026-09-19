Asegurando un servidor Linux en 5 minutos
==

**Nota:** Se tomará como ejemplo, un Ubuntu `24.04`, aunque aplica a `26.04` y a Debian 13 también.

1. Desactivar acceso con el usuario root

**Nota**: Ubuntu justo cuando instala, le permite a la persona que lo va a usar, crear un primer usuario. Este primer usuario tiene permisos `sudo`. Acá se asume que esto no sucedió, para demostrar el procedimiento. En Debían se debe entrar con `su -`.

##### Añadiendo el usuario

```bash
# Nos hacemos admin temporalmente
sudo su

# Si es Debian
su -
apt install -y sudo

#Añadimos el usuario
adduser sysadmin

# Para añadirlo al grupo sudo
usermod -aG sudo sysadmin

# Evitando pedir password cada vez que hagamos sudo
echo "sysadmin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# Y salimos de root
exit
```

##### Eliminando el acceso root:

**Nota**: Esto se ejecutará en el servidor, en este ejemplo `192.168.20.15`

```bash
# Eliminar acceso de usuario root
sudo passwd -l root

# Si necesitamos habilitar root de nuevo por alguna razón
sudo passwd -u root
```

Y a partir de este momento el usuario `sysadmin` será `sudo` y podra ejecutar comandos como root.   

2. Autentificación usando llaves ssh

##### En la máquina del administrador:

```bash
# Se copiará la llave pública del administrador, nos pedirá la contraseña del servidor 192.168.20.15. Aceptar y listo.
ssh-copy-id sysadmin@192.168.20.15
```

##### En el servidor

```bash
# Editamos la configuración
nano /etc/ssh/sshd_config

# Comentar la siguiente línea si existe en la configuración
Include /etc/ssh/sshd_config.d/*.conf

# La línea siguiente, descomentarla y ponerla a no
PermitRootLogin prohibit-password

# La línea siguiente, descomentarla solamente 
PubkeyAuthentication yes

# La línea siguiente, descomentarla y ponerla en no
PasswordAuthentication yes
```

Guardar y salir. ¿Qué se hizo? Habilitamos acceso por llave únicamente, deshabilitamos el login del usuario root y evitamos que alguien pueda conectarse usando contraseña. Ahora debemos reiniciar SSH:

```bash
systemctl restart ssh
```

Y listo, habilitamos el acceso por llaves.

3. Firewall

##### Instalamos el firewall

```bash
sudo apt install -y iptables-persistent

# Lo habilitamos para que inicie con el sistema
sudo systemctl enable netfilter-persistent

# Salvamos la configuración base
sudo mv /etc/iptables/rules.v4{,.orig}
```

Y la configuración, la ajustamos a nuestras necesidades:
```bash
nano /etc/iptables/rules.v4
```

Dejándola de la siguiente forma:

```text
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -s 192.168.20.0/24 -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
COMMIT
```

Explicación. Drop a todas las conexiones INPUT/FORWARD y habilitado el OUTPUT:
```
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
```

**Detalles:**
> INPUT: Cadena de entrada de conexión/paquetes de la red
> 
> OUTPUT: Salida de paquetes
> 
> FORWARD: Reenvío de paquetes

Tráfico local, habilitado:

```bash
-A INPUT -i lo -j ACCEPT
```

SSH sólo desde la lan:

```bash
-A INPUT -s 192.168.20.0/24 -p tcp -m tcp --dport 22 -j ACCEPT
```

Descartar conexiones que el kernel considera inválidas. Aceptar paquetes que pertenecen a conexiones ya establecidas o relacionadas.

```bash
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Permitir tráfico TCP entrante hacia los puertos 80 y 443, tanto para conexiones nuevas como para conexiones ya establecidas:

```bash
-A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
```

Una vez salvadas las reglas, reiniciamos el firewall:

```bash
sudo systemctl restart netfilter-persistent
```

Y para ver las reglas aplicadas:

```bash
iptables -n -L -v
iptables -S
```

4. Fail2ban

##### Instalando Fail2Ban

```bash
sudo apt install -y fail2ban

# Salvamos la configuración
sudo cp /etc/fail2ban/fail2ban.{conf,local}
sudo cp /etc/fail2ban/jail.{conf,local}
```

Ajustamos un poco:

```bash
sudo nano /etc/fail2ban/jail.local
# por default se banea 60 minutos la ip que intenta atacarnos, si este valor se pone a -1, sera baneado para siempre.
bantime  = 60m

backend = auto
```

Ahora, creamos la configuración a aplicar a un servicio, es como enjaular el servicio para protegerlo, en este caso SSH:

```bash
cat << EOF | sudo tee /etc/fail2ban/jail.d/sshd.local > /dev/null
[sshd]
enabled = true
filter = sshd
port    = ssh
logpath = %(sshd_log)s
backend = auto
EOF
```

Y listo. Si deseamos chequear el status de Fail2Ban o del servicio:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

5. Actualizaciones periódicas

##### Instalamos unattended-upgrades

```bash
sudo apt install -y unattended-upgrades
```

Reconfiguramos:

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Verifica el archivo de configuración de unattended-upgrades con tu editor de texto. Por default no debemos tocar nada, pero por si se necesita:

```bash
nano /etc/apt/apt.conf.d/20auto-upgrades
```

Para deshabilitar los reinicios automáticos provocados por la configuración de actualizaciones automáticas, edite el siguiente archivo:

```bash
nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Y descomenta la siguiente línea eliminando los slash iniciales:

```text
//Unattended-Upgrade::Automatic-Reboot "false";
```
