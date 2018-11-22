[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/yusorokin_infra.svg?branch=master)](https://travis-ci.com/Otus-DevOps-2018-09/yusorokin_infra)

# yusorokin_infra

yusorokin Infra repository


## Homework 3
### Самостоятельное задание
Подключение к целевому хосту через бастион-хост одной командой:
```sh
$ ssh -A -J <user>@<bastion_IP> <someinternalhost_IP>
```

**Дополнительное задание**

Подключение к целевому хосту с использованием алиаса:
```sh
$ cat <<TXT > ~/.ssh/config
> Host someinternalhost
>   HostName <someinternalhost_IP>
>   ForwardAgent yes
>   ProxyJump <user>@<bastion_IP>
> TXT
$ ssh someinternalhost
```
```sh
bastion_IP = 104.155.80.250
someinternalhost_IP = 10.142.0.2
```

## Homework 4
```sh
testapp_IP = 35.189.238.37
testapp_port = 9292
```

### Дополнительное задание

**Startup-scipt из файла:**

```sh
gcloud compute instances create reddit-app \
  --boot-disk-size=10GB \
  --image-family ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=g1-small \
  --tags puma-server \
  --restart-on-failure \
  --metadata-from-file startup-script=./startup_script.sh
```

**Startup-scipt по ссылке:**

```sh
gcloud compute instances create reddit-app \
  --boot-disk-size=10GB \
  --image-family ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=g1-small \
  --tags puma-server \
  --restart-on-failure \
  --metadata startup-script-url=https://gist.github.com/yusorokin/e529f16590f13b28106aaa1ea29d3ee4/raw/48acffb1c4e75df64a86c4e430974ca4fe01a95f/startup_script.sh
```

**Создание правила firewall через gcloud:**

```sh
gcloud compute firewall-rules create default-puma-server \
    --action allow \
    --target-tags puma-server \
    --source-ranges 0.0.0.0/0 \
    --rules tcp:9292
```

## Homework 5

### Основное задание
* Создан шаблон базового образа packer-base;
* Создан сам базовый образ;
* Развернут инстанс из этого образа;
* Задеплоено приложение на созданном инстансе;
* Шаблон дополнен параметрами и файлом параметров.

### Задание со *
* Создан шаблон baked образа immutable.json на основе базового образа, созданного в основном задании;
* В шаблоне использован измененный скрипт deploy.sh и новый скрипт systemd_service.sh, который создает файл сервиса puma и запускает его через systemd;
* Создан скрипт для поднятия инстанса ВМ create-redditvm.sh из созданного образа семейства reddit-full.


## Homework 6

### Задание со *
Нашел как минимум два способа добавления ключей в проект:
1. Команда:
    ```sh
    resource "google_compute_project_metadata_item" "default" {
      project = "${var.project}"
      key = "ssh-keys"
      value = "appuser1:${file(var.public_key_path)}"
    }
    ```
2. Команда:
    ```sh
    resource "google_compute_project_metadata" "default" {
      project = "${var.project}"
      metadata {
        ssh-keys = "appuser1:${file(var.public_key_path)}"
      }
    }
    ```

Добавил три ключа командой
```sh
resource "google_compute_project_metadata_item" "default" {
  project = "${var.project}"
  key = "ssh-keys"
  value = "appuser1:${file(var.public_key_path)}appuser2:${file(var.public_key_path)}appuser3:${file(var.public_key_path)}"
}
```

Добавил ключ пользователя appuser_web через веб-интерфейс. Затем применил `terraform apply`.

Какие проблемы выявил:
* *Очевидная:* Ключ appuser_web удалился;
* *Менее очевидная:* Перебрал несколько вариантов добавления нескольких ключей и получил следующие проблемы:
* * Добавляются лишние переносы строк (`\n`) после каждого ключа и еще один для последнего ключа при следующем формате:
    ```sh
    value = <<EOF
      appuser1:${file(var.public_key_path)}
      appuser2:${file(var.public_key_path)}
      appuser3:${file(var.public_key_path)}
    EOF
    ```
    Скрин https://drive.google.com/open?id=1Wj2vz1MoV2MCg4qOQWO49cNGvwl3HVhF
* * Добавляется лишь один лишний перенос строки после последнего ключа, и выглядит нечитабельно:
    ```sh
    value = "appuser1:${file(var.public_key_path)}appuser2:${file(var.public_key_path)}appuser3:${file(var.public_key_path)}"
    ```
    Скрин https://drive.google.com/open?id=1mlX2D0ERR0NKqBDCAqGqPiyVkOmsMppk
  Возможно ничего страшного в этом и нет, но выглядит странно.

### Задание с **

### Создание балансировщика веб-сервера

Для создания балансировщика потребовалось:
* Включить инстанс веб-сервера в группу инстансов google_compute_instance_group;
* Настроить для группы инстансов перенаправление трафика на порт 9292;
* Создать google_compute_health_check по порту tcp-9292 определения работающего инстанса;
* Создать google_compute_backend_service, который перенаправляет трафик на инстанс группы google_compute_instance_group в зависимости от состояния google_compute_health_check;
* Обозначить в google_compute_backend_service именованый порт port_name = "puma-9292" для перенаправления HTTP-трафика на tcp-9292, который описан в google_compute_instance_group;
* Создать правило google_compute_url_map для перенаправления всего трафика на google_compute_backend_service;
* Создать google_compute_target_http_proxy, который перенаправляет трафик исходя из правил, заданных в google_compute_url_map;
* Создать глобальное правило перенаправления трафика (google_compute_global_forwarding_rule) по HTTP, которому и присваивается внешний ip-адрес и которое перенаправляет трафик на google_compute_target_http_proxy.

Для шаблона инастанса была создана переменная count, в которой определяется количество создаваемых инстансов.

## Homework 7

### Задание со * (1)
* Настроил remote-backend в файле backends.tf, выполнил `terraform init`;
* Перенес конфигурационные файлы из terraform/prod в другую директорию, поправил пути к модулям, запустил terraform init, запустил terraform state pull. В результате загрузился удаленный стейт-файл и вывелся в stdout;
* Запустил во временной папке terraform destroy, не подтверждая действие; внес небольшие изменения в конфигурацию в исходной папке, запустил terraform apply, на что получил сообщение о блокировке стейт-файла:
  ```
  Error: Error locking state: Error acquiring the state lock: writing "gs://backend-prod-infra-777/terraform/state/default.tflock" failed: googleapi: Error 412: Precondition Failed, conditionNotMet
  ```

### Задание со * (2)
* Добавил ресурс `null_resource` для подключаемых по условию провиженеров;
* Настроил переменную для подключения провиженера, в ресурсе обозначил count, равный этой переменной;
* Настроил провиженеры:
* * Для **app**:
* * * Копирование файла сервиса puma;
* * * Запись значения `DATABASE_URL` в переменные окружения (`~/.profile`);
* * * Запуск скрипта деплоя приложения;
* * Для **db**:
* * * Команда для изменения настроек монго `bindIp: 127.0.0.1` на `bindIp: 0.0.0.0` для разрешения внешних подключений;
* * * Команда для перезапуска сервиса монго.

## Homework 8

### Основное задание
* Установил и настроил ansible;
* Создал инвентори-файл в формате INI;
* Разбил на группы хосты в инвентори, проверил работу с группами;
* Переформатировал инвентори в YAML;
* Проверил работу модулей ping, command, service, systemd, shell и git;
* Написал первый простой плейбук.

При выполнении плейбука после удаления репозитория с удаленной машины ansible возвращает состояние changed=1, так как он действительно внес изменения на удаленной машине в отличие от предыдущего запуска плейбука, когда репозиторий уже присутствовал в нужной папке и ansible просто проверил ее наличие.

### Задание со *
* Ознакомился с документацией по использованию динамического инвентори;
* Написал баш-скрипт по созданию динамического инвентори в двух вариациях:
* * `inv_script.sh` - получает информацию об инстансах через gcloud, парсит название хостов и в зависимости от того app это или db, включает из в определенную группу. Скрипт адаптирован под конкретное задание с конкретным количеством хостов, т.к. в задании сказано "Создайте файл inventory.json в формате, описанном в п.1 и
перенесите в него записи **аналогично уже созданному inventory**.". Полученный инвентори-файл - **inventory.json**;
* * `inv_full_auto.sh` - получает через gcloud любое количество инстансов и записывает их в json без группировки, более правильный на мой взгляд. Полученный инвентори-файл - **inventory_full_auto.json**;

В обоих случаях адреса хостов передаются через переменные hostvars в _meta.

## Homework 9

### Основное задание
* Научился работать с плейбуками, шаблонами, хендлерами;
* Создал плейбук с одним сценарием для MongoDB, настройки и деплоя приложения;
* Создал один плейбук с несколькими сценариями для тех же целей;
* Создал отдельные плейбуки для MongoDB, настройки приложения и деплоя;
* Создал главный плейбук, включающий три вышеописанных;
* Выполнил первое задание со *;
* Создал плейбуки для провижининга packer;
* Синтегрировал плейбуки провижининга с файлами конфигурации packer;
* Создал новые образы packer-ом;
* Проверил плейбук site.yml.

### Задание со *
* Для dynamic inventory решил использовать gce.py;
* Настроил сервисный аккаунт в gce, сохранил на свое машине json-файл с ключами аккаунта;
* Настроил в gce.ini данные для работы с gce;
* Проиписал в ansible.cfg использование в качестве инвентори-файла gce.py;
* Настроил плейбуки на использование новый имен групп, основанных на тегах сети

## Homework 10

### Основное задание
* Создал структуру ролей с помощью `ansible init <role_name>`;
* Настроил роли db и app;
* Настроил использование ролей в созданных ранее плейбуках;
* Настроил окружения stage и prod;
* Настроил переменные окружений;
* Настроил вывод информации об окружении;
* Навел порядок в директории ansible, рассортировав файлы по нужным директориям;
* Настроил nginx с помощью коммьюнити роли jdauphant.nginx;
* Создал плейбук для создания пользователей;
* Использовал ansible vault для хранения паролей пользователей.

### Задание со *
* Настроил окружения на работу с динамическим инвентори.

### Задание со **
* Установил и настроил trytravis;
* С работой trytravis возникла проблема с часовыми поясами, он не мог найти запущенный билд. Решил проблему поправив код trytravis (https://otus-devops.slack.com/archives/CCVTC0LT1/p1542567324547500);
* Настроил .travis.yml:
* * Описал создание недостающих файлов ключей;
* * Определил переменные окружения с версиями используемых приложений;
* * Описал переименование шаблонов .example в обычные файлы;
* * Описал установкку packer, terraform, tflint, ansible, ansible-lint;
* * Описал тестирование кода вышеописанными средствами;
* * Установил бейдж со статусом билда в шапку README.
