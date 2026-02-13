# Установка Self-Hosted RustDesk Server на Ubuntu LTS 24.*

**Полная пошаговая инструкция** от чистой системы до работающего сервера с Docker, Nginx Proxy Manager, автозапуском и безопасностью.

---

## Параметры установки

> **Перед началом замените эти значения на свои!**

| Параметр | Значение для замены | Описание |
|----------|-------------------|----------|
| **ОС** | `Ubuntu 24.04` | Версия операционной системы |
| **Имя пользователя** | `username` | Пользователь, под которым будет работать сервер |
| **Домен** | `rustdesk.domain.name` | Ваш домен, направленный на сервер |
| **Ваш IP** | `1.2.3.4` | Ваш домашний статический IP для доступа к SSH и админке |

---

## Содержание

1. [Подготовка системы](#-шаг-0-подготовка-системы)
2. [Установка Docker](#-шаг-1-установка-docker)
3. [Развертывание RustDesk + Nginx Proxy Manager](#-шаг-2-развертывание-rustdesk--nginx-proxy-manager)
4. [Настройка SSL и стримов](#-шаг-3-настройка-ssl-и-стримов)
5. [Автозапуск и мониторинг](#-шаг-4-автозапуск-и-мониторинг)
6. [Безопасность](#-шаг-5-безопасность)
7. [Шпаргалка команд](#-шпаргалка-команд)
8. [Решение проблем](#-решение-проблем)

---

## ШАГ 0: Подготовка системы

### 0.1. Создание пользователя (если зашли под root)

```bash
# Создать пользователя
useradd -m -s /bin/bash -G sudo username
passwd username

# Настроить SSH только с вашего IP
echo "AllowUsers username@1.2.3.4 root@1.2.3.4" >> /etc/ssh/sshd_config.d/allow-user.conf
systemctl restart sshd

# Выйти и зайти под username
exit
ssh username@rustdesk.domain.name
```

### 0.2. Базовая настройка системы

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Необходимые пакеты
sudo apt install -y curl wget git net-tools ufw nano htop fail2ban

# Настройка fail2ban
sudo tee /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
EOF

sudo systemctl restart fail2ban
```

### 0.3. Настройка UFW (фаервол)

```bash
# Сброс и базовые правила
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH только с вашего IP
sudo ufw allow from 1.2.3.4 to any port 22 proto tcp

# Порты для RustDesk (пока открыты всем)
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21116/udp

# Порты для HTTP/HTTPS (нужны для Let's Encrypt)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Включение UFW
sudo ufw --force enable
sudo ufw status verbose
```

### 0.4. Часовой пояс (опционально)

```bash
sudo timedatectl set-timezone Europe/Moscow  # замените на свой
```

---

## ШАГ 1: Установка Docker

### 1.1. Установка Docker Engine

```bash
# Удаление старых версий
sudo apt remove -y docker docker-engine docker.io containerd runc

# Установка зависимостей
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Добавление репозитория Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Добавление пользователя в группу docker
sudo usermod -aG docker $USER

# Применение группы (выйти и зайти)
exit
ssh username@rustdesk.domain.name

# Проверка
docker --version
docker compose version
docker run hello-world
docker rm $(docker ps -a -q)  # удалить тестовый контейнер

# Автозапуск Docker
sudo systemctl enable docker
```

---

## ШАГ 2: Развертывание RustDesk + Nginx Proxy Manager

### 2.1. Создание структуры папок

```bash
mkdir -p ~/rustdesk-docker/{data,backup,scripts,npm/data,npm/letsencrypt}
cd ~/rustdesk-docker
```

### 2.2. Создание docker-compose.yml

```bash
vim docker-compose.yml
```

**Содержание файла `docker-compose.yml`:**

```yaml
networks:
  rustdesk-net:
    driver: bridge

services:
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'              # HTTP (для Let's Encrypt)
      - '443:443'            # HTTPS
      - '8081:81'            # Админка NPM
      # Порты RustDesk (обязательно!)
      - '21115:21115'
      - '21116:21116'
      - '21116:21116/udp'
      - '21117:21117'
      - '21118:21118'
      - '21119:21119'
    environment:
      DB_SQLITE_FILE: "/data/database.sqlite"
      DISABLE_IPV6: 'true'
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
    networks:
      - rustdesk-net

  hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbs
    command: hbbs -r rustdesk.domain.name:21117
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped
    # Порты НЕ пробрасываем - они идут через NPM

  hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbr
    command: hbbr
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped
    # Порты НЕ пробрасываем
```

### 2.3. Запуск NPM

```bash
docker compose up -d nginx-proxy-manager
docker compose ps
```

### 2.4. Создание администратора NPM

1. Открой браузер: `http://rustdesk.domain.name:8081`
2. Первый вход:
   - Твой актуальный email: `admin@example.com`
   - Password: `надёжный_пароль`
3. Сохрани пароль в надежном месте!

---

## ШАГ 3: Настройка SSL и стримов

### 3.1. Получение SSL сертификата

В NPM:
1. **SSL Certificates** → **Add SSL Certificate** → **Let's Encrypt**
2. **Domain Names**: `rustdesk.domain.name`
3. **Email**: твой email
4. Включить согласие с условиями
5. **Save**

*Через 10-30 секунд статус станет "Valid" или "In Use"*

### 3.2. Создание Proxy Host (опционально, для веб-доступа)

В NPM:
1. **Hosts** → **Proxy Hosts** → **Add Proxy Host**
2. **Domain Names**: `rustdesk.domain.name`
3. **Scheme**: `http`
4. **Forward Hostname**: `hbbs`
5. **Forward Port**: `21119`
6. Вкладка **SSL**: выбери созданный сертификат, включи **Force SSL**
7. **Save**

### 3.3. Настройка Streams (самое важное!)

В NPM:
1. **Streams** → **Add Stream**
2. Создай **6 стримов** строго по таблице:

| № | Порт | Протокол | Forward Host | Forward Port |
|---|------|----------|--------------|--------------|
| 1 | 21115 | TCP | hbbs | 21115 |
| 2 | 21116 | TCP | hbbs | 21116 |
| 3 | 21116 | **UDP** | hbbs | 21116 |
| 4 | 21117 | TCP | hbbr | 21117 |
| 5 | 21118 | TCP | hbbs | 21118 |
| 6 | 21119 | TCP | hbbr | 21119 |

*Для всех стримов на вкладке SSL оставь **HTTP Only**!*

### 3.4. Запуск RustDesk

```bash
cd ~/rustdesk-docker
docker compose up -d hbbs hbbr
sleep 10
docker compose ps
```

### 3.5. Получение и сохранение ключей

```bash
# Посмотреть публичный ключ
cat ~/rustdesk-docker/data/id_ed25519.pub
# Запиши его! Пример: RnNeJekB3BW...

# Сохранить ключ в backup
echo "Публичный ключ RustDesk:" > ~/rustdesk-docker/backup/rustdesk-key.txt
cat ~/rustdesk-docker/data/id_ed25519.pub >> ~/rustdesk-docker/backup/rustdesk-key.txt

# Создать архив ключей
cd ~/rustdesk-docker
tar -czf ~/rustdesk-docker/backup/rustdesk-keys-$(date +%Y%m%d-%H%M%S).tar.gz -C data id_ed25519 id_ed25519.pub

# Проверить
ls -la ~/rustdesk-docker/backup/
```

### 3.6. Добавление ключа в команду запуска (рекомендуется)

```bash
# Остановить контейнеры
docker compose down hbbs hbbr

# Отредактировать docker-compose.yml
vim docker-compose.yml
```

**Измени строки command, добавив ключ:**

```yaml
  hbbs:
    command: hbbs -r rustdesk.domain.name:21117 -k RnNeJekB3BW...  # твой ключ

  hbbr:
    command: hbbr -k RnNeJekB3BW...  # твой ключ
```

**Запустить снова:**

```bash
docker compose up -d hbbs hbbr
```

### 3.7. Проверка работоспособности

```bash
# Проверка портов на хосте
sudo ss -tulpn | grep -E ':(2111|80|443|8081)'

# Проверка стримов внутри NPM
docker exec npm ls -la /data/nginx/stream/ | wc -l
# Должно быть 6

# Проверка доступности из Docker сети
docker run --rm --network rustdesk-docker_rustdesk-net alpine sh -c "apk add nmap-ncat > /dev/null 2>&1 && nc -zv hbbs 21115 && nc -zv hbbs 21116 && nc -zv hbbr 21117"
```

---

## ШАГ 4: Автозапуск и мониторинг

### 4.1. Systemd сервис для автозапуска

```bash
sudo tee /etc/systemd/system/rustdesk-docker.service << 'EOF'
[Unit]
Description=RustDesk Docker Compose
Requires=docker.service
After=docker.service network.target

[Service]
Type=oneshot
RemainAfterExit=yes
User=username
Group=docker
WorkingDirectory=/home/username/rustdesk-docker
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

# Активация
sudo systemctl daemon-reload
sudo systemctl enable rustdesk-docker.service
sudo systemctl start rustdesk-docker.service
sudo systemctl status rustdesk-docker.service --no-pager
```

### 4.2. Скрипт мониторинга

```bash
cd ~/rustdesk-docker
vim check-rustdesk.sh
```

**Содержание скрипта `check-rustdesk.sh`:**

```bash
#!/bin/bash
echo "========================================="
echo "    RustDesk Server Status Check"
echo "========================================="
echo ""

echo "Docker Containers:"
docker compose ps --format "table {{.Name}}\t{{.Status}}"

echo ""
echo "Ports listening on host:"
sudo ss -tulpn | grep -E ':(2111|80|443|8081)' | sed 's/^/  /'

echo ""
echo "Public Key:"
cat ./data/id_ed25519.pub | sed 's/^/  /'

echo ""
echo "NPM Streams Status:"
COUNT=$(docker exec npm ls -la /data/nginx/stream/ 2>/dev/null | grep -c ".conf")
echo "  Active stream configs: $COUNT (должно быть 6)"

echo ""
echo "========================================="
```

**Делаем исполняемым и создаем ссылку:**

```bash
chmod +x ~/rustdesk-docker/check-rustdesk.sh
sudo ln -s ~/rustdesk-docker/check-rustdesk.sh /usr/local/bin/rustdesk-status
```

**Проверка:**

```bash
rustdesk-status
```

---

## ШАГ 5: Безопасность (финальные штрихи)

### 5.1. Ограничение доступа к NPM админке

```bash
sudo ufw allow from 1.2.3.4 to any port 8081 proto tcp comment 'NPM admin'
```

### 5.2. Лимиты на порты RustDesk (защита от DoS)

```bash
sudo ufw limit 21115:21119/tcp
sudo ufw limit 21116/udp
sudo ufw reload
sudo ufw status verbose
```

### 5.3. Автоматические обновления безопасности

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Выбери "Yes"
```

### 5.4. Мониторинг доступа к ключам

```bash
sudo apt install -y auditd
sudo auditctl -w /home/username/rustdesk-docker/data/ -p wa -k rustdesk-keys
sudo systemctl enable auditd
```

### 5.5. Финальная проверка безопасности

```bash
# Статус UFW
sudo ufw status verbose

# Статус fail2ban
sudo fail2ban-client status sshd

# Проверка auditd
sudo ausearch -k rustdesk-keys | tail -5
```

---

## Настройка клиентов RustDesk

### Windows / Linux / macOS / Android / iOS

1. Скачай клиент с [официального сайта](https://rustdesk.com/)
2. Открой **Настройки** → **Сеть**
3. Введи:

```
Сервер ID:    rustdesk.domain.name
Ретранслятор: rustdesk.domain.name
API Server:   (оставь пустым)
Key:          RnNeJekB3BW... (твой публичный ключ)
```

4. Нажми **OK**
5. Должна загореться **зеленая точка** 🟢 и статус **"Готов"**

---

## Шпаргалка команд

### Мониторинг и диагностика

```bash
# Быстрая проверка всего сервера
rustdesk-status

# Статус контейнеров
cd ~/rustdesk-docker && docker compose ps

# Логи контейнеров
docker compose logs hbbs --tail 20
docker compose logs hbbr --tail 20
docker compose logs nginx-proxy-manager --tail 50

# Логи в реальном времени
docker compose logs -f

# Проверка портов
sudo ss -tulpn | grep -E ':(2111|80|443|8081)'

# Публичный ключ
cat ~/rustdesk-docker/data/id_ed25519.pub
```

### Управление сервером

```bash
# Остановить всё
cd ~/rustdesk-docker && docker compose down

# Запустить всё
cd ~/rustdesk-docker && docker compose up -d

# Перезапустить конкретный сервис
docker compose restart hbbs
docker compose restart nginx-proxy-manager

# Перезагрузить Nginx внутри NPM
docker exec npm nginx -s reload
```

### Бэкап ключей

```bash
cd ~/rustdesk-docker
tar -czf ~/rustdesk-docker/backup/rustdesk-keys-$(date +%Y%m%d-%H%M%S).tar.gz -C data id_ed25519 id_ed25519.pub
```

### Логи безопасности

```bash
# Заблокированные UFW подключения
sudo grep "UFW BLOCK" /var/log/ufw.log | tail -20

# Статус fail2ban
sudo fail2ban-client status sshd

# Неудачные попытки SSH
sudo grep "Failed password" /var/log/auth.log | tail -20

# Доступ к ключам (auditd)
sudo ausearch -k rustdesk-keys | tail -30
```

---

## Решение проблем

### Клиент не подключается (оранжевая точка)

```bash
# 1. Проверить, слушаются ли порты
sudo ss -tulpn | grep 2111

# 2. Проверить стримы в NPM
docker exec npm ls -la /data/nginx/stream/ | wc -l  # должно быть 6

# 3. Проверить логи hbbs
docker compose logs hbbs --tail 50 | grep -i error

# 4. Проверить доступность снаружи (с другого компа)
telnet rustdesk.domain.name 21115
```

### После перезагрузки ничего не работает

```bash
# Проверить статус systemd сервиса
sudo systemctl status rustdesk-docker.service

# Перезапустить вручную
cd ~/rustdesk-docker && docker compose up -d
```

### Забыл публичный ключ

```bash
# Посмотреть ключ
cat ~/rustdesk-docker/data/id_ed25519.pub

# Или в бэкапе
cat ~/rustdesk-docker/backup/rustdesk-key.txt
```

### NPM не открывает порты

```bash
# Проверить, что порты проброшены в compose
grep -A 10 "ports:" ~/rustdesk-docker/docker-compose.yml

# Перезапустить NPM
docker compose restart nginx-proxy-manager

# Проверить порты
sudo ss -tulpn | grep 2111
```

---

## Бэкап и восстановление

### Создание полного бэкапа

```bash
cd ~
tar -czf rustdesk-backup-$(date +%Y%m%d).tar.gz rustdesk-docker/
```

### Восстановление на новом сервере

1. Установить Docker (ШАГ 1)
2. Скопировать папку `rustdesk-docker` на новый сервер
3. Запустить контейнеры:

```bash
cd ~/rustdesk-docker
docker compose up -d
```

4. Изменить DNS запись `rustdesk.domain.name` на новый IP
5. Через 10-30 минут клиенты переподключатся автоматически

---

## Заключение

**Ваш сервер готов!**

✅ Все контейнеры работают  
✅ Автозапуск настроен  
✅ Безопасность обеспечена  
✅ Мониторинг работает  
✅ Клиенты подключаются  

---

*Документация создана при поддержке IT Adekvat. Февраль 2026.*

**⭐ Если инструкция была полезной, поставь звезду на GitHub!**
