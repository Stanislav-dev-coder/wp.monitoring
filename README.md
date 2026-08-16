# WordPress Monitoring

[Тестовое задание Intern/Junior DevOps-инженера от «RocketDev»](./Тестовое%20DevOps.md)

## Установка Ansible

**Debian/Ubuntu:**

```bash
sudo apt install ansible
```

**Arch Linux:**

```bash
yay -S ansible
```

**macOS:**

```bash
brew install ansible
```

## Подготовка

1. Скопировать `ansible/inventory.example.yml` в `ansible/inventory.yml` и заполнить реальными адресами серверов, пользователем и путём к SSH-ключу:

   ```bash
   cp ansible/inventory.example.yml ansible/inventory.yml
   ```

2. Скопировать `nginx/allowed_ips.example.conf` в `nginx/allowed_ips.conf` и указать IP, с которого разрешён доступ к `wp-admin` и `wp-login.php`:

   ```bash
   cp nginx/allowed_ips.example.conf nginx/allowed_ips.conf
   ```

3. Установить роли-зависимости:

```bash
cd ansible
ansible-galaxy install -r requirements.yml
```

4. Заполнить секреты в `ansible/group_vars/server/vault.yml` и зашифровать файл:

```bash
ansible-vault encrypt group_vars/server/vault.yml --vault-password-file ~/.vault_pass.txt
```

5. Проверить синтаксис плейбука:

```bash
ansible-playbook -i inventory.yml site.yml --syntax-check --vault-password-file ~/.vault_pass.txt
```

## Запуск плейбука

1. Запустить плейбук:

```bash
ansible-playbook -i inventory.yml site.yml --vault-password-file ~/.vault_pass.txt
```

2. После деплоя добавить в `/etc/hosts` на своей машине:

```
<IP-сервера>   site.local
<IP-сервера>   metrics.local
```

## Проверка

1. Проверка с помощью `curl`:

```bash
curl -Ikv https://site.local/
curl -Iv http://metrics.local/
```

- `-I`: только заголовки ответа, без тела страницы;
- `-k`: не проверять валидность SSL-сертификата (нужен, так как сертификат самоподписанный);
- `-v`: подробный вывод: TCP-соединение, TLS-хендшейк и данные сертификата, заголовки запроса и ответа.

2. Для проверки `wp-admin` и `wp-login.php` можно выполнить запрос с самого сервера через Ansible, который пойдёт с адреса `127.0.0.1`:

```bash
ansible -i inventory.yml server_timeweb -m command -a "curl -I -H 'Host: site.local' http://localhost/wp-admin/" --vault-password-file ~/.vault_pass.txt
ansible -i inventory.yml server_timeweb -m command -a "curl -I -H 'Host: site.local' http://localhost/wp-login.php" --vault-password-file ~/.vault_pass.txt
```

- `-H "Host: site.local"`: подставляет нужный домен в заголовок запроса, так как на сервере он не резолвится через DNS.

Ожидаемый результат `403 Forbidden`. С разрешённого IP те же пути будут отдавать `302` - редирект на `install.php`.

## Принятые решения

### swap

Тестирование проводилось на серверах с минимальной конфигурацией, поэтому для запуска WordPress и Grafana необходимо было установить swap объемом 2GB. Использовал роль Ansible `geerlingguy.swap`.

### fail2ban + iptables

Установил минимальную конфигурацию `fail2ban` для защиты от брутфорса с баном навсегда. Для дополнительной защиты сервера настроил цепочку INPUT `iptables`: разрешил loopback, ICMP запросы, подключение к 22 порту, established/related-соединения и 9100 порт `node_exporter` с подсети docker, которую установил для подсети мониторинга. Правила применяются через модуль `community.general.iptables_state` с `noflush: true`, чтобы не перезаписывать правила Docker.

### Домен .local

Использовал самоподписанный сертификат для домена `.local`, так как на этот домен нельзя получить сертификат от публичных удостоверяющих центров.

### Сохранение конфигурации Grafana

Для автоматической установки источника данных в Grafana добавил `provisioning` конфигурацию. Дашборд `grafana/dashboard/os-general.json` импортируется вручную через Dashboards → Import в интерфейсе Grafana.

### Разделение на несколько плейбуков

Провел разделение плейбуков для лучшей читаемости. `system.yml` с установкой swap и пакетов, `network.yml` с установкой fail2ban и iptables и `deploy.yml` с генерацией секретов, сертификатов и запуском контейнеров.

### Vault-пароль

Пароли храню зашифрованными в `ansible/group_vars/server/vault.yml` через Ansible Vault. Зашифрованный `vault.yml` можно закоммитить в репозиторий, они будут воспроизводимы на любой машине с доступом к vault-паролю, но не читаемы без него. Файл `.env` на сервере генерируется из шаблона `ansible/templates/env.j2` во время выполнения плейбука, поэтому пароли нигде не хранятся в открытом виде внутри репозитория. Ключ хранится вне репозитория и передаётся в команду запуска через `--vault-password-file`.

## Трудозатраты

| Этап                                                           | Время (часы) |
| -------------------------------------------------------------- | ------------ |
| Ручная настройка инфраструктуры                                | 4            |
| Перенос ручных шагов в Ansible-плейбуки                        | 3            |
| Переход на готовые роли и внедрение Ansible Vault для секретов | 2            |
| Тестирование на чистом сервере, healthcheck, доработка CI      | 2            |
| Документация и финальная проверка репозитория                  | 1            |
| Итого                                                          | 12           |
