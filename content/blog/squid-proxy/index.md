---
title: "Настройка кеширующего squid-прокси"
date: 2024-02-08T15:40:20+04:00
draft: false
tags: ["linux", "network", "squid", "proxy"]
categories: ["blog"]
---

В этой заметке разберемся, как поднять прокси для `dnf`/`apt` или в целом для http/https и направить траффик через него.

Для начала расскажу зачем вообще занялся этим, так как проксировать траффик можно кучей способов, почему именно
симпл-димпл-прокси, а не просто VPN или что-то такое.

## Вместо вступления

У нас было:

- Компьютер, подключённый в локальную сеть через маршрутизатор и к внешней сети.
- Сервер, на котором крутятся базы данных, который подключён в локальную сеть и не имеющий доступ к внешней;
- Сервер, на котором крутятся приложения, который подключен в локальную сеть и не имеющий доступ к внешней.

{{< alert "comment" >}}
На компьютере настроены маршруты,
чтобы только определенный траффик шел в локальную сеть (запросы типа 192.168.ххх.ххх и похожие), а остальной траффик
тёк через внешнюю сеть. Компьютер смотрит на внешнюю сеть **через VPN** чтобы иметь безопасный доступ до моего локального
домашнего сервера (он не имеет выделенного IP, а с [ngrok](https://ngrok.com/) не хочется заморачиваться). Cхема замудренная, но
что стоит вынести из этого, что это **единственный** компьютер в сети, который может смотреть на внешнюю сеть.
Таким образом мы можем задавать жесткие правила, чтобы остальная (локальная) сеть могла ходить только на разрешенные
зеркала с пакетами для сборки сервисов (rubygem-зеркало, apt-зеркало), размещенные на моем домашнем сервере.
{{< /alert >}}

~~Я просто в край \_\_\_\_\_\_\_\_ бегать с флешкой, подключать монитор к серверам и работать руками прям там, или же
заливать на какую-то шару необходимые пакеты, выкачивать их и возиться с установкой через SSH, а потом еще и какой-то
зависимости как обычно не хватит и ничего не поставить и тд.~~ Короче, кто хоть раз сталкивался с автономно работающим
сервером, на который требуется установить какой-то пакет сейчас явно почувствовал жжение чуть ниже поясницы. Хочется
простого и понятного `dnf install -y the-best-app-pkg`.

## Начальная конфигурация сети

В моём примере будет несколько маршрутизаторов в одной локальной сети, это нужно будет в дальнейшем, чтобы проксировать
траффик из одной "локальной" сети в другую.

На компьютере настроены маршруты, чтобы только определенный траффик шел в локальную сеть (запросы типа 192.168.ххх.ххх
и похожие), а остальной траффик тёк через внешнюю сеть.

```bash
sudo ip route add 192.168.10.0/24 dev wlo1 via 192.168.90.1
sudo ip route add 172.15.0.0/24 dev wlo1 via 192.168.90.1
```

Где:

- `192.168.10.0/24` локальная сеть **до моего маршрутизатора**;
- `172.15.xxx.0/24` локальная сеть **до маршрутизатора сети** `192.168.10.0/24`;
- `192.168.90.0/24` локальная сеть **после моего маршрутизатора** (в которой и находится мой компьютер);
- `192.168.90.1` **gateway** для маршрутов на моём компьютере.

Следите за руками, как говорится, потому что важно не запутаться на этом этапе из какой сети в какую мы будем двигать
траффик.

## Кто такой Squid

![Логотип](logo.webp)

[Squid-proxy](https://www.squid-cache.org/) - это популярный, высокопроизводительный и бесплатный прокси-сервер с открытым исходным кодом, предназначенный для
кэширования и фильтрации трафика. Он работает как посредник между пользователями и интернетом, поддерживая HTTP, HTTPS и FTP,
что ускоряет загрузку сайтов, оптимизирует пропускную способность сети и повышает безопасность корпоративных сетей.

## Настраиваем squid

### Устанавливаем пакет

Будем настраивать прокси на машине с fedora. Нам требуется установить 5-ю версию squid:

```bash
sudo dnf install -y squid-7:5.8-1.fc38.x86_64
```

Возможно настройка адекватно будет работать и на 6 и на 7 версии, но мы проверили только на 5.

### Генерируем сертификаты

Для поддержки HTTPS нам требуется сгенерировать SSL-сертификаты.

Предварительно установим OpenSSL, если его не было в системе:

```bash
sudo dnf install -y openssl
```

Далее создадим папку для сертификатов, перейдем в нее и сгенерируем SSL-сертификат:

```bash
sudo mkdir /etc/squid/ssl_cert && \
 sudo cd /etc/squid/ssl_cert && \
 sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout myCA.pem -out myCA.pem
```

В процессе генерации сертификата нас попросят заполнить регион, область, страну, организацию, домен и нашу почту, заполняем примерно так:

```bash
Country Name (2 letter code) [XX]:RU
State or Province Name (full name) []:Leningrad Oblast
Locality Name (eg, city) [Default City]:St. Petersburg
Organization Name (eg, company) [Default Company Ltd]:local
Organizational Unit Name (eg, section) []:local
Common Name (eg, your name or your server's hostname) []:proxy.local
Email Address []:proxy@server.local
```

Далее даем права на папку с сертификатами squid:

```bash
chown -R :squid /etc/squid/ssl_cert
```

Далее обновляем нашу базу ssl-сертификатов:

```bash
sudo rm -rf /var/lib/ssl_db /var/spool/squid/ssl_db
 sudo /usr/lib64/squid/security_file_certgen -c -s /var/lib/ssl_db -M 4MB && \
 sudo /usr/lib64/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 4MB
```

На выходе должны получить:

```bash
Initialization SSL db...
Done
```

Проверяем какие права имеет эта папка:

```bash
sudo ls -lah /var/spool/squid | grep ssl_db
```

Если видим такую картину:

```bash
drwxr-xr-x. 1 root root   36 Oct 21 08:39 ssl_db
```

Следует поменять права:

```bash
sudo chown -R squid:squid /var/spool/squid/ssl_db
```

Если же:

```bash
drwxr-xr-x. 1 squid squid   36 Oct 21 08:39 ssl_db
```

То все впорядке, можно ничего не менять.

### Настройка selinux для squid

Добавим новые политики:

```bash
sudo semanage fcontext -a -t squid_conf_t '/var/spool/squidssl_db/index.txt' && \
 sudo semanage fcontext -a -t squid_conf_t '/var/spool/squid/ssl_db/size' && \
 sudo /sbin/restorecon -v /var/spool/squidssl_db/index.txt && \
 sudo /sbin/restorecon -v /var/spool/squidssl_db/size
```

После этого требуется перезагрузить операционную систему, чтобы политики вступили в силу.

### Конфигурация squid

_Можно сразу взять мой конфиг, заменить им свой и перейти к созданию каталогов для кеша._

```bash
acl localnet src 0.0.0.1-0.255.255.255 # RFC 1122 "this" network (LAN)
acl localnet src 10.0.0.0/8  # RFC 1918 local private network (LAN)
acl localnet src 100.64.0.0/10  # RFC 6598 shared address space (CGN)
acl localnet src 169.254.0.0/16  # RFC 3927 link-local (directly plugged) machines
acl localnet src 172.16.0.0/12  # RFC 1918 local private network (LAN)
acl localnet src 192.168.0.0/16  # RFC 1918 local private network (LAN)
acl localnet src fc00::/7        # RFC 4193 local private network range
acl localnet src fe80::/10       # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80  # http
acl Safe_ports port 21  # ftp
acl Safe_ports port 443  # https
acl Safe_ports port 70  # gopher
acl Safe_ports port 210  # wais
acl Safe_ports port 1025-65535 # unregistered ports
acl Safe_ports port 280  # http-mgmt
acl Safe_ports port 488  # gss-http
acl Safe_ports port 591  # filemaker
acl Safe_ports port 777  # multiling http

http_access allow all
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager
http_access allow localhost
http_access deny to_localhost
http_access deny to_linklocal
http_access deny all

http_port 3128 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/ssl_cert/myCA.pem
ssl_bump peek all
ssl_bump bump all

cache_dir ufs /var/spool/squid 4096 32 256
coredump_dir /var/spool/squid

refresh_pattern ^ftp:  1440 20% 10080
refresh_pattern ^gopher: 1440 0% 1440
refresh_pattern -i (/cgi-bin/|\?) 0 0% 0
refresh_pattern .  0 20% 4320
```

После блока `acl Safe_ports` разрешаем все соединения (эта настройка имеет самый высокий преоритет, так как находится выше всех остальных правил):

```bash
http_access allow all
```

Ищем строку `http_port 3128` и меняем настройку на эту:

```
http_port 3128 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/ssl_cert/myCA.pem
ssl_bump peek all
ssl_bump bump all
```

Далее настроим размер кеша. Ищем строку `cache_dir` и меняем ее на:

```bash
cache_dir ufs /var/spool/squid 4096 32 256
```

Где:

- `4096` -- размер кеша в мегабайтах;
- `32` -- количество каталогов первого уровня, которое будет создано для размещения кэша;
- `256` -- количество каталогов второго уровня, которое будет создано для размещения кэша.

Сохраняем и останавливаем сервис, теперь нужно создать все нужные каталоги:

```bash
sudo systemctl stop squid && \
 sudo squid -z
```

Запускаем сервис и смотрим логи, все должно завестись без ошибок:

```bash
sudo systemctl restart squid.service && \
 sudo systemctl status squid.service
```

### Пробрасываем порты на роутере

Так как машина с прокси стоит в сети ЗА маршрутизатором (сеть `192.168.90.0/24`), а наши целевые машины стоят в сети `192.168.10.0/24`, на маршрутизаторе следует прокинуть порты. В нашем случае используется Mikrotik c RouterOS v7.6.

Для проброса портов идем в WebFig -> IP -> Firewall -> NAT, нажимаем Add New и добавляем новое правило, указав настройки:

```bash
Chain: dstnat
Protocol: 6 (tcp)
Dst. Port: 3128
In. Interface: ether1 #Это мой интерфейс, куда приходит сеть 192.168.10.0
Action: dst-nat
To Adresses: 192.168.90.250 #Это адрес машины с прокси в сети 192.168.90.0
To Ports: 3128
```

## Проксируем трафик

Настроим целевые машины, с которыз должен идти запрос на прокси и дальше в сеть до зеркал.

Наша машина, на которой крутится прокси, в сети `192.168.10.0/24` доступна по адресу `192.168.10.250` и порту `3128`,
следовательно адрес нашей машины с прокси будет `192.168.10.250:3128`. _Если тут стало что-то непонятно, обратитесь к начальной конфигурации,
где освещалась схема сети._

### Настройка пакетных менеджеров

Логинимся по ssh или панель управления сервером, в терминале прокидываем переменную окружения:

```bash
export HTTP_PROXY="http://192.168.10.250:3128" && \
 export HTTPS_PROXY="http://192.168.10.250:3128"
```

В случае с apt нужно дополнительно подкинуть прокси в файл настроек. Создаем файл `proxy.conf`:

```bash
sudo vim /etc/apt/apt.conf.d/proxy.conf
```

И вставляем конфиг:

```bash
Acquire::http::Proxy "http://192.168.10.250:3128";
Acquire::https::Proxy "http://192.168.10.250:3128";
```

### Настройка Docker

Добавляем настройку для сервиса:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

Добавляем наш прокси в настройки сервиса:

```bash
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
```

```bash
[Service]
Environment="HTTP_PROXY=http://192.168.10.250:3128"
Environment="HTTPS_PROXY=http://192.168.10.250:3128"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"
```

Перезапускаем демоны и перезапускаем Docker:

```bash
sudo systemctl daemon-reload && \
 sudo systemctl restart docker
```

### Настройка Yarn

По умолчанию [Yarn](https://yarnpkg.com/) не знает, что у вас используется прокси и вы будете получать ошибку, вроде:

```bash
➤ YN0000: · Yarn 4.9.1
➤ YN0000: ┌ Resolution step
➤ YN0000: └ Completed
➤ YN0000: ┌ Fetch step
➤ YN0000: ⠹ ==============------------------------------------------------------------------
    at ClientRequest.<anonymous> (/home/<username>/.cache/node/corepack/v1/yarn/4.9.1/yarn.js:147:14258)
    at Object.onceWrapper (node:events:622:26)
    at ClientRequest.emit (node:events:519:35)
    at c.emit (/home/armstrong/.cache/node/corepack/v1/yarn/4.9.1/yarn.js:142:14855)
    at emitErrorEvent (node:_http_client:104:11)
    at TLSSocket.socketErrorListener (node:_http_client:518:5)
    at TLSSocket.emit (node:events:507:28)
    at emitErrorNT (node:internal/streams/destroy:170:8)
    at emitErrorCloseNT (node:internal/streams/destroy:129:3)
    at process.processTicksAndRejections (node:internal/process/task_queues:90:21)AggregateError [EHOSTUNREACH]:
    at internalConnectMultiple (node:net:1139:18)
    at internalConnectMultiple (node:net:1215:5)
    at afterConnectMultiple (node:net:1714:7)
```

При этом, как в случае и с Docker, у вас уже были прокинуты все прокси...
Сами разработчики [Yarn](https://yarnpkg.com/), по непонятной для меня причине, [предлагают использовать](https://yarnpkg.com/configuration/yarnrc#httpProxy) глобальный или локальный (в проекте) файл настроек `.yarnrc.yml`, и в нем указывать нужные нам настройки. Это не совсем удобно, тем более у Yarn очень хорошая CLI. Мы просто сделаем так:

```bash
yarn config set httpProxy http://192.168.20.78:3128 && \
 yarn config set httpsProxy http://192.168.20.78:3128
```

{{< alert "note" >}}
Если у вас Yarn < 4, например v1.22.xx, ключи будут `http-proxy` и `https-proxy` соответственно.
{{</ alert >}}

## Благодарности серому волшебнику

Если текст был полезен и ты не можешь усмирить желание быть благодарным, то можешь:

Воспользоваться моей реферальной ссылкой на DigitalOcean:

{{< digitalocean >}}

Или же закинуть монету в мой кошелёк (USDT и TRX кошельки одинаковые, да, это не ошибка):

**Tether (TRC-20, USDT):**

```markdown
TYvFYUV3h5HwqfyTxskGQK7nDbUHTcwPn2
```

**Tron (TRX):**

```markdown
TYvFYUV3h5HwqfyTxskGQK7nDbUHTcwPn2
```

**Monero (XMR):**

```markdown
4AbxbT9vrNQTUDCQEPwVLYZq2zTEYzNr9ZzTLaq9YcwVfdxwkWjZ6FsewuXVDXPk7x2rE6FZACmLePPgJEcY4rm1GSHkwTZ
```
