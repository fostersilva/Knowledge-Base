# Monitor geral do sistema

```bash
nano /home/phablet/monitor_geral.sh
```

```bash
#!/bin/bash

# --- PASSO 0: VIGILANTE DA REDE ---
if ! ip addr show lxcbr0 > /dev/null 2>&1; then
    brctl addbr lxcbr0
    ifconfig lxcbr0 10.0.3.1 netmask 255.255.255.0 up
    sleep 1
fi

# --- PASSO 1: VIGILANTE MULTI-LXC ---
# Obtém a lista de todos os contentores que deveriam arrancar sozinhos (AUTOSTART=1)
mapfile -t CONTAINERS < <(lxc-ls -f | awk '$3 == "1" {print $1}')

for CT in "${CONTAINERS[@]}"; do
    STATUS=$(lxc-info -n "$CT" -s | awk '{print $2}')
    if [ "$STATUS" != "RUNNING" ]; then
        lxc-start -n "$CT"
        /home/phablet/telegram.sh "⚠️ Contentor [$CT] estava em baixo e foi reiniciado." &
    fi
done

# --- PASSO 2: VERIFICAÇÃO FÍSICA E ENERGIA ---
HW_DETECT=$(cat /sys/class/power_supply/usb/hw_detect)
VOLTAGEM=$(cat /sys/class/power_supply/usb/voltage_now)

if [ "$HW_DETECT" -eq 0 ]; then
    /home/phablet/telegram.sh "🔌 CABO DESLIGADO! O cabo saltou." &
    exit 0
fi

if [ "$VOLTAGEM" -lt 4000 ]; then
    /home/phablet/telegram.sh "⚠️ TOMADA OFF! Sem energia ($VOLTAGEM mV)." &
    exit 0
fi

# --- PASSO 3: GESTÃO 20/80 ---
CAPACITY=$(cat /sys/class/power_supply/battery/capacity)
CURRENT_LIMIT=$(cat /sys/class/power_supply/battery/charge_control_limit)

if [ "$CAPACITY" -ge 80 ] && [ "$CURRENT_LIMIT" != "4" ]; then
    echo 4 | tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
    /home/phablet/telegram.sh "🔋 80% atingidos. Modo preservação OK." &
elif [ "$CAPACITY" -le 20 ] && [ "$CURRENT_LIMIT" != "0" ]; then
    echo 0 | tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
    /home/phablet/telegram.sh "🪫 20% atingidos. A carregar..." &
fi
```

Atribuir as permissoes necessarias

```bash
chmod +x /home/phablet/monitor_geral.sh
```

agenda o serviço no cron
```bash
sudo crontab -e
* * * * * /bin/bash /home/phablet/monitor_geral.sh
```