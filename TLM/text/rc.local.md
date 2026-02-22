#!/bin/bash

# DAR TEMPO AO SISTEMA (O segredo da estabilidade)
sleep 20

# 1. Remontar o sistema
mount -o remount,rw /

# 2. Montar o Portal dos 106GB
mount --bind /home/phablet/full_var /var

# 3. CRIAR A PASTA QUE FALTA (Resolve o erro da imagem b68e99)
mkdir -p /run/sshd
chmod 0755 /run/sshd

# 4. Lancar o SSH
/usr/sbin/sshd

# Criar a bridge para os contentores
brctl addbr lxcbr0
ifconfig lxcbr0 192.168.3.1 netmask 255.255.255.0 up

# 2. Ativar o reencaminhamento de pacotes (IP Forwarding)
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE

# acertar a hora
ntpdate -u pool.ntp.org

# 3. Pequena pausa para o sistema estabilizar (opcional)
sleep 2

# 3. Partilhar a internet do Wi-Fi (wlan0) com os contentores
# Nota: Verifica se a tua interface de rede é wlan0 com o comando 'ifconfig'
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o lxcbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i lxcbr0 -o wlan0 -j ACCEPT

# 4. Iniciar os servidores em segundo plano
lxc-start -n mariadb.fostersilva.com
lxc-start -n test4test.fostersilva.com
lxc-start -n www.fostersilva.com

# Dar 5 segundos para a rede estabilizar
sleep 5

# Forçar o arranque do Porteiro (Nginx)
systemctl restart nginx

# Garantir que o limite de carga da bateria é ativado no arranque
chmod 644 /sys/class/power_supply/battery/charge_control_limit
echo 1 > /sys/class/power_supply/battery/charge_control_limit

# 4. Mensagem para o Telegram (opcional mas fixe)
/home/phablet/telegram.sh "LXC Bridge ativa. IPs .10, .11 e .100 prontos para decolar!"

# 5. Notificar o dono
/home/phablet/telegram.sh "Reboot concluído. Cluster FosterSilva ONLINE!"

exit 0
