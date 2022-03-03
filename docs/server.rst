.. role:: bash(code)
    :language: bash

Инструкция по настройке персонального VPN-сервера на базе XTLS/Xray
###################################################################

Инструкция написана для RHEL8-клонов. Проверялось на Oracle Linux 8 в Oracle Cloud. 

1. Хостинг
**********
Для запуска персонального сервера потребуется виртуальная машина с операционной системой Linux. Приобрести можно у любого облачного провайдера на Ваш вкус. Я использую Vultr, Oracle Cloud, Alibabacloud.

2. Регистрация домена
*********************

Домен подойдет любой, вопрос в регистраторе. 
Можно попробовать Cloudflare.
Заходим в панель управления и делаем A-запись указывающую на адрес виртуальной машины, которую Вы создали в п.1

3. Подготовка
*************
Обычно хостер тем или иным способом предоставляет удаленный доступ к ВМ (виртуальной машине). Если при покупке ВМ Вам предложили сразу установить в ВМ Ваш публичный ключ для SSH - настоятельно рекомендую воспользоваться этим.
Можете работать в консоли с сайта хостера, но это будет неудобно, не факт, что Вам не придется набирать все команды руками. 
Детальная настройка удаленного доступа по SSH не входит в это руководство, но я постараюсь при наличии свободного времени дописать это. 

Обновление операционной системы
===============================

.. code-block:: bash

    dnf -y update

Установка дополнительных приложений
===================================

Для удобства я ставлю mc, wget, nano, tmux.

.. code-block:: bash

    dnf install -y mc wget nano tmux


Безопасность
============
Лучше убрать из SSH возможность авторизации по паролям и перенести с порта по умолчанию (22) на любой другой случайный порт. 


4. Установка и настройка
************************

Установка nginx, certbot.
=========================

.. code-block:: bash

    dnf install -y nginx certbot

Ставим xray.

.. code-block:: bash

    bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install -u root

Получаем сертификат. domain.name - домен, который вы зарегистрировали, server - имя сервера, может быть любым. Не забудьте создать А-запись в панели управления доменом.

.. code-block:: bash

    certbot certonly -d server.domain.name

Обращаем внимание на строчки `Certificate is saved at` и `Key is saved at`. Там указаны пути к сертификату и закрытому ключу. Они понадобятся позже.

Настраиваем автоматическое обновление сертификата.

.. code-block:: bash

    crontab -e

В открывшемся редакторе (vim) нажимаем Ins и вставляем строчку.

    `0 12 * * * /usr/bin/certbot renew --quiet`

Нажимаем ESC, вводим команду **:wq** (с двоеточием).

Настраиваем firewall. ВНИМАНИЕ! У некоторых хостеров встроенный firewall внутри виртуалки выключен и правила настраиваются в панели управления ВМ! Иногда нужно настраивать и там, и там!

.. code-block:: bash

    firewall-cmd --permanent --zone=public --add-service=https
    firewall-cmd --permanent --zone=public --add-service=http
    firewall-cmd --reload

Настраиваем xray.

.. code-block:: bash

    nano /usr/local/etc/xray/config.json

Пример конфига

.. code-block:: json

    {
        "log": {
            "loglevel": "warning"
        },
        "inbounds": [
            {
                "port": 443,
                "protocol": "vless",
                "settings": {
                    "clients": [
                        {
                            "id": "<uuid>",
                            "level": 1,
                            "flow": "xtls-rprx-direct",
                            "email": "<email>"
                        }

                    ],
                    "decryption": "none",
                    "fallbacks": [
                        {
                            "dest": 80
                        },
                        {
                            "path": "/websocket",
                            "dest": 1234,
                            "xver": 1
                        }
                    ]
                },
                "streamSettings": {
                    "network": "tcp",
                    "security": "xtls",
                    "xtlsSettings": {
                        "alpn": [
                            "http/1.1"
                        ],
                        "certificates": [
                            {
                                "certificateFile": "<путь к сертификату>",
                                "keyFile": "<путь к закрытому ключу>"
                            }
                        ]
                    }
                }
            },
            {
                "port": 1234,
                "listen": "127.0.0.1",
                "protocol": "vless",
                "settings": {
                    "clients": [
                        {
                            "id": "<uuid>",
                            "level": 1,
                            "email": "<email>"
                        }
                    ],
                    "decryption": "none"
                },
                "streamSettings": {
                    "network": "ws",
                    "security": "none",
                    "wsSettings": {
                        "acceptProxyProtocol": true,
                        "path": "/websocket"
                    }
                }
            }
        ],
        "outbounds": [
            {
                "protocol": "freedom"
            }
        ]
    }



uuid генерируем на https://www.uuidgenerator.net/
email - просто для идентификации, любое слово английскими буквами

