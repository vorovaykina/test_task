# Тестовое задание: Корзина, API магазинов, Push-уведомления

**Кандидат:** Уварова Евгения 
**Дата:** 29 мая 2026  
**Заказчик:** Интернет-магазин "Петрушка Зеленая"

---

## Содержание

- [Задание 1: Анализ требований](#задание-1-анализ-требований)
- [Задание 2: Проектирование API](#задание-2-проектирование-api)
- [Задание 3: Архитектура Push-уведомлений](#задание-3-архитектура-push-уведомлений)

---

## Задание 1: Анализ требований

### Найденные противоречия

Пункты ТЗ:1 и 4. Проблема: Можно добавить 5 товаров по 10 штук = 50, но лимит 20 штук.
Пункты ТЗ:2 и 9. Проблема: Удаление либо отдельной кнопкой, либо при уменьшении до 0.
Пункты ТЗ: 7 и 12. Проблема: Цена фиксируется при добавлении И автоматически обновляется из каталога. 
Пункты ТЗ: 10, 11. Проблема: Реклама не описана технически, пункт 11 - не техническое требование. 
Пункт ТЗ: 6. Проблема: Непонятно, какой именно лимит превышен. 

### Исправленная версия ТЗ

1. Пользователь может добавить от 1 до 10 единиц одного товара за раз.
2. В корзине - не более 5 разных товаров.
3. Общее количество - не более 20 штук.
4. Количество товара можно менять от 1 до остатка (не более 10).
5. Удаление — только кнопкой «Удалить».
6. Цена фиксируется при добавлении в корзину.
7. Сообщение об ошибке: «Нельзя добавить: превышен лимит корзины (макс. 5 товаров, общее количество не более 20 шт.)».
8. Реклама - отдельным блоком, не влияет на лимиты.

### Вопросы к заказчику

- Что именно нужно бизнесу: фиксация цены при добавлении или автоматическое обновление?
- Что делать, если пользователь уже имеет 5 разных товаров и пытается добавить 6-й?
- Как обрабатывать товары, удалённые из каталога, но лежащие в корзине?
- Реклама — это отдельный блок в корзине или перемешанные с товарами карточки?
- Реклама кликабельна? Куда ведет ссылка - внутри приложения или внешний браузер?
- Нужна ли аналитика по рекламе?
- Кто управляет расписанием рекламы?
- Что делать с корзиной при возврате товара? Восстанавливать позиции?

---

## Задание 2: Проектирование API

### Запрос
GET /api/v1/partners/stores?lat=55.751244&lon=37.618423
Authorization: Bearer <token>

### Пример ответа (JSON)

```json
{
  "stores": [
    {
      "id": "metro_1",
      "name": "METRO",
      "delivery_info": "Ближайшая доставка сегодня 21:00–23:00",
      "external_url": "https://shop.metro.ru/partner/petrushka"
    },
    {
      "id": "ashan_2",
      "name": "Ашан",
      "delivery_info": "Ближайшая доставка сегодня 18:00–20:00",
      "external_url": "https://ashan.ru/petrushka"
    },
    {
      "id": "vkusvill_3",
      "name": "ВкусВилл",
      "delivery_info": "Быстрая доставка от 20 до 60 минут",
      "external_url": "https://vkusvill.ru/green-parsley"
    },
    {
      "id": "victoria_4",
      "name": "ВИКТОРИЯ",
      "delivery_info": "Ближайшая доставка сегодня 17:00–19:00",
      "external_url": "https://victoria.ru/store/petrushka"
    }
  ]
}
```

---

## Задание 3: Архитектура Push-уведомлений

```mermaid
graph TD
    classDef mobile fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef gateway fill:#cce5ff,stroke:#007bff,stroke-width:2px;
    classDef broker fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    classDef service fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    classDef external fill:#e2e3e5,stroke:#6c757d,stroke-width:2px;
    classDef db fill:#e8daef,stroke:#8e44ad,stroke-width:2px;

    Client[" Мобильное приложение\n(iOS / Android)"]:::mobile
    API_GW["API Gateway / BFF"]:::gateway
    
    subgraph Microservices ["Микросервисы-Инициаторы (Бизнес-логика)"]
        Cart[" Сервис Корзины\n(Брошенная корзина)"]:::service
        Order[" Сервис Заказов\n(Отмена заказа)"]:::service
        Promo[" Сервис Маркетинга\n(Рекламная рассылка)"]:::service
    end

    Broker{"Брокер сообщений\n(Kafka / RabbitMQ)"}:::broker
    
    PushService[" Push Notification Service"]:::service
    TokenDB[(" БД Токенов\n(User_ID -> Device_Token)")]:::db

    APNS[" APNs\n(Apple Push Service)"]:::external
    FCM[" FCM\n(Firebase Cloud Messaging)"]:::external

    Client -->|1. Передает Device Token| API_GW
    API_GW -->|2. Роутит запрос| PushService
    PushService -->|3. Сохраняет| TokenDB

    Cart -->|Событие: cart_abandoned| Broker
    Order -->|Событие: order_cancelled| Broker
    Promo -->|Событие: promo_campaign| Broker

    Broker -->|Слушает топики| PushService
    PushService -.->|Получает Device Token| TokenDB
    PushService -->|Формирует Payload| APNS
    PushService -->|Формирует Payload| FCM

    APNS -->|Доставка iOS| Client
    FCM -->|Доставка Android| Client
