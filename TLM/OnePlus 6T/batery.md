## Criar o Serviço
```bash
chmod 666 /sys/class/power_supply/battery/charge_control_limit
```

Copia e cola isto tudo. Ele vai criar o serviço apontando exatamente para a lógica que tu tinhas:
```bash
sudo bash -c 'cat <<EOF > /etc/systemd/system/reset-usb-zen.service
[Unit]
Description=Reset Zen Imediato ao detetar USB
After=multi-user.target

[Service]
Type=simple
ExecStart=/bin/bash /home/droidian/cluster_monitor_events.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF'
```
Abre o ficheiro:
```bash
nano /home/droidian/cluster_monitor_events.sh
```
adiciona:

## V03
```bash
#!/bin/bash

# FORÇAR PERMISSÕES NO ARRANQUE (O SEGREDO)
sudo chmod 666 /sys/class/power_supply/battery/charge_control_limit
sudo chmod 666 /sys/class/power_supply/battery/input_suspend 

FICHEIRO_USB="/sys/class/power_supply/usb/present"
FICHEIRO_BAT="/sys/class/power_supply/battery/capacity"
ESTADO_ANTERIOR=$(cat "$FICHEIRO_USB")

echo "[$(date)] Sentinela USB Ativo. Foco: Reset Total (0) ao detetar cabo."

while true; do
    ESTADO_ATUAL=$(cat "$FICHEIRO_USB")
    NIVEL_BAT=$(cat "$FICHEIRO_BAT")

    if [ "$ESTADO_ATUAL" != "$ESTADO_ANTERIOR" ]; then
        # RESET TOTAL: Não importa se liga ou desliga, limpa tudo para 0
        echo 0 > /sys/class/power_supply/battery/charge_control_limit
        echo 0 > /sys/class/power_supply/battery/input_suspend
        
        ESTADO_ANTERIOR=$ESTADO_ATUAL
    fi
    sleep 0.5
done
```

Ativar o serviço
```bash
sudo systemctl daemon-reload
sudo systemctl enable reset-usb-zen.service
sudo systemctl start reset-usb-zen.service
```

```bash
nano /home/droidian/battery_monitor.sh
```
```bash
#!/bin/bash

# Caminhos diretos
F_BAT="/sys/class/power_supply/battery/capacity"
F_LIM="/sys/class/power_supply/battery/charge_control_limit"
F_STA="/sys/class/power_supply/battery/status"
F_LOG="/home/droidian/gestor_bateria.log"

CAPACITY=$(cat "$F_BAT")
STATUS=$(cat "$F_STA")

# LÓGICA DE PRESERVAÇÃO (Se >= 80%)
if [ "$CAPACITY" -ge 80 ]; then
    # Se o Status disser "Charging", o hardware ignorou o limite 4.
    if [ "$STATUS" == "Charging" ]; then
        echo "[$(date)] Bat: $CAPACITY% | ALERTA: Hardware ignorou bloqueio. A forçar ciclo 0 -> 4..." >> "$F_LOG"
        
        # O Ciclo de Choque
        echo 0 > "$F_LIM"
        sleep 2
        echo 4 > "$F_LIM"
        
        echo "[$(date)] Ciclo 0->4 concluído. Status atual: $(cat $F_STA)" >> "$F_LOG"
    fi

# LÓGICA DE RECARGA (Se <= 20%)
elif [ "$CAPACITY" -le 20 ]; then
    CURRENT_LIMIT=$(cat "$F_LIM")
    if [ "$CURRENT_LIMIT" != "0" ]; then
        echo "[$(date)] Bat: $CAPACITY% | Limite 20% atingido. Reset para 0 (Carga Total)." >> "$F_LOG"
        echo 0 > "$F_LIM"
    fi
fi
```

Atribuir as permissoes necessarias

```bash
chmod +x /home/droidian/battery_monitor.sh
```

agenda o serviço no cron
```bash
sudo crontab -e
* * * * * /bin/bash /home/droidian/battery_monitor.sh
```

