# Тестовое задание Helastel

Разработчика попросили получить данные из REST-API стороннего сервиса.
Данные необходимо было кешировать, ошибки логировать. Разработчик с задачей справился, ниже предоставлен его код.
* Проведите максимально подробный Code Review. Необходимо написать, с чем вы не согласны и почему.
* Исправьте обозначенные ошибки, предоставив свой вариант кода.

``` php
<?php

namespace src\Integration;

class DataProvider
{
    private $host;
    private $user;
    private $password;

    /**
     * @param $host
     * @param $user
     * @param $password
     */
    public function __construct($host, $user, $password)
    {
        $this->host = $host;
        $this->user = $user;
        $this->password = $password;
    }

    /**
     * @param array $request
     *
     * @return array
     */
    public function get(array $request)
    {
        // returns a response from external service
    }
}
```

``` php
<?php

namespace src\Decorator;

use DateTime;
use Exception;
use Psr\Cache\CacheItemPoolInterface;
use Psr\Log\LoggerInterface;
use src\Integration\DataProvider;

class DecoratorManager extends DataProvider
{
    public $cache;
    public $logger;

    /**
     * @param string $host
     * @param string $user
     * @param string $password
     * @param CacheItemPoolInterface $cache
     */
    public function __construct($host, $user, $password, CacheItemPoolInterface $cache)
    {
        parent::__construct($host, $user, $password);
        $this->cache = $cache;
    }

    public function setLogger(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    /**
     * {@inheritdoc}
     */
    public function getResponse(array $input)
    {
        try {
            $cacheKey = $this->getCacheKey($input);
            $cacheItem = $this->cache->getItem($cacheKey);
            if ($cacheItem->isHit()) {
                return $cacheItem->get();
            }

            $result = parent::get($input);

            $cacheItem
                ->set($result)
                ->expiresAt(
                    (new DateTime())->modify('+1 day')
                );

            return $result;
        } catch (Exception $e) {
            $this->logger->critical('Error');
        }

        return [];
    }

    public function getCacheKey(array $input)
    {
        return json_encode($input);
    }
}
```

## Разбор задания

Самое критичное это использование `$this->logger->critical('Error');`
т.к. программист может забыть создать логгер и добавить его нашему классу
(и в случае отлова исключений программа без инстанса логгера программа просто завершится ошибкой).

Т.к. логгер это обязательно для нашего ДатаМенеджера, то это требование лучше перенести в конструктор Менеджера.

Неправильно использование композиции `class DecoratorManager extends DataProvider`. Лучше создавать DataProvider отдельно и передавать объект в конструктор DataManager.
(предлагаю создать отдельный интерфейс для DataProvider и использовать его в конструкторе).

Получится так
``` php
public function __construct(DataProviderInterface $provider, CacheItemPoolInterface $cache, LoggerInterface $logger)
{
    $this->cache = $cache;
    $this->provider = $provider;
    $this->logger = $logger;
}
```

Неправильно указаны области видимости $cache и $logger. Напрямую менять и перезаписывать эти объекты нельзя давать.

Делаем так, не забываем отдельную переменную для ДатаПровайдера.
``` php
/**
 * @var CacheItemPoolInterface
*/
private $cache;
/**
 * @var LoggerInterface
*/
private $logger;
/**
 * @var DataProviderInterface
*/
private $dataProvider;
```

В аннотациях метода getResponse и getCacheKey используется {@inheritdoc}, однако в родительском классе нет такого метода.

DecoratorManager я бы переименовал в DataManager, чтобы не путать программистов в применении паттерна Декоратор.

`$this->logger->critical('Error');` лучше писать понятную причину ошибки, а не отписку типа Error.

Метод getCacheKey использует json_encode, возможно генерируемый ключ для одних и тех же данных может не совпадать в случае
разного порядка в массиве $input.

В случае ошибок метод getResponse возвращает пустой массив, что не совсем правильно для конечного пользователя.
Лучше всего дать бизнес-логике реальную причину.

Такой небольшой код-ревью ждал бы программиста от меня. Переписывать его код и тратить свое время на исправление
его ошибок я бы конечно не стал.

