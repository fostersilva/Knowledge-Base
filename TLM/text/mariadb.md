# 1. Atualizar os repositórios
apt update

# 2. Instalar o MariaDB
apt install mariadb-server -y

# Configurar a "Fortaleza"
mysql_secure_installation

# Edita o ficheiro de configuração:
nano /etc/mysql/mariadb.conf.d/50-server.cnf
#alterar
bind-address = 0.0.0.0

-- 1. Criar a base de dados
CREATE DATABASE `test4test.fostersilva.com`;

-- 2. Criar o utilizador com permissão para aceder de qualquer lado (%)
CREATE USER 'fostersilva'@'%' IDENTIFIED BY 'xxxx';

-- 3. Dar todas as permissões apenas nesta base de dados
GRANT ALL PRIVILEGES ON `test4test.fostersilva.com`.* TO 'fostersilva'@'%';

-- 4. Aplicar as alterações
FLUSH PRIVILEGES;

-- 5. Sair
EXIT;


# Na Box: Extrair tudo do contentor para um ficheiro local
sudo lxc-attach -n mariadb.fostersilva.com -- mysqldump -u root -p --all-databases > joomla_backup.sql

# Na Box Transferir entre Hosts:
scp joomla_backup.sql phablet@192.168.1.95:~/

# No OnePlus (Restaurar no novo Contentor)
cat ~/joomla_backup.sql | sudo lxc-attach -n mariadb.fostersilva.com -- mariadb -u root -p

sudo lxc-attach -n mariadb.fostersilva.com -- mariadb-upgrade -u root -p

# Verificar
sudo lxc-attach -n mariadb.fostersilva.com -- mariadb -u root -p -e "SHOW DATABASES; USE \`test4test.fostersilva.com\`; SHOW TABLES;"
