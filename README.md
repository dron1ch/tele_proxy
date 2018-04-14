# Установка тачки

## Предисловие
*Мне казалось самым логичным воспользоваться сервисами любого крупного зарубежного провайдера, поэтому я выбрал amazon. Шансы, что его заблочат, есть - но тогда у нас будут гораздо более серьезные проблемесы, чем блокировка телеги. Энивей...*

## Поднимаем сервер
- Регаемся на [Amazon](https://aws.amazon.com)
- Заходим в меню services, выбираем ec2
- Слева в меню выбираем "instances"
- Кликаем launch instance, из предложенных выбираем Ubuntu Server 16.04
- Конфиги оставляем базовыми

## Настраиваем группу
- Идем в Security Groups
- Тыкаем в create security group
- Указываем название группы
- В inbound-правила добавляем два правила:
  - Custom TCP Rule, port range удобный вам, желательно выше 9000, source указываем статический ip, если вам его предоставляет ваш провайдер и вы не ходите в телегу с телефона и если вы не хотите шарить свой сервер ни с кем. В противном случае ставим 0.0.0.0/0.
  - Добавляем еще одно правило - на SSH подключение (протокол tcp, порт 22, айпишник тот же, что и выше).
  - Outbound не трогаем

## Подключение к удаленной машине
- После этого логинимся через SSH
  - [Linux](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)
  - [Windows](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html)

## Установка SOCKS-сервера
*Т.к. в стандартном репо почему-то лежит какой-то крокодил (dante-server), мне пришлось собирать данте из сырцов. Алгоритм приведен ниже:*

- `mkdir dante_src && cd ./dante_src`
- `wget https://www.inet.no/dante/files/dante-1.4.2.tar.gz`
- `tar -xvf dante-1.4.2.tar.gz`
- `sudo apt-get update`
- `cd dante-1.4.2`
- `sudo apt-get install gcc make`
- `sudo mkdir /home/dante`
- `sudo ./configure —prefix=/home/dante`
- `sudo make`
- `sudo make install`

Бинарь, который мы будем запускать ==> `/home/dante/sbin/sockd`

## Конфигурируем сервер
- Нужно отконфигурить вот этот файл `/etc/sockd.conf`
  - `sudo vim /etc/sockd.conf` (Если файл не пустой - все закомментировать)

- Добавим:
```logoutput: /var/log/socks.log

internal: eth0 port = 8565
external: eth0

method: username
user.privileged: root
user.notprivileged: nobody

client pass {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: error connect disconnect
}


client block {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: connect error
}

pass {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: error connect disconnect
}

block {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        log: connect error
}
```

- Запустим Данте
  - `sudo /home/dante/sbin/sockd -D`

- Проверим, что он включился
  - `netstat -tulp`

- Процесс можно убить так
  - `sudo pkill sockd`

- Добавим юзера, под которым будем ходить
  - `sudo useradd -s /sbin/nologin "ВАШ ЛОГИН"`

- Добавим ему пароль, под которым будем логиниться
  - `sudo passwd "ВАШ ЛОГИН"`

- Прикрутим автозапуск
  - `sudo mkdir /home/dante/scripts && cd /home/dante/scripts`

- Создадим файл start.sh
  - `sudo touch start.sh`

- В start.sh напишем
```
#!/bin/bash
sleep 10
/home/dante/sbin/sockd
```

- Сделаем их исполняемыми
-  `sudo chmod +x start.sh`

- Настроим планировщик
  - `sudo crontab -e`

- Добавим запланированный запуск
  - `@reboot /home/dante/scripts/start.sh > /dev/null 2>&1`
