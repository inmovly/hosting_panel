#!/bin/bash

# Kolory dla lepszej czytelności
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

echo -e "${GREEN}Rozpoczynam instalację panelu hostingowego...${NC}"

# Sprawdzenie czy skrypt jest uruchomiony jako root
if [ "$EUID" -ne 0 ]; then 
    echo -e "${RED}Uruchom skrypt jako root (sudo)${NC}"
    exit 1
fi

# Aktualizacja systemu
echo -e "${GREEN}Aktualizacja systemu...${NC}"
apt update && apt upgrade -y

# Instalacja wymaganych pakietów
echo -e "${GREEN}Instalacja wymaganych pakietów...${NC}"
apt install -y python3 python3-pip python3-venv apache2 mysql-server certbot \
    python3-certbot-apache git supervisor nginx

# Konfiguracja MySQL
echo -e "${GREEN}Konfiguracja MySQL...${NC}"
mysql_secure_installation

# Tworzenie użytkownika dla aplikacji
echo -e "${GREEN}Tworzenie katalogu aplikacji...${NC}"
mkdir -p /var/www/hosting-panel
cd /var/www/hosting-panel

# Tworzenie wirtualnego środowiska Python
echo -e "${GREEN}Konfiguracja środowiska Python...${NC}"
python3 -m venv venv
source venv/bin/activate

# Instalacja wymaganych bibliotek Pythona
echo -e "${GREEN}Instalacja wymaganych bibliotek Python...${NC}"
pip install flask flask-sqlalchemy flask-login flask-mail flask-limiter \
    mysqlclient psutil dnspython pyotp qrcode cryptography schedule gunicorn

# Tworzenie struktury katalogów
echo -e "${GREEN}Tworzenie struktury katalogów...${NC}"
mkdir -p {logs,backups/{files,databases},ssl,uploads}

# Konfiguracja Gunicorn
echo -e "${GREEN}Konfiguracja Gunicorn...${NC}"
cat > /etc/supervisor/conf.d/hosting-panel.conf << EOF
[program:hosting-panel]
directory=/var/www/hosting-panel
command=/var/www/hosting-panel/venv/bin/gunicorn -w 4 -b 127.0.0.1:8000 app:app
user=www-data
autostart=true
autorestart=true
stderr_logfile=/var/www/hosting-panel/logs/gunicorn.err.log
stdout_logfile=/var/www/hosting-panel/logs/gunicorn.out.log
EOF

# Konfiguracja Nginx
echo -e "${GREEN}Konfiguracja Nginx...${NC}"
cat > /etc/nginx/sites-available/hosting-panel << EOF
server {
    listen 80;
    server_name twoja-domena.pl www.twoja-domena.pl;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }

    location /static {
        alias /var/www/hosting-panel/static;
    }

    location /uploads {
        alias /var/www/hosting-panel/uploads;
    }
}
EOF

# Aktywacja konfiguracji Nginx
ln -s /etc/nginx/sites-available/hosting-panel /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

# Ustawienie uprawnień
echo -e "${GREEN}Ustawianie uprawnień...${NC}"
chown -R www-data:www-data /var/www/hosting-panel
chmod -R 755 /var/www/hosting-panel

# Tworzenie pliku .env dla konfiguracji
echo -e "${GREEN}Tworzenie pliku konfiguracyjnego...${NC}"
cat > /var/www/hosting-panel/.env << EOF
FLASK_APP=app.py
FLASK_ENV=production
SECRET_KEY=$(python3 -c 'import secrets; print(secrets.token_hex(16))')
DATABASE_URL=mysql://root:haslo_root@localhost/hosting_panel
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USE_TLS=True
MAIL_USERNAME=twoj-email@gmail.com
MAIL_PASSWORD=twoje-haslo-aplikacji
EOF

# Restart usług
echo -e "${GREEN}Restart usług...${NC}"
systemctl restart supervisor
systemctl restart nginx

# Konfiguracja firewalla
echo -e "${GREEN}Konfiguracja firewalla...${NC}"
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw enable

echo -e "${GREEN}Instalacja zakończona!${NC}"
echo -e "Następne kroki:"
echo -e "1. Edytuj plik /var/www/hosting-panel/.env i ustaw właściwe dane dostępowe"
echo -e "2. Zmień 'twoja-domena.pl' w konfiguracji Nginx na właściwą domenę"
echo -e "3. Uruchom certbot --nginx dla skonfigurowania SSL"
echo -e "4. Zrestartuj Nginx: systemctl restart nginx"
