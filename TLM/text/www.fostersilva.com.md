sudo lxc-create -n www.fostersilva.com -t download -- -d ubuntu -r jammy -a arm64
sudo lxc-start -n www.fostersilva.com

# adicionar em
sudo nano /var/lib/lxc/www.fostersilva.com/config
____________________________________________
lxc.net.0.ipv4.address = 192.168.3.10/24
lxc.net.0.ipv4.gateway = 192.168.3.1
_____________________________________________

sudo lxc-attach -n www.fostersilva.com

echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" > /etc/resolv.conf

# PHP 8
apt install software-properties-common -y

# Adicionar o repositório do PHP (o mais fiável para Debian/Ubuntu)
add-apt-repository ppa:ondrej/php -y
apt update

# Instalar o PHP 8.4 e Dependências
apt update
apt install php8.4-fpm php8.4-mysql php8.4-xml php8.4-curl php8.4-gd php8.4-zip php8.4-intl php8.4-mbstring unzip wget -y

mkdir -p /var/www/html

wget https://downloads.joomla.org/cms/joomla6/6-0-2/Joomla_6-0-2-Stable-Full_Package.zip

# Descompactar
unzip Joomla_6-0-2-Stable-Full_Package.zip
rm Joomla_6-0-2-Stable-Full_Package.zip

# Dar permissões ao PHP para ler/escrever
chown -R www-data:www-data /var/www/html
chmod -R 755 /var/www/html

########################################################################
#                                                                      #
#                      MARIADB                                         #
#                                                                      #
########################################################################

# Criar a Base de Dados (No Contentor .100)
sudo lxc-attach -n mariadb.fostersilva.com
mysql -u root -p

# cria a nova BD para o site principal
-- 1. Criar a base de dados
CREATE DATABASE www_fostersilva_com;

-- 2. Criar o utilizador
CREATE USER 'www_fostersilva_com'@'%' IDENTIFIED BY 'A_TUA_PASSWORD';

-- 3. Atribuir as permissões na nova base de dados
GRANT ALL PRIVILEGES ON www_fostersilva_com.* TO 'www_fostersilva_com'@'%';

-- 4. Aplicar as alterações
FLUSH PRIVILEGES;

EXIT;

########################################################################
#                                                                      #
#                      PHP-FM                                          #
#                                                                      #
########################################################################

lxc-attach -n www.fostersilva.com
Dentro do contentor (lxc-attach -n www.fostersilva.com):

Edita o ficheiro: nano /etc/php/8.4/fpm/pool.d/www.conf

Procura a linha listen = e altera para: listen = 9000

Procura listen.allowed_clients e adiciona o IP da bridge:
listen.allowed_clients = 127.0.0.1, 192.168.3.1

Reinicia o PHP: systemctl restart php8.4-fpm

########################################################################
#                                                                      #
#                      NGINX                                           #
#                                                                      #
########################################################################

# Configurar o Nginx no Host (O Porteiro)
sudo nano /etc/nginx/sites-available/www.fostersilva.com
________________________________________________________________________
server {
    listen 80;
    server_name www.fostersilva.com fostersilva.com;

    # Onde o Nginx lê as imagens e CSS diretamente do "Portal" (visto do Host)
    root /var/lib/lxc/test4test.fostersilva.com/rootfs/var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Ligar ao PHP 8.4-FPM na máquina .10 (Igual ao teu exemplo)
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass 192.168.3.10:9000;
        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
        include fastcgi_params;
    }
}
________________________________________________________________________

# Dá permissão de leitura/atravessamento na pasta do contentor para o grupo
sudo chmod 755 /var/lib/lxc/www.fostersilva.com
sudo chmod 755 /var/lib/lxc/www.fostersilva.com/rootfs

# Ativa o site e reinicia o Nginx:
sudo ln -s /etc/nginx/sites-available/www.fostersilva.com /etc/nginx/sites-enabled/
sudo systemctl restart nginx

########################################################################
#                                                                      #
#                      Certbot                                         #
#                                                                      #
########################################################################

# Ativar o HTTPS (Certbot) Como já tens o Certbot instalado, é só pedir o novo certificado:
sudo certbot --nginx -d www.fostersilva.com -d fostersilva.com









1. Configuração Inicial
Nome do Sítio: Escolhe o nome para o teu portal.

Dados de Autenticação: Define o e-mail, utilizador e password do administrador (guarda estes dados bem!).

2. Configuração da Base de Dados (O passo mais importante)
Quando avançares, o Joomla vai pedir os dados de ligação. Como tens o MariaDB num contentor separado, usa estas definições:

Tipo de Base de Dados: MySQLi ou MySQL (PDO).

Nome do servidor (Host): 192.168.3.100 (O IP do teu contentor MariaDB).

Nome de utilizador: fostersilva

Palavra-passe: A password que definiste quando criaste o utilizador no MariaDB.

Nome da base de dados: joomla_www

3. Ajuste de Segurança Pós-Instalação
Assim que terminares a instalação, o Joomla vai pedir para remover a pasta installation. Podes fazê-lo diretamente no terminal do contentor .10:

Bash
rm -rf /var/www/html/installation
