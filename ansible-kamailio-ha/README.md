# Ansible: Asterisk HA stack

Это рабочая ветка интеграции Ansible из исходного проекта `c4m1r/kamailio-ha` с целевой архитектурой проекта Asterisk 22.

> **Статус:** bootstrap / migration-in-progress. Это не production-ready playbook. Не запускать на production. Целевые ОС, версии пакетов, адреса узлов, интерфейсы, firewall policy, PostgreSQL topology и параметры SIP/RTP должны быть утверждены до первого применения.

## Что переносим из исходного проекта

Исходный `.ansible/` содержал роли Kamailio, RTPengine, Asterisk, Keepalived, Redis и общие системные настройки. Он ориентирован на Debian Buster, Kamailio 5.4, статические файлы и интерфейс `eth1`. В исходнике также были отдельные экспериментальные playbooks `.ansible_do/`.

Переносим подход и полезные роли, но не копируем без проверки:
- старые Debian Buster / Kamailio 5.4 repository definitions;
- статические IP и имена интерфейсов;
- старые конфиги Asterisk/Kamailio/RTPengine;
- публичные тестовые адреса из старого inventory;
- пользовательские SSH-файлы, локальные настройки и бинарные пакеты.

Исходная ветка `master` оставлена без изменений для сравнения.

## Целевая топология

| Группа | Количество | Назначение |
|---|---:|---|
| Kamailio | 2 | SIP ingress, dispatcher, OPTIONS health checks |
| RTPengine | 2 | RTP anchoring / relay, NG control |
| Asterisk 22 | 4 | Независимые PBX nodes, PJSIP |
| PostgreSQL + Patroni | 3 | HA database cluster |
| HAProxy | 2 | PostgreSQL writer/read endpoint |
| DCS quorum | 3 | Patroni consensus (etcd/Consul — TBD) |

Числа — проектная целевая схема, не подтверждённый production sizing.

## Структура

- `ansible.cfg` — локальные defaults.
- `inventories/lab/hosts.ini.example` — inventory-шаблон.
- `group_vars/all.yml.example` — переменные и адреса-заглушки.
- `playbooks/site.yml` — основной orchestration entrypoint.
- `playbooks/preflight.yml` — только проверка inventory/facts.
- `roles/*` — роли по компонентам.
- `docs/MIGRATION_NOTES.md` — перенос и известные несовместимости.

## Первые команды

```bash
cd ansible-kamailio-ha
cp inventories/lab/hosts.ini.example inventories/lab/hosts.ini
cp group_vars/all.yml.example group_vars/all.yml
ansible-inventory -i inventories/lab/hosts.ini --graph
ansible-playbook -i inventories/lab/hosts.ini playbooks/preflight.yml --syntax-check
ansible-playbook -i inventories/lab/hosts.ini playbooks/preflight.yml --check
```

До настройки адресов и проверки доступа запускать только `preflight.yml`. `site.yml` пока содержит отключённые deployment tags и не предназначен для применения.

## Архитектурные правила

1. SIP маршрутизируется через Kamailio; HAProxy не участвует в SIP path.
2. RTPengine обрабатывает media, но не реплицирует Asterisk channels/bridges.
3. Отказ Asterisk node теряет активные вызовы на этом node; новые INVITE после health-check могут идти на оставшиеся узлы.
4. `app_queue` локален для процесса Asterisk. Нельзя считать четыре независимых queue instances общей очередью без внешнего ACD/controller или строгого sharding.
5. Patroni/HAProxy дают DB failover, но не call-state failover.
6. Для восстановления активного разговора нужен отдельный HA B2BUA/call-anchor, который пока не входит в автоматизацию.
7. LAB IPs из старого проекта не являются адресным планом этой архитектуры.

## Ветки

- `master` — исходный демонстрационный проект, сохранён как baseline.
- `asterisk` — интеграция для целевого Asterisk 22 HA stack.
