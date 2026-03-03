# 🚀 OnePlus 6T (fajita) - Droidian Cluster Node Setup

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
 fastboot oem install halium-boot
```

### 4. Entrar na Recovery (TWRP)
```bash
fastboot boot twrp-3.3.1-32-fajita-Pie-mauronofrio.img
```

### 5. Preparação no TWRP
```bash
fastboot boot twrp-3.3.1-32-fajita-Pie-mauronofrio.img
```
No ecrã do telemóvel: **Wipe -> Format Data ->** escreve ```yes```.
Volta ao menu principal: **Advanced -> ADB Sideload**.

### 6. Instalação do Droidian
```bash
fastboot boot twrp-3.3.1-32-fajita-Pie-mauronofrio.img
```

## Fase 3: Configuração do Sistema (Pós-Instalação)

### 7. Acesso Remoto (SSH)
```bash
# No teu PC, se o host mudar:
ssh-keygen -R 192.168.x.xx
ssh droidian@192.168.x.xx
```

### 8. Limpeza de Repositórios e Atualização
Remove os repositórios mortos e configura a fonte oficial Droidian:
Remover fontes obsoletas
```bash
sudo rm /etc/apt/sources.list.d/mobian.sources
sudo rm /etc/apt/sources.list.d/gnome-clocks.list
sudo rm /etc/apt/sources.list.d/gst.list
sudo rm /etc/apt/sources.list.d/phone.list
```
Validar Repositório Droidian (Trusted)
```bash
sudo tee /etc/apt/sources.list.d/droidian.sources <<EOF
Types: deb
URIs: [https://production.repo.droidian.org](https://production.repo.droidian.org)
Suites: bullseye
Components: main
Trusted: yes
EOF
```
Atualizar o sistema
```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt update
sudo apt upgrade -y
```

## Fase 4: Estabilização do Cluster (Alta Disponibilidade)
### 9. Autostart em Energia (Modo Servidor)

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

