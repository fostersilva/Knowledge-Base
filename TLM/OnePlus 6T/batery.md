## Criar o Serviço
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
```bash
#!/bin/bash

FICHEIRO_USB="/sys/class/power_supply/usb/present"
ESTADO_ANTERIOR=$(cat "$FICHEIRO_USB")

echo "[$(date)] Sentinela USB Ativo. A vigiar o cabo..."

while true; do
    ESTADO_ATUAL=$(cat "$FICHEIRO_USB")

    # Se o estado mudou (algu  m ligou ou desligou o cabo)
    if [ "$ESTADO_ATUAL" != "$ESTADO_ANTERIOR" ]; then
        if [ "$ESTADO_ATUAL" -eq 1 ]; then
            # CABO LIGADO: Reset Zen Imediato
            echo 0 > /sys/class/power_supply/battery/charge_control_limit
            echo 0 > /sys/class/power_supply/battery/input_suspend
            echo "[$(date)]  ^=^t^l CABO DETETADO: Reset Zen aplicado instantaneamente!"
        else
            # CABO DESLIGADO
            echo 0 > /sys/class/power_supply/battery/input_suspend
            echo "[$(date)]  ^z   ^o CABO REMOVIDO: Limpeza efetuada."
        fi
        ESTADO_ANTERIOR=$ESTADO_ATUAL
    fi
    sleep 0.5 # Verifica 2 vezes por segundo (imediato para o humano, leve para o CPU)
done
```
Ativar o serviço
```bash
sudo systemctl daemon-reload
sudo systemctl enable reset-usb-zen.service
sudo systemctl start reset-usb-zen.service
```