# **Deploy** de aplicação Django na DigitalOcean

---

### **app** - Nome do projeto

### **deploy** - Nome do usuário

---

## Criando e configurando via root

---

- Criar droplet com Ubuntu 18.04
- Conectar via ssh --- ssh root@000.000.000.000
- apt update && apt upgrade
- adduser deploy
- usermod -aG sudo deploy
- cd /home/deploy
- mkdir .ssh
- chown deploy:deploy .ssh/
- cp ~/.ssh/authorized_keys /home/deploy/.ssh/
- cd .ssh/
- chown deploy:deploy authorized_keys
- exit

---

## Instalar ssh_keygen

---

- ssh-keygen

---

## Instalar Docker

---

- sudo apt install apt-transport-https ca-certificates curl software-properties-common
- curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
- sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
- sudo apt update
- apt-cache policy docker-ce
- sudo apt install docker-ce
- sudo systemctl status docker
- sudo usermod -aG docker ${USER}
- su - ${USER}
- id -nG
- sudo usermod -aG docker **deploy**
- docker run -d --name postgres -e POSTGRES_PASSWORD=29gu09il -e POSTGRES_USERNAME=postgres -e POSTGRES_DATABASE=delta -p 5432:5432 bitnami/postgresql:latest

---

## Abrir portas

---

- sudo ufw allow 8000
- sudo ufw allow 80
- sudo ufw allow 443

---

## Instalando Python

---

- sudo apt-get install python3.6-dev
- sudo apt-get install build-essential libssl-dev libffi-dev python-dev
- sudo apt-get install python3-venv
- python3 -m venv venv

---

## Clonando projeto

---

- git config --global user.email "guilhermeddc@gmail.com"
- git config --global user.name "Guilherme Rodrigues"
- git clone ...

---

## Configurando o projeto

---

- source venv/bin/activate
- pip install uwsgi
- pip install wheel
- pip install -r requirements.txt
- python manage.py collectstatic
- python manage.py migrate
- python manage.py createsuperuser

---

## Instalando Nginx

---

- sudo apt-get install nginx
- sudo /etc/init.d/nginx start

---

## Configurando Nginx / uWSGI

---

- vim uwsgi_params

```

uwsgi_param     QUERY_STRING    $query_string;
uwsgi_param     REQUEST_METHOD  $request_method;
uwsgi_param     CONTENT_TYPE    $content_type;
uwsgi_param     CONTENT_LENGTH  $content_length;

uwsgi_param     REQUEST_URI     $request_uri;
uwsgi_param     PATH_INFO       $document_uri;
uwsgi_param     DOCUMENT_ROOT   $document_root;
uwsgi_param     SERVER_PROTOCOL $server_protocol;
uwsgi_param     REQUEST_SCHEME  $scheme;
uwsgi_param     HTTPS           $https if_not_empty;

uwsgi_param     REMOTE_ADDR     $remote_addr;
uwsgi_param     REMOTE_PORT     $remote_port;
uwsgi_param     SERVER_PORT     $server_port;
uwsgi_param     SERVER_NAME     $server_name;

```

- cd /etc/nginx/sites-available/
- sudo vim django.conf

```
upstream django {
    server unix:///home/deploy/app/mysite.sock;
}

server {
    listen      80;
    server_name exemple.com;
    charset     utf-8;

    client_max_body_size 75M;

    location /media  {
        alias /home/deploy/app/media;
    }

    location /static {
        alias /home/deploy/app/static;
    }

    location / {
        uwsgi_pass  django;
        include     /home/deploy/app/uwsgi_params;
    }
}
```

- cd ../sites-enabled/
- sudo ln -s /etc/nginx/sites-available/django.conf /etc/nginx/sites-enabled/
- sudo rm default
- cd
- cd **app**/
- vim core/settings.py -> adicionar no ALLOWED_HOSTS = ['exemple.com', '000.000.000.000']
- sudo /etc/init.d/nginx restart
- uwsgi --socket mysite.sock --module core.wsgi --chmod-socket=666
- vim uwsgi.ini

```
[uwsgi]
chdir           = /home/deploy/app
module          = core.wsgi
home            = /home/deploy/venv
master          = true
processes       = 10
socket          = /home/deploy/app/mysite.sock
vacuum          = true
chmod-socket    = 666
```

- uwsgi --ini uwsgi.ini

---

## Configurar o uWSGI no modo Emperor

---

- sudo mkdir /etc/uwsgi
- sudo mkdir /etc/uwsgi/vassals
- sudo ln -s /home/**deploy**/**app**/uwsgi.ini /etc/uwsgi/vassals/
- uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data

---

## Configurar systemctl

---

- cd /etc/systemd/system/
- sudo vim django.service

```
[Unit]
Description=Django VPS uWSGI Emperor
After=syslog.target

[Service]
ExecStart=/home/deploy/venv/bin/uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data
RuntimeDirectory=uwsgi
Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all
User=deploy

[Install]
WantedBy=multi-user.target
```

- sudo chmod 664 /etc/systemd/system/django.service
- sudo systemctl daemon-reload
- sudo systemctl enable django.service
- sudo systemctl start django.service
- sudo systemctl status django.service
- journalctl -u django.service

---

## HTTPS --- https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx

---

- sudo snap install core; sudo snap refresh core
- sudo snap install --classic certbot
- sudo ln -s /snap/bin/certbot /usr/bin/certbot
- sudo certbot --nginx

---
