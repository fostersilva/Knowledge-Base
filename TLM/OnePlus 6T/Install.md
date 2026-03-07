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
Se o teu objetivo é ter um servidor estável, a regra de ouro no Linux é "se funciona, não mexe"
```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt update
sudo apt autoremove
```

Desativar o "Spam" da Câmara (O passo vital): Antes de removeres as apps, tens de calar o driver da câmara, senão o CPU continuará a 100%
```bash
echo "kernel.printk = 3 4 1 3" | sudo tee -a /etc/sysctl.conf
```
```bash
sudo sysctl -p
```
Remover/Desativar o Chatty: Como vimos, o Chatty tenta processar demasiados eventos e devora a RAM.
```bash
sudo mv /usr/bin/chatty /usr/bin/chatty.bak
```
```bash
sudo killall -9 chatty
```

```bash
sudo apt purge firefox-esr
sudo apt autoremove
```

```bash
sudo reboot -f
```

Configurar a Internet da MEO (Se a lista estiver vazia)
```bash
dbus-send --system --print-reply --dest=org.ofono /ril_0/context1 org.ofono.ConnectionContext.SetProperty string:"Active" variant:boolean:true
ping -I rmnet_data0 -c 4 8.8.8.8
```


O teu SIM está identificado pela pasta 268069666419730
```bash
sudo ls -l /var/lib/ofono/
```

Ficheiro de Estado do SIM
```bash
sudo nano /var/lib/ofono/268069666419730/gprs
```

Apaga tudo o que lá estiver e deixa exatamente assim:
```bash
[Settings]
Powered=true
RoamingAllowed=true

[context1]
Name=Tmn Internet
AccessPointName=internet
Username=tmn
Password=tmn
AuthenticationMethod=any
Type=internet
Protocol=dual
Active=true
```
Trancar o ficheiro (Para o sistema não o apagar)
```bash
sudo chattr +i /var/lib/ofono/268069666419730/gprs
```

O Droidian desliga o hardware no arranque. Este serviço volta a ligá-lo 15 segundos depois de ligares o aparelho.
```bash
sudo nano /etc/systemd/system/ligar-4g.service
```
inserir
```bash
[Unit]
Description=Forçar 4G - Loop de Conexão
After=ofono.service
Wants=ofono.service

[Service]
Type=simple
# O segredo está no "set -e": se um comando falhar, o serviço crasha e o systemd reinicia-o
ExecStart=/bin/sh -c 'set -e; sleep 20; \
dbus-send --system --print-reply --dest=org.ofono /ril_0 org.ofono.Modem.SetProperty string:"Powered" variant:boolean:true; \
sleep 5; \
dbus-send --system --print-reply --dest=org.ofono /ril_0 org.ofono.Modem.SetProperty string:"Online" variant:boolean:true; \
sleep 5; \
dbus-send --system --print-reply --dest=org.ofono /ril_0/context1 org.ofono.ConnectionContext.SetProperty string:"Active" variant:boolean:true'

# Reinicia sempre que falhar (ex: modem ocupado ou SIM não pronto)
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable ligar-4g.service
```

```bash
sudo reboot -f
```
```bash
sudo journalctl -u ligar-4g.service -f
ip addr show rmnet_data0
```

Para teres a certeza que o servidor tem internet sem depender de Wi-Fi, tenta fazer um ping
forçando a interface de rede móvel (rmnet_data0 ou similar):
```bash
ping -I rmnet_data0 -c 4 8.8.8.8
```

