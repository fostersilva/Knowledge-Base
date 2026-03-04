# OnePlus 6T (fajita) - Droidian Cluster Node Setup

Este guia documenta o procedimento de instalação e estabilização do **Droidian (Debian Bullseye)** no OnePlus 6T para utilização como nó de cluster.

---

## Fase 1: Preparação de Hardware & Firmware

### 1. O Alicerce (MSM Tool)
Antes de começar, usa o **MSM Download Tool** para flashar o **OxygenOS 9.x (Android 9)**.
> **Porquê?** Isto garante os drivers de rádio (Modem/Wi-Fi) e firmware corretos para a compatibilidade com a API 28.

### 2. Abrir o Bootloader
```bash
adb reboot bootloader
fastboot oem unlock
```
## Fase 2: Instalação do Sistema (Recovery & Sideload)

### 3. Instalar o Kernel (Halium-Boot)
Este ficheiro faz a ponte entre o hardware OnePlus e o Linux.
 ```bash
fastboot flash boot halium-boot.img
```

### 5. Preparação no TWRP
```bash
fastboot boot twrp-3.3.1-32-fajita-Pie-mauronofrio.img
```
No ecrã do telemóvel: **Wipe -> Format Data ->** escreve ```yes```.
Volta ao menu principal: **Advanced -> ADB Sideload**.

### 6. Instalação do Droidian
```bash
adb sideload droidian-rootfs-api28gsi-arm64_20230603.zip
```

## Fase 3: Configuração do Sistema (Pós-Instalação)

**Acesso Remoto (SSH):**
```bash
# No teu PC, se o host mudar:
ssh-keygen -R 192.168.x.xx
ssh droidian@192.168.x.xx
```

**Impedir que o servidor entre em modo de suspensão (Sleep/Suspend)**
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

**Garantir que o logind não suspende o sistema por inatividade**
Edita o ficheiro de configuração do login:
```bash
sudo nano /etc/systemd/logind.conf
```
Procura (ou adiciona) estas linhas e retira o # da frente:
```bash
IdleAction=ignore
HandleSuspendKey=ignore
HandleLidSwitch=ignore
```

### Localização e Tempo
Para garantir a consistência dos logs e da base de dados num contexto internacional:

```bash
sudo rm /etc/localtime
sudo ln -s /usr/share/zoneinfo/UTC /etc/localtime
date
```

### Limpeza de Repositórios e Atualização
Remove os repositórios mortos e configura a fonte oficial Droidian:
Remover fontes obsoletas
```bash
sudo rm /etc/apt/sources.list.d/mobian.sources
sudo rm /etc/apt/sources.list.d/gnome-clocks.list
sudo rm /etc/apt/sources.list.d/gst.list
sudo rm /etc/apt/sources.list.d/phone.list
sudo rm /etc/apt/sources.list.d/gtk4.list
```
Validar Repositório Droidian (Trusted)
```bash
sudo tee /etc/apt/sources.list.d/droidian.sources <<EOF
Types: deb
URIs: https://production.repo.droidian.org
Suites: bookworm
Components: main
Trusted: yes
EOF
```

Limpeza Final dos "Fantasmas"
```bash
sudo rm -f /etc/apt/sources.list.d/mobian.list /etc/apt/sources.list.d/gnome-clocks.list /etc/apt/sources.list.d/gst.list /etc/apt/sources.list.d/phone.list /etc/apt/sources.list.d/gtk4.list /etc/apt/sources.list.d/libaperture.list /etc/apt/sources.list.d/ofono2mm.list /etc/apt/sources.list.d/callaudiod.list
```

Agora sim, limpa a cache e tenta o update:
```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt update
```
### O Grande Passo: Full Upgrade
Agora que a base do Debian 12 (Bookworm) está sólida e os repositórios estão sincronizados,
vamos atualizar esses 314 pacotes. Como este é um salto de versão e de imagem,
recomendo usares o full-upgrade para garantir que as dependências novas são resolvidas corretamente.
```bash
sudo apt full-upgrade -y
```

```bash
chmod 666 /sys/class/power_supply/battery/charge_control_limit
```
## hh

