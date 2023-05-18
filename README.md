# Проект Kittygram

## Описание
Проект Kittygram — социальная сеть для обмена фотографиями любимых питомцев. Это полностью рабочий проект, который состоит из бэкенд-приложения на Django и фронтенд-приложения на React.

Проект доступен по адресу:
``` 
https://bestkittygram.sytes.net
```

### Стек:
```
Python 3.8.6, Django, DRF, SQLite, Simple-JWT, Docker, Docker compose.
```

### Деплой проекта:
Размещаем проект на удаленном облачном сервере, используя контейнеризацию.
Необходимые данные для подключения к удаленному серверу:
- имя пользователя;
- IP-адрес сервера;
- пароль от закрытого SSH-ключа;
- открытый SSH-ключ;
- закрытый SSH-ключ.

Позже допишу, извините, двое суток боролся с глюком сервера.
Подключаемся к удаленному серверу:
```bash
ssh -i путь_до_файла_с_SSH_ключом/название_файла_с_SSH_ключом_без расширения имя_пользователя@ip_адрес_сервера
```

Устанавливаем git на сервер:
```bash
sudo apt update
```

После [настройки SSH для GitHub](https://code.s3.yandex.net/backend-developer/learning-materials/Настройка%20SSH%20для%20GitHub.pdf), подключаем сервер к аккаунту на GitHub:
```bash
git clone git@github.com:SpaiK89/infra_sprint1.git
```

Установим пакетный менеджер и утилиту для создания виртуального окружения:
```bash
sudo apt install python3-pip python3-venv -y
```

Переходим в директорию с проектом, создаем и активируем виртуальное окружение, устанавливаем пакеты
из файла requirements.txt:
```bash
cd infra_sprint1/backend/
source venv/bin/activate
pip install -r requirements.txt
```

Выполняем миграции и создаем суперюзера:
```bash
python manage.py migrate
python manage.py createsuperuser
```

Для запуска и корректной работы фронтенд-части устанавливаем на сервер Node.js,
устанавливаем зависимости: 
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - &&\
sudo apt-get install -y nodejs
npm i
```

Устанавливаем и запускаем Gunicorn:
```bash
pip install gunicorn==20.1.0

```

Для автоматического запуска и постоянной работы WSGI-сервера настроим работу утилиты systemd,
создав пользовательский юнит:
```bash
sudo nano /etc/systemd/system/gunicorn-kittygram.service
```

Пропишем в нём все необходимые конфигурации и сохраним:
```
[Unit]
# Это текстовое описание юнита, пояснение для разработчика.
Description=gunicorn daemon 

# Условие: при старте операционной системы запускать процесс только после того, 
# как операционная система загрузится и настроит подключение к сети.
# Ссылка на документацию с возможными вариантами значений 
# https://systemd.io/NETWORK_ONLINE/
After=network.target 

[Service]
# От чьего имени будет происходить запуск:
# укажите имя, под которым вы подключались к серверу.
User=<имя-пользователя-в-системе> 

# Путь к директории проекта
# /home/<имя-пользователя-в-системе>/
# <директория-с-проектом>/<директория-с-файлом-manage.py>/
# Например:
WorkingDirectory=/home/yc-user/infra_sprint1/backend/

# автоматический запуск systemd
# /home/<имя-пользователя-в-системе>/
# <директория-с-проектом>/<путь-до-gunicorn-в-виртуальном-окружении> --bind 0.0.0.0:8000 backend.wsgi
ExecStart=/home/yc-user/infra_sprint1/backend/venv/bin/gunicorn --bind 0.0.0.0:8000 kittygram_backend.wsgi

[Install]
# В этом параметре указывается вариант запуска процесса.
# Значение <multi-user.target> указывает, чтобы systemd запустил процесс
# доступный всем пользователям и без графического интерфейса.
WantedBy=multi-user.target
```

Запускаем WSGI-сервер и добавляем процесс Gunicorn в список автозапуска:
```bash
sudo systemctl start gunicorn-kittygram
sudo systemctl enable gunicorn-kittygram
```

Устанавливаем веб-сервер Nginx и запускаем его:
```bash
sudo apt install nginx -y
sudo systemctl start nginx
```

Оставляем возможность отправлять запросы только на порты 80, 443, 22 с помощью встроенного файрвола ufw
и запускаем его:
```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

Продолжаем настройку Nginx и собираем статику фронтенд-приложения:
```bash
cd infra_sprint1/frontend
npm run build
sudo cp -r <имя_пользователя>/infra_sprint1/frontend/build/. /var/www/kittygram/
```

Открываем файл конфигурации веб-сервера
```bash
sudo nano /etc/nginx/sites-enabled/default
```

Настраиваем веб-сервер, записав в файл конфигурации следующее:
```bash

server {

    server_name <публичный_ip_вашего_удаленного_сервера> <доменное_имя>;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;

    }

    location /admin/ {
        proxy_pass http://127.0.0.1:8080;

    }

    location /media/ {
        alias /var/www/kittygram/media/;

    }
    location / {
        root   /var/www/kittygram;
        index  index.html index.htm;
        try_files $uri /index.html;
    }
}
```
Cобираем статику бэкенд-приложения:
```bash
cd infra_sprint1/backend
python manage.py collectstatic
sudo cp -r infra_sprint1/backend/static_backend/ /var/www/kittygram/
```

Создадим директорию media в директории /var/www/kittygram/:
```bash
cd /var/www/kittygram/
sudo mkdir media
```

Установим и настроим certbot для получения SSL-сертификата:
```bash
sudo apt install snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

Перезагрузим веб-сервер и WSGI-сервер:
```bash
sudo systemctl reload nginx
sudo systemctl restart gunicorn-kittygram
```


### Работу по деплою выполнил
- [Богомолов Игорь (тимлид, разработка ресурсов Auth и Users)](https://github.com/SpaiK89)
