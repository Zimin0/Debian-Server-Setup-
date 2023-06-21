# Настройка Debian сервера под Django проект
Основано на (https://github.com/alexey-goloburdin/debian-set-up-for-django)


## :anger: SSH :anger:


Обновите список пакетов с помощью команды:
```
sudo apt-get update && apt-get upgrade
```

Установите OpenSSH:
```
sudo apt-get install openssh-server
```

Разрешите удаленное подключение через порт 22 для SSH в настройках UFW:
```
sudo apt-get -y ssh 
sudo ufw allow ssh
```
Проверьте работу SSH с помощью команды:
```
sudo systemctl status ssh
```
Статус «active (running)» означает, что SSH запущен. <br>
Также в строке «Server listening on» вы увидите порт, через который работает SSH. <br>
<br>
В случае unactive выполните:
```
sudo system ssh restart
```

## :anger: Установка полезных пакетов :anger:
```
sudo apt-get update ; 
sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make 
sudo apt install -y git ufw python3-pip python3-dev nginx

sudo apt-get install python-pil python3-pil

sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python3-lxml libxslt-dev libffi-dev libssl-dev python-dev-is-python3 gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor 
```

Установка zsh как дефолтного shell
```
chsh -s $(which zsh)
```

Установка красивого python-shell
```
pip install ipython
```

## :anger: Установка Python-3.7 :anger:
```
mkdir ~/code

wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz ; \
tar xvf Python-3.7.* ; \
cd Python-3.7.3 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/www/.python ; \
make -j8 ; \
sudo make altinstall

sudo /home/www/.python/bin/python3.7 -m pip install -U pip
```
После установки можно удалить обе папки - архив и Python-3.7.3
```
apt-get install -y python3-venv
/home/www/
cd code
git pull project_git
cd project_dir
python3.7 -m venv env
. ./env/bin/activate
```
Активируется виртуальное окружение <br>
Для проверки работы проекта можно выполнить и в браузере открыть server-ip:8181: 
```
python manage.py runserver 0.0.0.0:8181
```

## :anger: Установка и настройка Gunicorn :anger:

```
pip install gunicorn
pip freeze > requirements.txt
```
В папке с manage.py
```
touch gunicorn_config.py
```
В него вставляем код (кол-во воркеров = кол-во ядер процессора * 2 + 1): 
```
command = '/home/www/code/project/env/bin/gunicorn'
pythonpath = '/home/www/code/project/project' # до папки с manage.py
bind = '127.0.0.1:8001'
workers = 3 
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=project.settings'
workers = 2 * n + 1, где n – кол-вао ядер
```
Pwd - /home/www/code/project ( уровень manage.py )

Создаем юзера, через которого будет запускаться Gunicorn
```
adduser www
usermod -aG sudo www
```

```
mkdir bin
touch bin/start_gunicorn.sh
```
В файл записываем:
```
#!/bin/bash 
source /home/www/code/monstory_project/env/bin/activate
gunicorn -c “/home/www/code/monstory_project/ monstory_project/gunicorn_config.py” monstory_project.wsgi
```
Добавляем права запуска для bash скрипта
```
chmod +x bin/start_gunicorn.sh
```
Запускаем Gunicorn: 
```
./bin/start_gunicorn.sh
```

## :anger: Настройка Nginx :anger:

```
nano /etc/nginx/sites-available/default
```

Добавляем код: /home/cloudadmin/vkbotbook

!!!!!!!!!!!!!! Поправить кавычки, заменить на базовый локейшн !!!!!!!!!!!!!!

location / {
proxy_pass http://127.0.0.1:8001;
proxy_set_header X-Forwarded-Host $server_name;
proxy_set_header X-Real-IP $remote_addr;
add_header P3P ‘CP-“ALL DSP COR PSAa PSDa OUR NOR UNI COM NAV”’;
add_header Access-Control-Allow-Origin *;
}

Перезапускаем Nginx, чтобы он подтянул конфиги:
```
service nginx restart
```
По ip сервера должно появиться окно 502 с nginx
логи доступа nginx 
nano  /var/log/nginx/access.log
Запустить gunicorn 
Приложение заработает на 8001 порту

Запуск supervisor
nano /etc/supervisor/conf.d/Craft-Site.conf

## :anger: Настройка раздачи статического/медиа контента :anger:
В том же файле Nginx (sites-available)
```
location /static/ {
        root /home/theo/mywebsite;
    }
```
Прописать chmod 777 для базы данных проекта, если это SQLite


## :anger: Настройка доменного имени для хостинга Reg.ru :anger:
Найти нужный домен на www.reg.ru 
В разделе Управление -> DNS-серверы и управление зоной поменять DNS сервера на ns5.hosting.reg.ru, ns6.hosting.reg.ru
Зайти на dnsadmin.hosting.reg.ru -> Доменные имена -> Создать -> вписываем домен и IP адрес. 



## :anger: Немного теории: :anger:
Почему стоит использовать Gunicorn?
Команда разработчиков рекомендует использовать Gunicorn в связке с Nginx, где Nginx используется в качестве прокси-сервера.
Периодически между веб-сервером и веб-приложением существуют одна или несколько промежуточных прослоек. Такие прослойки позволяют осуществить, например, балансировку между несколькими веб-приложениеми или предпроцессинг(предобработку) отдаваемого контента.

WSGI-сервера были разработаны чтобы обрабатывать множество запросов одновременно. А фреймворки не предназначены для обработки тысяч запросов и не дают решения того, как наилучшим образом маршрутизировать их(запросы) с веб-сервера. Поэтому требуется Gunicorn.


