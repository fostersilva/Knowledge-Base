NO HOST
sudo apt update
sudo apt install nginx -y

sudo nano /etc/nginx/sites-available/test4test.fostersilva.com
------------------------------------------------------------------------
server {
    listen 80;
    server_name test4test.fostersilva.com;

    # Caminho para os ficheiros estáticos do Joomla na máquina .11
    root /var/lib/lxc/test4test.fostersilva.com/rootfs/var/www/html;
    index index.php index.html index.htm;

    # Regra para os URLs bonitos do Joomla
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Passar os ficheiros PHP para o motor PHP-FPM na .11
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass 192.168.3.11:9000;
        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
    }

    # Segurança: negar acesso a ficheiros .htaccess
    location ~ /\.ht {
        deny all;
    }
}
------------------------------------------------------------------------
# Ativar o site novo
sudo ln -s /etc/nginx/sites-available/test4test.fostersilva.com /etc/nginx/sites-enabled/

# Desativar a página padrão (Welcome to nginx)
sudo rm /etc/nginx/sites-enabled/default

# Testar a configuração (Se aparecer "syntax is ok", estás no bom caminho!)
sudo nginx -t

# Reiniciar o Nginx
sudo systemctl restart nginx
