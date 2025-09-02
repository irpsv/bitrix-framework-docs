---
title: Создание контроллера
---

Контроллеры в Bitrix Framework обрабатывают входящие HTTP-запросы, выполняют бизнес-логику и формируют ответ клиенту.

В статье приведен пример создания контроллера для модуля `my.module`. Этот модуль устанавливает компонент `my:user.card`, который выводит карточку пользователя. Мы добавим функционал лайков в этот компонент с помощью контроллера, в результате пользователи смогут отмечать понравившиеся карточки.

Контроллер будет:

-  обрабатывать запросы на добавление и удаление лайков,

-  хранить отметки о лайках в cookie-файлах.

{% note tip "" %}

Подробно про модуль `my.module` и компонент `my:user.card` читайте в статьях:

[Создание модуля](./sozdanie-modulya#пример-создания-модуля)

[Создание компонента](./sozdanie-komponenta)

{% endnote %}

## Структура модуля с контроллером

Файлы контроллера необходимо размещать в папке `lib` модуля. Для функционала лайков создайте два файла:

-  `/Services/LikeService.php` -- для класса-сервиса лайков,

-  `/controller/user.php` -- для описания контроллера.

```
/local/modules/my.module/
├── install/
│   ├── components/
│   │   └── my/
│   │       └── user.card/          // Компонент карточки пользователя
│   │           ├── .description.php
│   │           ├── .parameters.php
│   │           ├── class.php
│   │           ├── templates/
│   │           │   └── .default/
│   │           │       ├── template.php
│   │           │       ├── script.js
│   │           │       └── style.css
│   │           └── lang/
│   │               └── ru/
│   │                   └── messages.php
│   ├── index.php                   // Файл установки
│   └── version.php                 // Версия модуля
├── lib/
│   ├── controller/
│   │   └── user.php                // Контроллер для обработки лайков
│   └── Services/
│       └── LikeService.php         // Сервис для бизнес-логики
├── lang/
│   └── ru/
│       ├── install/
│       │   └── index.php           // Языковой файл установки
│       └── messages.php            // Общие языковые файлы
└── .settings.php                   // Файл с конфигурацией
```

## Сервис для бизнес-логики

В файле `/lib/Services/LikeService.php` опишите класс `LikeService` для реализации логики работы с лайками. Чтобы обрабатывать лайки пользователей, добавьте в класс:

-  константу `COOKIE_NAME` -- имя cookie для хранения `ID` пользователей, в чьих карточках поставлены лайки.

-  метод `isLiked` -- проверяет, поставлен ли лайк в карточке пользователя с идентификатором `userId`.

-  метод `likeUser` -- добавляет идентификатор пользователя в список пользователей, в чьих карточках поставлен лайк. Если пользователь уже в списке, он не будет добавлен повторно.

-  метод `dislikeUser` -- удаляет идентификатор пользователя из списка пользователей, которым поставлен лайк.

-  метод `getLikedUsersIds` -- извлекает и декодирует список идентификаторов пользователей из cookie.

-  метод `setLikedUsersIds` -- кодирует массив идентификаторов пользователей в формат JSON и сохраняет его в cookie, которые будут храниться 30 дней.

```php
<?php

namespace My\Module\Services;

use Bitrix\Main\ArgumentException;
use Bitrix\Main\Context;
use Bitrix\Main\Web\Cookie;
use Bitrix\Main\Web\Json;

class LikeService
{
	private const COOKIE_NAME = 'liked_users';

	public function isLiked(int $userId): bool
	{
		return in_array($userId, $this->getLikedUsersIds());
	}

	public function likeUser(int $userId): void
	{
		$likedUsers = $this->getLikedUsersIds();
		$likedUsers[] = $userId;

		$this->setLikedUsersIds(
			array_unique($likedUsers)
		);
	}

	public function dislikeUser(int $userId): void
	{
		$likedUsers = array_filter($this->getLikedUsersIds(), static fn($i) => (int)$i !== $userId);

		$this->setLikedUsersIds(
			array_unique($likedUsers)
		);
	}

	private function getLikedUsersIds(): array
	{
		try
		{
			$cookieValue = Context::getCurrent()->getRequest()->getCookie(self::COOKIE_NAME);
			if (empty($cookieValue))
			{
				return [];
			}

			$value = Json::decode($cookieValue);
			if (!is_array($value))
			{
				return [];
			}

			return $value;
		}
		catch (ArgumentException)
		{
			return [];
		}
	}

	private function setLikedUsersIds(array $likedUsers): void
	{
		Context::getCurrent()->getResponse()->addCookie(
			new Cookie(
				self::COOKIE_NAME,
				Json::encode($likedUsers),
				time() + 60 * 60 * 24 * 30 // 30 days
			)
		);
	}
}
```

## Контроллер обработки лайков

В файле `/lib/controller/user.php` опишите контроллер `User`, который управляет действиями для обработки лайков. Добавьте в контроллер два метода:

-  `getDefaultPreFilters` -- определяет стандартные фильтры для всех действий,

-  `likeAction` -- обрабатывает действие лайка пользователя.

```php
<?php

namespace My\Module\Controller;

use Bitrix\Main\Engine\Controller;
use Bitrix\Main\Engine\ActionFilter;
use Bitrix\Main\Error;
use My\Module\Services\LikeService;

class User extends Controller
{
 	/**
	 * Настройка фильтров для действий
	 *
	 * @return array
	 */
	protected function getDefaultPreFilters()
	{
		return [
			new ActionFilter\Authentication(),
			new ActionFilter\HttpMethod([
				ActionFilter\HttpMethod::METHOD_POST,
			]),
			new ActionFilter\Csrf(),
		];
	}

	/**
	 * Действие для обработки лайков
	 *
	 * @param  int $likedUserId
	 *
	 * @return void
	 */
	public function likeAction(LikeService $service, int $likedUserId)
	{
		if ($likedUserId < 1)
		{
			$this->addError(new Error('Неверный ID пользователя'));

			return null;
		}

		$isLikeAction = !$service->isLiked($likedUserId);
		if ($isLikeAction)
		{
			$service->likeUser($likedUserId);
		}
		else
		{
			$service->dislikeUser($likedUserId);
		}

		return [
			'liked' => $isLikeAction,
		];
	}
}
```

## Файлы компонента

Компонент `my:user.card` был создан [ранее](./sozdanie-komponenta). Чтобы он показывал информацию о лайках, внесите изменения в установочные файлы.

### Файл template.php

В файле `/install/components/my/user.card/templates/.default/template.php` добавьте блок лайка с текстом Нравится.

```php
<?php
if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true)
die();

use Bitrix\Main\Localization\Loc;

/**
 * @var array $arParams
 * @var array $arResult
 * @var CMain $APPLICATION
 * @var CBitrixComponent $component
 * @var CBitrixComponentTemplate $this
 */

$additionalUserCardContainers = $arResult['HAS_LIKE'] ? 'my-user-card__like-text--liked' : '';

?>

<div class="my-user-card">
    <?php if (isset($arResult['PERSONAL_PHOTO_SRC'])): ?>
    <div class="my-user-card__avatar">
        <img
            class="js-my-user-card-avatar-show-in-new-page"
            src="<?= $arResult['PERSONAL_PHOTO_SRC'] ?>"
            alt="<?= $arResult['NAME'] ?>"
        >
    </div>
    <?php endif; ?>
    
    <div class="my-user-card__info">
        <h2 class="my-user-card__name">
            <?= $arResult['NAME'] ?>
        </h2>
        
        <?php if ($arParams['SHOW_EMAIL'] === 'Y'): ?>
        <p class="my-user-card__email">
            <span><?= Loc::getMessage('USER_CARD_EMAIL_LABEL') ?></span>
            <span><?= $arResult['EMAIL'] ?></span>
        </p>
        <?php endif; ?>
    </div>
    <?php if ($USER->IsAuthorized()): ?>
    <div class="my-user-card__like-container">
        <span 
            class="my-user-card__like-text js-my-user-card-like <?= $additionalUserCardContainers ?>"
            data-user-id="<?= $arParams['USER_ID'] ?>"
        >
            Нравится
        </span>
    </div>
    <?php endif; ?>
</div>
```

### Файл script.js

В файл `/install/components/my/user.card/templates/.default/script.js` добавьте обработку клика на текст Нравится.

```php
BX.ready(() => {
    // Обработка клика на аватар
    document.querySelectorAll('.js-my-user-card-avatar-show-in-new-page').forEach((item) => {
        BX.Event.bind(item, 'click', (e) => {
            window.open(item.src, '_blank');
        });
    });

    // Обработка клика на текст "Нравится"
    document.querySelectorAll('.js-my-user-card-like').forEach((element) => {
        BX.Event.bind(element, 'click', (e) => {
            const userId = element.dataset.userId;
            
            BX.ajax.runAction(
                'my:module.user.like',
                {
                    data: { likedUserId: userId }
                }
            ).then((response) => {
                if (response.status === 'success') {
					if (response.data.liked)
					{
						element.classList.add('my-user-card__like-text--liked');
					}
					else
					{
						element.classList.remove('my-user-card__like-text--liked');
					}
                }
            }).catch((response) => {
                console.error('Error:', response.errors);
            });
        });
    });
});
```

### Файл style.css

В файл `/install/components/my/user.card/templates/.default/style.css` добавьте стили для текста лайка .

```php
.my-user-card {
    background: #cff9f2;
    padding: 15px;
    width: 230px;
}

.my-user-card__avatar {
    margin: 0 0 15px 0;
}

.my-user-card__avatar img {
    cursor: pointer;
    max-width: 200px;
}

.my-user-card__info {
    text-align: left;
}

.my-user-card__name {
    font-size: 1.5rem;
    margin: 0 0 10px 0;
}

.my-user-card__email {
    font-size: 1rem;
}

.my-user-card__like-container {
    margin-top: 10px;
}

.my-user-card__like-text {
    cursor: pointer;
    transition: color 0.3s ease, font-weight 0.3s ease;
    user-select: none;
	color: #999;
	font-weight: normal;
}

.my-user-card__like-text:hover {
    text-decoration: underline;
}

.my-user-card__like-text--liked {
	color: #0066cc;
	font-weight: bold;
}
```

### Файл class.php

Внесите изменения в основной класс компонента `/install/components/my/user.card/class.php`:

-  подключите `ServiceLocator` и `LikeService`,

-  в выборку пользователя добавьте поле `ID`,

-  в `arResult` добавьте проверку лайков `HAS_LIKE`,

-  добавьте метод `isUserLiked()`, который проверяет лайк через `LikeService`.

```php
<?php

use Bitrix\Main\DI\ServiceLocator;
use Bitrix\Main\Loader;
use My\Module\Services\LikeService;

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true)
{
	die();
}

class UserCardComponent extends CBitrixComponent
{
	/**
	 * Подготавливаем входные параметры
	 *
	 * @param  array $arParams
	 *
	 * @return array
	 */
	public function onPrepareComponentParams($arParams)
	{
		$arParams['USER_ID'] ??= 0;
		$arParams['SHOW_EMAIL'] ??= 'Y';

		return $arParams;
	}
	/**
	 * Основной метод выполнения компонента
	 *
	 * @return void
	 */

	public function executeComponent()
	{
		if (!Loader::includeModule('my.module'))
		{
			ShowError('Модуль my.module не установлен');

			return;
		}

		// кешируем результат, чтобы не делать постоянные запросы к базе
		if ($this->startResultCache())
		{
           $this->initResult();

			// в случае если ничего не найдено, отменяем кеширование
			if (empty($this->arResult))
			{
				$this->abortResultCache();
				ShowError('Пользователь не найден');

				return;
			}
			$this->includeComponentTemplate();
		}
	}

	/**
	 * Инициализируем результат
	 *
	 * @return void
	 */
	private function initResult(): void
	{
		$userId = (int)$this->arParams['USER_ID'];
		if ($userId < 1)
		{
			return;
		}

		$user = \Bitrix\Main\UserTable::query()
			->setSelect([
				'ID',
				'NAME',
				'EMAIL',
				'PERSONAL_PHOTO',
			])
			->where('ID', $userId)
			->fetch()
		;
		if (empty($user))
		{
			return;
		}

		$this->arResult = [
			'NAME' => $user['NAME'],
			'EMAIL' => $user['EMAIL'],
			'HAS_LIKE' => $this->isUserLiked((int)$user['ID']),
		];

		// получаем путь до аватарки, в случае если она указана
		if (!empty($user['PERSONAL_PHOTO']))
		{
			$this->arResult['PERSONAL_PHOTO_SRC'] = \CFile::GetPath($user['PERSONAL_PHOTO']);
		}
	}

 	/**
	 * Проверяем, поставил ли текущий пользователь лайк
	 *
	 * @return void
	 */
	private function isUserLiked(int $userId): bool
	{
		/**
		 * @var LikeService $service
		 */
		$service = ServiceLocator::getInstance()->get(LikeService::class);

		return $service->isLiked($userId);
	}
}
```

## Настройки модуля

Добавьте регистрацию контроллера в файл `.settings.php`:

```
<?php
return [
    'controllers' => [
        'value' => [
            'defaultNamespace' => '\\My\\Module\\Controller',
        ],
        'readonly' => true,
    ]
];
```

## Размещение компонента на сайте

1. В административном разделе откройте страницу *Marketplace > Установленные решения*.

2. Установите модуль *Мой модуль (my.module)*.

3. Создайте страницу и разместите на ней компонент *Карточка пользователя*.

После настройки компонента авторизованные пользователи увидят текст «Нравится» в карточке. Изначально текст будет серым, но, если поставить лайк, цвет изменится на синий.

![](./sozdanie-kontrollera-3.png){width=548px height=424px}

## Архив с примером модуля

Все файлы модуля с контроллером можно [скачать в архиве](https://dev.1c-bitrix.ru//docs/chm_files/my.module.zip). Для работы распакуйте архив в папку `/local/modules/`.