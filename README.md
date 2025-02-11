# Шпаргалка по настройке Flask + Gunicorn + Nginx + HTTPS

## 1. Установка Flask
Flask устанавливаем в систему (не через venv):
```sh
sudo apt update && sudo apt install python3-pip -y
pip3 install flask
```

## 2. Установка и конфигурация Gunicorn
Gunicorn также ставим в систему:
```sh
pip3 install gunicorn
```

## 3. Flask-приложение (исполняемый файл)
Файл приложения должен находиться в `/home/site/app.py`:
```python
from flask import Flask, send_from_directory

app = Flask(__name__, static_folder="static")

@app.route("/")
def serve_index():
    return send_from_directory("static", "index.html")

@app.route("/<path:filename>")
def serve_static(filename):
    return send_from_directory("static", filename)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

## 4. Настройка systemd-сервиса для автоматического запуска
Создаём сервис:
```sh
sudo nano /etc/systemd/system/site.service
```

Добавляем содержимое:
```ini
[Unit]
Description=Gunicorn server for Flask site
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/site
ExecStart=/usr/local/bin/gunicorn -w 2 -b 0.0.0.0:8000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

Запускаем и включаем автозапуск:
```sh
sudo systemctl daemon-reload
sudo systemctl start site
sudo systemctl enable site
```

## 5. Настройка Nginx для работы с доменом pospelov-med.ru
Открываем конфигурацию:
```sh
sudo nano /etc/nginx/sites-available/pospelov-med.ru
```

Добавляем:
```nginx
server {
    listen 80;
    server_name pospelov-med.ru www.pospelov-med.ru;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Активируем конфиг:
```sh
sudo ln -s /etc/nginx/sites-available/pospelov-med.ru /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

## 6. Подключение HTTPS (SSL-сертификат Let’s Encrypt)
Устанавливаем Certbot:
```sh
sudo apt install certbot python3-certbot-nginx -y
```

Получаем и устанавливаем SSL-сертификат:
```sh
sudo certbot --nginx -d pospelov-med.ru -d www.pospelov-med.ru
```

Добавляем автоматическое обновление SSL-сертификатов:
```sh
echo "0 3 * * * certbot renew --quiet" | sudo tee -a /etc/crontab > /dev/null
```

## 7. Проверка работы сервиса
Проверяем работу systemd-сервиса:
```sh
sudo systemctl status site
```

Проверяем работу Nginx:
```sh
sudo systemctl status nginx
```

Проверяем SSL-сертификат:
```sh
sudo certbot certificates
```

Теперь сайт доступен по **[https://pospelov-med.ru](https://pospelov-med.ru)** 🚀
