Reset	Voltar à fábrica	MSM Download Tool
Unlock	Libertar o hardware	fastboot flashing unlock
Install	Ubuntu Touch 20.04	UBports Installer (Stable)
Espaço	O Portal de 106GB	mount --bind /home/phablet/full_var /var
Limpeza	Salvar a Raiz (Root)	rm -rf /usr/share/man/*
Auto	Persistência total	rc.local + sleep 20
Energia	Imortalidade	off-mode-charge 0


1.Ferramentas
1.1. Qualcom Drivers
1.2. ADB FastBoot (Latest-adb-fastboot-installer-for-windows - https://github.com/fawazahmed0/Latest-adb-fastboot-installer-for-windows)
1.3. SDK Platform-Tools (que contêm o adb e o fastboot https://developer.android.com/tools/releases/platform-tools)
1.3. UBports Installer

2. O Reset Total (MSM Download Tool): Objetivo: Voltar à estaca zero.
2.1 Coloca o telemóvel em EDL Mode (Desliga-o e mantém premidos Volume Up + Volume Down enquanto ligas o cabo USB ao PC).
2.2 No PC (gestor de dispositivos), o dispositivo deve aparecer como Qualcomm HS-USB QDLoader 9008
2.3 Corre a ferramenta, clica em Start e espera pelo "Download Complete".

3. Preparação para o Ubuntu Touch (Depois de reviveres o telemóvel com a MSM)
3.1. Faz o setup inicial básico do Android (podes saltar tudo, não precisas de conta Google).
3.2. Ativa as Opções de Programador (Settings -> About Phone -> Build Number (Carregar 7 vezes seguidas para ativar o Developer Mode))
3.3. Ativa o OEM Unlocking e o USB Debugging (Settings -> System -> Developer Options -> Stay awake, OEM Unlocking e USB Debugging).
3.3.1. Desbloqueia o Bootloader
3.3.1.1. Instala e inicia a ferramenta ADB FastBoot (Latest ADB Launcher.bat)
3.3.1.2. # adb devices
3.3.1.3. # adb reboot bootloader (este comando reinicia o tlm e manda-o para o "Flasboot Mode")
3.3.1.3. # fastboot devices (Com o Tlm em Flasboot Mode lançar este comando no ADB FastBoot)
3.3.1.4. # fastboot flashing unlock
3.3.1.5. No telemovel: UNLOCK THE BOOTLOADER (esta acção vai instalar novamente o software com o bootloader desbloqueado)
3.3.1.6. Depois de o Tlm pronto com o software instalado repetir os passos de 3.1. a 3.3

4. Instalação do Ubuntu Touch
4.1. Liga o cabo ao PC e Lança o UBports Installer
4.2. Na fase "Install Options" selecionar "20.04/stable" (recomendado), "Wipe User data" e "Bootstrap" e clickar em ok e seguir as instruçoes do instalador

5. Preparação inicial
5.1 Os proximos passos executam-se das seguinte formas alternativas
5.1.1. No próprio telemóvel: Abre a gaveta de aplicações (arrasta da esquerda para a direita) e procura pela aplicação "Terminal".
5.1.2. Via Cabo USB (ADB): Se o telemóvel estiver ligado ao PC com o USB Debugging ativo (ligar e desligar o cabo se necessario, para ser detetado pelo Ubunt Touch), podes usar o comando adb shell no terminal do teu computador. É mais fácil porque podes copiar e colar os comandos.
5.1.2.1. System Settings -> About -> Developer Mode ->  activar "Developer Mode" () (aceitar no tlm "Allow USB Debugging")
5.1.2.2. Abrir a Linha de Comandos do Windows e executar os comandos.
5.1.2.2.1. # adb shell
5.1.2.2.2. # sudo mount -o remount,rw /
5.1.2.2.2.1. O "Portal" dos 106GB (Fundamental!)
5.1.2.2.2.2. # mkdir -p /home/phablet/full_var (Criar a pasta na partição de utilizador)
5.1.2.2.2.3. # sudo cp -a /var/* /home/phablet/full_var/ (Mover o que existe no /var atual para lá (para não perderes as bases de dados do sistema))
5.1.2.2.2.4. # sudo mount --bind /home/phablet/full_var /var (Montar o "Portal")

# acerta a hora a mao mete a data "à mão" via SSH (usa o formato MMDDhhmmYYYY):
sudo date 021823352026

# atualiza os pacotes e instala o tempo
sudo apt update && sudo apt install ntpdate -y

5.1.2.2.3.1. Limpeza de Segurança (Libertar a Raiz)
5.1.2.2.3.2. # sudo rm -rf /usr/share/man/* /usr/share/doc/* /usr/share/locale/*
# Apaga logs antigos e comprimidos
sudo rm -f /var/log/*.gz
sudo rm -f /var/log/*.1

# Apaga traduções de línguas (ex: Chinês, Russo, etc.)
sudo rm -rf /usr/share/locale/*

# Apaga wallpapers e ícones pesados
sudo rm -rf /usr/share/backgrounds/*
sudo rm -rf /usr/share/icons/Adwaita
sudo rm -rf /usr/share/icons/suru

# Eliminar Documentação e Manuais 
sudo rm -rf /usr/share/man/*
sudo rm -rf /usr/share/doc/*

# Eliminar Temas e Ícones Adicionais
sudo rm -rf /usr/share/icons/Humanity
sudo rm -rf /usr/share/icons/ubuntu-mobile
sudo rm -rf /usr/share/icons/gnome

#Eliminar Sons de Sistema e Teclado
sudo rm -rf /usr/share/sounds/*

#Eliminar Bases de Dados de Localização
sudo rm -rf /usr/share/geoclue*
sudo rm -rf /usr/share/zoneinfo/Etc

#Remover Dicionários e Verificadores Ortográficos
sudo rm -rf /usr/share/dict/*
sudo rm -rf /usr/share/hunspell/*
sudo rm -rf /usr/share/myspell/*

# Limpar a Cache de Ícones do Sistema
sudo rm -rf /var/cache/fontconfig/*

# 2. Instalar o servidor (Garante que tens net no tlm!)
sudo apt update
sudo apt install openssh-server -y

# 3. Permitir o acesso por password
# Edita o ficheiro de configuração
sudo nano /etc/ssh/sshd_config
Dentro do editor nano:
Garante que estas linhas estão assim (sem o # à frente):
Port 22
ListenAddress 0.0.0.0
PermitRootLogin yes
PasswordAuthentication yes
TCPKeepAlive yes

Reiniciar o SSH
# sudo service ssh restart

o servidor SSH por vezes ignora o sshd_config principal se houver ficheiros na pasta .d
apagar (ou renomear) o ficheiro:
sudo rm /etc/ssh/sshd_config.d/usb-debug.conf

Ligar remotamente ao ssh
# ssh phablet@192.168.1.95

Se der erro WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
ssh-keygen -R 192.168.x.xx
ssh phablet@192.168.1.95

Arranque automatico
sudo nano /etc/rc.local
Colar o Script de Ativação
Copia e cola exatamente este conteúdo. Ele garante que o espaço de 106GB é montado primeiro e que o SSH arranca logo a seguir:
# sudo nano /etc/rc.local
========================================================================
#!/bin/bash

# DAR TEMPO AO SISTEMA (O segredo da estabilidade)
sleep 20

#!/bin/bash
# 1. Tenta forçar a hora logo no início
/usr/sbin/ntpdate -u pool.ntp.org

# 2. Garante que o servico continuo esta a correr
systemctl restart systemd-timesyncd

# 1. Remontar o sistema
sudo mount -o remount,rw /

# 2. Montar o Portal dos 106GB
sudo mount --bind /home/phablet/full_var /var

# 3. CRIAR A PASTA QUE FALTA (Resolve o erro da imagem b68e99)
mkdir -p /run/sshd
chmod 0755 /run/sshd

# 4. Lançar o SSH
/usr/sbin/sshd

exit 0
========================================================================

Dar Permissões de Execução
sudo chmod +x /etc/rc.local
sudo systemctl enable rc-local
sudo systemctl start rc-local
sudo /etc/rc.local

Resiliência de Energia: Para garantir que o servidor liga sozinho após uma falha de energia, configurar o off-mode-charge via Fastboot.
Entrar em modo Fastboot
# adb reboot bootloader
# fastboot oem off-mode-charge 0

# Se precisares de reiniciar via terminal, usa para evitar que ele fique preso no ecrã preto
sudo reboot -f 

#######################################################################
7. Comandos de Ouro (Manutenção)
Reiniciar: sudo reboot -f (Evita bloqueios no ecrã preto).

Desligar: sudo poweroff -f.

Auto-Boot (Energia): No modo Fastboot: fastboot oem off-mode-charge 0.

Erro SSH no Windows: ssh-keygen -R 192.168.1.95.




======================================================================
# a ferramenta de tempo
sudo apt update
sudo apt install ntpdate -y
sudo ntpdate -u pool.ntp.org
sudo timedatectl set-timezone Europe/Lisbon

# precisamos de abrir a orta 80 para o mundo


LXC
# Antes de criares as máquinas, vamos ver se o teu sistema está pronto para o LXC:
# Instala as ferramentas:
sudo apt update
sudo apt install lxc lxc-templates bridge-utils -y

# O Teste de Stress do Kernel: Executa este comando para ver se o kernel do OnePlus suporta tudo o que o LXC precisa:
lxc-checkconfig

# Mudar o caminho do LXC para o Portal
# Por defeito, o LXC guarda as máquinas em /var/lib/lxc. Como o teu /var
# já está montado nos 106GB, estás safo! As tuas máquinas vão ter espaço
# para crescer à vontade

sudo lxc-create -n mariadb.fostersilva.com -t download -- -d ubuntu -r jammy -a arm64
sudo lxc-create -n test4test.fostersilva.com -t download -- -d ubuntu -r jammy -a arm64
sudo lxc-create -n www.fostersilva.com -t download -- -d ubuntu -r jammy -a arm64

sudo lxc-ls -f

# Criar a Nova Ponte (Gama 10.0.3.x)
sudo brctl addbr lxcbr0
sudo ifconfig lxcbr0 10.0.3.1 netmask 255.255.255.0 up

#Criar a "Autoestrada" (Bridge)
sudo nano /var/lib/lxc/www.fostersilva.com/config
lxc.net.0.ipv4.address = 192.168.3.10/24
lxc.net.0.ipv4.gateway = 192.168.3.1

sudo nano /etc/rc.local
-----------------------------------------------------------------------
#!/bin/bash

# DAR TEMPO AO SISTEMA (O segredo da estabilidade)
sleep 20

# 1. Remontar o sistema
mount -o remount,rw /

# 2. Montar o Portal dos 106GB
mount --bind /home/phablet/full_var /var

# 3. CRIAR A PASTA QUE FALTA (Resolve o erro da imagem b68e99)
mkdir -p /run/sshd
chmod 0755 /run/sshd

# 4. Lancar o SSH
/usr/sbin/sshd

# Criar a bridge para os contentores
brctl addbr lxcbr0
ifconfig lxcbr0 192.168.3.1 netmask 255.255.255.0 up

# 2. Ativar o reencaminhamento de pacotes (IP Forwarding)
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE

# acertar a hora
ntpdate -u pool.ntp.org

# 3. Pequena pausa para o sistema estabilizar (opcional)
sleep 2

# 3. Partilhar a internet do Wi-Fi (wlan0) com os contentores
# Nota: Verifica se a tua interface de rede é wlan0 com o comando 'ifconfig'
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o lxcbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i lxcbr0 -o wlan0 -j ACCEPT

# 4. Iniciar os servidores em segundo plano
lxc-start -n mariadb.fostersilva.com
lxc-start -n test4test.fostersilva.com
lxc-start -n www.fostersilva.com

# Dar 5 segundos para a rede estabilizar
sleep 5

# Forçar o arranque do Porteiro (Nginx)
systemctl restart nginx

# 4. Mensagem para o Telegram (opcional mas fixe)
/home/phablet/telegram.sh "LXC Bridge ativa. IPs .10, .11 e .100 prontos para decolar!"

# 5. Notificar o dono
/home/phablet/telegram.sh "Reboot concluído. Cluster FosterSilva ONLINE!"

exit 0
-----------------------------------------------------------------------

comandos no terminal do OnePlus para forçar a zona horária de Portugal dentro do MariaDB:

sudo lxc-stop -n mariadb.fostersilva.com
sudo lxc-start -n mariadb.fostersilva.com

echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" > /etc/resolv.conf
