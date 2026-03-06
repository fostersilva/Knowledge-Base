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

# --- GESTÃO 20/80 ---
CAPACITY=$(cat /sys/class/power_supply/battery/capacity)
CURRENT_LIMIT=$(cat /sys/class/power_supply/battery/charge_control_limit)

if [ "$CAPACITY" -ge 80 ] && [ "$CURRENT_LIMIT" != "4" ]; then
    echo 4 | tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
    # /home/phablet/telegram.sh "🔋 80% atingidos. Modo preservação OK." &
elif [ "$CAPACITY" -le 20 ] && [ "$CURRENT_LIMIT" != "0" ]; then
    echo 0 | tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
    # /home/phablet/telegram.sh "🪫 20% atingidos. A carregar..." &
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




## V01

```bash
#!/bin/bash

## V02
```bash
#!/bin/bash

FICHEIRO_USB="/sys/class/power_supply/usb/present"
FICHEIRO_BAT="/sys/class/power_supply/battery/capacity"
ESTADO_ANTERIOR=$(cat "$FICHEIRO_USB")

echo "[$(date)] Sentinela USB Ativo. A vigiar o cabo..."
while true; do
    ESTADO_ATUAL=$(cat "$FICHEIRO_USB")
    NIVEL_BAT=$(cat "$FICHEIRO_BAT")

    # Se o estado mudou (alguém ligou ou desligou o cabo)
    if [ "$ESTADO_ATUAL" != "$ESTADO_ANTERIOR" ]; then
        if [ "$ESTADO_ATUAL" -eq 1 ]; then
            # CABO LIGADO: Primeiro faz o RESET total
            echo 0 > /sys/class/power_supply/battery/charge_control_limit
            echo 0 > /sys/class/power_supply/battery/input_suspend
            
            # Depois verifica se deve bloquear
            if [ "$NIVEL_BAT" -ge 80 ]; then
                echo 4 > /sys/class/power_supply/battery/charge_control_limit
                echo "[$(date)] Bat: $NIVEL_BAT% | CABO DETETADO: Reset feito e Bloqueio Zen (80%+) aplicado!"
            else
                echo "[$(date)] Bat: $NIVEL_BAT% | CABO DETETADO: Reset Zen aplicado (Carga Normal)!"
            fi
        else
            # CABO DESLIGADO
            echo 0 > /sys/class/power_supply/battery/input_suspend
            echo "[$(date)] Bat: $NIVEL_BAT% | CABO REMOVIDO: Limpeza efetuada."
        fi
        ESTADO_ANTERIOR=$ESTADO_ATUAL
    fi
    sleep 0.5
done
```
## V01

```bash
#!/bin/bash

FICHEIRO_USB="/sys/class/power_supply/usb/present"
FICHEIRO_BAT="/sys/class/power_supply/battery/capacity"
ESTADO_ANTERIOR=$(cat "$FICHEIRO_USB")

echo "[$(date)] Sentinela USB Ativo. A vigiar o cabo..."

while true; do
    ESTADO_ATUAL=$(cat "$FICHEIRO_USB")
    NIVEL_BAT=$(cat "$FICHEIRO_BAT")

    # Se o estado mudou (alguém ligou ou desligou o cabo)
    if [ "$ESTADO_ATUAL" != "$ESTADO_ANTERIOR" ]; then
        if [ "$ESTADO_ATUAL" -eq 1 ]; then
            # CABO LIGADO: Reset Zen Imediato
            echo 0 > /sys/class/power_supply/battery/charge_control_limit
            echo 0 > /sys/class/power_supply/battery/input_suspend
            echo "[$(date)] Bat: $NIVEL_BAT% | CABO DETETADO: Reset Zen aplicado!"
        else
            # CABO DESLIGADO
            echo 0 > /sys/class/power_supply/battery/charge_control_limit
            echo 0 > /sys/class/power_supply/battery/input_suspend
            echo "[$(date)] Bat: $NIVEL_BAT% | CABO REMOVIDO: Limpeza efetuada."
        fi
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

Para apagares o serviço da memória do Systemd (fazer com que ele "esqueça" que o serviço existe ou que falhou), o comando é:

Bash
sudo systemctl reset-failed reset-usb-zen.service
O que isto faz:
Limpa o estado "failed" ou "active" da memória RAM do Systemd.

Remove os logs de erro temporários que ficam no status.

Libera o nome do serviço para que, se alteraste o ficheiro, ele seja lido do zero.

Se queres que ele desapareça da lista de serviços ativos de vez:

sudo systemctl stop reset-usb-zen.service (para o parar).

sudo systemctl daemon-reload (para o Systemd ler que o ficheiro mudou ou foi removido).

sudo systemctl reset-failed (para limpar o "lixo" da memória).