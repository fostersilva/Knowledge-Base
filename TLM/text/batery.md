# Monitor de Bateria e Alertas Telegram (Ubuntu Touch)

Este conjunto de scripts monitoriza a saúde da bateria do **OnePlus 6T** e envia alertas via Telegram caso o cabo seja desligado ou a voltagem baixe.

## 1. Configuração do Bot Telegram
* Cria o ficheiro `~/telegram.sh` para gerir as notificações:

```bash
nano ~/telegram.sh
```
Copia e ccola o conteúdo abaixo para dentro do ficheiro:

```bash
#!/bin/bash
TOKEN="8551512704:AAHH99VMNDCDVYnVWW2boHAo031cJaK0YYI"
ID="7395856647"
MSG=$1

if [ -z "$MSG" ]; then
  MSG="Alerta: Mensagem vazia enviada pelo servidor."
fi

curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
     -d "chat_id=$ID" \
     -d "text=$MSG" > /dev/null
```

Após guardar o ficheiro, define as permissões necessárias:
```bash
chown phablet:phablet /home/phablet/telegram.sh
chmod 700 /home/phablet/telegram.sh
```

## 2. Configuração gestao energia 20% 80%
Cria o ficheiro `~/check_battery.sh` para gerir o carregamento da bateria:
```bash
nano /home/phablet/check_battery.sh
```
Conteudo de `~/check_battery.sh`:
```bash
#!/bin/bash

# --- PASSO 1: VERIFICAÇÃO FÍSICA ---
HW_DETECT=$(cat /sys/class/power_supply/usb/hw_detect)
if [ "$HW_DETECT" -eq 0 ]; then
    /home/phablet/telegram.sh "🔌 CABO DESLIGADO! O cabo saltou do telemóvel." &
    exit 0
fi

# --- PASSO 2: VERIFICAÇÃO DE CORRENTE ---
VOLTAGEM=$(cat /sys/class/power_supply/usb/voltage_now)
if [ "$VOLTAGEM" -lt 4000 ]; then
    /home/phablet/telegram.sh "⚠️ TOMADA OFF! Cabo ligado mas sem energia ou voltagem muito baixa (Valor: $VOLTAGEM mV)." &
    exit 0
fi

# --- PASSO 3: LÓGICA 20% / 80% COM TRAVÃO DE SPAM ---
CAPACITY=$(cat /sys/class/power_supply/battery/capacity)
# Lemos o estado atual do limite para não repetir comandos nem mensagens
CURRENT_LIMIT=$(cat /sys/class/power_supply/battery/charge_control_limit)

if [ "$CAPACITY" -ge 80 ] && [ "$CURRENT_LIMIT" != "4" ]; then
    # Só entra aqui se estiver >= 80% E o limite ainda não for 4
    echo 4 | sudo tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
    /home/phablet/telegram.sh "Status: Limite de 80% atingido ($CAPACITY%). Carga Bloqueada." &
    echo "Status: Limite de 80% atingido. Bloqueado."

elif [ "$CAPACITY" -le 20 ] && [ "$CURRENT_LIMIT" != "0" ]; then
    # Só entra aqui se estiver <= 20% E o limite ainda não for 0
    echo 0 | sudo tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
    /home/phablet/telegram.sh "Status: Bateria em 20% ($CAPACITY%). Carga Libertada." &
    echo "Status: Bateria em 20%. Libertado."
fi
```
atribui as  permissoes necessarias:
```bash
chmod +x /home/phablet/check_battery.sh
```
agenda o serviço no cron
```bash
sudo crontab -e
*/5 * * * * /bin/bash /home/phablet/check_battery.sh
```

# Outros comandos
## Repor os valores

* desligar o cabo e executar
```
# 1. Repor o limite de carga para o padrão (0 = sem limite)
echo 0 | sudo tee /sys/class/power_supply/battery/charge_control_limit

# 2. Garantir que a suspensão de entrada está desligada (0 = a receber energia)
echo 0 | sudo tee /sys/class/power_supply/battery/input_suspend

# 3. Verificar se a voltagem voltou ao normal
cat /sys/class/power_supply/usb/voltage_now
```

```bash
if [ $(cat /sys/class/power_supply/battery/capacity) -ge 80 ]; then
    echo 4 | sudo tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
fi


sudo bash -c "echo 1 > /sys/class/leds/led:switch_0/brightness"

while true; do 
  t=$(cat /sys/class/power_supply/battery/temp); 
  c=$(cat /sys/class/power_supply/battery/capacity); 
  u=$(cat /sys/class/power_supply/usb/online); 
  p=$(cat /sys/class/power_supply/pc_port/online); 
  s=$(cat /sys/class/power_supply/battery/status);
  h=$(date +"%H:%M:%S"); 
  
  # Lógica de discriminação da fonte
  if [ "$p" -eq 1 ]; then 
    fonte="PC (USB)"; 
  elif [ "$u" -eq 1 ]; then 
    fonte="TOMADA (AC)"; 
  else 
    fonte="DESLIGADO"; 
  fi
  
  echo -n "[$h] Bat: $c% | Status: $s | Fonte: $fonte | Temp: "; expr $t / 10; 
  sleep 5; 
done
```