```bash
#!/bin/bash

# Configurações de Caminhos
LOG_FILE="/home/droidian/bateria_monitor.log"
NOW=$(date +"%Y-%m-%d %H:%M:%S")

# --- PASSO 1: DETEÇÃO DE CABO ---
# Usamos o 'present' para saber se o carregador está lá
USB_PRESENT=$(cat /sys/class/power_supply/usb/present 2>/dev/null || echo 0)

if [ "$USB_PRESENT" -eq 0 ]; then
    # Se tiraste o cabo à mão, limpamos o suspender para quando voltares a ligar
    echo 0 | sudo tee /sys/class/power_supply/battery/input_suspend > /dev/null
    echo "[$NOW] Cabo removido. Reset do input_suspend efetuado." >> "$LOG_FILE"
    exit 0
fi

# --- PASSO 2: LÓGICA DE CLUSTER (80% / 50%) ---
CAPACITY=$(cat /sys/class/power_supply/battery/capacity)
# Lemos o estado atual do suspender (0 = a carregar, 1 = suspenso/discharging)
CURRENT_SUSPEND=$(cat /sys/class/power_supply/battery/input_suspend)

# LIMITE SUPERIOR: Chegou aos 80%, "puxamos a ficha" via software
if [ "$CAPACITY" -ge 80 ]; then
    if [ "$CURRENT_SUSPEND" != "1" ]; then
        echo 1 | sudo tee /sys/class/power_supply/battery/input_suspend > /dev/null
        echo "[$NOW] 🛑 Limite 80% atingido ($CAPACITY%). Entrada de energia SUSPENSA." >> "$LOG_FILE"
        # /home/droidian/telegram.sh "🛑 Nó OnePlus: Carga suspensa aos $CAPACITY%." &
    fi

# LIMITE INFERIOR: Baixou dos 50%, deixamos o gajo "comer"
elif [ "$CAPACITY" -le 50 ]; then
    if [ "$CURRENT_SUSPEND" != "0" ]; then
        echo 0 | sudo tee /sys/class/power_supply/battery/input_suspend > /dev/null
        echo "[$NOW] ⚡ Limite 50% atingido ($CAPACITY%). Entrada de energia REATIVADA." >> "$LOG_FILE"
        # /home/droidian/telegram.sh "⚡ Nó OnePlus: A carregar desde os $CAPACITY%." &
    fi
fi
```





## Fase 4: Estabilização do Cluster (Alta Disponibilidade)
### 9. Autostart em Energia (Modo Servidor)
```bash
fastboot oem off-mode-charge 0
```

Configura o telemóvel para ligar automaticamente quando recebe energia via USB.
EXECUTAR NO PC (Modo Fastboot):
```bash
fastboot oem off-mode-charge 0
```

### 10. Persistência de Erros (Kernel Panic)
Configura o nó para reiniciar sozinho em caso de crash do sistema.

Tornar o Auto-Restart Permanente
```bash
sudo tee /etc/sysctl.d/99-restart-on-panic.conf <<EOF
kernel.panic = 10
kernel.panic_on_oops = 1
EOF
```

Aplicar agora
```bash
sudo sysctl -p /etc/sysctl.d/99-restart-on-panic.conf
```

### Desativar Interface Gráfica (Opcional - Poupança de RAM)
```bash
sudo systemctl disable phosh
sudo systemctl stop phosh
```
if [ "$CAPACITY" -ge 80 ]; then
if [ "$CURRENT_LIMIT" != "4" ]; then
sudo chmod 666 /sys/class/power_supply/battery/charge_control_limit
echo 4 | sudo tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
echo "[$NOW] Limite 80% ($CAPACITY%). Limit definido para 4." >> "$LOG_FILE"
# /home/droidian/telegram.sh "🔋 Carga Bloqueada ($CAPACITY%)." &
sleep 1
fi

elif [ "$CAPACITY" -le 20 ]; then
if [ "$CURRENT_LIMIT" != "0" ]; then
sudo chmod 666 /sys/class/power_supply/battery/charge_control_limit
echo 0 | sudo tee /sys/class/power_supply/battery/charge_control_limit > /dev/null
echo "[$NOW] Bateria 20% ($CAPACITY%). Limit definido para 0." >> "$LOG_FILE"
# /home/droidian/telegram.sh "🪫 Carga Libertada ($CAPACITY%)." &
sleep 1
fi
fi
