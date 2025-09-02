---
title: Пре- и постфильтры
---

Фильтры -- это обработчики, которые выполняются до или после действия (Action). Они могут отменить выполнение действия или изменить его результат. Фильтры помогают сделать код более модульным и управляемым, обеспечивая дополнительный уровень контроля над выполнением действий.

## Типы фильтров

-  Префильтры (prefilter) -- выполняются до запуска действия. Могут отменить выполнение.

-  Постфильтры (postfilter) -- выполняются после действия. Могут изменить результат.

## Основные фильтры

### Фильтр HTTP-методов

`\Bitrix\Main\Engine\ActionFilter\HttpMethod`

Проверяет HTTP-метод действия и блокирует выполнение, если метод не разрешен. Полезен, когда необходимо ограничить выполнение действия только определенными HTTP-методами, например, POST для изменения данных.

Конструктор:

```php
__construct(array $allowedMethods = [self::METHOD_GET])
```

-  `$allowedMethods` -- список допустимых HTTP-методов. По умолчанию GET.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\HttpMethod;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new HttpMethod([HttpMethod::METHOD_POST]);
        // Дальнейшая реализация
    }
}
```

### Фильтр аутентификации

`\Bitrix\Main\Engine\ActionFilter\Authentication`

Проверяет аутентификацию пользователя и блокирует действие, если проверка не пройдена (HTTP статус 401). Может перенаправить на страницу авторизации. Используется для защиты действий, требующих аутентификации, например, доступ к личным данным пользователя.

Конструктор:

```php
__construct($enableRedirect = false)
```

-  `$enableRedirect` -- включает редирект на авторизацию. По умолчанию `false`.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\Authentication;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new Authentication(true);
        // Дальнейшая реализация
    }
}
```

### Фильтр CSRF-защиты

`\Bitrix\Main\Engine\ActionFilter\Csrf`

Проверяет наличие и корректность CSRF-токена и блокирует действие, если проверка не пройдена. Необходим для защиты от CSRF-атак, особенно в формах и AJAX-запросах.

Конструктор:

```php
  __construct($enabled = true, $tokenName = 'sessid', $returnNew = true)
```

-  `$enabled` -- включает проверку токена. По умолчанию `true`.

-  `$tokenName` -- имя токена. По умолчанию `'sessid`'.

-  `$returnNew` -- возвращает новый токен при неудаче. По умолчанию `true`.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\Csrf;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new Csrf();
        // Дальнейшая реализация
    }
}
```

### Фильтр закрытия сессии

`\Bitrix\Main\Engine\ActionFilter\CloseSession`

Закрывает сессию перед выполнением действия.  Полезен для повышения производительности, когда изменения в сессии не требуются. Будьте внимательны, так как изменения в сессии после закрытия не сохранятся.

Конструктор:

```php
__construct($enabled = true)
```

-  `$enabled` -- включает фильтр. По умолчанию `true`.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\CloseSession;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new CloseSession();
        // Дальнейшая реализация
    }
}
```

### Фильтр области действия

`\Bitrix\Main\Engine\ActionFilter\Scope`

Блокирует действия для определенного scope. Полезен для ограничения доступа к действиям в зависимости от контекста выполнения, например, только для AJAX-запросов.

Конструктор:

```php
__construct($scopes)
```

-  `$scopes` -- перечисление допустимых scopes.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\Scope;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new Scope(Scope::AJAX);
        // Дальнейшая реализация
    }
}
```

### Фильтр CORS-настроек

`\Bitrix\Main\Engine\ActionFilter\Cors`

Устанавливает заголовки для управления CORS. Используется для настройки CORS, когда требуется доступ к ресурсам с другого домена.

Конструктор:

```php
__construct(string $origin = null, bool $credentials = false)
```

-  `$origin` -- заголовок Access-Control-Allow-Origin. По умолчанию `null`.

-  `$credentials` -- устанавливает заголовок Access-Control-Allow-Credentials. По умолчанию `false`.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\Cors;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new Cors('https://example.com', true);
        // Дальнейшая реализация
    }
}
```

### Фильтр типа контента

`\Bitrix\Main\Engine\ActionFilter\ContentType`

Разрешает выполнение действия только для допустимых content-type. Полезен для обеспечения корректной обработки данных в зависимости от типа контента, например, JSON.

Конструктор:

```php
__construct(array $allowedTypes)
```

-  `$allowedTypes` -- допустимые content-type.

Пример использования:

```php
use \Bitrix\Main\Engine\ActionFilter\ContentType;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new ContentType(['application/json']);
        // Дальнейшая реализация
    }
}
```

### Фильтр декодирования POST-запросов

`\Bitrix\Main\Engine\ActionFilter\PostDecode`

Перекодирует данные из POST-запроса, если кодировка проекта отличается от UTF-8.

### Фильтр типа пользователя

`\Bitrix\Intranet\ActionFilter\UserType`

Проверяет, принадлежит ли пользователь к разрешенным типам, например, экстранет или интранет. Используется для ограничения доступа к действиям в зависимости от типа пользователя.

Конструктор:

```php
__construct(array $allowedUserTypes)
```

-  `$allowedUserTypes` -- допустимые типы пользователей.

Пример использования:

```php
use \Bitrix\Intranet\ActionFilter\UserType;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new UserType(['extranet', 'intranet']);
        // Дальнейшая реализация
    }
}
```

{% note info "" %}

При использовании фильтра UserType убедитесь, что модуль intranet подключен.

{% endnote %}

### Фильтр интранет-пользователя

`\Bitrix\Intranet\ActionFilter\IntranetUser`

Проверяет, является ли пользователь интранет-пользователем. Упрощенный вариант UserType.

Пример использования:

```php
use \Bitrix\Intranet\ActionFilter\IntranetUser;

class ExampleClass
{
    public function exampleMethod()
    {
        $filter = new IntranetUser();
        // Дальнейшая реализация
    }
}
```

{% note info "" %}

При использовании фильтра IntranetUser убедитесь, что модуль intranet подключен.

{% endnote %}

## Как использовать фильтры в контроллерах

Фильтры можно применять в контроллерах через метод `configureActions()`.

```php
use \Bitrix\Main\Engine\Controller;
use \Bitrix\Main\Engine\ActionFilter;

class ExampleController extends Controller
{
    public function configureActions()
    {
        return [
            'exampleAction' => [ // Название экшена
                'prefilters' => [ // Фильтры, которые выполнятся до вызова экшена
					// Разрешаем только POST-запросы
                    new ActionFilter\HttpMethod([ActionFilter\HttpMethod::METHOD_POST]),
					// Требуем авторизацию + редирект на страницу входа
                    new ActionFilter\Authentication(true),
                ],
                'postfilters' => [ // Фильтры, выполняемые после экшена
					// Проверка CSRF-токена, параметры по умолчанию
                    new ActionFilter\Csrf(),
                ],
            ],
        ];
    }

    public function exampleAction()
    {
        // Логика действия
        return ['result' => 'success'];
    }
```