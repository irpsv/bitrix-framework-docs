---
title: Service Locator
---

Service Locator -- это шаблон проектирования для управления сервисами приложения. Вместо создания сервисов напрямую, используется специальный объект -- Service Locator. Он отвечает за создание и поиск сервисов, что упрощает их использование и замену.

Класс [\\Bitrix\\Main\\DI\\ServiceLocator](https://dev.1c-bitrix.ru/api_d7/bitrix/main/di/servicelocator/index.php) реализует интерфейс PSR-11. Доступен с версии main 20.5.400.

Пример использования:

```php
$serviceLocator = \Bitrix\Main\DI\ServiceLocator::getInstance(); // Получаем экземпляр сервис локатора
if ($serviceLocator->has('someService')) { // Проверяем наличие сервиса
    $someService = $serviceLocator->get('someService'); // Получаем сервис
    // Использование $someService
}
```

## Конфигурация сервиса

Конфигурация определяет способ создания объекта. Возможны три варианта:

1. **Указать класс сервиса.** Service Locator создаст сервис вызвав `new $className`.

   ```php
   'someModule.someServiceName' => [
       'className' => \VendorName\SomeModule\Services\SomeService::class,
   ]
   ```

2. **Указать класс сервиса и параметры конструктора.** Service Locator создаст сервис вызвав `new $className('foo', 'bar')`.

   ```php
   'someModule.someServiceName' => [
       'className' => \VendorName\SomeModule\Services\SomeService::class,
       'constructorParams' => ['foo', 'bar'],
   ]
   ```

3. **Указать замыкание-конструктор.** Он создаст и вернет объект сервиса.

   ```php
   'someModule.someAnotherServiceName' => [
       'constructor' => static function () {
           return new \VendorName\SomeModule\Services\SecondService('foo', 'bar');
       },
   ]
   ```

## Регистрация сервиса

Чтобы обратиться к сервису, его нужно зарегистрировать одним из способов.

1. **Через файл настроек bitrix/.settings.php**

   В Bitrix Framework файлы `.settings.php` используются для хранения конфигураций. Они находятся в корне проекта и содержат секции для различных настроек, таких как базы данных, кеширование и сервисы.

   Сервисы регистрируются в секции services.

   ```php
   // /bitrix/.settings.php
   return [
       'services' => [
           'value' => [
               'someServiceName' => [
                   'className' => \VendorName\Services\SomeService::class,
               ],
               'someGoodServiceName' => [
                   'className' => \VendorName\Services\SecondService::class,
                   'constructorParams' => ['foo', 'bar'],
               ],
           ],
           'readonly' => true,
       ],
   ];
   ```

   После инициализации ядра сервисы становятся доступны.

   ```php
   $serviceLocator = \Bitrix\Main\DI\ServiceLocator::getInstance();
   $someGoodServiceName = $serviceLocator->get('someGoodServiceName');
   $someServiceName = $serviceLocator->get('someServiceName');
   ```

2. **Через файл настроек модуля \{moduleName}/.settings.php**

   Файл `.settings.php` в корне модуля описывает сервисы модуля. Это позволяет модулям иметь свои собственные настройки и сервисы, которые не зависят от глобальных настроек.

   ```php
   // someModule/.settings.php
   return [
       'services' => [
           'value' => [
               'someModule.someServiceName' => [
                   'className' => \VendorName\SomeModule\Services\SomeService::class,
               ],
               'someModule.someAnotherServiceName' => [
                   'constructor' => static function () {
                       return new \VendorName\SomeModule\Services\SecondService('foo', 'bar');
                   },
               ],
               'someModule.someGoodServiceName' => [
                   'className' => \VendorName\SomeModule\Services\SecondService::class,
                   'constructorParams' => static function () {
                       return ['foo', 'bar'];
                   },
               ],
           ],
           'readonly' => true,
       ],
   ];
   ```

   Сервисы регистрируются после подключения модуля. Используйте префикс имени модуля для уникальности, например: `disk.urlManager`, `crm.urlManager`.

3. **Через API**

   Регистрация через API осуществляется методами класса [\\Bitrix\\Main\\DI\\ServiceLocator](https://dev.1c-bitrix.ru/api_d7/bitrix/main/di/servicelocator/index.php).

   -  `getInstance()` -- получить экземпляр локатора.

   -  `addInstance(string $code, $service)` -- зарегистрировать созданный сервис.

   -  `addInstanceLazy(string $code, $configuration)` -- выполнить регистрацию с конфигурацией.

   -  `has(string $code)` -- проверить наличие сервиса.

   -  `get(string $code)` -- получить сервис. Метод создает сервис при первом обращении. Если сервиса нет, выбрасывается исключение \\Psr\\Container\\NotFoundExceptionInterface.

   Подробное описание методов -- в [справочнике API](https://dev.1c-bitrix.ru/api_d7/bitrix/main/di/servicelocator/index.php).