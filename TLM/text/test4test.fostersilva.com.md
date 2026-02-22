sudo lxc-attach -n www.fostersilva.com
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" > /etc/resolv.conf
apt update

# PHP 8
apt install software-properties-common -y

# Adicionar o repositório do PHP (o mais fiável para Debian/Ubuntu)
add-apt-repository ppa:ondrej/php -y
apt update

# Instalar o PHP 8.4 e os módulos que o Joomla exige
apt install php8.4 php8.4-mysql php8.4-xml php8.4-gd php8.4-mbstring php8.4-zip php8.4-curl php8.4-intl php8.4-bcmath php8.4-fpm -y

# Configurar o PHP para "ouvir" fora da máquina
nano /etc/php/8.4/fpm/pool.d/www.conf
Altera estas 3 linhas (muito importante):
Procura listen = /run/php/php8.4-fpm.sock e muda para: listen = 9000
Procura ;listen.allowed_clients (tira o ;) e deixa assim: listen.allowed_clients = 127.0.0.1,192.168.3.1 (Isto permite que o Host/OnePlus fale com ele).

# Reinicia o PHP para assumir as mudanças:
systemctl restart php8.4-fpm

mkdir -p /var/www/html

# Transferência do Joomla Na BOX
cd /var/www/html
sudo tar -cvzf joomla_site.tar.gz .
scp joomla_site.tar.gz phablet@192.168.1.95:~/

# NO ONEPLUS
# 1. Mover o ficheiro para dentro do rootfs do contentor
sudo cp ~/joomla_site.tar.gz /var/lib/lxc/test4test.fostersilva.com/rootfs/var/www/html/

# 2. Entrar no contentor para tratar do resto
sudo lxc-attach -n test4test.fostersilva.com

cd /var/www/html/

# 3. Descompactar (o -v mostra o progresso, podes tirar se preferires)
tar -xvzf joomla_site.tar.gz

# 4. Limpeza e Permissões
rm joomla_site.tar.gz
chown -R www-data:www-data /var/www/html
chmod -R 755 /var/www/html

# Verifica se o PHP 8.4-FPM está a correr
systemctl status php8.4-fpm

3. Configurar o PHP para "ouvir" a rede
Como o teu PHP 8.4-FPM está ativo, precisamos apenas de garantir que ele aceita ligações do Nginx que vamos instalar no Host.

Edita a configuração: nano /etc/php/8.4/fpm/pool.d/www.conf
