---
title: "Запуск Python приложения в качестве службы на Linux-сервер с systemd или причём тут экзорцизм"
date: 2022-07-19T10:00:20-08:00
draft: false
tags: ["dev", "python", "linux"]
categories: ["blog"]
---

Рассмотрим, как демонизировать наше python-приложение под Linux.

## Дьявол в деталях

{{< alert >}}
В начале сотворил он небо и землю
На второй день разделил он свет и тьму

...

На седьмой день разобрался он с systemd, запустил в фоне службы, автоматизирующие процессы, и больше никогда не был online.
{{< /alert >}}

Привет-привет, оклахома кидс! В этой заметке мы будем разбираться, что нужно делать дальше, когда мы написали приложение и у нас есть сервер.

К сожалению, недостаточно просто выгрузить приложение на удаленный сервер и запустить скрипт, например так:

```bash
Python3 ~/MyFirstDeploy/best_fucking_idea.py
```

Приложение запустится и даже будет работать. Но есть одно но: после того, как вы вышли с сервера, закрыв ssh-соединение, вы закрыли все запущенные вами приложения в этой сессии, в том числе и запущенный скрипт, потому что вы закрыли не только [ssh](https://manpages.org/ssh)-соединение, а вообще закрыли сессию, которая была создана для этого пользователя.

Что бы такого не происходило приложение требуется демонизировать и вынести его выполнение за рамки конкретного пользователя. Это можно сделать как средствами операционной системы (этот метод описан в этой заметке), так и [запрограммировать руками](https://stackoverflow.com/questions/16420092/how-to-make-python-script-run-as-service/16420140#16420140) (~~но это пипец какой костыль, не находите?~~)

В операционных системах на базе Linux организация пользователей координально отличается от Windows. Это является сильной стороной Linux-base систем и одновременно с этим, по первости, добавляет нам проблем в копилку ~~непонимания и ахуевания~~, но освоившись вы почувствуете все приемущества такого подхода к разграничению пользователей.

{{< alert >}}
Кстати, благодаря такому подходу к организации пользователей в Linux-base системах и появился [Docker](https://docs.docker.com/), который стал стандартом дефакто в современном мире web-разработки, привнеся в неё контейнеризацию, которая позволяет в 1 клик поднять огромный сервис на тысячах машинах, а вторым кликом почти мгновенно масштабировать мощности сервиса.
{{< /alert >}}

В операционных Linux-base системах все привычные в windows службы (service) называют демонами (daemon). Здесь и дальше под словом демон я буду подразумевать службу, которая выполняется в фоновом режиме.

[Демон (daemon)](https://en.wikipedia.org/wiki/Daemon_(computing))  — это всего лишь компьютерная программа, которая под управлением систем инициализации блоков способна запускать процессы в фоновом режиме. И для того, чтобы наше приложение могло запускаться в виде демона / быть демоном, его нужно демонизировать. Этим мы и займёмся дальше, но сначала мы обратим внимание на заголовок статьи. Причём тут какой-то systemd?

[Systemd](https://manpages.org/systemd) — это системный диспетчер. Аналогию можно провести с диспетчером задач, который является частью менеджера процессов в windows. Этот парень позволяет нам управлять почти всеми демонами в нашей операционной системе, помогает создавать конфигурации этих демонов и прч. Не все операционные системы на базе Linux используют systemd, так что это нужно учитывать. Systemd в свою очередь работает с помощью systemctl.

[Systemctl](https://manpages.org/systemctl) — это инструмент центрального управления для контроля системы инициализациии. С помощью этого парня мы будем запускать, останавливать и проверять статус нашего процесса.

Вводную получили, поехали стучать по клавишам!

## Мой лучший друг — демон

Итак, у нас было:

- Приложение на Python
- Сервер на linux с systemd
- Огромное желание что-то запускать

Я буду использовать свой droplet на digital ocean, вы можете использовать хоть виртуальную машину на своём компьютере, разницы 0.

Итак, мы зашли по ssh на наш сервер, предварительно загрузили на этот сервер файлы приложения, например по этому пути:

```bash
/root/telegram-bot
```

И нам нужно, чтобы запускался скрипт `[bot.py](http://bot.py)`. Это точка входа в наше приложение.

Получаем такой путь до нашей точки входа: `/root/telegram-bot/bot.py`. *Этот путь я буду дальше использовать для примеров.*

Для того, что бы создать демона нашего скрипта, требуется сделать ряд вещей:

1. Создать файл `<name>.service`, в котором будет описана конфигурация нашего демона
2. Попросить systemctl перечитать все файлы настроек демонов
3. Рассказать systemctl о нашем демоне (создаст [semilink](https://www.freecodecamp.org/news/symlink-tutorial-in-linux-how-to-create-and-remove-a-symbolic-link/))
4. Запустить демона!

План понятен и приятен. Но где вообще живут эти демоны и чёэтатакое?!

Заходим на наш сервер:

```bash
ssh <user>@<server_ip_address>
```

Идем смотреть на демонов:

```bash
cd /etc/systemd/system/ && ls -lah ./
```

И видим примерно следующую картину:

```bash
drwxr-xr-x 21 root root 4.0K May  6 08:43 .
drwxr-xr-x  6 root root 4.0K May  6 05:50 ..
...
drwxr-xr-x  2 root root 4.0K May  6 05:50 sshd-keygen@.service.d
lrwxrwxrwx  1 root root   31 Jan 31 22:28 sshd.service -> /lib/systemd/system/ssh.service
drwxr-xr-x  2 root root 4.0K Jan 31 22:28 sysinit.target.wants
lrwxrwxrwx  1 root root   35 Jan 31 22:26 syslog.service -> /lib/systemd/system/rsyslog.service
drwxr-xr-x  2 root root 4.0K Jan 31 22:28 timers.target.wants
lrwxrwxrwx  1 root root   41 Jan 31 22:28 vmtoolsd.service -> /lib/systemd/system/open-vm-tools.service
```

Файлы `<name>.service` — описание демона. Там лежит вся его настройка

Папки `<name>.service.d` — это папка, в которой лежит дополнительная настройка демона. По умолчанию она не создается, но мы в нашем примере будем её создавать и положим туда часть настройки демона.

Создадим новый файл конфигурации демона для нашего приложения.

### Создаём конфигурацию демона

Без лишних слов, будучи в папке `/etc/systemd/system/`

```bash
vim bot.service
```

Откроется редактор `[VIM](https://www.vim.org/)`, в котором мы будем описывать нашего демона:

```bash
[Unit]
Description=My telegram bot
After=multi-user.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/bin/python3 /root/telegram-bot/bot.py

[Install]
WantedBy=multi-user.target
```

Вот в целом - то и всё. После обновления `systemctl` можно запустить сервис. Но мы разберемся немного подробнее, за что отвечает каждый ключ в каждой секции.

Секция `Unit`:

- `Description` — Просто описание вашего сервиса, которое будет выведено при выполнении `systemctl status <name>` в статус-строке, после названия демона. Может быть вообще произвольным.
- `After` — Определяет порядок запуска блоков. Здесь `multi-user.target` указывает на цель, в которой описаны другие службы. Т.е. наш демон запустится только после того, как все службы, указанные в `multi-user.target` были успешно запущены. Например, в `multi-user.target` важные системные службы, такие как `NetworkManager` (`NetworkManager.service`) или `D-Bus` (`dbus.service`) и инструкция запуска другого целевого блока с именем `basic.target`.

Секция `Service`:

- `Type` — Настраивает тип демона, этот ключ на прямую влияет на то, как будет выполняется инструкция в `ExecStart`.
    - `simple` — Значение по умолчанию. Процесс начинается с инструкции `ExecStart` и является основным процессом
    - `forking` — Процесс начинается с `ExecStart` и порождает дочерний процесс, который замещает собой родительский процесс.
    - `oneshot` — аналогичен `simple`, только процесс завершается до запуска последующих блоков.
    - ... и т.д. Ссылку на описание всех ключей оставлю ниже. Нас же интересует только `simple`.
- `Restart` — Указывает на то, будет ли перезапущена служба после завершения процесса. Указываем `always`, что бы не быть неприятно удивленными каким-нибудь вечером.
- `ExecStart` — Инструкция, которая исполнится при запуске демона. В данном примере я указал абсолютные пути до Python и до моего скрипта, который является точкой входа в приложение. Можно упростить до `python3 /root/telegram-bot/bot.py`, но лучше перестраховаться на случай, если на сервере не прокинуты `$PATH`.

Секция `Install`:

- `WantedBy` — указывает на зависимости, причем делает это мягче, чем ключ `RequareBy`, который бы прервал выполнение, в случае неудачного запуска одной из служб.

[Описание всех ключей тут.](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/chap-managing_services_with_systemd#sect-Managing_Services_with_systemd-Unit_File_Structure)

### Прячем от чужих глаз чувствительные данные

Бывает такое, что приложение использует какие-то пароли / ключи / токены и тд, которые не стоит держать в открытом виде в скрипте или где-то рядом с ним, что бы случайно не выгрузить на github в публичный репозиторий. В таком случае можно восппользоваться переменными окружения или `Environment variable`.

Так как наше приложение будет запущено в качестве демона, то нельзя просто так прокинуть эти переменные, допустим, пользователю `root`. Мы будем использовать дополнительную настройку нашего демона, в которой переопределим этот блок (вспоминаем про папки `<name>.service.d`)

Для этого воспользуемся аргументом `edit` у `systemctl` и создадим для нашей настройки `bot.service` доп настройку с нашими переменными окружения:

```bash
systemctl edit bot.service
```

Откроется редактор `nano` с пустым файлом. Внесем туда свои переменный, переопределив блок `[Service]`:

```bash
[Service]
Environment="telegram=<your_token>"
Environment="openweathermap=<your_token>"
Environment="resources_path=/root/telegram-bot/resources"
```

Далее нажимаем `Ctrl + O`, чтобы записать наш файл на диск, назовем его `local.conf`, и нажимаем `Ctrl + X` для выхода.

Можем проверить, что у нас получилось:

```bash
.
├── bot.service
├── bot.service.d
│   └── local.conf
```

Видим наш файл настройки `bot.service` и дополнительная конфигурация `local.conf` в папке `bot.service.d`.

Теперь при запуске службы будут подтянуты переменные окружения и ваш код приложения, который требовал эти самые `environment variable` будет отрабатывать как положено.

### Запускаем и проверяем статус нашего демона

Мы сделали все нужные настройки, теперь надо рассказать нашему `systemctl` о нашем демоне, добавить его в автозапуск и запустить наконец-то.

Обновляем информацию о демонах:

```bash
systemctl daemon-reload
```

Добавляем в автозагрузку:

```bash
systemctl enable bot.service
```

Запускаем:

```bash
systemctl start bot.service
```

Проверим статус, запустился ли вообще сервис или упал:

```bash
systemctl status bot.service
```

Должны увидеть вот такое сообщение:

```bash
● bot.service - My telegram bot
     Loaded: loaded (/etc/systemd/system/bot.service; enabled; vendor preset: enabled)
     Drop-In: /etc/systemd/system/bot.service.d
             └─local.conf
		Active: active (running) since Fri 2022-05-06 09:17:11 UTC; 24h ago
...
```

Если в статусе `Failed`, значит либо что-то с приложением, либо что-то с настройкой. Можно пойти почитать логи коммандой `[dmesg](https://manpages.org/dmesg)` или `cat /var/log/syslog`.

## Полезные ссылки

- [Простейший способ превратить скрипт в демона](https://lincolnloop.com/blog/joy-upstart/), [Linkolnloop](http://lincolnloop.com), Graham King
- [Руководство для начинающих по systemctl](https://bitlaunch.io/blog/a-beginners-guide-to-systemctl/), BitLaunch
- [Использование Systemctl для управления системой и юнитами](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units), библиотека [Digital Ocean](http://digitalocean.com)
- [Управление службами с помощью systemd](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/chap-managing_services_with_systemd), [документация Red Hat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/)
- [Описание всех ключей файла конфигурации службы](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/chap-managing_services_with_systemd#sect-Managing_Services_with_systemd-Unit_File_Structure), [документация Red Hat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/)
- [Как создать и удалить символическую ссылку](https://www.freecodecamp.org/news/symlink-tutorial-in-linux-how-to-create-and-remove-a-symbolic-link/), freecodecamp, [Dillion Megida](https://www.freecodecamp.org/news/author/dillionmegida/)
- [Как читать и устанавливать переменные среды и  shell в Linux](https://www.digitalocean.com/community/tutorials/how-to-read-and-set-environmental-and-shell-variables-on-linux), библиотека [Digital Ocean](http://digitalocean.com)

## Благодарности серому волшебнику

Если текст был полезен и ты не можешь усмирить желание быть благодарным, то можешь: 

Воспользоваться моей реферальной ссылкой на TimeWeb:

{{< timeweb >}}

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
