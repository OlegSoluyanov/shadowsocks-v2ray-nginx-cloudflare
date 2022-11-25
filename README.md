# Конфигурация сервера shadowsocks c плагином обфускации v2ray c использованием CDN Cloudflare
Зашифрованный прокси позволяющий перенаправлять траффик клиента как обычный HTTPS к доменному имени за CDN Cloudflare

Подготовка системы.
Конфигурация реализована на **`Ubuntu 22.04`**. Предварительно настроен вход по **SSH key**. Создан новый пользователь, отключен вход root и вход по паролю. Изменен стандартный порт ssh на кастомный. Настроен файервол `UFW`, закрыты все порты на вход за исключением порта ***SSH*** и портов ***HTTPS (443)*** и ***HTTP (80)***.

## 🔲Установка `shadowsocks-rust`
* :one: Апдейт и апгрейд системы
```bash
sudo apt update && sudo apt upgrade -y
```
* :two: Проверка наличия `wget` для загрузки
```bash
sudo apt install wget
```
* :three: Переход во временную папку
```bash
cd /tmp
```
* :four: Загрузка актуальной версии `shadowsock-rust`.  [Тут нужно выбрать `shadowsocks-rust`](https://github.com/shadowsocks/shadowsocks-rust/releases/ "список релизов shadowsocks") для своей архитектуры. Рекомедуется загружать версию с тегом `latest`.
```bash
dpkg --print-architecture # Определение архитектуры
```
```bash
# Загрузка соответсвующей архитектуре версии, в этом примере для amd64
sudo wget https://github.com/shadowsocks/shadowsocks-rust\
/releases/download/v1.14.3/shadowsocks-v1.14.3.x86_64-unknown-linux-gnu.tar.xz
```
* :five: Распаковка скачанного архива
```bash
sudo tar -xf shadowsocks-v1.14.3.x86_64-unknown-linux-gnu.tar.xz
```
* :six: Копирование бинарного файла shadowsocks-rust в `/usr/local/bin`
```bash
sudo cp ssserver /usr/local/bin
```
* :seven: Создание папки `shadowsocks`
```bash
sudo mkdir /etc/shadowsocks
```
* :eight: Cоздание файла конифигурации `shadowsocks`
```bash
sudo nano /etc/shadowsocks/shadowsocks-rust.json
```
Первичная настройка конфигурации
```javascript
{
"server": "0.0.0.0",               # IP адрес сервера, по умолчанию указаны все интерфейсы
"server_port": port,                # Кастомный порт сервера по выбору
"password": "password",             # пароль
"timeout": 300,                     # Желательно не более 600, время закрытия соединения
"method": "chacha20-ietf-poly1305", # Метод шифрования, при выборе исходить из уровня потребления ресурсов, можно остваить.
"no_delay": true,
"fast_open": true,                  # Быстрое открытие соединения на ядрах старше 3.16
"reuse_port": true,                 # Ускоряет приемку пакетов, позволяет привязку к TCP 
"workers": 1,                       # Количество ядер доступных серверу
"ipv6_first": true,                 # В случае поддержки сервером ipv6 - true, иначе - false
"nameserver": "DNS url",            # url предпочитаемого dns провайдера
"mode": "tcp_only"                  # Режим работы в TCP， UDP не поддерживается.
}
```
* :nine: Оптимизация конфигурации `sysctl` под `shadowsocks-rust`
```bash
sudo nano /etc/sysctl.conf
```
Cодержимое [файла sysctl.conf](https://github.com/OlegSoluyanov/shadowsocks-v2ray-nginx-cloudflare/blob/8f9cf5cd1f5c40bcb24010052700903a58464b6c/sysctl.conf "дополнение sysctl.conf") необходимо добавить в конец файла открытого выше.
* :one::zero: Применение изменений `sysctl` без перезагрузки
```bash
sudo sysctl -p
```
* :one::one: Контроль изменений `sysctl`
```bash
sudo sysctl net.ipv4.tcp_available_congestion_control
```
Вывод должен быть таким:
`net.ipv4.tcp_available_congestion_control = reno cubic bbr`
* :one::two: Cоздание системного юнита (daemon) `shadowsock-rust.service` для автозапуска сервера shadowsocks
```bash
sudo nano /usr/lib/systemd/system/shadowsocks-rust.service
```
Содержимое юнита
```
[Unit]
Description=shadowsocks-rust service
After=network.target
[Service]
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks/shadowsocks-rust.json
ExecStop=/usr/bin/killall ssserver
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ssserver
User=nobody
Group=nogroup
[Install]
WantedBy=multi-user.target
```
* :one::three: Активация и старт демона `shadowsocks-rust`
```bash
sudo systemctl enable shadowsocks-rust && sudo systemctl start shadowsocks-rust
```
* :one::four: Проверка работы сервера `shadowsocks-rust`
```bash
sudo systemctl status shadowsocks-rust
```
<details>
<summary>Вывод должен быть таким: (развернуть) ⬇️</summary>
     
```
● shadowsocks-rust.service - shadowsocks-rust service
     Loaded: loaded (/lib/systemd/system/shadowsocks-rust.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-11-23 20:44:29 +10; 1 day 20h ago
   Main PID: 746 (ssserver)
      Tasks: 11 (limit: 1030)
     Memory: 14.9M
        CPU: 1min 48.171s
     CGroup: /system.slice/shadowsocks-rust.service
             ├─746 /usr/local/bin/ssserver -c /etc/shadowsocks/shadowsocks-rust.json
```
</details>    
     
**`Внимание!`** В случае ошибки необходимо строго проверить файл конфигурации, синтаксис, наличие кавычек и запятых в строках, также нужно проверить под какую *архитектуру* скачан архив `shadowsocks-rust`.
## 🔲Установка плагина обфускации траффика `v2ray`
* :one: Загрузка актуальной версии плагина v2ray. [Тут нужно выбрать `v2ray plugin`](https://github.com/shadowsocks/v2ray-plugin/releases/ "список релизов v2ray plugin") для своей архитектуры. Рекомедуется загружать версию с тегом `latest`.
```bash
sudo wget https://github.com/shadowsocks/v2ray-plugin/releases/download/v1.3.2/v2ray-plugin-linux-amd64-v1.3.2.tar.gz
```
* :two: Распаковка архива с плагином `v2ray`
```bash
sudo tar -xf v2ray-plugin-linux-amd64-v1.3.2.tar.gz
```
* :three: Перенос плагина в папку с shadowsocks переименование файла плагина в v2ray-plugin
```bash
sudo mv v2ray-plugin_linux_amd64 /etc/shadowsocks/v2ray-plugin
```
* :four: Установка прав файла плагина `v2ray` и возможности его подключения к привелигерованным портам
```bash
sudo setcap "cap_net_bind_service=+eip" /etc/shadowsocks/v2ray-plugin && sudo chmod +x /etc/shadowsocks/v2ray-plugin
```
* :five: Корректировка файла конфигурации `shadowsocks-rust.json` для работы с плагином `v2ray`
```bash
sudo nano /etc/shadowsocks/shadowsocks-rust.json
```
Необходимо добавить строки `plugin` с указанием пути к плагину и `plugin_opts` со значением `"server"` в конец файла. 
```javascript
{
"server": "server ip",                        # IP адрес сервера
"server_port": port,                          # Кастомный порт сервера по выбору
"password": "password",                       # пароль
"timeout": 300,                               # Желательно не более 600, время закрытия соединения
"method": "chacha20-ietf-poly1305",           # Метод шифрования, при выборе исходить из уровня потребления ресурсов, можно остваить.
"no_delay": true,
"fast_open": true,                            # Быстрое открытие соединения на ядрах старше 3.16
"reuse_port": true,                           # Ускоряет приемку пакетов, позволяет привязку к TCP 
"workers": 1,                                 # Количество ядер доступных серверу
"ipv6_first": true,                           # В случае поддержки сервером ipv6 - true, иначе - false
"nameserver": "DNS url",                      # url предпочитаемого dns провайдера
"mode": "tcp_only",                           # Режим работы в TCP， UDP не поддерживается.
"plugin": "/etc/shadowsocks/v2ray-plugin",    # Путь к плагину
"plugin_opts": "server"                       # Опция плагина, при использовании Сlouflare.
}
```
* :six: Проверка работы `shadowsocks-rust` c плагином `v2ray`
```bash
sudo systemctl restart shadowsocks-rust && sudo systemctl status shadowsocks-rust
```
<details>
<summary>Вывод должен быть такой: (развернуть) ⬇️</summary>
     
```
● shadowsocks-rust.service - shadowsocks-rust service
     Loaded: loaded (/lib/systemd/system/shadowsocks-rust.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-11-23 20:44:29 +10; 1 day 21h ago
   Main PID: 746 (ssserver)
      Tasks: 12 (limit: 1030)
     Memory: 14.9M
        CPU: 1min 49.233s
     CGroup: /system.slice/shadowsocks-rust.service
             ├─746 /usr/local/bin/ssserver -c /etc/shadowsocks/shadowsocks-rust.json
             └─766 /etc/shadowsocks/v2ray-plugin
 ```
 </details>     
     
 Нижняя строчка и статус ***active(runnung)*** показывает что плагин работает.
 <br>**`Внимание!`** В случае ошибки, необходимо проверить синтаксис конфигурационнго файла и соответствие скачанной версии плагина архитектуре сервера.
 
 ## 🔲 Установка и конфигурация веб сервера `Nginx` для работы с доменом проксируемым `CDN Cloudflare`
 * :one: Установка сертификатов `Cloudflare` на сервер
 ```bash
 # Папка под сертификаты конкретного домена
 sudo mkdir /etc/ssl/example_certs
 ```
 В папку example_certs необходимо установить:
 
 - - `ssl` сертификат cloudflare, example.com.pem
 * * `ssl key` сертификат cloudflare, example.com.key
 * * сертификат `authenticated_origin_pull_ca` (опционально) - для [проксирования запросов к серверу только с IP адресов CDN Cloudflare](https://blog.cloudflare.com/protecting-the-origin-with-tls-authenticated-origin-pulls/ "описание функции")
 * * `origin_ca_rsa_root` (trusted_certificate) Cloudflare (опционально) для включения функции [OCSP Stapling](https://blog.cloudflare.com/high-reliability-ocsp-stapling/ "описание функции")
 * :two: Генерация параметра [DH (Диффи-Хеллмана)](https://tproger.ru/translations/diffie-hellman-key-exchange-explained/ "описание") для шифрования соединения.
 ```bash
sudo openssl dhparam -out /etc/ssl/example_certs/dhparams.pem 4096
```
* :three: Установка `nginx` и проверка статуса
```bash
sudo apt install nginx && sudo systemctl status nginx
```
<details>
<summary>Вывод должен быть такой: (развернуть) ⬇️</summary>
     
```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-11-23 20:44:30 +10; 2 days ago
       Docs: man:nginx(8)
   Main PID: 787 (nginx)
      Tasks: 2 (limit: 1030)
     Memory: 16.9M
        CPU: 2min 32.561s
     CGroup: /system.slice/nginx.service
             ├─787 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─788 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Nov 23 20:44:29 server name systemd[1]: Starting A high performance web server and a reverse proxy server...
```
</details>   
     
* :four: Редактирование конфигурационного файла `nginx` `default` в дирректории `sites-available`
```bash
sudo nano etc/nginx/sites-available/default
```

Необходимо заменить файл `default` [новой версией](https://github.com/OlegSoluyanov/shadowsocks-v2ray-nginx-cloudflare/blob/02f50aeb22cbd5d3e03b5d62e5c877a16efad375/default.txt "новый файл default") для проксирования трффика через домен example.com размещенный за `СDN Cloudflare`

**`Внимание!`**
* * `example.com` нужно заменить на свое доменное имя размещенное за **cloudflare**
* * `xxxxx` в диррективе `location` необходимо заменить на `номер порта` из строки `server port` конфигурационного файла **/etc/shadowsocks/shadowsocks-rust.json**
* * `example_cert.pem` необходимо заменить на название файла своего `ssl` сертификата
* * `example_key.key` необходимо заменить на название файла своего `ssl key` сертификата

<details>
<summary> Возможные ошибки: ..... ⬇️</summary>
     
* * Неправильный `путь` до любого из сертификатов
* * Неправильные `имена файлов` сертификатов
* * Неправильный `порт` сервера shadowsocks В диррективе `proxy_pass`
* * синтаксические ошибки в файле
* * настройки домена в кабинете `cloudflare`
</details>

* :five: Редактирование дефолтного файла конфигурации `nginx.conf` в дирректории `nginx`
```bash
sudo nano /etc/nginx/nginx.conf
```

Необходимо заменить файл `nginx.conf` [новой версией](https://github.com/OlegSoluyanov/shadowsocks-v2ray-nginx-cloudflare/blob/4eb039187ba9df83f09335211d7b3826eae44bfd/nginx.conft "новый файл nginx.conf")

<details>
<summary> ВНИМАНИЕ!: ..... ⬇️</summary>
     
* * Nginx будет выполнять все конфигурации указанные в диррективе `include`, в данной версии - все файлы из дирректории `/etc/nginx/conf.d` и `/etc/nginx/sites-available`. Если в `conf.d` есть другие файлы конфигурации нужно заккоментировать строку `include` c этой дирректорией
* * Включить логгирование, расположение и название файла лога можно в блоке "Настройки логгирования"
</details>

* :six: Запрет входящих запросов напрямую в сервер shadowsocks по ip минуя `nginx`. (все запросы будут приходить по домену за cloudflare
```bash
sudo nano /etc/shadowsocks/shadowsocks-rust.json
```

В первой строке необходимо указать `"server": "127.0.0.1",` - так shadowsocks будет доступен только на `localhost`

* :seven: Проверка корректности конфигурации `nginx`
```bash
sudo nginx -t
```
Вывод должен быть такой:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Если в конфигурации есть ошибка nginx в выводе отобразит номер строки

* :eight: Перезапуск `nginx` и вывод статуса
```bash
sudo systemctl restart nginx && sudo systemctl status nginx
```
<details>
<summary>Вывод должен быть такой: (развернуть) ⬇️</summary>
     
```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-11-23 20:44:30 +10; 2 days ago
       Docs: man:nginx(8)
   Main PID: 787 (nginx)
      Tasks: 2 (limit: 1030)
     Memory: 18.4M
        CPU: 3min 1.443s
     CGroup: /system.slice/nginx.service
             ├─787 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─788 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Nov 23 20:44:29 server name systemd[1]: Starting A high performance web server and a reverse proxy server...
```

 
