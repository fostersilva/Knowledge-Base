# Instalar LXC

```bash
sudo apt install lxc lxc-templates bridge-utils
```

As maquinas vao ter os ips na gama 192.168.3.0/24
Editar /etc/default/lxc-net
```bash
sudo nano /etc/default/lxc-net
```
Conteudo de /etc/default/lxc-net:
```bash
LXC_BRIDGE="lxcbr0"
LXC_ADDR="192.168.3.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="192.168.3.0/24"
LXC_DHCP_RANGE="192.168.3.2,192.168.3.254"
LXC_DHCP_MAX="253"
```
Editar /etc/rc.local
```bash
sudo nano /etc/rc.local
```
Adicionar a /etc/rc.local
```bash
# Criar a ponte de rede para o LXC
brctl addbr lxcbr0
ifconfig lxcbr0 10.0.3.1 netmask 255.255.255.0 up
sleep 2
lxc-start -n mariadb
```
Criar o monitor do LXC
Ver monitor_geral.md