# 🛒 Тестовый стенд 1С-Битрикс для интеграции с платежным шлюзом Газпромбанка (СПЭК)

Комплексное развертывание изолированной среды для тестирования платежного модуля `pgamerchant` и отработки интеграции с эквайрингом.

---

## 📋 **О проекте**

**Цель:** Создание полностью изолированного, контролируемого тестового стенда для тестирования интеграции интернет-магазина с эквайрингом Газпромбанка (СПЭК).

**Задачи:**
- Развернуть физический сервер с белым статическим IP
- Установить и настроить 1С-Битрикс в Docker-контейнерах
- Интегрировать платежный модуль pgamerchant
- Отработать полный цикл платежной интеграции
- Подготовить документацию и инструменты для диагностики

---

## 🛠 **Технологический стек**

| Компонент | Технология |
|-----------|------------|
| **Аппаратная платформа** | Raspberry Pi 4 (ARM64, 4GB RAM) |
| **ОС** | Debian/Ubuntu Linux |
| **Контейнеризация** | Docker, Docker Compose |
| **Веб-сервер** | Apache (в контейнере) |
| **База данных** | MySQL 8.0 |
| **CMS** | 1С-Битрикс: Управление сайтом (редакция «Бизнес») |
| **Платежный модуль** | pgamerchant (Газпромбанк, СПЭК) |
| **Криптография** | OpenSSL, PEM-сертификаты |

---

## 🚀 **Архитектура решения**
5.102.159.220 (Raspberry Pi 4)
├── Docker контейнеры
│ ├── bitrix_php:8443 # 1С-Битрикс (интернет-магазин)
│ └── bitrix_mysql:3306 # MySQL для Битрикса
│
└── Системные сервисы
├── Nginx:443 # Reverse proxy для основного сайта
├── Flask-приложение:8081 # Основной сайт tda-photo.ru
└── API-сервисы:7443 # Дополнительные обработчики

text

### **Сетевая инфраструктура**
- Белый статический IP: `5.102.159.220`
- Домен: `tda-photo.ru` с валидным SSL-сертификатом
- Локальный IP Raspberry: `192.168.32.116`
- Проброс портов на роутере: `8443` → `8443`

---

## 🔧 **Развертывание**

### **Docker-контейнеры для Битрикса**

```yaml
# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: arm64v8/mysql:8.0
    container_name: bitrix_mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: bitrix
      MYSQL_USER: bitrix
      MYSQL_PASSWORD: bitrix
    volumes:
      - ./mysql:/var/lib/mysql
    ports:
      - "3306:3306"
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci

  php:
    image: arm64v8/php:8.2.28-apache
    container_name: bitrix_php
    restart: unless-stopped
    ports:
      - "8443:80"  # Порт из списка разрешенных СПЭК
    volumes:
      - ./www:/var/www/html
      - ./php.ini:/usr/local/etc/php/php.ini
    depends_on:
      - mysql
    environment:
      DB_HOST: mysql
      DB_NAME: bitrix
      DB_USER: bitrix
      DB_PASSWORD: bitrix
Настройка PHP
ini
; php.ini
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 120
max_input_vars = 10000
date.timezone = Europe/Moscow
short_open_tag = Off
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.revalidate_freq = 60
💳 Интеграция с платежным модулем
Установка модуля pgamerchant
bash
# Создать папку для модуля
mkdir -p /home/ioxnsun/bitrix/www/bitrix/modules/pgamerchant

# Распаковать архив модуля
unzip pga_bitrix_module-1.0.5.zip -d /home/ioxnsun/bitrix/www/bitrix/modules/pgamerchant/

# Установить права
chmod -R 755 /home/ioxnsun/bitrix/www/bitrix/modules/pgamerchant/
Настройка сертификата
bash
# Создать папку для сертификатов
mkdir -p /home/ioxnsun/bitrix/www/local/php_interface/include/certificates/

# Загрузить сертификат (PEM-формат)
scp -P 2222 pgacert.crt ioxnsun@5.102.159.220:/home/ioxnsun/bitrix/www/local/php_interface/include/certificates/

# Проверить сертификат
openssl x509 -in /home/ioxnsun/bitrix/www/local/php_interface/include/certificates/pgacert.crt -text -noout
🔍 Диагностические скрипты
scripts/cert_test.php - проверка сертификата
php
<?php
$certPath = $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/include/certificates/pgacert.crt';

if (!file_exists($certPath)) {
    die("❌ Сертификат не найден");
}

$certContent = file_get_contents($certPath);
echo "📏 Размер: " . filesize($certPath) . " байт\n";

if (strpos($certContent, '-----BEGIN CERTIFICATE-----') !== false) {
    echo "✅ Формат: PEM (корректный)\n";
}

$certData = openssl_x509_read($certContent);
if ($certData) {
    $info = openssl_x509_parse($certData);
    echo "📅 Действует до: " . date('d.m.Y', $info['validTo_time_t']) . "\n";
    echo "🏷️ Субъект: " . $info['subject']['CN'] . "\n";
}
?>
scripts/host_test.php - проверка HTTP-заголовков
php
<?php
echo "=== Диагностика HTTP-заголовков ===\n\n";
echo "HTTP_HOST: " . $_SERVER['HTTP_HOST'] . "\n";
echo "SERVER_NAME: " . $_SERVER['SERVER_NAME'] . "\n";
echo "REQUEST_URI: " . $_SERVER['REQUEST_URI'] . "\n";
echo "DOCUMENT_ROOT: " . $_SERVER['DOCUMENT_ROOT'] . "\n";
?>
scripts/callback_test.php - тестирование callback-ов
php
<?php
$logFile = $_SERVER['DOCUMENT_ROOT'] . '/bitrix/tools/callback_debug.log';

if (file_exists($logFile)) {
    echo "=== Последние callback-запросы ===\n\n";
    $logs = file_get_contents($logFile);
    echo "<pre>" . htmlspecialchars($logs) . "</pre>";
} else {
    echo "Лог-файл не найден";
}
?>
🌐 Сетевая конфигурация
Проброс портов на роутере
Параметр	Значение
Внешний порт	8443
Внутренний IP	192.168.32.116
Внутренний порт	8443
Протокол	TCP
Проверка доступности
powershell
# PowerShell (с ПК)
Test-NetConnection 5.102.159.220 -Port 8443
# Должен быть результат: TcpTestSucceeded : True
📌 Адреса и доступ
Назначение	Адрес
Витрина магазина	http://5.102.159.220:8443
Админка Битрикса	http://5.102.159.220:8443/bitrix/admin/
Callback для банка	http://5.102.159.220:8443/bitrix/tools/sale_ps_result.php
Данные для входа в админку:


🚀 Быстрый старт
bash
# 1. Клонировать репозиторий
git clone https://github.com/yourusername/bitrix-payment-testing.git
cd bitrix-payment-testing

# 2. Запустить контейнеры
docker compose up -d

# 3. Установить Битрикс через браузер
open http://localhost:8443

# 4. Установить модуль pgamerchant
# Копировать модуль в /bitrix/modules/ и установить через админку
📌 Полезные команды
bash
# Управление контейнерами
docker compose up -d        # запустить
docker compose down         # остановить
docker compose restart      # перезапустить
docker ps                   # статус

# Логи
docker logs bitrix_php --tail 50 -f

# Доступ к БД
docker exec -it bitrix_mysql mysql -u root -p


Проект демонстрирует полный цикл инженерной работы: от развертывания физического сервера до настройки платежной интеграции.
