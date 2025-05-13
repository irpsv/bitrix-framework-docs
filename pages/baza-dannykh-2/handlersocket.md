---
title: HandlerSocket
---

*  {% colwidth=[248] %}

   MySQL через HandlerSocket

*  {% colwidth=[184] %}

   750 000

*  {% colwidth=[175] %}

   45%

*  {% colwidth=[171] %}

   53%

{% /table %}

Результаты могут меняться в зависимости от версии MySQL и аппаратных характеристик сервера.

## Как установить HandlerSocket

-  **Cоберите плагин из исходников MySQL.** Этот способ подходит, если нужно добавить HandlerSocket к существующей MySQL.

-  **Используйте MariaDB.** Это форк MySQL, в котором HandlerSocket включен по умолчанию. Способ подходит для новых проектов, где можно выбрать СУБД с нуля.

После установки добавьте параметры HandlerSocket в конфигурационный файл MySQL.

{% note warning %}
 

Чтобы настроить пул потоков, укажите значение `handlersocket_threads` в диапазоне от 8 до 32 в конфигурационном файле `my.cnf`.


{% endnote %}

## Как подключить HandlerSocket

Добавьте указание на HandlerSocket в конфигурационный файл `bitrix/.settings.php` под ключом `'default'`:

```php
'default' => [
    'className' => \Bitrix\Main\DB\MysqliConnection::class,
    'handlersocket' => [
        'read' => 'handlersocket', // Использовать HandlerSocket для чтения
    ],
],
```

Добавьте отдельное соединение для HandlerSocket с указанием хоста и порта:

```php
'handlersocket' => [
    'className' => \Bitrix\Main\Data\HsphpReadConnection::class,
    'host' => 'localhost',  // Хост сервера HandlerSocket
    'port' => 9998,        // Порт для чтения (обычно 9998)
    'port_write' => 9999,  // Порт для записи (обычно 9999)
],
```

## Пример конфигурации с HandlerSocket

Этот конфигурационный файл позволяет настроить два типа подключений к БД -- обычное через MySQL и быстрое через HandlerSocket:

```php
'connections' => [ 
    'value' => [
        // Основное соединение с MySQL базой данных через MySQLi
        'default' => [ 
            'className' => \Bitrix\Main\DB\MysqliConnection::class, // Используем драйвер MySQLi для соединения
            'host' => 'localhost:31006',     // Хост базы данных с указанием порта (нестандартный порт 31006)
            'database' => 'admin_bus',       // Имя базы данных
            'login' => 'admin_bus',          // Имя пользователя для доступа к БД
            'password' => 'admin_bus',       // Пароль пользователя
            'options' => 2,                  // Дополнительные опции соединения (2 - отложенное соединение)
            'handlersocket' => [             // Настройки HandlerSocket (альтернативный метод доступа к MySQL)
                'read' => 'handlersocket',  // Использование HandlerSocket для операций чтения
            ], 
        ], 
        // Настройки соединения HandlerSocket для чтения данных
        'handlersocket' => [ 
            'className' => \Bitrix\Main\Data\HsphpReadConnection::class, // Специальный класс для работы с HandlerSocket
            'host' => 'localhost',    // Хост HandlerSocket
            'port' => '9998',         // Порт HandlerSocket
        ], 
    ], 
    'readonly' => true, // Запрещает изменение настроек во время выполнения скрипта
],
```

-  `default` -- основное подключение, использует стандартный MySQL-драйвер.

-  `handlersocket` -- дополнительное подключение, использует специальный драйвер для быстрого чтения.