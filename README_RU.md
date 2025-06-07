[![en](https://img.shields.io/badge/lang-en-red.svg)](README_EN.md)
[![ua](https://img.shields.io/badge/lang-ua-yellow.svg)](README.md)
[![ru](https://img.shields.io/badge/lang-ru-blue.svg)](README_RU.md)

# Captive Portal для интернет-провайдеров

**Простая реализация Captive Portal**, задача которого — отобразить страницу-заглушку клиентам с ограниченным доступом к интернету.

<img src="https://raw.githubusercontent.com/Nekkoy/Captive-Portal/main/img_ru.jpg" width="560" height="380">

---

## 🔧 Как это работает

1. Устройство клиента при подключении к сети проверяет наличие доступа к интернету через DNS-запросы к специальным доменам (например, `www.msftconnecttest.com`, `captive.apple.com`, и др.).
2. Мы перехватываем эти запросы и отвечаем на них с локального DNS-сервера (Unbound), указывая на IP Captive Portal.
3. Клиент делает HTTP запрос по полученному IP. Если сервер возвращает `HTTP 302 Found` с нужным URL — браузер клиента автоматически открывает страницу с этим URL.

---

## 📦 Установка и настройка

### 1. Установка nginx и unbound

#### Debian/Ubuntu:
```bash
apt-get install nginx unbound
```

#### RHEL/CentOS:
```bash
yum install nginx unbound
```

### 2. Настройка nginx
#### a. Страница-заглушка
Скопируйте файл `nginx_captive_portal_page.conf` в `/etc/nginx/conf.d/`:
```bash
cp -f ./nginx_captive_portal_page.conf /etc/nginx/conf.d/
```
Измените доменное имя `captiveportal.example.com` на ваш домен и установите SSL-сертификат.

Создайте директорию для статики:
```bash
mkdir -p /var/www/captive_portal
```

И положите туда HTML-файл:
```bash
cp -f ./index.html /var/www/captive_portal/index.html
```

#### b. Редирект
Скопируйте файл `nginx_captive_portal_redirect.conf` в ту же папку:
```bash
cp -f ./nginx_captive_portal_redirect.conf /etc/nginx/conf.d/
```
Замените домен `captiveportal.example.com` на ваш в строке:
```bash
 set $portal_domain captiveportal.example.com;
```

Перезапустите `nginx`:
```bash
systemctl restart nginx
```

### 3. Настройка unbound
Скопируйте конфигурационный файл:
```bash
cp -f ./unbound.conf /etc/unbound/
```

Перезапустите `Unbound`:
```bash
systemctl restart unbound
```

## 🔐 Перехват DNS запросов через iptables
#### a. Перенаправление DNS-клиентов на локальный DNS (192.168.1.254) с подменой адреса на адрес шлюза (192.168.1.1)
```bash
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p udp --dport 53 -j DNAT --to-destination 192.168.1.254:53
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p tcp --dport 53 -j DNAT --to-destination 192.168.1.254:53
```
#### b. SNAT для исходящих DNS-запросов
```bash
iptables -t nat -A POSTROUTING -p udp -d 192.168.1.254 --dport 53 -j SNAT --to-source 192.168.1.1
iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.254 --dport 53 -j SNAT --to-source 192.168.1.1
```

## 🔒 Ограничения для клиентов
Для корректной работы Captive Portal клиенты должны быть ограничены следующим образом:

1. Разрешены:
 - DNS (UDP/TCP 53)
 - HTTP (80) и HTTPS (443) доступ только к:
   - Captive Portal
   - Личный кабинет
   - Платёжные системы и прочие «белые» сайты

2. Запрещены:
 - HTTP(S) доступ к другим хостам в интернете
