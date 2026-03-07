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
```bash
sudo apt upgrade -y
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
Neutralizar o "Batman" (Gestão de Energia)
```bash
sudo apt remove batman
```
```bash
sudo apt purge firefox-esr
sudo apt autoremove
```

 Configurar a Internet da MEO (Se a lista estiver vazia)
```bash
/usr/share/ofono/scripts/create-context /ril_0 internet "internet"
```

Ativar os Dados Móveis
```bash
/usr/share/ofono/scripts/activate-context /ril_0 1
```
Este comando diz ao gestor de ligações para manter a rádio de dados ligada:
```bash
dbus-send --system --print-reply --dest=org.ofono /ril_0 org.ofono.ConnectionManager.SetProperty string:"Powered" variant:boolean:true
```

Para teres a certeza que o servidor tem internet sem depender de Wi-Fi, tenta fazer um ping
forçando a interface de rede móvel (rmnet_data0 ou similar):
```bash
ping -I rmnet_data0 -c 4 8.8.8.8
```

```bash
sudo reboot -f
```