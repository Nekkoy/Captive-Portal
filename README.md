[![en](https://img.shields.io/badge/lang-en-red.svg)](README_EN.md)
[![ua](https://img.shields.io/badge/lang-ua-yellow.svg)](README.md)
[![ru](https://img.shields.io/badge/lang-ru-blue.svg)](README_RU.md)

# Captive Portal для інтернет-провайдерів

**Проста реалізація Captive Portal**, завдання якого – відобразити сторінку-заглушку клієнтам з обмеженим доступом до інтернету.

<img src="https://raw.githubusercontent.com/Nekkoy/Captive-Portal/main/img_ua.jpg" width="560" height="380">

---

## 🔧 Як це працює

1. Пристрій клієнта при підключенні до мережі перевіряє наявність доступу до інтернету через DNS-запити до спеціальних доменів (наприклад, `www.msftconnecttest.com`, `captive.apple.com`, та ін).
2. Ми перехоплюємо ці запити та відповідаємо на них з локального DNS-сервера (Unbound), вказуючи на IP Captive Portal.
3. Клієнт робить HTTP запит по отриманому IP. Якщо сервер повертає `HTTP 302 Found` з потрібною URL - браузер клієнта автоматично відкриває сторінку з цим URL.

---

## 📦 Встановлення та налаштування

### 1. Установка nginx та unbound

#### Debian/Ubuntu:
```bash
apt-get install nginx unbound
```

#### RHEL/CentOS:
```bash
yum install nginx unbound
```

### 2. Налаштування nginx
#### a. Сторінка-заглушка
Скопіюйте файл `nginx_captive_portal_page.conf` у `/etc/nginx/conf.d/`:
```bash
cp -f ./nginx_captive_portal_page.conf /etc/nginx/conf.d/
```
Змініть доменне ім'я `captiveportal.example.com` на ваш домен та встановіть SSL-сертифікат.

Створіть директорію для статики:
```bash
mkdir -p /var/www/captive_portal
```

Та покладіть туди HTML-файл:
```bash
cp -f ./index.html /var/www/captive_portal/index.html
```

#### b. Редирект
Скопіюйте файл `nginx_captive_portal_redirect.conf` у ту саму папку:
```bash
cp -f ./nginx_captive_portal_redirect.conf /etc/nginx/conf.d/
```
Замініть домен `captiveportal.example.com` на ваш рядок:
```bash
 set $portal_domain captiveportal.example.com;
```

Перезапустіть `nginx`:
```bash
systemctl restart nginx
```

### 3. Налаштування unbound
Скопіюйте конфігураційний файл:
```bash
cp -f ./unbound.conf /etc/unbound/
```

Тепер потрібно замінити в конфізі IP адресу Captive Portal
Для цього виконаємо команду замінивши IP `10.0.0.1` на IP свого Captive Portal
```bash
sed -i 's|192\.168\.1\.254|10.0.0.1|g' /etc/unbound/unbound.conf
```

Перезапустіть `Unbound`:
```bash
systemctl restart unbound
```

## 🔐 Перехоплення DNS запитів через iptables
#### a. Перенаправлення DNS-клієнтів на локальний DNS (192.168.1.254) із заміною адреси на адресу шлюзу (192.168.1.1)
```bash
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p udp --dport 53 -j DNAT --to-destination 192.168.1.254:53
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p tcp --dport 53 -j DNAT --to-destination 192.168.1.254:53
```
#### b. SNAT для вихідних DNS-запитів
```bash
iptables -t nat -A POSTROUTING -p udp -d 192.168.1.254 --dport 53 -j SNAT --to-source 192.168.1.1
iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.254 --dport 53 -j SNAT --to-source 192.168.1.1
```

## 🔒 Обмеження для клієнтів
Для коректної роботи Captive Portal клієнти повинні бути обмежені таким чином:

1. Дозволено:
 - DNS (UDP/TCP 53)
 - HTTP (80) та HTTPS (443) доступ тільки до:
   - Captive Portal
   - Особистий кабінет
   - Платіжні системи та інші «білі» сайти

2. Заборонено:
 - HTTP(S) доступ до інших хостів в інтернеті
