# PostgreSQL High-Availability Cluster :elephant: :sparkling_heart:

[<img src="https://github.com/vitabaks/postgresql_cluster/workflows/Ansible-lint/badge.svg?branch=master">](https://github.com/vitabaks/postgresql_cluster/actions?query=workflow%3AAnsible-lint)
[<img src="https://github.com/vitabaks/postgresql_cluster/workflows/Yamllint/badge.svg?branch=master">](https://github.com/vitabaks/postgresql_cluster/actions?query=workflow%3AYamllint)
[<img src="https://github.com/vitabaks/postgresql_cluster/workflows/Flake8/badge.svg?branch=master">](https://github.com/vitabaks/postgresql_cluster/actions?query=workflow%3AFlake8)
[<img src="https://github.com/vitabaks/postgresql_cluster/workflows/Molecule/badge.svg?branch=master">](https://github.com/vitabaks/postgresql_cluster/actions?query=workflow%3AMolecule)
[![GitHub license](https://img.shields.io/github/license/vitabaks/postgresql_cluster)](https://github.com/vitabaks/postgresql_cluster/blob/master/LICENSE) 
![GitHub stars](https://img.shields.io/github/stars/vitabaks/postgresql_cluster)

### Production-ready PostgreSQL High-Availability Cluster (based on Patroni). Automating with Ansible.

`postgresql_cluster` автоматизирует развертывание и управление высокодоступными кластерами PostgreSQL в производственных средах. Это решение предназначено для использования на выделенных физических серверах, виртуальных машинах, а также в локальных и облачных инфраструктурах.

Вы можете найти версию этой документации с возможностью поиска и более удобной навигацией по адресу [postgresql-cluster.org](http://postgresql-cluster.org)

<a href="https://www.producthunt.com/posts/postgresql-cluster-org?embed=true&utm_source=badge-featured&utm_medium=badge&utm_souce=badge-postgresql&#0045;cluster&#0045;org" target="_blank"><img src="https://api.producthunt.com/widgets/embed-image/v1/featured.svg?post_id=583645&theme=light" alt="postgresql&#0045;cluster&#0046;org - The&#0032;open&#0045;source&#0032;alternative&#0032;to&#0032;cloud&#0045;managed&#0032;databases | Product Hunt" style="width: 250px; height: 54px;" width="250" height="54" /></a>

:trophy: **Use the [sponsoring](https://github.com/vitabaks/postgresql_cluster#sponsor-this-project) program to get personalized support, or just to contribute to this project.**

---

### Supported setups of Postgres Cluster

![postgresql_cluster](images/postgresql_cluster.png#gh-light-mode-only)
![postgresql_cluster](images/postgresql_cluster.dark_mode.png#gh-dark-mode-only)

У вас есть три схемы, доступные для развертывания:

#### 1. PostgreSQL High-Availability only

Это простая схема без балансировки нагрузки.

##### Components:

- [**Patroni**](https://github.com/zalando/patroni) это шаблон для создания собственного специализированного решения высокой доступности с использованием Python и - для максимальной доступности - распределенного хранилища конфигурации, например ZooKeeper, etcd, Consul или Kubernetes. Используется для автоматизации управления экземплярами PostgreSQL и автоматического обхода отказа.

- [**etcd**](https://github.com/etcd-io/etcd) это распределенное надежное хранилище ключевых значений для наиболее важных данных распределенной системы. etcd написан на языке Go и использует [Raft](https://raft.github.io/) алгоритм консенсуса для управления высокодоступным реплицированным журналом. Он используется Patroni для хранения информации о состоянии кластера и параметров конфигурации PostgreSQL. [What is Distributed Consensus?](http://thesecretlivesofdata.com/raft/)

- [**vip-manager**](https://github.com/cybertec-postgresql/vip-manager) (_optional, if the `cluster_vip` variable is specified_) это служба, которая запускается на всех узлах кластера и подключается к DCS. Если локальный узел владеет ключом лидера, vip-manager запускает сконфигурированный VIP. В случае обхода отказа vip-manager удаляет VIP на старом лидере, а соответствующая служба на новом лидере запускает его. Используется для обеспечения единой точки входа (VIP) для доступа к базе данных.

- [**PgBouncer**](https://pgbouncer.github.io/features.html) (optional, if the `pgbouncer_install` variable is `true`) является пулом соединений для PostgreSQL.

#### 2. PostgreSQL High-Availability with Load Balancing

Эта схема позволяет распределить нагрузку на операции чтения, а также масштабировать кластер с помощью реплик, предназначенных только для чтения.

При развертывании на облачных провайдерах, таких как AWS, GCP, Azure, DigitalOcean и Hetzner Cloud, по умолчанию автоматически создается облачный балансировщик нагрузки для обеспечения единой точки входа в базу данных (управляется переменной `cloud_load_balancer`).

Для необлачных сред, например при развертывании на собственных машинах, можно использовать балансировщик нагрузки HAProxy. Чтобы включить его, установите значение `with_haproxy_load_balancing: true` в файле vars/main.yml.

> [!NOTE]
> Ваше приложение должно поддерживать отправку запросов на чтение на пользовательский порт 5001, а на запись - на порт 5000.

Список портов при использовании HAProxy:
- порт 5000 (чтение / запись) мастер
- порт 5001 (только чтение) все реплики
- порт 5002 (только чтение) только синхронные реплики
- порт 5003 (только чтение) только асинхронные реплики

##### Components of HAProxy load balancing:

- [**HAProxy**](http://www.haproxy.org/) это бесплатное, очень быстрое и надежное решение, обеспечивающее высокую доступность, балансировку нагрузки и проксирование для TCP- и HTTP-приложений. 

- [**confd**](https://github.com/kelseyhightower/confd) Управление локальными конфигурационными файлами приложений с помощью шаблонов и данных из etcd или consul. Используется для автоматизации управления конфигурационными файлами HAProxy.

- [**Keepalived**](https://github.com/acassen/keepalived)  (_опционально, если указана переменная `cluster_vip`) предоставляет виртуальный высокодоступный IP-адрес (VIP) и единую точку входа для доступа к базам данных.
Реализация протокола VRRP (Virtual Router Redundancy Protocol) для Linux. В нашей конфигурации keepalived проверяет статус службы HAProxy и в случае сбоя делегирует VIP другому серверу в кластере.

#### 3. PostgreSQL High-Availability с Consul Service Discovery

Чтобы использовать эту схему, укажите `dcs_type: consul` в файле переменных vars/main.yml

Эта схема подходит для доступа только к мастеру и для балансировки нагрузки (с помощью DNS) для чтения между репликами. В качестве клиентской точки доступа к базе данных используется Consul [Service Discovery](https://developer.hashicorp.com/consul/docs/concepts/service-discovery) с [DNS resolving ](https://developer.hashicorp.com/consul/docs/discovery/dns).

Клиентская точка доступа (пример):

- `master.postgres-cluster.service.consul`.
- `replica.postgres-cluster.service.consul`.

Кроме того, это может быть полезно для распределенного кластера по разным дата-центрам. Мы можем заранее указать, в каком дата-центре находится сервер базы данных, а затем использовать его для приложений, работающих в том же дата-центре. 

Пример: `replica.postgres-cluster.service.dc1.consul`, `replica.postgres-cluster.service.dc2.consul`.

Требуется установка consul в режиме клиента на каждом сервере приложений для разрешения служебных DNS (или использование [forward DNS](https://developer.hashicorp.com/consul/tutorials/networking/dns-forwarding?utm_source=docs) на удаленный сервер consul вместо установки локального клиента consul).

## Совместимость
Дистрибутивы на базе RedHat и Debian (x86_64)

###### Поддерживаемые дистрибутивы Linux:
- **Debian**: 11, 12
- **Ubuntu**: 22.04, 24.04
- **CentOS Stream**: 9
- **Oracle Linux**: 8, 9
- **Rocky Linux**: 8, 9
- **AlmaLinux**: 8, 9

###### Версии PostgreSQL: 
все поддерживаемые версии PostgreSQL

:white_check_mark: проверено, работает нормально: PostgreSQL 10, 11, 12, 13, 14, 15, 16, 17

Таблица результатов ежедневного автоматизированного тестирования развертывания кластера:_
| Distribution | Test result |
|--------------|:----------:|
| Debian 11    | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_debian11.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_debian11.yml) |
| Debian 12    | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_debian11.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_debian12.yml) |
| Ubuntu 22.04 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_ubuntu2204.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_ubuntu2204.yml) |
| Ubuntu 24.04 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_ubuntu2204.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_ubuntu2404.yml) |
| CentOS Stream 9 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_centosstream9.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_centosstream9.yml) |
| Oracle Linux 8 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_oracle_linux8.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_oracle_linux8.yml) |
| Oracle Linux 9 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_oracle_linux9.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_oracle_linux9.yml) |
| Rocky Linux 8 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_rockylinux8.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_rockylinux8.yml) |
| Rocky Linux 9 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_rockylinux9.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_rockylinux9.yml) |
| AlmaLinux 8 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_almalinux8.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_almalinux8.yml) |
| AlmaLinux 9 | [![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/vitabaks/postgresql_cluster/schedule_pg_almalinux9.yml?branch=master)](https://github.com/vitabaks/postgresql_cluster/actions/workflows/schedule_pg_almalinux9.yml) |

###### Версия Ansible 
Минимальная поддерживаемая версия Ansible: 8.0.0 (ansible-core 2.15.0)

## Требования

<details><summary>Нажмите здесь, чтобы развернуть...</summary><p>

Для работы с этим плейбуком требуются права root или sudo.

Ansible ([Что такое Ansible](https://www.ansible.com/how-ansible-works/)?)

if dcs_type: «consul», пожалуйста, установите требования роли consul на управляющем узле:

`ansible-galaxy install -r roles/consul/requirements.yml`.

### Требования к портам
Список необходимых TCP-портов, которые должны быть открыты для кластера баз данных:

- `5432` (postgresql)
- `6432` (pgbouncer)
- `8008` (patroni rest api)
- `2379`, `2380` (etcd)

для схемы «[Type A] PostgreSQL High-Availability with Load Balancing»:

- `5000` (haproxy - (чтение/запись) master)
- `5001` (haproxy - (только чтение) все реплики)
- `5002` (haproxy - (только чтение) только синхронная реплика)
- `5003` (haproxy - (только чтение) только асинхронные реплики)
- `7000` (опционально, статистика haproxy)

для схемы «[Type C] PostgreSQL High-Availability with Consul Service Discovery (DNS)»:

- `8300` (Consul Server RPC)
- `8301` (Consul Serf LAN)
- `8302` (Consul Serf WAN)
- `8500` (Consul HTTP API)
- `8600` (Consul DNS сервер)

</p></details>

## Рекомендации

<details><summary>Click here to expand...</summary><p>

- **linux (операционная система)**: 

Перед развертыванием обновите операционную систему на целевых серверах;

Убедитесь, что у вас настроена синхронизация времени (NTP).
Укажите `ntp_enabled:'true'` и `ntp_servers`, если вы хотите установить и настроить службу ntp.

- **DCS (Distributed Consensus Store)**: 

Быстрые диски и надежная сеть являются наиболее важными факторами для производительности и стабильности кластера etcd (или consul).

Избегайте хранения данных etcd (или consul) на одном диске с другими процессами (например, базой данных), которые интенсивно используют ресурсы дисковой подсистемы! 
Храните данные etcd и postgresql на **разных** дисках (см. переменные `etcd_data_dir`, `consul_data_path`), по возможности используйте ssd-диски.
См. руководства [рекомендации по оборудованию](https://etcd.io/docs/v3.3/op-guide/hardware/) и [настройка](https://etcd.io/docs/v3.3/tuning/).

Кластер DCS рекомендуется размещать на выделенных серверах, отдельно от серверов баз данных.

- **Размещение участников кластера в разных дата-центрах**:

Если вы предпочитаете кросс-центровую установку, когда реплицируемые базы данных расположены в разных дата-центрах, размещение членов etcd становится критически важным.

Если вы хотите создать действительно надежный etcd-кластер, необходимо учитывать множество моментов, но есть одно правило: *не размещайте все члены etcd в вашем основном центре обработки данных*. Посмотрите некоторые [примеры](https://www.cybertec-postgresql.com/en/introduction-and-how-to-etcd-clusters-for-patroni/).


- **Как предотвратить потерю данных при автоотказе (synchronous_modes)**:

По соображениям производительности синхронная репликация по умолчанию отключена.

Чтобы минимизировать риск потери данных при автоотказе, вы можете настроить параметры следующим образом:
- synchronous_mode: 'true'
- synchronous_mode_strict: 'true'
- synchronous_commit: 'on' (или 'remote_apply')

</p></details>

## Начало работы

У вас есть возможность легко развернуть кластеры Postgres с помощью консоли (UI) или из командной строки с помощью команды ansible-playbook.

### Консоль (пользовательский интерфейс)

Чтобы запустить консоль кластера PostgreSQL, выполните следующую команду:

```
docker run -d --name pg-console \
  --publish 80:80 \
  --publish 8080:8080 \
  --env PG_CONSOLE_API_URL=http://localhost:8080/api/v1 \
  --env PG_CONSOLE_AUTHORIZATION_TOKEN=secret_token \
  --env PG_CONSOLE_DOCKER_IMAGE=vitabaks/postgresql_cluster:latest \
  --volume console_postgres:/var/lib/postgresql \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --volume /tmp/ansible:/tmp/ansible \
  --restart=unless-stopped \
  vitabaks/postgresql_cluster_console:2.0.0
```

> [!ПРИМЕЧАНИЕ]
> Если вы запускаете консоль на выделенном сервере (а не на ноутбуке), замените `localhost` на IP-адрес сервера в переменной `PG_CONSOLE_API_URL`.

> [!СОВЕТ]
> Рекомендуется запускать консоль в той же сети, что и серверы баз данных, чтобы иметь возможность отслеживать состояние кластера.


**Откройте пользовательский интерфейс консоли**:

Перейдите по адресу http://localhost:80 (или адресу вашего сервера) и используйте `secret_token` для авторизации.

![Демонстрация создания кластера](images/pg_console_create_cluster_demo.gif)

Обратитесь к разделу [Развертывание](https://postgresql-cluster.org/docs/category/deployment), чтобы узнать больше о различных методах развертывания.

### Командная строка

<details><summary>Нажмите здесь, чтобы развернуть... если вы предпочитаете командную строку.</summary><p>

0. [Установите Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) на один управляющий узел (которым вполне может быть ноутбук)

```
sudo apt update && sudo apt install -y python3-pip sshpass git
pip3 install ansible
```

1. Скачайте или клонируйте этот репозиторий

```
git clone https://github.com/vitabaks/postgresql_cluster.git
```

2. Перейдите в каталог автоматизации

```
cd postgresql_cluster/automation
```

3. Установите требования на узле управления

```
ansible-galaxy install --force -r requirements.yml 
```

Примечание: Если вы планируете использовать Consul (`dcs_type: consul`), установите требования к роли consul
```
ansible-galaxy install -r roles/consul/requirements.yml
```

4. Отредактируйте файл инвентаризации

Укажите (непубличные) IP-адреса и параметры подключения (`ansible_user`, `ansible_ssh_pass` или `ansible_ssh_private_key_file` для вашего окружения

```
nano inventory
```

5. Отредактируйте файл переменных vars/[main.yml](./automation/vars/main.yml)

```
nano vars/main.yml
```

Минимальный набор переменных: 
- `proxy_env` для загрузки пакетов в окружении без прямого доступа в интернет (опционально)
- `patroni_cluster_name`
- `postgresql_version`
- `postgresql_data_dir`
- `cluster_vip` для обеспечения единой точки входа для клиентского доступа к базам данных в кластере (опционально)
- `with_haproxy_load_balancing` для включения балансировки нагрузки (необязательно)
- `dcs_type` «etcd» (по умолчанию) или «consul»

Подробнее см. в файлах vars/[main.yml](./automation/vars/main.yml), [system.yml](./automation/vars/system.yml) и ([Debian.yml](./automation/vars/Debian.yml) или [RedHat.yml](./automation/vars/RedHat.yml)).

6. Попробуйте подключиться к хостам

```
ansible all -m ping
```

7. Запустите плейбук:

```
ansible-playbook deploy_pgcluster.yml
```

#### Развертывание кластера с TimescaleDB

Чтобы развернуть кластер PostgreSQL High-Availability Cluster с расширением [TimescaleDB](https://github.com/timescale/timescaledb), добавьте переменную `enable_timescale`:

Пример:
```
ansible-playbook deploy_pgcluster.yml -e «enable_timescale=true»
```

[![asciicast](https://asciinema.org/a/251019.svg)](https://asciinema.org/a/251019?speed=5)

### Как начать с нуля

Если вам нужно начать с самого начала, вы можете воспользоваться плейбуком `remove_cluster.yml`.

Доступные переменные:
- `remove_postgres`: остановить службу PostgreSQL и удалить данные.
- `remove_etcd`: остановить службу ETCD и удалить данные.
- `remove_consul`: остановить службу Consul и удалить данные.

Выполните следующую команду для удаления определенных компонентов:

``bash
ansible-playbook remove_cluster.yml -e «remove_postgres=true remove_etcd=true»
```

Эта команда удалит указанные компоненты, что позволит вам начать новую установку с нуля.

:warning: **Осторожно:** будьте внимательны при выполнении этой команды в производственной среде.

</p></details>

## Star us

Если вы считаете наш проект полезным, поставьте ему звезду на GitHub! Ваша поддержка помогает нам развиваться и мотивирует нас продолжать совершенствоваться. Пометить проект звездой - это простой, но эффективный способ выразить свою признательность и помочь другим обнаружить его.

<a href=«https://star-history.com/#vitabaks/postgresql_cluster&Date»>
<картинка>
<source media=«(prefers-color-scheme: dark)» srcset=«https://api.star-history.com/svg?repos=vitabaks/postgresql_cluster&type=Date&theme=dark» />
<source media=«(prefers-color-scheme: light)» srcset=«https://api.star-history.com/svg?repos=vitabaks/postgresql_cluster&type=Date» />
<img alt=«Диаграмма звездной истории» src=«https://api.star-history.com/svg?repos=vitabaks/postgresql_cluster&type=Date» />
</picture
</a>

## Спонсируйте этот проект

Спонсируя наш проект, вы вносите непосредственный вклад в его постоянное совершенствование и инновации. В качестве спонсора вы получите эксклюзивные преимущества, включая индивидуальную поддержку, ранний доступ к новым функциям и возможность влиять на направление развития проекта. Ваша спонсорская поддержка бесценна для нас и помогает обеспечить устойчивость и прогресс проекта.

Станьте спонсором сегодня и помогите нам вывести проект на новый уровень!

Поддержите нашу работу через [GitHub Sponsors](https://github.com/sponsors/vitabaks)

[![GitHub Sponsors](https://img.shields.io/github/sponsors/vitabaks?style=for-the-badge)](https://github.com/sponsors/vitabaks)

Поддержите нашу работу через [Patreon](https://www.patreon.com/vitabaks)

[![Поддержите меня на Patreon](https://img.shields.io/endpoint.svg?url=https%3A%2F%2Fshieldsio-patreon.vercel.app%2Fapi%3Fusername%3Dvitabaks%26type%3Dpatrons&style=for-the-badge)](https://patreon.com/vitabaks)

Поддержите нашу работу через криптокошелек:

USDT (TRC20): `TSTSXZzqDCUDHDjZwCpuBkdukjuDZspwjj`.

## Лицензия
Лицензируется в соответствии с лицензией MIT. Подробности см. в файле [LICENSE](./LICENSE).

## Автор
Виталий Кухарик (PostgreSQL DBA) \
vitabaks@gmail.com

## Отзывы, баг-репорты, запросы, ...
[приветствуются](https://github.com/vitabaks/postgresql_cluster/issues)!
