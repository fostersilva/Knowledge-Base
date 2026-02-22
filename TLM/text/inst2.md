# Setup Inicial Ubuntu Touch

* System Settings -> About -> Developer Mode ->  activar "Developer Mode" ()
* (aceitar no tlm "Allow USB Debugging")
* Abrir a Linha de Comandos do Windows e executar os comandos.
```bash
adb shell
sudo mount -o remount,rw /
```

Criar a base para os 106GB
```bash
mkdir -p /home/phablet/full_var
sudo cp -a /var/* /home/phablet/full_var/
sudo mount --bind /home/phablet/full_var /var
```

Acerto inicial para o APT não falhar
```bash
sudo date 022210562026
sudo apt update
sudo apt install ntpdate openssh-server cron curl -y
```

Configurar ssh
```bash
sudo bash -c 'cat <<EOF > /etc/ssh/sshd_config
Port 22
ListenAddress 0.0.0.0
PermitRootLogin prohibit-password
PasswordAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
TCPKeepAlive yes
ClientAliveInterval 60
ClientAliveCountMax 3
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
EOF'
```

Garantir que o ficheiro da hora existe com permissões certas antes do reboot
```bash
touch /home/phablet/last_time.txt
chmod 666 /home/phablet/last_time.txt
```

editar o crontab
```bash
sudo bash -c 'cat <<EOF | crontab -
# 1. Tenta acertar pela NET e guarda no ficheiro (tudo a cada 1 minuto)
* * * * * /usr/sbin/ntpdate -u pool.ntp.org; /bin/date +"\%Y-\%m-\%d \%H:\%M:\%S" > /home/phablet/last_time.txt

# 2. Sincronizar com a internet a cada hora (já que o serviço nativo falhou)
0 * * * * /usr/sbin/ntpdate -u pool.ntp.org

EOF'
```

editar o /etc/rc.local com as scripts de arranque
```bash
sudo bash -c 'cat <<EOF > /etc/rc.local
#!/bin/bash
exec > /home/phablet/rc_local.log 2>&1

# 1. Recuperar hora (Ficheiro > Data Manual > 1970)
if [ -s /home/phablet/last_time.txt ]; then
    date -s "\$(cat /home/phablet/last_time.txt)"
else
    date 022111002026
fi

# 2. Configurações de Sistema
timedatectl set-timezone Europe/Lisbon
mount -o remount,rw /
mountpoint -q /var || mount --bind /home/phablet/full_var /var

# 3. Persistência de Serviços
mkdir -p /run/sshd
chmod 755 /run/sshd
systemctl daemon-reload
service ssh restart
service cron restart

# Garantir que o limite de carga da bateria é ativado no arranque
sudo chmod 644 /sys/class/power_supply/battery/charge_control_limit

# 4. Sincronismo Online (assim que houver rede)
(
    until ping -c 1 8.8.8.8 &>/dev/null; do sleep 5; done
    /usr/sbin/ntpdate -u pool.ntp.org && date +"%Y-%m-%d %H:%M:%S" > /home/phablet/last_time.txt
) &
exit 0
EOF'
```
Dar as permissoes necessarias:
```bash
sudo chmod +x /etc/rc.local
```

Correr o /etc/rc.local:
```bash
sudo /etc/rc.local
```
Checklist Final antes do Reboot:
```bash
df -h /var
date
sudo sshd -t
```

Se der erro WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```bash
ssh-keygen -R 192.168.x.xx
```
reboot:
```bash
sync && sudo reboot -f
```


