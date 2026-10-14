# Конфигурация виртуальной машины devops-vm

## 1. Параметры машины
- Гипервизор: VirtualBox 7.x
- ОС: Ubuntu Server 24.04 LTS (64-bit), установка не minimized
- ОЗУ: 2048 МБ
- Ядра CPU: 2
- Диск: 25 ГБ, динамически расширяемый, разметка Use an entire disk
- Имя узла: devops-vm

## 2. Сетевые интерфейсы
| Адаптер | Тип | Адрес | Назначение |
|---|---|---|---|
| 1 | NAT | 10.0.2.15 | выход в Интернет, репозитории пакетов |
| 2 | Host-only | 192.168.56.101 (<проверьте свой через `ip -brief address`>) | доступ с хоста, веб-сервер (работа № 6) |

## 3. Правило проброса портов
Адаптер 1 (NAT): имя `ssh`, TCP, хост 127.0.0.1:**2222** → гость **2222**
(изначально гостевой порт был 22, изменён после смены порта sshd).

## 4. Учётные записи
| Имя | Группы | Аутентификация |
|---|---|---|
| student | student, sudo | пароль (создана при установке) |
| devops | devops, sudo | только ключ ed25519 (`~/.ssh/devops_vm` на хосте) |

Создание:
```bash
sudo adduser devops
sudo usermod -aG sudo devops
```
Ключ: `ssh-keygen -t ed25519 -C "devops-vm-key" -f ~/.ssh/devops_vm`, затем
`ssh-copy-id -i ~/.ssh/devops_vm.pub -p 2222 devops@127.0.0.1`.
Права: `~/.ssh` — 700, `authorized_keys` — 600.
Клиентский `~/.ssh/config`: Host devops, HostName 127.0.0.1, Port 2222, User devops, IdentityFile ~/.ssh/devops_vm, IdentitiesOnly yes.

## 5. Служба SSH
- Файл конфигурации: `/etc/ssh/sshd_config.d/99-hardening.conf` (основной `sshd_config` не менялся, резервная копия `sshd_config.backup`)
- Директивы:

| Директива | Значение |
|---|---|
| Port | 2222 |
| PermitRootLogin | no |
| PasswordAuthentication | no |
| PubkeyAuthentication | yes |
| PermitEmptyPasswords | no |
| MaxAuthTries | 3 |
| LoginGraceTime | 30 |
| AllowUsers | devops |
| X11Forwarding | no |
| ClientAliveInterval | 300 |
| ClientAliveCountMax | 2 |

- Конфликтующая строка `PasswordAuthentication yes` в `/etc/ssh/sshd_config.d/50-cloud-init.conf` (если есть) закомментирована.
- Сокетная активация отключена:
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh
```
- Проверка: `sudo sshd -t`, `sudo sshd -T`, `sudo ss -tlnp | grep sshd`

## 6. Правила межсетевого экрана
- Политики: `default deny incoming`, `default allow outgoing`
- Правила (порядок важен: разрешающие правила до `ufw enable`):
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw delete allow 2222/tcp
sudo ufw limit 2222/tcp comment 'SSH rate-limited'
sudo ufw logging medium
```
- Итог: 2222/tcp LIMIT, 80/tcp ALLOW, 443/tcp ALLOW

## 7. Локальное имя
- На хосте в файл hosts: `192.168.56.101 devops.local`
- На сервере: `sudo hostnamectl set-hostname devops-vm`, в `/etc/hosts` строка `127.0.1.1 devops-vm.devops.local devops-vm`

## 8. Снимки состояния
| Наименование | Момент создания |
|---|---|
| 01-clean-install | <дата, время>, чистая ОС после обновления, до создания devops |
| 02-keys-configured | <дата, время>, создан devops, работает вход по ключу |
| 03-ssh-hardened | <дата, время>, применён 99-hardening.conf |

## 9. Порядок восстановления с нуля (после отката на 01-clean-install)
1. Создать пользователя `devops`, добавить в `sudo`
2. Сгенерировать ключ на хосте, скопировать на сервер, настроить `~/.ssh/config`
3. Создать `99-hardening.conf`, проверить `sshd -t`, отключить `ssh.socket`, перезапустить `ssh`
4. Изменить проброс порта в VirtualBox на 2222 → 2222
5. Проверить вход новой сессией (старую не закрывать)
6. Настроить UFW (сначала allow, потом enable), заменить правило SSH на limit
7. Настроить hosts на хосте и hostname на сервере
