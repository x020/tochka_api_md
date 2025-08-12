# Open API 

> Version v1.78.1

# Про API

**Tochka OpenApi** — продукт, который поможет вам автоматизировать бизнес-процессы.
С API вы сможете объединить сервисы Точки со своими приложениями и сервисами.

С помощью API Точки можно:

 - получать список счетов, подключённых к аккаунту;
 - получать выписку по счёту;
 - создавать черновики платёжек на подпись;
 - получать вебхуки о входящих и исходящих платежах;
 - подписывать платежи из интегрированного сервиса;
 - регистрироваться в СБП (Системе быстрых платежей), создавать QR-коды, отслеживать оплаты по ним и делать возвраты;
 - получать балансы счетов;
 - создавать счета и закрывающие документы;
 - создавать платёжные ссылки для приёма оплаты по карте и/или СБП.


# Терминология

**Access Token Client** — токен, дающий доступ к методам Tochka OpenApi. С токеном можно вызывать методы в течение 24 часов, потом его нужно будет получить заново.

**Access Token Hybrid** — токен доступа, полученный после прохождения гибридного потока. Применяется для доступа к счетам, балансу, выпискам и так далее. Срок действия также 24 часа.

**Refresh Token Hybrid** — токен, который отправляет новый Access Token Hybrid, если предыдущий устарел. Refresh Token активен 30 дней.

**Redirect Uri** — uri для перехода, который указывали при регистрации в API в интернет-банке. На него будет произведён редирект для получения Access и Refresh токенов.

**Code** — код авторизации, полученный через редирект браузера пользователя приложения. Срок действия — 5 минут.

**Id Token** — токен безопасности, который содержит параметры аутентификации конечного пользователя сервером авторизации. Id token представляется в виде структуры JWT.

**Client Id** — идентификатор приложения.

**Client Secret** — пароль приложения.

**Consent ID** — идентификатор списка разрешений.

**Customer Code** — идентификатор компании в запросе.

**AccountId** — номер расчётного счёта и БИК банка.

**JWT** — веб-токен, который дает долгоживущий доступ к методам Tochka OpenApi. Токену можно задать произвольное время жизни вручную или сделать бессрочным, не указывая дату окончания срока жизни.


# Алгоритм работы по OAuth 2.0

## Принцип работы

1. [Зарегистрируйте своё приложение](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Zaregistrirujte-svoyo-prilozhenie). В ответ вы получите `client_id` и `client_secret`, с которыми и будете запрашивать данные;

2. [Получите токен для работы с разрешениями](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Poluchite-token-dlya-raboty-s-razresheniyami);

3. [Задайте разрешения](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Zadajte-razresheniya);

4. [Сформируйте запрос на подписание списка разрешений и попросите пользователя подтвердить список](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Sformirujte-zapros-na-podpisanie-spiska-razreshenij-i-poprosite-polzovatelya-podtverdit-spisok);

5. [Обменяйте код на токен для доступа к API](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Obmenyajte-kod-na-token-dlya-dostupa-k-API);

6. [Обменяйте refresh токен на новую пару access/refresh токенов](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Obmenyajte-refresh-token-na-novuyu-paru-accessrefresh-tokenov);

7. [При необходимости проверьте Access Token Hybrid](/doc/v2/redoc/section/Algoritm-raboty-po-OAuth-2.0#Kak-proverit-Access-Token-Hybrid).

Также можно настроить интеграцию упрощённым методом с помощью [веб-токена](/doc/v2/redoc/section/Algoritm-raboty-s-JWT-tokenom)

### Простая схема

На схеме представлен общий флоу работы от первого запроса разрешений до получения результата. Пройти этот путь нужно только один раз, чтобы создать токен oAuth.

```mermaid
sequenceDiagram
    participant C as Клиент (владелец счёта)
    participant P as Партнёр (приложение)
    participant T as Точка
    C->>+P: Хочу подключить Точку в твоём приложении
    P-->>+T: Получение токена для работы с разрешениями
    T-->>-P: Успех, получен токен для работы с разрешениями
    Note over P,T: полученный токен подходит только для работы с разрешениями
    P-->>+T: Создай ресурс на доступ
    Note over P,T: в теле запроса передаём модель разрешения
    T-->>-P: Успех, получен id разрешения
    P->>P: Подписываем запрос, в котором содержится информация о разрешениях
    P->>-C: Получена ссылка, перейдите по ней и подтвердите разрешения
    C->>+T: Переходим на страницу подтверждения разрешений
    T->>T: Проверяем авторизацию
    T->>T: Генерируем страницу с разрешениями
    T->>-C: Показываем страницу с разрешениями и ждём подтверждения
    C-->>+T: Подтверждение разрешений
    T-->>-C: Успех, получена ссылка, по которой вернутся обратно
    C->>+P: Переходим по ссылке и передаём все необходимые данные
    Note over C,P: Ждем получения `code` на `redirect_uri`
    P-->>+T: Меняем `code` на `Access Token Hybrid``
    T-->>-P: И снова успех, получен токен для дальнейших запросов
```

Далее добавляйте `Access Token Hybrid` в заголовок `Authorization: Bearer <Access Token Hybrid>` при совершении запросов.


### Модель запросов

При совершении запросов всегда нужно добавлять токен в заголовок авторизации.
Если вы создаёте запрос по компании, обязательно указывайте идентификатор этой компании в заголовке `CustomerCode`.
Запросить список компаний клиента можно вот [так](/doc/v2/redoc/tag/Rabota-s-klientami#get_customers_list_open_banking__apiVersion__customers_get).

## Зарегистрируйте своё приложение

Если у вашей компании ещё нет счёта в Точке, оставьте заявку на нашем [сайте](https://tochka.com/public-api/).
Мы свяжемся с вами в течение 2 рабочих дней, а затем отправим необходимые данные на вашу электронную почту.

Если вы уже работаете в Точке, зарегистрируйте приложение в интернет-банке: раздел «Интеграции и API» — [«Подключить»](https://i.tochka.com/bank/services/m/integration/new).  
Нажмите на кнопку «Зарегистрировать oAuth 2.0 приложение» и выберите необходимые разрешения для приложения.


## Получите токен для работы с разрешениями

Пройти oAuth авторизацию по принципу `client_credentials`.

### Примеры запросов

Пример запроса Партнёр -> Точка:
```
curl --request POST 'https://enter.tochka.com/connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=test_app' \
--data-urlencode 'client_secret=test_secret' \
--data-urlencode 'grant_type=client_credentials' \
--data-urlencode 'scope=accounts balances customers statements sbp payments' 
```

Пример ответа:
```
{
    "token_type": "bearer",
    "access_token": "p6awaKAleytQh6mKc61vXd22YniDerme",
    "expires_in": 86400
}
```

## Задайте разрешения

`expirationDateTime` — это срок действия разрешений. Если нужны бессрочные, не передавайте этот параметр.

### Примеры запросов

Пример запроса Партнёр -> Точка:
```
curl --request POST 'https://enter.tochka.com/uapi/v1.0/consents' \
--header 'Authorization: Bearer p6awaKAleytQh6mKc61vXd22YniDerme' \
--header 'Content-Type: application/json' \
--data-raw '{
  "Data": {
    "permissions": [
      "ReadAccountsBasic",
      "ReadAccountsDetail",
      "MakeAcquiringOperation",
      "ReadAcquiringData",
      "ReadBalances",
      "ReadStatements",
      "ReadCustomerData",
      "ReadSBPData",
      "EditSBPData",
      "CreatePaymentForSign",
      "CreatePaymentOrder",
      "ManageWebhookData",
      "ManageInvoiceData"
    ],
    "expirationDateTime": "2030-10-03T00:00:00+00:00"
  }
}
'
```

Пример ответа:
```
{
    "Data": {
        "status": "AwaitingAuthorisation",
        "creationDateTime": "2024-01-10T11:40:16.941480+00:00",
        "statusUpdateDateTime": "2024-01-10T11:40:16.941498+00:00",
        "permissions": [
            "ReadAccountsBasic",
            "ReadAccountsDetail",
            "MakeAcquiringOperation",
            "ReadAcquiringData",
            "ReadBalances",
            "ReadStatements",
            "ReadCustomerData",
            "ReadSBPData",
            "EditSBPData",
            "CreatePaymentForSign",
            "CreatePaymentOrder",
            "ManageWebhookData",
            "ManageInvoiceData"
        ],
        "expirationDateTime": null,
        "consentId": "705ba15f-a109-4156-bb7f-50c21413c158",
        "customerCode": null,
        "applicationName": null,
        "consumerId": "0b5efd83-2f20-4350-b777-c2525efc68bc",
        "clientId": null
    },
    "Links": {
        "self": "https://enter.tochka.com/uapi/v1.0/consents"
    },
    "Meta": {
        "totalPages": 1
    }
}
```

## Сформируйте запрос на подписание списка разрешений и попросите пользователя подтвердить список

Сформируйте ссылку для подтверждения списка разрешений и перенаправьте пользователя на неё. В параметрах нужно передать следующий набор:
 - client_id — идентификатор клиента;
 - response_type — код авторизации;
 - state — произвольная строка (подойдёт для связи запроса и ответа, то есть идентификации клиента);
 - redirect_uri — URI, предварительно зарегистрированный на авторизационном сервере. На него перенаправим пользователя;
 - scope — `accounts balances customers statements sbp payments acquiring` то есть запрашиваемая область действия токена доступа;
 - consent_id — id разрешения из ответа с предыдущего шага.


Пример URL, куда направляем пользователя:
```
https://enter.tochka.com/connect/authorize?client_id=test_app&response_type=code&state=Vuihvsds&redirect_uri=http://localhost:8000/&scope=accounts%20balances%20customers%20statements%20sbp%20payments&consent_id=705ba15f-a109-4156-bb7f-50c21413c158
```

## Обменяйте код на токен для доступа к API

Подхватываем пользователя на `redirect_url` и меняем код на токен запросом:

```
curl --request POST 'https://enter.tochka.com/connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=test_app' \
--data-urlencode 'client_secret=test_secret' \
--data-urlencode 'grant_type=authorization_code' \
--data-urlencode 'scope=accounts balances customers statements sbp payments' \
--data-urlencode 'code=' \
--data-urlencode 'redirect_uri=http://localhost:8000/'
```

Ответ:
```
{
    "refresh_token": "ViYnf8sDILwKpvUMcIQJEqCYy88zW1ys",
    "access_token": "JlvB7rgB0Q9vuuh0YNVQ4hBkGChIqovR",
    "token_type": "bearer",
    "expires_in": 86400,
    "state": "Vuihvsds",
    "user_id": "b5fb8f68-10b3-48cd-a3e9-642162432724" 
}
```


## Обменяйте refresh токен на новую пару access/refresh токенов

Для обмена токена выполните запрос:

```
curl --request POST 'https://enter.tochka.com/connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=test_app' \
--data-urlencode 'client_secret=test_secret' \
--data-urlencode 'grant_type=refresh_token' \
--data-urlencode 'refresh_token=ViYnf8sDILwKpvUMcIQJEqCYy88zW1ys'
```

Ответ:
```
{
    "refresh_token": "7lsLqrr2ug3j4KPPWgkUC8zPQ9W2HKH6",
    "token_type": "bearer",
    "access_token": "scD56TmsyrCV9kpExKlY2p61BrCY0tNj",
    "expires_in": 86400
}
```

## Как проверить Access Token Hybrid
Для этого проверьте аутентификацию клиента по полученному токену. В ответе будет [jwt](https://jwt.io/introduction), в котором зашифрован Customer Code (sub) и Client ID (aud). Этот метод — не обязательный для работы с API.
```
curl --request POST 'https://enter.tochka.com/connect/introspect' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'access_token=dnjsndamklcmda8n3m2jkndsa76'
```
Ответ:
```
"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.ewogICJpc3MiOiAiZW50ZXIudG9jaGthLmNvbSIsCiAgInN1YiI6ICIzMDk4NzY1NDMiLAogICJhdWQiOiAiQU1KanMzbmFkamFuMjFqa2Exc211NyIKfQ.vPOLUlTzMm3HJz4D76JfO3Y8dJHf57v2oMMhbWJowVqAIhs2wAAOGouORzoo2Az28bbEYxaHeITDpMWg035UVwGQYskL7qeFBO2m5bFlPoAnt1RyFMblhjG8iLb5yAFWtJUD1tyJDR1FYAdJshVMddY4_ZDyMUIK3fJElbr2xAWuk_cX9Y5hoGe1plB0JpWLEgPaFuB9TjCjvpz9sWbMBvczC3bx_07-t0I8Jodh82THaz7o63IctEwlJ3vqNyr4l01JA6cZidmpAgQJ-448wQbOWBT_WOHMHWwc2EuQy1vDni3jHm6AvWlnnXJyKGBXClMqpj67m2lpuPfIoqyC_g"
```

# Алгоритм работы с JWT-токеном

Это упрощённый метод авторизации — в нём не используется _redirect_uri_.  

Чтобы получить токен в интернет-банке Точки перейдите в раздел «Интеграции и API» — [«Подключить»](https://i.tochka.com/bank/services/m/integration/new).  
Нажмите на кнопку «Сгенерировать JWT-ключ» и выберите необходимые разрешения для приложения.  

Когда JWT сгенерируется, поместите ключ в заголовок `Authorization: Bearer <jwt_token>`, а также скопируйте `client_id` — он понадобится для некоторых запросов в будущем.
```mermaid
sequenceDiagram
    participant C as Клиент (владелец счёта)
    participant T as Точка
    C-->+T: Хотим подключить Точку в своём приложении
    C->>+T: Вводим название токена, указываем срок его жизни, выбираем необходимые разрешения и подтверждаем
    T-->>-C: Высылаем смс с кодом
    C-->>+T: Вводим смс-код
    T->>-C: Успех! Показываем сгенерированный JWT-токен и client_id, используйте их для дальнейших запросов
```

# Коллекция запросов

## Ссылки

- [Коллекция запросов](/doc/openapi/static/postman/OpenAPI.postman_collection.json)
- [Настройка окружения](/doc/openapi/static/postman/Setting.postman_environment.json)
- [Настройка окружения для песочницы](/doc/openapi/static/postman/Sandbox.postman_environment.json)

## Как использовать коллекцию

Поставьте Postman не ниже версии 2.1. на свой компьютер. Затем импортируйте коллекцию и набор необходимых переменных с помощью кнопки `Import`.

![Import](/doc/openapi/static/image/import.png)

Выберите `Upload files`

![Upload file](/doc/openapi/static/image/import-2.png)

Другой вариант: сначала выберите вариант `Link`, в нём укажите сначала ссылку на коллекцию, а затем — на настройку окружения.

## Как настроить окружение

Теперь осталось лишь указать `client_id` и `client_secret` своего приложения в настройках переменных.

![Set env](/doc/openapi/static/image/env.png)

## Как выполнять запросы

Чтобы получить полноценный токен, нужно пройти авторизацию. Для этого сделайте три следующих запроса:
 - Получите токен работы с разрешениями с помощью первого POST запроса;
 - Создайте разрешение — клиенту нужно будет позднее его подтвердить. В ответ вы получите `consent_id`, который надо передать в ссылке и перенаправить клиента для подтверждения.

> **Алгоритм уже готов: выполните запрос, откройте в Postman консоль из нижнего левого края и скопируйте автоматически сформированную ссылку в браузер.**

 - Когда клиент пройдёт авторизацию в браузере и подтвердит все настройки, он перенаправится на `redirect_uri` вашего приложения.
Вы получите `code`, который далее меняется на полноценный `Access Token Hybrid` для выполнения запросов.


# Описание методов

## Баланс счёта
Для работы с балансами счетов должно быть выдано разрешение `ReadBalances`.

С помощью методов: [Get Balance Info](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_balance_info_open_banking__apiVersion__accounts__accountId__balances_get) и [Get Balances List](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_balances_list_open_banking__apiVersion__balances_get), можно получать информацию о банковском остатке по счетам.  

Обратите внимание: доступный остаток по счёту может отличаться от банковского.  
Доступный остаток может быть меньше, например, из-за блокировок на сумму или резерва по карточным операциям.

### Описание типов балансов

- **OpeningAvailable** — начальный баланс счёта без учёта овердрафта и блокировок;  
- **ClosingAvailable** — доступный баланс счёта без учёта овердрафта и за вычетом всех блокировок, сумма свободных к использованию денег;  
- **Expected** — сумма заблокированных на счёте денег;  
- **OverdraftAvailable** — сумма доступного лимита овердрафта на счёте. Отображается, только если на счёте есть выданный овердрафт.

Метод [Get Authorized Card Transactions](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_authorized_card_transactions_open_banking__apiVersion__accounts__accountId__authorized_card_transactions_get) позволяет получать информацию об авторизованных карточных транзакциях по конкретному счёту.  
Он поможет отслеживать операции по банковской карте, и это удобно, ведь в выписку карточные операции попадают совсем не моментально.

Авторизованная карточная операция встаёт в резерв. Резерв — это деньги, которые блокируются на вашем счету после покупки по карте или снятия наличных.  
Сумма становится недоступной для использования и «ждёт» окончательного списания со счёта. Пока деньги стоят в резерве, использовать их нельзя, рекомендуем учитывать это при планировании расходов.

Списание происходит, когда банк получает окончательное требование от платёжной системы. Обычно это занимает 3-5 рабочих дней, но иногда — и до 30. В банковскую выписку операция попадёт именно с датой списания, а не самой покупки.

## Вебхуки

Для работы с вебхуками должно быть выдано разрешение `ManageWebhookData`.

Вебхуки нужны для того, чтобы получать уведомления о важных событиях в вашем сервисе или телеграм-каналах, связанных с вашими счетами.
 
Вебхук — это POST запрос на указанный url, в теле которого находится jwt токен в виде строки.  
Этот jwt объект нужно расшифровать — алгоритм шифрования тела вебхука `RS256`.  

Для того чтобы убедиться, что вебхук пришёл от Точки, используйте наш [публичный ключ OpenAPI](/doc/openapi/static/keys/public).

Сейчас доступны четыре вида событий: `incomingPayment`, `incomingSbpPayment`, `outgoingPayment` и `acquiringInternetPayment`.

<details>
    <summary>incomingPayment — о входящих платежах</summary>

Вебхук нужен для отправки информации о входящих платежах.

В течение 20 секунд после каждого входящего платежа будет отправлено уведомление с основными данными: 
- реквизиты получателя и отправителя платежа,  
- **purpose** — назначение,  
- **amount** — сумма,  
- **webhookType** — incomingPayment,  
- **customerCode** - уникальный идентификатор клиента,  
- **paymentId** — уникальный идентификатор платежа, который также содержится в выписке.

**Пример с телом вебхука по событию incomingPayment:**

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJTaWRlUGF5ZXIiOiB7ImJhbmtDb2RlIjogIjAwMDAwMDAwMCIsICJiYW5rTmFtZSI6ICJcdTA0MWVcdTA0MWVcdTA0MWUgXHUwNDExXHUwNDMwXHUwNDNkXHUwNDNhIFx1MDQyMlx1MDQzZVx1MDQ0N1x1MDQzYVx1MDQzMCIsICJiYW5rQ29ycmVzcG9uZGVudEFjY291bnQiOiAiMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCAiYWNjb3VudCI6ICIwMDAwMDAwMDAwMDAwMDAwMDAwMCIsICJuYW1lIjogIlx1MDQxOFx1MDQxZiBcdTA0MjJcdTA0MzVcdTA0NDFcdTA0NDIiLCAiYW1vdW50IjogIjQwLjAiLCAiY3VycmVuY3kiOiAiUlVCIiwgImlubiI6ICIwMDAwMDAwMDAwIiwgImtwcCI6ICIwMDAwMDAwMDAwIn0sICJTaWRlUmVjaXBpZW50IjogeyJiYW5rQ29kZSI6ICIwMDAwMDAwMDAiLCAiYmFua05hbWUiOiAiXHUwNDFlXHUwNDFlXHUwNDFlIFx1MDQxMVx1MDQzMFx1MDQzZFx1MDQzYSBcdTA0MjJcdTA0M2VcdTA0NDdcdTA0M2FcdTA0MzAiLCAiYmFua0NvcnJlc3BvbmRlbnRBY2NvdW50IjogIjAwMDAwMDAwMDAwMDAwMDAwMDAwIiwgImFjY291bnQiOiAiMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCAibmFtZSI6ICJcdTA0MThcdTA0MWYgXHUwNDIyXHUwNDM1XHUwNDQxXHUwNDQyIiwgImFtb3VudCI6ICI0MC4wIiwgImN1cnJlbmN5IjogIlJVQiIsICJpbm4iOiAiMDAwMDAwMDAwMCIsICJrcHAiOiAiMDAwMDAwMDAwMCJ9LCAicHVycG9zZSI6ICJcdTA0MjJcdTA0MzVcdTA0NDFcdTA0NDJcdTA0M2VcdTA0MzJcdTA0M2VcdTA0MzUgXHUwNDNkXHUwNDMwXHUwNDM3XHUwNDNkXHUwNDMwXHUwNDQ3XHUwNDM1XHUwNDNkXHUwNDM4XHUwNDM1IFx1MDQzZlx1MDQzYlx1MDQzMFx1MDQ0Mlx1MDQzNVx1MDQzNlx1MDQzMCIsICJkb2N1bWVudE51bWJlciI6ICIwMDAwMCIsICJwYXltZW50SWQiOiAiMDAwMDAwMDAwMCIsICJkYXRlIjogIjIwMTgtMTAtMDEiLCAid2ViaG9va1R5cGUiOiAiaW5jb21pbmdQYXltZW50IiwgImN1c3RvbWVyQ29kZSI6ICIzMDAxMjMxMjMifQ.j7FCYrHL6pmR7m8TZxtBPBdsHG3uOu6bBl7HZq-VaK_LDj7Lyjb_B0L6zWZjVTRWvyfi6CZ-T-yT8IWzXdN5csoIEiwSuzLeC17oW-9c359Z5AYbL9x4SHSlYd3Q_hPM4DQVxPnPKb_IYpZGDCTovU_wtlAxiBdXQbY3qWEzzDzCuqZNjlVQalu7XipuSt7nNDBuwcDWJAC8Ry0U7UwRH6wboufhL7WcrQgsEn-2ZV--sKzinhUyfyYMw8_cFt9MNX3x_x3Fhu56708MDviu57O5u9t-diWrw2X75QKjnlf-PAamb0idK_8bJ5XbLXymnaBuYgSZSd-HZMHYLWiMHDL-z1OIsLgPKanUPJeKsSlmiVA1VJH2oZVRMv9Pf05O_cbN26d-LIjqn9z_m8XeZ1w0I9sfGP96IfT7xONsTLdbTCiKAJ-he4nzscxpzQc3tYnaeqETTZiyjB1U_chh88bG2n_0tNlQEtRT3j0sPyCWX3qLSFlkp6kKmQkIfeBn
```
</details>

<details>
    <summary>outgoingPayment — об исходящих платежах</summary>

Вебхук нужен для отправки информации об исходящих платежах.
    
В течение 20 секунд после каждого исходящего платежа будет отправлено уведомление с основными данными:  
- реквизиты получателя и отправителя платежа,  
- **purpose** — назначение,  
- **amount** — сумма,  
- **webhookType** — outgoingPayment,  
- **customerCode** - уникальный идентификатор клиента,  
- **paymentId** — уникальный идентификатор платежа, который также содержится в выписке.

**Пример с телом вебхука по событию outgoingPayment:**

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJTaWRlUGF5ZXIiOiB7ImJhbmtDb2RlIjogIjAwMDAwMDAwMCIsICJiYW5rTmFtZSI6ICJcdTA0MWVcdTA0MWVcdTA0MWUgXHUwNDExXHUwNDMwXHUwNDNkXHUwNDNhIFx1MDQyMlx1MDQzZVx1MDQ0N1x1MDQzYVx1MDQzMCIsICJiYW5rQ29ycmVzcG9uZGVudEFjY291bnQiOiAiMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCAiYWNjb3VudCI6ICIwMDAwMDAwMDAwMDAwMDAwMDAwMCIsICJuYW1lIjogIlx1MDQxOFx1MDQxZiBcdTA0MjJcdTA0MzVcdTA0NDFcdTA0NDIiLCAiYW1vdW50IjogIjQwLjAiLCAiY3VycmVuY3kiOiAiUlVCIiwgImlubiI6ICIwMDAwMDAwMDAwIiwgImtwcCI6ICIwMDAwMDAwMDAwIn0sICJTaWRlUmVjaXBpZW50IjogeyJiYW5rQ29kZSI6ICIwMDAwMDAwMDAiLCAiYmFua05hbWUiOiAiXHUwNDFlXHUwNDFlXHUwNDFlIFx1MDQxMVx1MDQzMFx1MDQzZFx1MDQzYSBcdTA0MjJcdTA0M2VcdTA0NDdcdTA0M2FcdTA0MzAiLCAiYmFua0NvcnJlc3BvbmRlbnRBY2NvdW50IjogIjAwMDAwMDAwMDAwMDAwMDAwMDAwIiwgImFjY291bnQiOiAiMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCAibmFtZSI6ICJcdTA0MThcdTA0MWYgXHUwNDIyXHUwNDM1XHUwNDQxXHUwNDQyIiwgImFtb3VudCI6ICI0MC4wIiwgImN1cnJlbmN5IjogIlJVQiIsICJpbm4iOiAiMDAwMDAwMDAwMCIsICJrcHAiOiAiMDAwMDAwMDAwMCJ9LCAicHVycG9zZSI6ICJcdTA0MjJcdTA0MzVcdTA0NDFcdTA0NDJcdTA0M2VcdTA0MzJcdTA0M2VcdTA0MzUgXHUwNDNkXHUwNDMwXHUwNDM3XHUwNDNkXHUwNDMwXHUwNDQ3XHUwNDM1XHUwNDNkXHUwNDM4XHUwNDM1IFx1MDQzZlx1MDQzYlx1MDQzMFx1MDQ0Mlx1MDQzNVx1MDQzNlx1MDQzMCIsICJkb2N1bWVudE51bWJlciI6ICIwMDAwMCIsICJwYXltZW50SWQiOiAiMDAwMDAwMDAwMCIsICJkYXRlIjogIjIwMTgtMTAtMDEiLCAid2ViaG9va1R5cGUiOiAib3V0Z29pbmdQYXltZW50IiwgImN1c3RvbWVyQ29kZSI6ICIzMDAxMjMxMjMifQ.UCRqWxMHocDG83NeHxL9DdaZuLdnvWKC-zcbQL80PM3LAauP9H98tC80krR_jfGP7aXa8kOKrP2Qq5JSy4_lITViSuJKhQW9tHxQA0zJwwHmUOGIU6RNDmPVKQScoh2LlTcCpu-94v8O2GSa5nssVVj4_3mZT8VdjagpxMCuISYFRaUTnvjskkNOSmrXZIuoUHHL9h5sc6o7E8i6mMI0tMHRArgsM2Ge7de4ajkEsGQsJNBXEJV9rA4Ba5a4XdPlo1wE5nMLziKh5w-CVkotO2KyKCTLgIlg6tNwmBs0saRMJU3kJfiBrLsuPoW34ZKPoqvgQEP8q9AayjC-eTrtsC94cNoiwz2I8-KLhnZmIXKahO_p-6kCBzQfJyz-H0sfJQjDBC1zFQieP79sarBPqYmCJ7eNNttyoGdJz2xDoLFStBRU2ua198PYSHsuBsdoBQ_6FO6BWCnnL66mzMKXPRpQ-uOIa-t73wTuzCre3HZVUpFKlzrfts4BxuG4Yy6o
```
</details>

<details>
    <summary>incomingSbpPayment — о входящих платежах через СБП</summary>

По этому типу вебхука мы отправляем уведомление только, если деньги зачисляются через Систему быстрых платежей. В нём указываются:  
- **operationId** — идентификатор операции,  
- **qrcId** — идентификатор QR-кода,  
- **payerName** — данные покупателя: имя, отчество и первая буква фамилии,  
- **payerMobileNumber** — номер телефона,  
- **amount** — сумма операции,  
- **brandName** — наименование ТСП,  
- **webhookType** — incomingSbpPayment,  
- **customerCode** — уникальный идентификатор клиента,  
- **merchantId** — идентификатор ТСП,  
- **refTransactionId** — идентификатор транзакции, по которой осуществляется возврат,  
- **purpose** — назначение платежа.
    
Уведомление обычно приходит в течение 5 секунд с момента зачисления денег по QR-коду.
    
**Пример с телом вебхука по событию incomingSbpPayment:**
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJvcGVyYXRpb25JZCI6ICJBMjIwMDExMDAyNjM4MjAxMDAwMDA1MzNFNjI1RkNCMyIsICJxcmNJZCI6ICJBUzEwMDA2RFBSVEVGUEZTOUhKOVNRU0RTVlJISkQzTCIsICJhbW91bnQiOiAiMC4zMyIsICJwYXllck1vYmlsZU51bWJlciI6ICIrOTk5OTEyMzQ1NjciLCAicGF5ZXJOYW1lIjogIlx1MDQxOFx1MDQzMlx1MDQzMFx1MDQzZCBcdTA0MThcdTA0MzJcdTA0MzBcdTA0M2RcdTA0M2VcdTA0MzJcdTA0MzhcdTA0NDcgXHUwNDE4LiIsICJicmFuZE5hbWUiOiAiXHUwNDFhXHUwNDNlXHUwNDQ0XHUwNDM1XHUwNDM5XHUwNDNkXHUwNDRmIFx1MDQ0MyBcdTA0MTBcdTA0NDBcdTA0NDJcdTA0MzVcdTA0M2NcdTA0MzAiLCAibWVyY2hhbnRJZCI6ICJNRjAwMDAwMDAwMDEiLCAicHVycG9zZSI6ICJcdTA0MWVcdTA0M2ZcdTA0M2JcdTA0MzBcdTA0NDJcdTA0MzAgXHUwNDNmXHUwNDNlIFx1MDQ0MVx1MDQ0N1x1MDQzNVx1MDQ0Mlx1MDQ0MyBcdTIxMTYgMSBcdTA0M2VcdTA0NDIgMDEuMDEuMjAyMS4gXHUwNDExXHUwNDM1XHUwNDM3IFx1MDQxZFx1MDQxNFx1MDQyMSIsICJ3ZWJob29rVHlwZSI6ICJpbmNvbWluZ1NicFBheW1lbnQiLCAiY3VzdG9tZXJDb2RlIjogIjMwMDEyMzEyMyIsICJyZWZUcmFuc2FjdGlvbklkIjogImNiMWVmMTBiLTM5ZTktNGU1NS1hZWY5LWQ2YmNjMzk2ZmYwZiJ9.QroEOEYw-fxniFzvjcrgihSeYOhE9GLVuMwI9Uuc0ubuwc-wYP2pHRYF4Lu2-bqHZk2jP5QHn2aDYWFVWi3vBi6sbVTa_n3Y9qZ1ROjbRaF-Uuah5XyzijwNG0I--0jj-vvn3D3caC1mw6dOCP1Ehkd5ipZWtcsrzu0wPxni8bsiqDIBVcA4hg3a6Iu5AORANwy4whiFo295bzf1Y_fdBhZO1QjZqv3-Cc_fF9liKd56UMfljm3ChqVwCdNF85e69PXAjkeicp-pH12kkgeflA2zLq_LFurAZj9_JkrOS9qSpfDqYf0m-ciHYFVYJNGH4PZnIPOa7iYNlGi0WjVbxE6uJX2EDHnYCZ1qXAkdGlm3K1CNvmBzjIUEowWOmD9bkq4INGsd9MMmRRSo-K3fqqjPNXkVGw2-_ZbIgemG9TD4bGeZCWN4shXpzrWGcyfQ7mrhRW0XiECGxTy8K4QuK-WsijGBlkYGd-bZuBa9qyZXLnsX4FhTIdEHYDc_eW6v
```
</details>

<details>
    <summary>incomingSbpB2BPayment — о входящих платежах через B2B QR-код по СБП</summary>

По этому типу вебхука мы отправляем уведомление только при зачислении денег через B2B QR-коды, созданные через Систему быстрых платежей. 
В вебхуке указываются:
- **qrcId** — идентификатор QR-кода,
- **customerCode** — уникальный идентификатор клиента,
- **webhookType** — incomingSbpB2BPayment,
- **amount** — сумма операции,
- **purpose** — назначение платежа.

Уведомление обычно приходит в течение 10 секунд с момента зачисления денег по QR-коду.

    
**Пример с телом вебхука по событию incomingSbpB2BPayment:**
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJxcmNJZCI6ICJBUzEwMDA2RFBSVEVGUEZTOUhKOVNRU0RTVlJISkQzTCIsICJhbW91bnQiOiAiMC4zMyIsICJwdXJwb3NlIjogIlx1MDQxZVx1MDQzZlx1MDQzYlx1MDQzMFx1MDQ0Mlx1MDQzMCBcdTA0M2ZcdTA0M2UgXHUwNDQxXHUwNDQ3XHUwNDM1XHUwNDQyXHUwNDQzIFx1MjExNiAxIFx1MDQzZVx1MDQ0MiAwMS4wMS4yMDIxLiBcdTA0MTFcdTA0MzVcdTA0MzcgXHUwNDFkXHUwNDE0XHUwNDIxIiwgIndlYmhvb2tUeXBlIjogImluY29taW5nU2JwQjJCUGF5bWVudCIsICJjdXN0b21lckNvZGUiOiAiMzAwMTIzMTIzIn0.dty4QICBwDnglK9VKFchdKxG_QbOPHKHmqxSan5uDUadMT61391lNiRj7cKGiH35heZc7GEKp3rwmuqLIEwLcUQts2t90Baw7gVeHQSQMyeBjIEFoBXgkDCNsRwqJISmW8NfEv6xX87IEjLp6wKJi3VmfF_sffW6EpaXzyScA9DPIKjtUA9U-W5t8JmjgWMbXK41ghfiSlYiCOpOVOleod3R-jjzsGI3nEYExDqWWz3VgjsFGwHLGKoWBzhqtSL-NBTp5rE82N5eZrWF_4n8tkAmPzShQY9Rh2VxFVwAyqIezqYHYAcBBW_6vbZvA_XFVB8UkjOqJKRWSFHEOsmu7o3JYOq0WnI1iJ_Hfsi0W1HjjgaZuMhyezqNsGWib6u_g-86ekEPxXGFFlKglHg52t8iU-YE8dt7e7s890l_4_C2tRBbOuLrdnITrHzepiby6A74Qdt29j2Ffo0Vkd8qRNWx7RmWjy1X7NN0h595imLbBFWeYPYNOdG0M7CBQUox
```
</details>

<details>
    <summary>acquiringInternetPayment — о входящих платежах только по платёжным ссылкам</summary>

По этому типу вебхука мы отправляем уведомление, только если по ссылке платят с карты или через Систему быстрых платежей.  
Доставляется он обычно за 5-10 секунд с момента оплаты.
    
При платеже по номеру карты мы передадим следующую информацию: 
- **amount** — сумма платежа,  
- **paymentType** — card,  
- **operationId** — идентификатор платежа,  
- **webhookType** — acquiringInternetPayment,  
- **customerCode** — уникальный идентификатор клиента,  
- **merchantId** — идентификатор торговой точки в интернет-эквайринге,  
- **consumerId** — идентификатор покупателя,  
- **purpose** — назначение платежа,
- **status** — статус платежа. Может иметь следующие значения:
	- `AUTHORIZED` — деньги успешно заморожены на счёте плательщика. Значение актуально только для [двухэтапной оплаты](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Platyozhnye-ssylki).
	- `APPROVED` — деньги успешно списаны со счёта, оплата завершена.
    
**Пример с телом вебхука по событию acquiringInternetPayment при оплате картой:**
    
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJjdXN0b21lckNvZGUiOiAiMzAwMTIzMTIzIiwgImFtb3VudCI6ICIwLjMzIiwgInBheW1lbnRUeXBlIjogImNhcmQiLCAib3BlcmF0aW9uSWQiOiAiYmVlYWM4YTQtNjA0Ny0zZjM4LTg5MjItYTY2NGU2YjVjNDNiIiwgInB1cnBvc2UiOiAiXHUwNDFlXHUwNDNmXHUwNDNiXHUwNDMwXHUwNDQyXHUwNDMwIFx1MDQzZlx1MDQzZSBcdTA0NDFcdTA0NDdcdTA0MzVcdTA0NDJcdTA0NDMgXHUyMTE2IDEgXHUwNDNlXHUwNDQyIDAxLjAxLjIwMjEuIFx1MDQxMVx1MDQzNVx1MDQzNyBcdTA0MWRcdTA0MTRcdTA0MjEiLCAid2ViaG9va1R5cGUiOiAiYWNxdWlyaW5nSW50ZXJuZXRQYXltZW50IiwgIm1lcmNoYW50SWQiOiAiMjAwMDAwMDAwMDAxMjM0IiwgImNvbnN1bWVySWQiOiAiOTE3ZWQzODktYTEyMC00MjkxLThlNzMtMzhjNmVmN2Q2NzcwIiwgInN0YXR1cyI6ICJBUFBST1ZFRCJ9.cFFVd_rpmBFSrefGm2jRM7NhahAY2Jhb_7vwQXs952q18SiIGfyJZdFqGh2uicwBNKOnv_Bto0gDoTZcToEfqPXbHCKCYsA7uu8JUR_DHJtxjYuiilXRgkyDfUxNb0jCqWEfMJz6sGM2AA7kKyQ-Ds9fnGipX9GzYBtksO38PWPmIAJmjYSXsnXYM6bqE8cl-EhbbBbHkCg7vDMV2kwiA0AoW7_SniT4LR0H_NosyaVe0WHq9d7pTipW8rxKl69tFebCl8iz2ZulU0qWLVWPifcF1SbnIAw5WMhEbWBJVNuKYMiGrU_YX958iIOTZ0CyajmEwQutnDIJoMpHi1xOc747ba14H1l1rfDVRXiuNOmzbXYnGBuZ6Bzy2vlJ-54PB4ToQt9nthpi65uWB4YUck3yuoFJzntqkVrmp3ZztoyvVPkpR8ub7nzAhDxLWVLdbM7Q4HQACMcnJXRfkkHlGFR9P8Ygji3OATvNcP0wnxqNxp9P629vG5uICke-Ex0I
```

При платеже через Систему быстрых платежей (СБП) мы передадим следующую информацию: 
- **amount** — сумма платежа,  
- **paymentType** — sbp,  
- **transactionId** — идентификатор платежа в СБП,  
- **purpose** — назначение платежа,  
- **qrcId** — идентификатор QR-кода,  
- **merchantId** — идентификатор торговой точки в интернет-эквайринге,  
- **operationId** — идентификатор платежа,  
- **webhookType** — acquiringInternetPayment,  
- **customerCode** — уникальный идентификатор клиента,  
- **payerName** — данные покупателя: имя, отчество и первая буква фамилии,
- **status** — статус платежа. Для платежей через СБП всегда будет значение `APPROVED`.

**Пример с телом вебхука по событию acquiringInternetPayment при оплате через Систему быстрых платежей:**

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJjdXN0b21lckNvZGUiOiAiMzAwMTIzMTIzIiwgImFtb3VudCI6ICIwLjMzIiwgInBheW1lbnRUeXBlIjogInNicCIsICJvcGVyYXRpb25JZCI6ICJiZWVhYzhhNC02MDQ3LTNmMzgtODkyMi1hNjY0ZTZiNWM0M2IiLCAidHJhbnNhY3Rpb25JZCI6ICI0M2M2M2VjMS00MmZhLWE3MDQtZGRlNy02MDI1YzIwYjk2Y2UiLCAicHVycG9zZSI6ICJcdTA0MWVcdTA0M2ZcdTA0M2JcdTA0MzBcdTA0NDJcdTA0MzAgXHUwNDNmXHUwNDNlIFx1MDQ0MVx1MDQ0N1x1MDQzNVx1MDQ0Mlx1MDQ0MyBcdTIxMTYgMSBcdTA0M2VcdTA0NDIgMDEuMDEuMjAyMS4gXHUwNDExXHUwNDM1XHUwNDM3IFx1MDQxZFx1MDQxNFx1MDQyMSIsICJxcmNJZCI6ICJBUzEwMDA2RFBSVEVGUEZTOUhKOVNRU0RTVlJISkQzTCIsICJwYXllck5hbWUiOiAiXHUwNDE4XHUwNDMyXHUwNDMwXHUwNDNkIFx1MDQxOFx1MDQzMlx1MDQzMFx1MDQzZFx1MDQzZVx1MDQzMlx1MDQzOFx1MDQ0NyBcdTA0MTguIiwgIndlYmhvb2tUeXBlIjogImFjcXVpcmluZ0ludGVybmV0UGF5bWVudCIsICJtZXJjaGFudElkIjogIjIwMDAwMDAwMDAwMTIzNCIsICJzdGF0dXMiOiAiQVBQUk9WRUQifQ.mTw0f22N6XWE_mgJ1z6kv_4ASjACXYJUpPNHhz6crFT3KTeYIqQKRSWIFnQiqWLwneg8Mv3qfKWPdKSOjs-FnhM3HD20wq1rNWIF97HtzuT-kMTUZnProv7QDYU8iIBeMJmXYv91DW1WzE2OHJUHWEtYhwEI9yHLLIXvWkgnUYnTsiB1o8cACHxswT0drP09v607Jo31y58NmjqH1x5VYCr2YIaPunsTuY8KqusegTokqYlnQswZ3C41YZuROIXWcPetUuKp7l9m0c_y68REnMNw9KQTbyjzzjzb_cb3Qt8MYe16bWk8oe2bO4mlRs0I1tqHJtoIwcHoAPbKeMfG7qgQhz6xNnstlJsuhRh7g5OH9KgFmHYCqq3YAxrMQCRwyYKEoWa3N5c9KXMzpLkSwY6WG0UVjhJDlLghT2Rk1mSufpOFL_G5ED7gyTvdAJYzTtYMCTDSdR5sToH0EP7oijwMtlSEDV_v6-L5-NLZVeqM8_c4jRafICtO6S6ztIeF
```
</details>

### Несколько особенностей в работе вебхуков

- Если на отправленный вебхук мы не получили ответ с кодом состояния HTTP 200, а получили любой другой код, то будем отправлять его повторно 30 раз с периодичностью в 10 секунд.  

- Для проверки доступности указанного вами хоста при создании или изменении вебхука мы отправим на ваш url по одному тестовому вебхуку на каждое из событий, на которые вы подписаны.  
Если в статусе ответа не придёт код HTTP 200, то вебхук не будет создан или изменён. 

- Подключить или изменить вебхук можно только по протоколу HTTPS на порт 443. Создать вебхук на другой порт или протокол не получится.

- Вебхук отправляется с заголовком `Content-Type: text/plain`.


### Примеры обработки вебхуков

Примеры кода, который успешно обрабатывает вебхук

<details>
    <summary>Python</summary>

```python
from aiohttp import web
import jwt
from jwt import exceptions
import json

# Публичный ключ Точки. Может быть получен из https://enter.tochka.com/doc/openapi/static/keys/public
key_json = '{"kty":"RSA","e":"AQAB","n":"rwm77av7GIttq-JF1itEgLCGEZW_zz16RlUQVYlLbJtyRSu61fCec_rroP6PxjXU2uLzUOaGaLgAPeUZAJrGuVp9nryKgbZceHckdHDYgJd9TsdJ1MYUsXaOb9joN9vmsCscBx1lwSlFQyNQsHUsrjuDk-opf6RCuazRQ9gkoDCX70HV8WBMFoVm-YWQKJHZEaIQxg_DU4gMFyKRkDGKsYKA0POL-UgWA1qkg6nHY5BOMKaqxbc5ky87muWB5nNk4mfmsckyFv9j1gBiXLKekA_y4UwG2o1pbOLpJS3bP_c95rm4M9ZBmGXqfOQhbjz8z-s9C11i-jmOQ2ByohS-ST3E5sqBzIsxxrxyQDTw--bZNhzpbciyYW4GfkkqyeYoOPd_84jPTBDKQXssvj8ZOj2XboS77tvEO1n1WlwUzh8HPCJod5_fEgSXuozpJtOggXBv0C2ps7yXlDZf-7Jar0UYc_NJEHJF-xShlqd6Q3sVL02PhSCM-ibn9DN9BKmD"}'
key = json.loads(key_json)
jwk_key = jwt.jwk_from_dict(key)


async def handle(request: web.Request):
    payload = await request.text()

    try:
        # тело вебхука
        webhook_jwt = jwt.JWT().decode(
            message=payload,
            key=jwk_key,
        )
    except exceptions.JWTDecodeError:
        # Неверная подпись, вебхук не от Точки или с ним что-то не так
        pass

    return web.Response(status=200)


app = web.Application()
app.router.add_route('POST', '/', handle)

if __name__ == '__main__':
    web.run_app(app, port=80)

```
</details>

<details>
    <summary>PHP</summary>

```php
<?php
require __DIR__ . "/vendor/autoload.php";
use Firebase\JWT\JWK;
use Firebase\JWT\JWT;


$entityBody = file_get_contents("php://input");

// Публичный ключ Точки. Может быть получен из https://enter.tochka.com/doc/openapi/static/keys/public
$json_key = '{"kty":"RSA","e":"AQAB","n":"rwm77av7GIttq-JF1itEgLCGEZW_zz16RlUQVYlLbJtyRSu61fCec_rroP6PxjXU2uLzUOaGaLgAPeUZAJrGuVp9nryKgbZceHckdHDYgJd9TsdJ1MYUsXaOb9joN9vmsCscBx1lwSlFQyNQsHUsrjuDk-opf6RCuazRQ9gkoDCX70HV8WBMFoVm-YWQKJHZEaIQxg_DU4gMFyKRkDGKsYKA0POL-UgWA1qkg6nHY5BOMKaqxbc5ky87muWB5nNk4mfmsckyFv9j1gBiXLKekA_y4UwG2o1pbOLpJS3bP_c95rm4M9ZBmGXqfOQhbjz8z-s9C11i-jmOQ2ByohS-ST3E5sqBzIsxxrxyQDTw--bZNhzpbciyYW4GfkkqyeYoOPd_84jPTBDKQXssvj8ZOj2XboS77tvEO1n1WlwUzh8HPCJod5_fEgSXuozpJtOggXBv0C2ps7yXlDZf-7Jar0UYc_NJEHJF-xShlqd6Q3sVL02PhSCM-ibn9DN9BKmD"}';
$jwks = json_decode($json_key, true, 512, JSON_THROW_ON_ERROR);

try {
    $decoded = JWT::decode($entityBody, JWK::parseKey($jwks,"RS256"));
} catch (\UnexpectedValueException $e) {
    // Неверная подпись, вебхук не от Точки или с ним что-то не так
    echo "Invalid webhook";
}


// Тело вебхука
$decoded_array = (array) $decoded;


http_response_code(200);
```
</details>

<details>
    <summary>Java</summary>

```java
import com.nimbusds.jose.JOSEException;
import com.nimbusds.jose.crypto.RSASSAVerifier;
import com.nimbusds.jose.jwk.JWK;
import com.nimbusds.jwt.SignedJWT;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.text.ParseException;

@RestController
@RequestMapping("/")
public class WebhookResource {

    @PostMapping(value = "/")
    public void tochkaWebhook(@RequestBody String webhookData) throws ParseException, JOSEException {

        // Публичный ключ Точки. Может быть получен из https://enter.tochka.com/doc/openapi/static/keys/public
        var jsonKey = "{\"kty\":\"RSA\",\"e\":\"AQAB\",\"n\":\"raJQJyBXIgS1YzYFkmQGq5XtadLVvMcx5u-guR2r5ZgSb-HGUG7HF5NM-NJeL9YrVtjjGf8VNLpwGbeejsS9LRniPfKkCYaVqV1DSGOZ6RTOtqN3jKW1W86cVb-LffrQo3eFhPX5V464uduPu9RouFplQ7wprY5ewke0Yj0FCOr6Ebxlpql-aJp_wk8JSzzFN17IC5tfUXgGDjEmnMjxag_CntnJtKWmw69ivhrq5sTPspclL3Ij8K_Qk0MwAZFCci25WxIuKQe7Mk4dvay6CUfrCbAgEtqMcWUSqoG7pdBig59lo-kIMWvVQIAWjo2JhI7VlI_ssvFtiJg5T9myE914aESFZ8jEheQv-4kZ81F0qk02k2mJ4C7AasGhbzC4F8YQ7nbr49v1n_j8udNZZXA8vI2hacG517A66-uvEHIxXRUo_gIcubR-vdbJbaK_k8JRLJNmdf4B9HchJ6VD9aGjMT0GYfhQ8jf16E1L_U4G4XLB5cnb0h88PD2MaMGP\"}";
        var jwkKey = JWK.parse(jsonKey);

        var jwtData = SignedJWT.parse(webhookData);

        var verifier = new RSASSAVerifier(jwkKey.toRSAKey().toRSAPublicKey());
        if (!jwtData.verify(verifier)) {

            // Неверная подпись, вебхук не от Точки или с ним что-то не так
            System.out.println("Invalid webhook");
            return;
        }

        // Тело вебхука
        var webhookParsedData = jwtData.getJWTClaimsSet().getClaims();
        return;
    }
}
```
</details>


### Вебхуками можно управлять с помощью методов:

- [Get Webhooks](/doc/v2/redoc/tag/Rabota-s-vebhukami#get_webhooks_webhook__apiVersion___client_id__get) — чтобы получить список всех вебхуков, настроенных на приложение;
- [Create Webhook](/doc/v2/redoc/tag/Rabota-s-vebhukami#create_webhook_webhook__apiVersion___client_id__put) — чтобы создать новый вебхук;
- [Edit Webhook](/doc/v2/redoc/tag/Rabota-s-vebhukami#edit_webhook_webhook__apiVersion___client_id__post) — чтобы изменить URL или тип существующего вебхука;
- [Delete Webhook](/doc/v2/redoc/tag/Rabota-s-vebhukami#delete_webhook_webhook__apiVersion___client_id__delete) — чтобы удалить вебхук;
- [Send Webhook](/doc/v2/redoc/tag/Rabota-s-vebhukami#send_webhook_webhook__apiVersion___client_id__test_send_post) — чтобы отправить тестовый вебхук.

## Клиенты
Для работы с клиентами должно быть выдано разрешение `ReadCustomerData`.

С помощью метода [Get Customers List](/doc/v2/redoc/tag/Rabota-s-klientami#get_customers_list_open_banking__apiVersion__customers_get) вы получите список подключённых компаний, а с [Get Customer Info](/doc/v2/redoc/tag/Rabota-s-klientami#get_customer_info_open_banking__apiVersion__customers__customerCode__get) — детальную информацию по каждому клиенту.

## Счета
Для работы со счетами должны быть выданы разрешения `ReadAccountsBasic` `ReadAccountsDetail`.

 - [Get Accounts List](/doc/v2/redoc/tag/Rabota-so-schetami#get_accounts_list_open_banking__apiVersion__accounts_get) — получить информация по всем счетам;  
 - [Get Account Info](/doc/v2/redoc/tag/Rabota-so-schetami#get_account_info_open_banking__apiVersion__accounts__accountId__get) — получить информация по одному счёту.

## Платёжные ссылки
Для работы с платёжными ссылками нужны разрешения `MakeAcquiringOperation`, `ReadAcquiringData` и `ReadCustomerData`.

С методами openAPI можно формировать платёжные ссылки, по которым ваши покупатели смогут оплатить товар или услугу — и банковской картой, и через Систему быстрых платежей.

<details>
    <summary>методы по работе с платёжными ссылками без фискализации чеков</summary>

### Подключение платёжных ссылок

Перед работой с API платёжных ссылок нужно оставить заявку на подключение интернет-эквайринга внутри интернет-банка. Сделать это может руководитель или распорядитель с правом подписи в разделе «Сервисы» → [«Интернет-эквайринг»](https://i.tochka.com/a/internet_acquiring/).

В течение рабочего дня с момента заявки с вами свяжутся специалисты для подтверждения заявки.

Проверить статус подключения интернет-эквайринга можно с помощью метода [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get)\.  
Статус REG и значение isActive: «true» будут означать, что настройка завершена — можно приступать к работе с платёжными ссылками.  

### Генерация платёжной ссылки
Для создания платёжной ссылки нужно вызвать метод [Create Payment Operation](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post) и передать:
-  **amount** — сумма платежа в рублях.
-  **customerCode** — уникальный код клиента.
-  **purpose** — назначение платежа, которое увидит перешедший по платёжной ссылке покупатель.
-  **paymentMode** — способ оплаты. Можно выбрать оплату только по карте (card), через T-Pay (tinkoff) или по QR-коду (sbp), либо указать все три варианта, чтобы при переходе по ссылке покупатель сам выбирал способ оплаты.

#### Дополнительно можно указать:

- **redirectUrl** — URI-адрес, на который нужно переправить покупателя после оплаты.  
- **failRedirectUrl** — URI-адрес, на который нужно переправить покупателя если оплата не прошла.  
- **saveCard** — параметр, указывающий, будет ли покупателю предложено сохранить карту для следующих платежей.  
- **merchantId** — идентификатор торговой точки в интернет-эквайринге. Параметр является обязательным, если у вас подключено несколько торговых точек для работы с интернет-эквайрингом.  
- **consumerId** — идентификатор покупателя. Если ранее был указан параметр saveCard и клиент согласился с сохранением карты, то из ответа можно взять consumerId и подставить в запрос, чтобы покупатель мог выбрать карту, по которой уже происходила оплата.  
- **ttl** — время действия платёжной ссылки в минутах, можно указать от 1 до 44 640 минут. Если при создании ссылки не передать этот параметр, срок действия составит 10 080 минут.
- **preAuthorization** — поле показывает, будет ли использоваться [двухэтапная оплата](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Platyozhnye-ssylki). Неактуально для оплат через Систему быстрых платежей (СБП). Значение по умолчанию — `false`.

Чтобы сразу узнать об оплате по ссылке вы можете настроить вебхук с событием [acquiringInternetPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) на свой URL.
</details>

<details>
        <summary>методы по работе с платёжными ссылками с фискализацией чеков от партнёров Точки</summary>
        
### Подключение платёжных ссылок

Перед работой с API платёжных ссылок нужно отправить заявку на подключение интернет-эквайринга и выбрать партнёра, который будет фискализировать ваши чеки по 54-ФЗ.

Сделать это может руководитель или распорядитель с правом подписи в разделе «Сервисы» →  [«Интернет-эквайринг»](https://i.tochka.com/a/internet_acquiring/).  

В течение рабочего дня с момента заявки с вами свяжутся специалисты для подтверждения заявки.

Проверить статус подключения интернет-эквайринга можно с помощью метода [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get)\.  
Статус REG и значение isActive: «true» будут означать, что настройка завершена — можно приступать к работе с платёжными ссылками.  

### Генерация платёжной ссылки

Для создания платёжной ссылки нужно вызвать метод [Create Payment Operation With Receipt](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_with_receipt_acquiring__apiVersion__payments_with_receipt_post) и передать:
- **amount** — сумма платежа в рублях.
- **customerCode** — уникальный код клиента.
- **paymentMode** — способ оплаты. Можно выбрать оплату только по карте (card), через T-Pay (tinkoff) или по QR-коду (sbp), либо указать все три варианта, чтобы при переходе по ссылке покупатель сам выбирал способ оплаты.
- **purpose** — назначение платежа, которое увидит перешедший по платёжной ссылке покупатель.

**Client** — объект, содержащий данные покупателя.  
- **Email** — электронная почта покупателя, на которую будет отправлен чек.

**Items** — объект, содержащий список товаров в заказе.  
- **name** — название товара или услуги.
- **amount** — цена за единицу товара или услуги.
- **quantity** — количество товаров или оказанных услуг.

### Дополнительно можно указать:

- **redirectUrl** — URI-адрес, на который нужно переправить покупателя после оплаты.  
- **failRedirectUrl** — URI-адрес, на который нужно переправить покупателя если оплата не прошла.  
- **saveCard** — параметр, указывающий, будет ли покупателю предложено сохранить карту для следующих платежей.  
- **consumerId** — идентификатор покупателя. Если ранее был указан параметр saveCard и клиент согласился с сохранением карты, то из ответа можно взять consumerId и подставить в запрос, чтобы покупатель мог выбрать карту, по которой уже происходила оплата.  
- **taxSystemCode** — система налогообложения, которая используется в организации.  
- **merchantId** — идентификатор торговой точки в интернет-эквайринге. Параметр является обязательным, если у вас подключено несколько торговых точек для работы с интернет-эквайрингом.  
- **ttl** — время действия платёжной ссылки в минутах, можно указать от 1 до 44 640 минут. Если при создании ссылки не передать этот параметр, срок действия составит 10 080 минут.
- **preAuthorization** — поле показывает, будет ли использоваться [двухэтапная оплата](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Platyozhnye-ssylki). Неактуально для оплат через Систему быстрых платежей (СБП). Значение по умолчанию — `false`. 

**Client** — объект, содержащий данные покупателя.  
- **name** —  наименование плательщика. Для юрлица — название организации, для ИП и физического лица — ФИО.
- **phone** — телефон покупателя для отправки чека.

**Items** — объект, содержащий список товаров в заказе.  
- **vatType** — ставка НДС в позициях.  
- **paymentMethod** — тип оплаты. full_payment — полная оплата, full_prepayment — полная предоплата.  
- **paymentObject** — признак предмета расчёта. goods — товары, service — услуги, work — работа.  
- **measure** — единица измерения. По умолчанию ставится штуки.  

**Supplier** — объект, содержащий информацию о поставщике. Если вы работаете как агент и продаёте не свой товар, 54-ФЗ требует указывать в чеке данные поставщика.  
- **phone** — номер телефона поставщика.  
- **name** — наименование поставщика.  
- **taxCode** — ИНН поставщика.  

Чтобы сразу узнать об оплате по ссылке, вы можете настроить вебхук с событием [acquiringInternetPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) на свой URL.
</details>

### Получение списка торговых точек

Метод [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get) возвращает информацию о подключённых торговых точках для работы с интернет-эквайрингом. В этом методе передаётся следующая информация:
- **status** — статус регистрации. Работать с платёжными ссылками можно только при наличии статуса REG.
- **isActive** — статус готовности к работе у торговой точки. Работать с платёжными ссылками только при наличии статуса true.
- **mcc** — MCC-код, соответствующий вашей деятельности.
- **rate** — сумма взимаемой комиссии, при платеже на карту.
- **name** — наименование торговой точки.
- **url** — адрес сайта регистрации торговой точки.
- **merchantId** — идентификатор торговой точки.
- **terminalId** — идентификатор интернет-эквайринга.
- **paymentModes** — доступные способы оплаты у торговой точки.
- **cashbox** — информация о наличии подключённой онлайн-кассы.

### Получение списка операций по платёжным ссылкам

Метод [Get Payment Operation List](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_payment_operation_list_acquiring__apiVersion__payments_get) позволяет получить список всех операций и их статусы.  
Лучше всего указывать значения дат _fromDate_ и _toDate_, чтобы не получить огромный список операций за всё время работы с Точкой.

С методом [Get Payment Operation Info](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_payment_operation_info_acquiring__apiVersion__payments__operationId__get) вы сможете получить информацию по одной конкретной операции.  
Для вызова этого метода понадобится `operationId`, который вы можете получить из нашего вебхука с событием _acquiringInternetPayment_, либо найдя нужную операцию в полученном методом _Get Payment Operation List_ списке.

### Создание ссылки с двухэтапной оплатой

В некоторых случаях вам может понадобиться платёжная ссылка с двухэтапной оплатой. Например, если ваш бизнес — это турагентство, и прежде чем списать деньги со счёта плательщика, нужно заморозить их на его счёте. Затем получить данные о наличии свободного номера у отеля и, если всё хорошо, списать деньги.

За двухэтапную оплату отвечает поле `preAuthorization` в методах [Create Payment Operation](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post) и [Create Payment Operation With Receipt](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_with_receipt_acquiring__apiVersion__payments_with_receipt_post). Порядок действий для использования двухэтапной оплаты такой:

1. Создайте платёжную ссылку, указав в теле метода `"preAuthorization": true`.
2. Когда клиент перейдёт по ссылке и оплатит картой, если денег будет достаточно, они заморозятся на его счёте, но ещё не спишутся.
3. Вам отправится вебхук с событием `acquiringInternetPayment`. Внутри этого события будет поле `status` со значением `AUTHORIZED` — это означает, что деньги успешно заморожены.
4. Вызовите метод [Capture Payment](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#capture_payment_acquiring__apiVersion__payments__operationId__capture_post), чтобы списать замороженные деньги.
5. После успешного списания вам придёт вебхук с событием `acquiringInternetPayment`, внутри которого будет `"status": "APPROVED"` — подтверждение, что оплата прошла успешно.

> Для оплат через Систему быстрых платежей (СБП) двухэтапной оплаты не предусмотрено. Поэтому в событии `acquiringInternetPayment` поле `status` всегда будет приходить в значении `APPROVED`.

### Несколько особенностей в работе с платёжными ссылками

- если у вас подключено несколько торговых точек для работы с интернет-эквайрингом, то указывать `merchantId` нужно обязательно. Этот параметр можно получить, вызвав метод [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get). merchantId для работы с платёжными ссылками состоит из 15 цифр.
- для вызова большинства методов по работе с платёжными ссылками нужен параметр `customerCode` — это ваш уникальный код в системе банка.  
Узнать его можно, вызвав метод [Get Customers List](/doc/v2/redoc/tag/Rabota-s-klientami#get_customers_list_open_banking__apiVersion__customers_get). Параметр customerCode необходимо брать из поля customerType: "Business".
- платёжная ссылка действует 168 часов. С момента перехода по ссылке у покупателя есть 1 час на оплату. По истечении этого срока он увидит ошибку, но при повторном переходе по ссылке сформируется новая рабочая страничка для оплаты.

### Возврат платежа, поступившего через платёжную ссылку

Проводить возврат можно только для платежей в статусе `APPROVED`. 

Возврат может быть как полным, так и частичным. Главное — чтобы сумма возврата не превышала сумму поступления на расчётный счёт.

Для возврата платежа, полученного через платёжную ссылку, нужно вызвать метод [Refund Payment Operation](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#refund_payment_operation_acquiring__apiVersion__payments__operationId__refund_post) и передать `operationId`.

Получить `operationId` можно несколькими способами:
- Вызвать метод [Get Payment Operation List](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_payment_operation_list_acquiring__apiVersion__payments_get) и найти нужную вам транзакцию, взяв operationId.
- Если у вас настроен вебхук с событием [acquiringInternetPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki), в теле вебхука мы передадим operationId, который можно использовать для возврата.

### Скорость возврата платежа
Если платёж поступил с помощью QR-кода, то возврат происходит практически мгновенно.  
Если платили банковской картой, то возврат обычно проходит за 3-7 календарных дней, хотя по правилам платёжных систем может занимать до 30 дней.

## Подписки (рекуррентные платежи)

Для работы с подписками должны быть выданы разрешения `MakeAcquiringOperation`, `ReadAcquiringData` и `ReadCustomerData`.

Подписки (рекуррентные платежи) — это удобный сервис автооплаты, который позволяет вашим клиентам подписаться на регулярные платежи за товары или услуги.  
Благодаря возможностям openAPI ваш бизнес может настроить создание ссылок, автоматизирующих процесс оплаты подписок. Клиенты совершают первую транзакцию, а потом в течение заданного периода деньги будут списываться с их карты без дополнительных действий.

<details>
    <summary>метод по работе подписок без фискализации чеков</summary>

### Подключение подписок

Перед работой с API платёжных ссылок нужно оставить заявку на подключение интернет-эквайринга внутри интернет-банка. Сделать это может руководитель или распорядитель с правом подписи в разделе «Сервисы» → [«Интернет-эквайринг»](https://i.tochka.com/a/internet_acquiring/).

В течение рабочего дня с момента заявки с вами свяжутся специалисты для подтверждения заявки.

Проверить статус подключения интернет-эквайринга можно с помощью метода [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get)\.  
Статус REG и значение isActive: «true» будут означать, что настройка завершена — можно приступать к работе с платёжными ссылками.

### Создание подписки (рекуррентного платежа) без фискализации чеков

Для создания платёжной ссылки нужно вызвать метод [Create Subscription](/doc/v2/redoc/tag/Rabota-s-podpiskami#create_subscription_acquiring__apiVersion__subscriptions_post) и передать:  

- **amount** — сумма платежа в рублях.  
- **customerCode** — уникальный код клиента.  
- **purpose** — назначение платежа, которое увидит перешедший по ссылке покупатель.  

### Дополнительно можно указать

- **redirectUrl** — URI-адрес, на который нужно переправить покупателя после оплаты.  
- **failRedirectUrl** — URI-адрес, на который нужно переправить покупателя, если оплата не прошла.  
- **saveCard** — параметр, указывающий, будет ли покупателю предложено сохранить карту для следующих платежей.  
- **consumerId** — идентификатор покупателя. Если ранее был указан параметр saveCard и клиент согласился с сохранением карты, то из ответа можно взять consumerId и подставить в запрос, чтобы покупатель мог выбрать карту, по которой уже происходила оплата.  
- **merchantId** — идентификатор торговой точки в интернет-эквайринге. Параметр является обязательным, если у вас подключено несколько торговых точек для работы с интернет-эквайрингом.  

**Options** — объект, содержащий опции подпски.  
- **trancheCount** — количество списаний по подписке. По умолчанию установлено 12 списаний.  
- **period** — периодичность списания. По умолчанию устанавливается месяц (Month).  
- **daysInPeriod** — длина периодичности списания в днях. Работает только с периодом в днях (Day).  

Чтобы сразу узнать о поступлении оплаты, вы можете настроить вебхук с событием [acquiringInternetPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) на свой URL.

</details>

<details>
    <summary>метод по работе подписок с фискализацией чеков от партнёров Точки</summary>

### Подключение подписок

Перед работой с API платёжных ссылок нужно отправить заявку на подключение интернет-эквайринга и выбрать партнёра, который будет фискализировать ваши чеки по 54-ФЗ.

Сделать это может руководитель или распорядитель с правом подписи в разделе «Сервисы» →  [«Интернет-эквайринг»](https://i.tochka.com/a/internet_acquiring/).  

В течение рабочего дня с момента заявки с вами свяжутся специалисты для подтверждения заявки.

Проверить статус подключения интернет-эквайринга можно с помощью метода [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get)\.  
Статус REG и значение isActive: «true» будут означать, что настройка завершена — можно приступать к работе с платёжными ссылками.

### Создание подписки (рекуррентного платежа) с фискализацией чеков

Для создания ссылки нужно вызвать метод [Create Subscription With Receipt](/doc/v2/redoc/tag/Rabota-s-podpiskami#create_subscription_with_receipt_acquiring__apiVersion__subscriptions_with_receipt_post) и передать:

- **amount** — сумма платежа в рублях;
- **customerCode** — уникальный код клиента;
- **purpose** — назначение платежа, которое увидит перешедший по платёжной ссылке покупатель.

**Client** — объект, содержащий данные покупателя.  
- **Email** — электронная почта покупателя, на которую будет отправлен чек.  
 
**Items** — объект, содержащий список товаров в заказе.  
- **name** — название товара или услуги.  
- **amount** — цена за единицу товара или услуги.  
- **quantity** — количество товаров или оказанных услуг.  

### Дополнительно можно указать

- **redirectUrl** — URI-адрес, на который нужно переправить покупателя после оплаты;
- **failRedirectUrl** — URI-адрес, на который нужно переправить покупателя если оплата не прошла;
- **saveCard** — параметр, указывающий, будет ли покупателю предложено сохранить карту для следующих платежей;
- **consumerId** — идентификатор покупателя. Если ранее был указан параметр saveCard и клиент согласился с сохранением карты, то из ответа можно взять consumerId и подставить в запрос, чтобы покупатель мог выбрать карту, по которой уже происходила оплата;
- **merchantId** — идентификатор торговой точки в интернет-эквайринге. Параметр является обязательным, если у вас подключено несколько торговых точек для работы с интернет-эквайрингом;
- **taxSystemCode** — система налогообложения, которая используется в вашей организации;

**Options** — объект, содержащий опции подпски.  
- **trancheCount** — количество списаний по подписке. По умолчанию установлено 12 списаний.  
- **period** —периодичность списания. По умолчанию устанавливается месяц (Month).  
- **daysInPeriod** — длина периодичности списания в днях. Работает только с периодом Day.  

**Client** — объект, содержащий данные покупателя.  
- **name** — наименование плательщика. Для юрлица — название организации, для ИП и физического лица — ФИО.  
- **phone** — телефон покупателя для отправки чека.  

**Items** — объект, содержащий список товаров в заказе.  
- **vatType** — ставка НДС в позициях.  
- **paymentMethod** — тип оплаты. full_payment — полная оплата, full_prepayment — полная предоплата.  
- **paymentObject** — признак предмета расчёта. goods — товары, service — услуги, work — работа.  
- **measure** - единица измерения. По умолчанию ставится штуки.  

**Supplier** — объект, содержащий информацию о поставщике. Если вы работаете как агент и продаёте не свой товар, 54-ФЗ требует указывать в чеке данные поставщика.  
- **phone** — номер телефона поставщика.  
- **name** — наименование поставщика.  
- **taxCode** — ИНН поставщика.  

Чтобы сразу узнать о поступлении оплаты, вы можете настроить вебхук с событием [acquiringInternetPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) на свой URL.

</details>

### Работа с подписками без графика списания

Если вы хотите самостоятельно управлять периодичностью и суммами списаний, можно передать boolean параметр `recurring` со значением `true`. Так вы создадите подписку без графика и сможете получать оплату подписки в нужные вам даты.  

Чтобы провести списание, нужно вызвать метод [Charge Subscription](/doc/v2/redoc/tag/Rabota-s-podpiskami#charge_subscription_acquiring__apiVersion__subscriptions__operationId__charge_post) и передать:  
**operationId** — идентификатор подписки, полученный при создании подписки.  
**amount** — сумма платежа.  

**Обратите внимание**: создать подписку с параметром recurring и объектом Options нельзя.  

### Дополнительные методы для работы с подписками (рекуррентными платежами)

Метод [Get Subscription List](/doc/v2/redoc/tag/Rabota-s-podpiskami#get_subscription_list_acquiring__apiVersion__subscriptions_get) позволяет получать все созданные подписки. В ответе будут показаны как активные, так и неактивные подписки. Для удобства можно использовать параметры `page` — номер страницы и `perPage` — количество записей на странице.  
Метод [Get Subscription Status](/doc/v2/redoc/tag/Rabota-s-podpiskami#get_subscription_status_acquiring__apiVersion__subscriptions__operationId__status_get) покажет актуальный статус подписки. Для вызова этого метода необходимо передать `operationId` — идентификатор подписки.  
Метод [Set Subscription Status](/doc/v2/redoc/tag/Rabota-s-podpiskami#set_subscription_status_acquiring__apiVersion__subscriptions__operationId__status_post) позволяет отключить подписку, переведя её в статус `Cancelled`\.  
Обратите внимание, что при отключении подписки вернуть её назад не получится, и придётся создавать новую.  

### Несколько особенностей в работе с подписками (рекуррентными платежами)

- возврат платежа за подписку можно провести только в интернет-банке;
- продлить подписку пока нельзя. Если она закончится, нужно создать новую;
- если у вас подключено несколько торговых точек для работы с интернет-эквайрингом, то указывать `merchantId` нужно обязательно. Этот параметр можно получить, вызвав метод [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get). merchantId для работы с подписками (рекуррентными платежами) состоит из 15 цифр.
- для вызова большинства методов по работе с подписками нужен параметр `customerCode` — это ваш уникальный код в системе банка.  
Узнать его можно, вызвав метод [Get Customers List](/doc/v2/redoc/tag/Rabota-s-klientami#get_customers_list_open_banking__apiVersion__customers_get). Параметр customerCode необходимо брать из поля customerType: "Business".
- ссылка для подключения подписки (рекуррентных платежей) действует 168 часов. С момента перехода по ссылке у покупателя есть 1 час на оплату. По истечении этого срока он увидит ошибку, но при повторном переходе по ссылке сформируется новая рабочая страничка для оплаты.
- сейчас можно создать подписку (рекуррентный платёж) только для приёма платежей с банковских карт.

## Выставление счетов и создание закрывающих документов

Для выставления счетов и создания закрывающих документов нужно разрешение `ManageInvoiceData` и `ReadCustomerData`.

### Выставление счетов

Создать счёт на оплату можно с помощью метода [Create Invoice](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#create_invoice_invoice__apiVersion__bills_post).
 
- В параметре SecondSide укажите данные контрагента, которому вы выставляете счёт.
- В параметре Content укажите содержимое счёта на оплату.
- В ответе мы передадим documentId, который может понадобиться для получения информации о выставленном счёте, связывания счёта с закрывающим документом, удаления счёта или отправки его на электронную почту.

> Для вызова метода нужен `customerCode`. Чтобы узнать customerCode, нужно вызвать метод [Get Customers List](/doc/v2/redoc/tag/Rabota-s-klientami#get_customers_list_open_banking__apiVersion__customers_get)\.  
Параметр customerCode необходимо брать из поля customerType: "Business".

Получить выставленный счёт в PDF можно с помощью метода [Get Invoice](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#get_invoice_invoice__apiVersion__bills__customerCode___documentId__file_get).

Узнать, оплачен ли выставленный счёт, можно с помощью метода [Get Invoice Payment Status](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#get_invoice_payment_status_invoice__apiVersion__bills__customerCode___documentId__payment_status_get). Нужная информация будет в полученном в ответ параметре paymentStatus.

- `payment_waiting` — оплаты счёта ещё не было;
- `payment_expired` — оплата счёта просрочена. Статус может появиться только если при выставлении счёта с помощью метода Create Invoice вы передавали параметр `paymentExpiryDate`;
- `payment_paid` — оплата по счёту прошла.

Обратите внимание: для смены статуса мы сличаем входящие платежи со счетами по ИНН контрагента, сумме и наличию номера счёта в назначении. 

Если от контрагента придёт платёж на нужную сумму, то меняем статус оплаты счёта. Если сумма будет отличаться — например, при частичной оплате или переплате — меняться автоматически статус не будет.  
Статус можно изменить вручную прямо в интернет-банке, раздел «[Счета, акты, накладные](https://i.tochka.com/bank/m/document_flow)».

После создания счёт можно отправить на электронную почту контрагента через метод [Send Invoice To Email](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#send_invoice_to_email_invoice__apiVersion__bills__customerCode___documentId__email_post).

Если вам больше не нужен счёт или вы передумали отправлять его контрагенту, то его можно удалить с помощью метода [Delete Invoice](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#delete_invoice_invoice__apiVersion__bills__customerCode___documentId__delete).

### Выставление закрывающих документов

Сформировать акт, счёт-фактуру или накладную можно с помощью метода [Create Closing Document](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#create_closing_document_invoice__apiVersion__closing_documents_post).

- В параметре SecondSide укажите данные контрагента, которому предназначен документ.
- В параметре Content укажите указать содержимое акта, счёт-фактуры или накладной.
- В ответе мы передадим documentId, который может понадобиться для получения PDF закрывающего документа, удаления или отправки документа на электронную почту.

При необходимости можно связать выставленный счёт с закрывающим документом. Для этого заполните параметр `documentId`, указав в нём documentId из ответа метода Create Invoice.

Для получения PDF с закрывающим документом испоьзуйте метод [Get Closing Document](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#get_closing_document_invoice__apiVersion__closing_documents__customerCode___documentId__file_get).

После создания закрывающий документ можно отправить на электронную почту контрагента через метод [Send Closing Documents To Email](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#send_closing_documents_to_email_invoice__apiVersion__closing_documents__customerCode___documentId__email_post).

Если вам больше не нужен закрывающий документ или вы передумали отправлять его контрагенту, документ можно удалить с помощью метода [Delete Closing Documents](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#delete_closing_documents_invoice__apiVersion__closing_documents__customerCode___documentId__delete).

## Выписки
Для работы с выписками должно быть выдано разрешение `ReadStatements`.

С помощью методов вы можете запросить выписку по счёту в формате JSON. Как и в выписке из интернет-банка, в ней содержатся:
 - остаток на начало и конец заданного периода;
 - список транзакций.

### Чтобы запросить выписку, выполните следующий запрос:
[Init Statement](/doc/v2/redoc/tag/Rabota-s-vypiskami#init_statement_open_banking__apiVersion__statements_post) — этот метод позволяет создавать выписку по счёту за определённый период: в запросе нужно указать `accountId`, дату начала и конца периода.  
В ответ на запрос вы получите идентификатор выписки `statementId`.

Далее вызовите один из следующих методов:
 - [Get Statement](/doc/v2/redoc/tag/Rabota-s-vypiskami#get_statement_open_banking__apiVersion__accounts__accountId__statements__statementId__get) — метод, который формирует конкретную выписку. Он запрашивается через идентификатор statementId.
 - [Get Statements List](/doc/v2/redoc/tag/Rabota-s-vypiskami#get_statements_list_open_banking__apiVersion__statements_get) — метод, вызывающий список всех доступных выписок.

### Некоторые особенности:

Метод [Init Statement](/doc/v2/redoc/tag/Rabota-s-vypiskami#init_statement_open_banking__apiVersion__statements_post) отрабатывает асинхронно.  
Отражаются только операции, находящиеся в финальном статусе — `Ready`. Выписки, инициированные этим методом, доступны для получения в течение 24 часов, после — они удаляются.

Общая сумма транзакций в теле выписки отдаются двумя полями: `amount` — в валюте счёта и `amountNat` — в рублях при пересчёте по курсу Центробанка.
Исходящие и входящие платежи отличаются по полю `creditDebitIndicator`, где `credit` — приход, а `debit` — расход.

В некоторых случаях в списке операций в поле `transactionTypeCode` вместо платёжного поручения указан `мемориальный ордер.`  

Мемориальный ордер — это вид платёжного документа, означающий, что банк обязан перечислить деньги получателю.
В таком статусе платёж ещё не отправлен вашему контрагенту — он только проходит все статусы перед отправкой.  

Для того, чтобы получить актуальный статус платежа, нужно запросить выписку повторно.  Обычно уже через несколько секунд в новой выписке будет указан статус «Платёжное поручение».

## Платежи
Для работы с платежами должны быть выданы разрешения `CreatePaymentForSign` `CreatePaymentOrder`.

С помощью метода [Create Payment For Sign](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_for_sign_payment__apiVersion__for_sign_post) можно создать и отправить на подпись платёжку. Она будет в интернет-банке Точки — раздел «На подпись».

Метод [Create Payment](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_payment__apiVersion__order_post) позволяет создавать и подписывать платёж в интегрированном приложении, без необходимости дополнительно заходить в интернет-банк Точки.  
При вызове метода создаётся ссылка для перехода на страницу с параметрами платежа. Платёж нужно подписать кодом, который приходит в смс на номер авторизации.

Метод [Get Payment For Sign List](/doc/v2/redoc/tag/Rabota-s-platezhami#get_payment_for_sign_list_payment__apiVersion__for_sign_get) поможет проверить все платежи из раздела «На подпись» интернет-банка. В ответе вы получите всю информацию, которая содержится в платёжном поручении.  

**Обратите внимание**, что при вызове этого метода отображаются платежи, созданные как внутри интернет-банка, так и по API. Информация о платежах на подпись актуальна только на момент вызова метода.

Метод [Get Payment Status](/doc/v2/redoc/tag/Rabota-s-platezhami#get_payment_status_payment__apiVersion__status__requestId__get) проверяет статус создания платежа.  
Работает для платежей, созданных методами Create Payment For Sign и Create Payment.  
_Метод не отдаёт статус отправки платежа из банка._

## СБП — Система быстрых платежей
Оплата через QR-коды — это замена интернет-эквайрингу, которая позволяет вам получать оплату от физлиц и экономить на банковских комиссиях.  
Вы можете зарегистрировать вашу торговую точку в Системе быстрых платежей, генерировать QR-коды и делать возвраты через API.

Для работы с Системой быстрых платежей должны быть выданы разрешения `EditSBPData` `ReadSBPData`.

### Немного определений

**Динамический QR-код** — одноразовый QR-код с уникальным идентификатором.  
По этому коду можно принять оплату только один раз, и сумма платежа прописывается при создании.  
По умолчанию срок действия динамического QR-кода составляет 72 часа, но его можно увеличить или уменьшить, поменяв время в минутах в параметре ttl.

**Кассовый QR-код** — подтип статического QR-кода, имеющий признаки динамического QR-кода. При создании кассовой ссылки создаётся идентификатор QR-кода, по которому можно проводить неограниченное количество оплат.  
После каждой оплаты продавцу будет необходимо активировать ссылку заново.  
Срок действия кассового QR-кода по умолчанию 5 минут. Если передать атрибут ttl, его можно увеличить до 20 минут.

**Статический QR-код** — многоразовый QR-код с одним идентификатором. Код действует бессрочно и по нему можно провести неограниченное количество оплат.  
Статический QR-код можно создать с указанием цены — покупатели будут платить только указанную сумму. Или вы можете при создании QR-кода не задавать сумму оплаты: тогда она сможет быть разной при каждой операции, но покупателю придётся вводить её вручную.

**B2B QR-код** — одноразовый QR-код с уникальным идентификатором, который используется для приёма платежей исключительно от ИП и организаций.  
Каждый такой код уникальный: принять платёж на сумму, указанную при его создании, можно только один раз.  
По умолчанию B2B QR-код действует 72 часа, но срок можно увеличить или уменьшить, указав время в минутах в параметре ttl.

**ТСП** — это торгово-сервисное предприятие, то есть магазины или иные торговые точки.

**legalId** — уникальный идентификатор юрлица в Системе быстрых платежей.

**merchantId** —  уникальный идентификатор ТСП в Системе быстрых платежей.

**accountId** — номер счёта и БИК через слеш, например `40802810802000000008/044525104`. 

### Как зарегистрировать юрлицо и торговые точки
Чтобы проверить, зарегистрирована ли ваша компания в Системе быстрых платежей, вызовите метод [Get Customer Info](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-YuL#get_customer_info_sbp__apiVersion__customer__customerCode___bankCode__get)\.  
Если регистрация в СБП есть, вам останется только получить информацию о `legalId` и `merchantId` и приступить к работе.

Если регистрации в СБП нет, пройдите её и создайте торговую точку одним из следующих методов: 

#### Регистрация юрлица в СБП через метод [Register Legal Entity](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-YuL#register_legal_entity_sbp__apiVersion__register_sbp_legal_entity_post)\.  

В запросе необходимо передать:  
- **customerCode** — уникальный код клиента;  
- **bankCode** — БИК банка. У Точки это 044525104.

При регистрации в СБП юрлицу присваивается уникальный идентификатор — `legalId`.

#### Регистрация нового ТСП через метод [Register Merchant](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-TSP#register_merchant_sbp__apiVersion__merchant_legal_entity__legalId__post)\.  

В запросе необходимо передать:  
- **legalId** — идентификатор зарегистрированного юрлица в СБП;  
- **address** — юридический адрес компании или адрес ТСП;  
- **city** — город, в котором находится ТСП;  
- **countryCode** — нужно указать RU;  
- **countrySubDivisionCode** — код региона-регистрации юридического лица, это первые две цифры кода ОКТМО;  
- **zipCode** — индекс;  
- **brandName** — название ТСП;  
- **capabilities** — возможности ТСП по формированию QR-кодов:  
 - **001** — возможность формировать только статические QR-коды;  
 - **010** — возможность формировать только динамические QR-коды;  
 - **011** — возможность формировать только статические и динамические QR-коды;  
 - **100** — возможность формировать только кассовые QR-коды;  
 - **101** — возможность формировать только статические и кассовые QR-коды;  
 - **110** — возможность формировать только динамические и кассовые QR-коды;  
 - **111** — возможность формировать статические, динамические и кассовые QR-коды.  
- **mcc** — MCC-код, соответствующий вашей деятельности.  

Мы рекомендуем указывать `capabilities` — 111, чтобы вам не пришлось заново проводить регистрацию ТСП, если потребуется работать с другим типом QR-кодов.  
Также можно добавить `contactPhoneNumber` — номер телефона ТСП.

После регистрации ТСП каждой торговой точке присваивается уникальный идентификатор — `merchantId`\.  
Чтобы получить список всех ТСП и информацию по ним можно вызвать метод: [Get Merchants List](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-TSP#get_merchants_list_sbp__apiVersion__merchant_legal_entity__legalId__get).

#### Регистрация юрлица или ТСП в СБП через Точку  

Зарегистрировать юрлицо и создать торговую точку можно в интернет-банке Точки — раздел «[QR-платежи](https://i.tochka.com/bank/m/products/sbp/registration)».\.  
Раздел доступен только руководителю и распорядителю с правом подписи.

<details>
    <summary>Как сгенерировать статические и динамические QR-коды</summary>

QR-коды нужны для получения оплат от физлиц. Их можно сформировать только для торговых точек, которые уже зарегистрированы в СБП.  

Для регистрации динамического или статического QR-кода выполните запрос: [Register Qr Code](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami#register_qr_code_sbp__apiVersion__qr_code_merchant__merchantId___accountId__post)\.  

В запросе необходимо передать:  
- **accountId** — уникальный и неизменный идентификатор счёта юрлица.  
- **merchantId** — идентификатор ТСП.  
- **paymentPurpose** — назначение платежа. Покупатель его увидит при считывании QR-кода.  
- **amount** — сумма платежа в копейках. Обязательно при создании динамических QR-кодов, для статических QR-кодов данное поле можно не заполнять.  
- **qrcType** — тип запрашиваемого QR-кода:  
 - **qrcType**: 01 — статический QR-код;  
 - **qrcType**: 02 — динамический QR-код.  
- **imageParams** — параметры изображения:  
 - **width** — ширина изображения (>=200, по умолчанию: 300);  
 - **height** — высота изображения (>=200, по умолчанию: 300).   

При создании статических и динамических QR-кодов можно указать дополнительные поля:  
- **currency** — валюта операции. На данный момент можно указать только RUB.  
- **sourceName** — название вашей интеграции. Эту информацию покупатель не увидит, можно указывать для аналитики.  
- **ttl** — время действия QR-кода в минутах, можно указать от 1 до 129 600 минут. Данный параметр есть только для динамических QR-кодов. Если при создании динамического QR-кода не передать этот параметр, его срок действия составит 72 часа.  
- **redirectUrl** — это ссылка в формате https, перенаправляющая пользователя на выбранную страницу после проведения оплаты.  

Если ваши клиенты оплачивают покупки через приложение, удобнее перенаправлять их по ссылке для оплаты, а не показывать изображение QR-кода.  
Для этого в ответе метода [Register Qr Code](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami#register_qr_code_sbp__apiVersion__qr_code_merchant__merchantId___accountId__post) мы передаём атрибут `payload`.  

Статус динамического QR-кода можно отследить с помощью метода [Get Qr Codes Payment Status](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami#get_qr_codes_payment_status_sbp__apiVersion__qr_codes__qrc_ids__payment_status_get) в течение периода использования QR-кода или 24 часов с момента оплаты. По истечению этого времени узнать статус не получится.  
Для статических QR-кодов статус отследить не получится, так как этот QR-код всегда активен.
</details>

<details>
    <summary>Как сгенерировать кассовые QR-коды</summary>
QR-коды нужны для получения оплат от физлиц. Их можно сформировать только для торговых точек, которые уже зарегистрированы в СБП.  


Для регистрации кассового QR-кода выполните запрос: [Register Cashbox Qrcode](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-kassovymi-QR-kodami#register_cashbox_qrcode_sbp__apiVersion__cashbox_qr_code_post)

В запросе необходимо передать:  
- **accountId** — уникальный и неизменный идентификатор счёта юрлица;  
- **merchantId** — идентификатор ТСП;  
- **imageParams** — параметры изображения:  
 - **width** — ширина изображения (>=200, по умолчанию: 300);  
 - **height** — высота изображения (>=200, по умолчанию: 300).  

Дополнительно можно указать **redirectUrl** — это ссылка в формате https, перенаправляющая пользователя на выбранную страницу после проведения оплаты.  

Из ответа необходимо забрать и сохранить следующие параметры:  
- **qrcId** — идентификатор QR-кода;  
- **payload** — ссылка для перенаправления пользователей;  
- **image** — изображение QR-кода.  
Эти параметры закрепляются для кассовой ссылки.  

Чтобы принимать оплату по QR-коду, его нужно каждый раз активировать с помощью метода: [Activate Cashbox Qrcode](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-kassovymi-QR-kodami#activate_cashbox_qrcode_sbp__apiVersion__cashbox_qr_code__qrcId__activate_post)\.  
В запросе необходимо передать:  
- **qrcId** — идентификатор QR-кода, полученный в методе Register Cashbox Qrcode;  
- **amount** — сумма платежа в копейках.  

Дополнительно в запросе можно указать:  
- **currency** — валюта операции. На данный момент можно указать только RUB.  
- **paymentPurpose** — назначение платежа. Покупатель его увидит при считывании QR-кода.  
- **ttl** — время действия QR-кода в минутах, можно указать от 5 до 20 минут. Если не передавать этот параметр, кассовый QR-код по умолчанию будет действовать 5 минут.  
Когда вы активируете QR-код, покупатель сможет провести оплату. После оплаты или по истечению ttl, QR-код деактивируется — необходимо вызвать метод Activate Cashbox Qrcode заново.  

Если покупатель передумал совершать оплату или больше нет необходимости в активном QR-коде, можно вручную вызвать метод на деактивацию QR-кода: [Deactivate Cashbox Qrcode](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-kassovymi-QR-kodami#deactivate_cashbox_qrcode_sbp__apiVersion__cashbox_qr_code__qrcId__deactivate_post).

При работе с кассовыми QR-кодами есть возможность изменить расчётный счёт для зачисления денег — но в рамках одной компании. Изменить расчётный счёт одной компании на счёт другой не получится.

Для изменения счёта необходимо вызвать метод [Change Cashbox Qrcode Account](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-kassovymi-QR-kodami#change_cashbox_qrcode_account_sbp__apiVersion__cashbox_qr_code__qrcId__account_post) и передать следующие параметры:  
- **qrcId** — идентификатор QR-кода, полученный в методе Register Cashbox Qrcode;  
- **accountId** — уникальный и неизменный идентификатор счёта юрлица.  

Чтобы посмотреть информацию обо всех выпущенных кассовых QR-кодах, вызовите метод [Get Cashbox Qrcode List](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-kassovymi-QR-kodami#get_cashbox_qrcode_list_sbp__apiVersion__cashbox_qr_code_merchant__merchantId___accountId__get)\.  
Чтобы посмотреть информацию по конкретному кассовому QR-коду, вызовите метод [Get Cashbox Qrcode](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-kassovymi-QR-kodami#get_cashbox_qrcode_sbp__apiVersion__cashbox_qr_code__qrcId__post)\.  
Обратите внимание, что необязательные параметры передаются, только когда QR-код активен — по нему ещё не прошла оплата или ещё не закончился `ttl`.
</details>

<details>
    <summary>Как сгенерировать B2B QR-коды</summary>
B2B QR-коды нужны для приёма платежей от ИП и организаций. Формировать их можно только для торговых точек, которые уже зарегистрированы в СБП.  

Для регистрации B2B QR-кода выполните запрос [Register B2B Qr Code](/doc/v2/redoc/tag/Servis-SBP:-rabota-s-B2B-QR-kodami#register_b2b_qr_code_sbp__apiVersion__b2b_qr_code_merchant__merchantId___accountId__post) с такими параметрами:  
- **accountId** — уникальный и неизменный идентификатор счёта юрлица;  
- **merchantId** — идентификатор торговой Точки в СБП (ТСП);  
- **paymentPurpose** — назначение платежа, которое покупатель увидит при считывании QR-кода;  
- **amount** — сумма платежа в копейках. Максимальная сумма для B2B QR-кода составляет 1 миллион рублей;  
- **sourceName** — название вашей интеграции. Покупатель его не увидит, можно указывать для аналитики;  
- **takeTax** — информация об НДС (True — для получения платежей с НДС, false — без НДС).  

При создании B2B QR-кодов можно указать дополнительные параметры:  
- **totalTaxAmount** — сумма НДС в копейках. Необходимо, если в параметре takeTax стоит true.  
- **ttl** — время действия QR-кода в минутах, можно указать от 1 до 129 600 минут. Если при создании QR-кода не передать этот параметр, его срок действия составит 72 часа.  
- **redirectUrl** — ссылка в формате https, на которую нужно перенаправить пользователя после проведения оплаты.  
- **uip** —  назначаемый получателем уникальный идентификатор платежа.  

Чтобы посмотреть информацию по конкретному B2B QR-коду, вызовите метод [Get B2B Qr Code](/doc/v2/redoc/tag/Servis-SBP:-rabota-s-B2B-QR-kodami#get_b2b_qr_code_sbp__apiVersion__b2b_qr_code__qrcId__get).

Чтобы сразу узнать о поступлении оплаты, вы можете настроить вебхук с событием [incomingSbpB2BPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) на свой URL.
</details>

### Возврат через СБП

> На данный момент вернуть платежи поступившие через СБП можно только по динамическим QR-кодам.  
Платежи по статическим, кассовым и B2B QR-кодам возвращаются через интернет-банк Точки.

Принятые по QR-кодам платежи в статусе `Accepted` можно возвращать покупателю через Систему быстрых платежей.

Возврат может быть как полным, так и частичным. Главное — чтобы его сумма не превышала сумму поступления на расчётный счёт.

1. Получите refTransactionId

- Если у вас подключён вебхук с событием [incomingSbpPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki), то параметр _refTransactionId_ мы передадим при зачислении денег.

- Если у вас нет подключённого вебхука с событием incomingSbpPayment, используйте метод [Get Payments](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-vozvratami#get_payments_sbp__apiVersion__get_sbp_payments_get) — он найдёт платёж, который нужно вернуть.

В запросе укажите необходимый промежуток времени через поля `fromDate` и `toDate`. Период поиска не ограничен, но удобнее искать за день, неделю или месяц — в зависимости от того, что сообщил покупатель.  
Для фильтрации можете указать `qrcId` и `page`. 

В ответе вам нужно взять параметры refTransactionId и qrcId — идентификатора QR-кода в СБП.

2. Вызовите метод [Start Refund](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-vozvratami#start_refund_sbp__apiVersion__refund_post) — он отправляет запрос на возврат платежа по СБП. Укажите в запросе  

- **accountCode** и **bankCode** — это счёт и БИК вашей компании, то есть плательщика, со счёта которого происходит возврат.
- **amount** — сумма возврата. Указывается в рублях, а не копейках.
- **qrcId** — идентификатор QR-кода по которому зачислились деньги.
- **purpose** — назначение платежа — можно не заполнять, если ваш покупатель резидент РФ. Если платёж нужно вернуть нерезиденту, обязательно заполните поле кодом валютной операции {VO99020}. 

3. Отследите статус возврата платежа с помощью метода [Get Refund Data](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-vozvratami#get_refund_data_sbp__apiVersion__refund__request_id__get) .  
Финальный успешный статус  — `Accepted`. Если возврат не прошёл, будет статус `Rejected`.


# Песочница

### Для чего нужна
Песочница — это специальный режим работы OpenAPI для разработки. В нём все данные заменены на тестовые.

### Как получить доступ
К песочнице не нужно получать отдельный доступ, достаточно просто изменить url в запросах с `https://enter.tochka.com/uapi` на `https://enter.tochka.com/sandbox/v2`

Вместо токена для авторизации в заголовке `Authorization` нужно использовать строку `sandbox.jwt.token`, которая даёт все разрешения во всех методах. Пример: `Authorization: Bearer sandbox.jwt.token`.
Любая другая строка в этом заголовке определяется как отсутствие разрешения на выполнения метода

Пример готового окружения для песочницы есть в нашей [коллекции запросов](/doc/v2/redoc/section/Kollekciya-zaprosov)

### Работа с разрешениями
Для песочницы не нужно создавать разрешения, менять код на токен и обновлять токен.
`sandbox.jwt.token` всегда действителен и предоставляет все разрешения

### Реализованные методы
Сейчас в песочнице можно вызвать все методы из следующих разделов:
1. [Работа со счетами](/doc/v2/redoc/tag/Rabota-so-schetami)
2. [Работа с клиентами](/doc/v2/redoc/tag/Rabota-s-klientami)
3. [Работа с балансами счетов](/doc/v2/redoc/tag/Rabota-s-balansami-schetov)
4. [Работа с выписками](/doc/v2/redoc/tag/Rabota-s-vypiskami)
5. [Работа с платежами](/doc/v2/redoc/tag/Rabota-s-platezhami)
6. [Сервис СБП: Работа с ЮЛ](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-YuL)
7. [Сервис СБП: Работа с ТСП](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-TSP)
8. [Сервис СБП: Работа с QR-кодами](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami)
9. [Сервис СБП: Работа с возвратами](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-vozvratami)
10. [Работа с выставлением счетов](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov)
11. [Работа с закрывающими документами](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami)
12. [Работа с платёжными ссылками](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami)
# Обновления API

## 1.77.3
07.08.25

- Добавили возможность создавать платёжные ссылки с [двухэтапной оплатой](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Platyozhnye-ssylki) — с помощью поля `preAuthorization` в методах [Create Payment Operation](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post) и [Create Payment Operation With Receipt](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_with_receipt_acquiring__apiVersion__payments_with_receipt_post).
- В вебхуке с событием [acquiringInternetPayment](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) у поля `status` появилось значение `AUTHORIZED`. Событие с таким статусом приходит после заморозки денег на счёте плательщика при [двухэтапной оплате](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Platyozhnye-ssylki).

## 1.65.1
18.04.25

- Добавили в методы [Create Payment Operation](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post) и [Create Payment Operation With Receipt](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_with_receipt_acquiring__apiVersion__payments_with_receipt_post) параметр `ttl`, который позволяет задавать срок действия платёжной ссылки в минутах, и объект `Supplier`, в котором можно передавать данные поставщика.  
- Реализовали возможность создавать подписки без графика списания и с изменением суммы платежа. Для создания такой подписки в методах [Create Subscription](/doc/v2/redoc/tag/Rabota-s-podpiskami#create_subscription_acquiring__apiVersion__subscriptions_post) и [Create Subscription With Receipt](/doc/v2/redoc/tag/Rabota-s-podpiskami#create_subscription_with_receipt_acquiring__apiVersion__subscriptions_with_receipt_post) нужно указывать boolean параметр `recurring` со значением `true`, а для списания денег — вызывать метод [Charge Subscription](/doc/v2/redoc/tag/Rabota-s-podpiskami#charge_subscription_acquiring__apiVersion__subscriptions__operationId__charge_post).  
- В методы [Get Payment Operation Info](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_payment_operation_info_acquiring__apiVersion__payments__operationId__get) и [Get Payment Operation List](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_payment_operation_list_acquiring__apiVersion__payments_get) добавили параметр `paidAt`, который показывает время оплаты платёжной ссылки, и объект `Order`, содержащий информацию обо всех совершённых платежах и возвратах по платёжной ссылке.

## 1.63.1
31.03.25

- Добавили [метод для создания B2B QR-кодов](/doc/v2/redoc/tag/Servis-SBP:-rabota-s-B2B-QR-kodami#register_b2b_qr_code_sbp__apiVersion__b2b_qr_code_merchant__merchantId___accountId__post), с которыми у вас появляется возможность принимать платежи от своих контрагентов в любое время дня и ночи.  
- Добавили [метод для получения информации](/doc/v2/redoc/tag/Servis-SBP:-rabota-s-B2B-QR-kodami#get_b2b_qr_code_sbp__apiVersion__b2b_qr_code__qrcId__get) о созданных B2B QR-кодах.  
- Создали новый тип вебхука [incomingSbpB2BPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki), чтобы вы всегда были в курсе поступающих по B2B QR-кодам платежей.
- При [создании счёт-фактуры](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#create_closing_document_invoice__apiVersion__closing_documents_post) добавили объект `shipmentDocuments`, который подтверждает отгрузку товаров, работ или услуг.

## 1.59.1
11.02.25

- Добавили параметры `nds_5` и `nds_7` при [выставлении счетов и закрывающих документов](/doc/v2/redoc/section/Opisanie-metodov#Vystavlenie-schetov-i-sozdanie-zakryvayushih-dokumentov).
- Обновили вебхук [acquiringInternetPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki), теперь в нём содержится больше полезной информации.

## 1.58.0
03.02.25

- Добавили в вебхук [incomingSbpPayment](/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) параметр refTransactionId, с помощью которого вы можете проводить возвраты по СБП.

## 1.56.0
23.12.24

- Добавили в методы [Create Payment Operation](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post) и [Create Payment Operation With Receipt](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_with_receipt_acquiring__apiVersion__payments_with_receipt_post) возможность принимать платежи через T-Pay. Для этого у торговой точки в интернет-банке нужно выбрать Т-Pay как способ для приёма платежей, а в методах API использовать параметр paymentMode: tinkoff.
- Обогатили данными метод [Get Retailers](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#get_retailers_acquiring__apiVersion__retailers_get): теперь он отдаёт информацию о возможных способах оплаты и наличии онлайн-касс.


## 1.55.0
06.12.24

- Добавили новые методы для работы с подписками в интернет-эквайринге. С их помощью вы сможете создавать подписки (рекуррентные платежи) на определённый период, чтобы в течение этого периода у вашего покупателя списывались деньги с банковской карты.
- Добавили необязательное поле `merchantId` в метод [Create Payment Operation](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post). Это поле необходимо заполнять, если у вас есть несколько торговых точек в интернет-эквайринге. Оно позволяет нам понять, по какой торговой точке будут происходить зачисления на ваш счёт.
- Исправили задержки при отправке вебхуков, теперь они работают еще быстрее.
- Добавили новые методы в песочницу и коллекцию запросов Postman.
- Создали метод [Get Payment For Sign List](/doc/v2/redoc/tag/Rabota-s-platezhami#get_payment_for_sign_list_payment__apiVersion__for_sign_get), который позволяет просматривать все платежи из раздела «На подпись» интернет-банка.
- Обновили дизайн раздела «Интеграции и API» в интернет-банке.
- По каждому обновлению добавили информацию в документацию, чтобы вам было легче разобраться с нашим API.


## 1.46.0
30.05.24

- Добавили методы по работе с [кассовыми QR-кодами](/doc/v2/redoc/section/Opisanie-metodov#SBP-Sistema-bystryh-platezhej)\.  
- Обновили методы по работе с платёжными ссылками, выставлением счетов и закрывающих документов в [песочнице](/doc/v2/redoc/section/Pesochnica)\.  
- Добавили необязательный параметр `email` в метод [Create Payment For Sign](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_for_sign_payment__apiVersion__for_sign_post).


## 1.45.0
22.05.24

- Добавили новый метод по работе с платёжными ссылками [Create Payment Operation With Receipt](/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_with_receipt_acquiring__apiVersion__payments_with_receipt__post). Он позволяет формировать ссылки на оплату с автоматической фискализацией чеков от партнёров Точки.

## 1.44.6
05.04.24

Создали четыре новых метода для работы с выставлением счетов и формированием закрывающих документов:

- [Delete Invoice](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#delete_invoice_invoice__apiVersion__bills__customerCode___documentId__delete) — для удаления счёта.
- [Send Invoice To Email](/doc/v2/redoc/tag/Rabota-s-vystavleniem-schetov#send_invoice_to_email_invoice__apiVersion__bills__customerCode___documentId__email_post) — для отправки счёта в PDF на электронную почту.
- [Delete Closing Documents](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#delete_closing_documents_invoice__apiVersion__closing_documents__customerCode___documentId__delete) — для удаления закрывающих документов.
- [Send Closing Documents To Email](/doc/v2/redoc/tag/Rabota-s-zakryvayushimi-dokumentami#send_closing_documents_to_email_invoice__apiVersion__closing_documents__customerCode___documentId__email_post) — для отправки закрывающих документов на электронную почту.

## 1.44.1
01.02.24

- Добавили новый тип вебхука [outgoingPayment](https://enter.tochka.com/doc/v2/redoc/section/Opisanie-metodov#Vebhuki) для получения уведомлений об исходящих платежах.
- Добавили новый параметр в методы [Get Statements List](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-vypiskami#get_statements_list_open_banking__apiVersion__statements_get) и [Get Statement](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-vypiskami#get_statement_open_banking__apiVersion__accounts__accountId__statements__statementId__get), а также в вебхук типа incomingPayment. Это paymentId — уникальный идентификатор платежа.
- Сделали обязательным поле purpose в методе [Create Payment Operation](https://enter.tochka.com/doc/v2/redoc/tag/Rabota-s-platyozhnymi-ssylkami#create_payment_operation_acquiring__apiVersion__payments_post).
- Обновили в документации пример тела вебхука по событию incomingPayment.
- Доработали методы для выставления счетов и формирования закрывающих документов. Поле `totalNds` больше не является обязательным.

## 1.43.3
08.12.23

- Добавили методы для выставления счетов на оплату и создания закрывающих документов. Также с помощью API теперь можно получить PDF с выставленными счетами и закрывающими документами, а также посмотреть статус оплаты счетов.
- Создали метод [Get Authorized Card Transactions](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_authorized_card_transactions_open_banking__apiVersion__accounts__accountId__authorized_card_transactions_get), который позволяет получать информацию о карточных операциях, находящихся в резерве.
- Обновили разрешения для работы с вебхуками — теперь для создания, удаления или изменения вебхука нужно использовать разрешение [ManageWebhookData](/doc/v2/redoc/section/Authentication). Существующие вебхуки продолжат свою работу, но для новых интеграций необходимо использовать новое разрешение.

## 1.42.5
20.11.23

- Добавлена возможность работы с [платёжными ссылками](doc/v2/redoc/section/Opisanie-metodov#Platyozhnye-ssylki),
- в методах [Get Balance Info](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_balance_info_open_banking__apiVersion__accounts__accountId__balances_get) и [Get Balances List](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_balances_list_open_banking__apiVersion__balances_get) появился новый тип баланса — OverdraftAvailable.
- В методе [Create Payment](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_payment__apiVersion__order_post) обновлён способ авторизации для проведения платежа.
- Из документации и коллекции запросов в Postman удалены неактуальные методы.

## 1.38.5
07.07.2023

- Добавили в метод [Register Qr Code](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami#register_qr_code_sbp__apiVersion__qr_code_merchant__merchantId___accountId__post) необязательный атрибут - redirectUrl.  
С помощью этого атрибута вы сможете перенаправить покупателя после оплаты по QR-коду на ваш сайт или приложение.


## 1.38.3
27.06.2023

- Удалили из документации устаревшие методы для работы в СБП.

## 1.37.10
26.04.2023

После получения Точкой банковской лицензии мы изменили несколько методов:
- В методе для получения информации о клиенте в СБП [Get Customer Info](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-YuL#get_customer_info_sbp__apiVersion__customer__customerCode___bank_code__get) теперь необходимо передавать bankCode — БИК банка.
- В теле запроса метода для регистрации клиента в СБП [Register Legal Entity](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-YuL#register_legal_entity_sbp__apiVersion__register_sbp_legal_entity_post) теперь нужно будет передавать bankCode — БИК банка. 
- В методе для получения данных о QR-коде и ТСП по идентификатору QR-кода [Get Qr Code Payment Data](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami#get_qr_code_payment_data_sbp__apiVersion__qr_code__qrcId__payment_sbp_data_get) изменился эндпоинт запроса.  
Предыдущие методы объявлены **deprecated** и в ближайшее время перестанут работать.


## 1.37.9
13.04.2023

- Обновили тело вебхука по событию icomingSbpPayment, чтобы подпись корректно валидировалась с нашим публичным ключом openAPI.
- Удалили проверку длины названия юридического лица при запросе выписки.
- Добавили ограничение на попытки отправлять вебхук и создали проверку доступности URL вебхука при его создании или обновлении: если на отправленный вебхук мы не получили ответ с кодом состояния HTTP 200, а получили любой другой код, то будем отправлять его повторно 30 раз с периодичностью в 10 секунд.  
Для проверки доступности указанного вами хоста при создании или изменении вебхука мы отправим на ваш URL по одному тестовому вебхуку на каждое из событий, на которые вы подписаны.  
Если в статусе ответа не придёт код HTTP 200, то вебхук не будет создан или изменён.

- Добавили в песочницу все методы по работе в СБП, теперь вы сможете отладить работу с QR-кодами без регистрации QR-кодов на боевом слое.


## 1.37.0
01.02.2023

- Добавили пример с телом вебхука по событиям icomingPayment и icomingSbpPayment.
- Теперь при регистрации QR-кода на несуществующий merchantId мы отдаём ошибку 403 — forbidden by consent.

- Добавили для работы с API БИК Точки.
- Исправили ошибку в методах [Get Balance Info](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_balance_info_open_banking__apiVersion__accounts__accountId__balances_get) и [Get Balances List](/doc/v2/redoc/tag/Rabota-s-balansami-schetov#get_balances_list_open_banking__apiVersion__balances_get) — мы отдавали accountId без значения БИК.
- Поправили ошибку метода [Start Refund](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-vozvratami#start_refund_sbp__apiVersion__refund_post), которая позволяла вернуть деньги по чужому QR-коду.

## 1.36.4
13.12.2022

- Обновили в документации описание всех методов, чтобы вам было проще разобраться в нашем API.

## 1.36.0
20.10.2022

- Вы просили, мы сделали: добавили [JWT-токен](https://enter.tochka.com/doc/v2/redoc/section/Algoritm-raboty-s-JWT-tokenom) для упрощённой авторизации. А ещё он избавляет от необходимости обновлять токены каждый день — просто подставьте сгенерированный ключ в заголовок _Authorization_.

## 1.34.0
18.07.2022

- Добавили [методы для работы с тендерными спецсчетами](/doc/v2/redoc/section/Opisanie-metodov#Rabota-so-specschyotom). Теперь вы сможете получать актуальный баланс спецсчёта и видеть, сколько денег и какими электронными торговыми площадками (ЭТП) заблокированы.

## 1.30.0
22.03.2022

- Появился режим [песочницы](/doc/v2/redoc/section/Pesochnica). Сейчас доступны только основные методы.

## 1.29.0
24.02.2022

- Появились методы на совершение возвратов сбп платежей и отслеживания их статуса. [Подробнее](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-vozvratami).

## 1.28.0
01.12.2021

- Теперь у нас есть вебхуки, подробнее можно прочитать [здесь](/doc/v2/redoc/section/Opisanie-metodov#Rabota-s-vebhukami).


## 1.26.5
23.09.2021

- В [методе](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-TSP#register_merchant_sbp__apiVersion__merchant_legal_entity__legalId__post) регистрации ТСП в СБП теперь необходимо передавать customerCode как параметр, а не как заголовок.

- В [методе](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-YuL#register_legal_entity_sbp__apiVersion__register_legal_entity_post) регистрации юрлица в СБП изменился адрес и набор параметров.


## 1.25.0
23.07.2021

- Теперь сформировать ссылку для подписания списка разрешений можно передав в нее id разрешения, без необходимости генерировать jwt объект.

## 1.24.1
16.06.2021

- В [выписке](/doc/v2/redoc/tag/Rabota-s-vypiskami) появился баланс счёта на начало и конец запрашиваемого периода. Теперь вы можете посмотреть полную выписку: с начальными и итоговыми остатками на счёте.

- В [методы](/doc/v2/redoc/tag/Rabota-so-schetami) работы со счетами добавили поле `registrationDate` с датой открытия счёта. Можно запросить выписку за весь период работы, отдельно вводить дату открытия счёта не нужно.


## 1.23.1
21.05.2021

- Обновили [метод](/doc/v2/redoc/tag/Rabota-s-vypiskami#get_statement_open_banking__apiVersion__accounts__accountId__statements__statementId__get) работы с выписками. Теперь в поле *amountNat* лежит сумма транзакции в рублях, а в *amount* — в валюте счёта.

- Для методов [Create_payment](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_payment__apiVersion__order_post) и [Payment_for_sign](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_for_sign_payment__apiVersion__for_sign_post) добавили валидацию полей. Проверяем все поля перед отправкой платежа в банк. Если будет какая-то ошибка, вы сразу сможете показать её тому, кто создаёт платёж.


## 1.21.5
12.04.2021

- Добавлена валидация поля окончания срока действия разрешения `expirationDateTime`. Теперь нельзя указать дату меньше текущей.

## 1.20.0
25.02.2021

- Добавили [метод](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_payment__apiVersion__order_post) создания и подписания платежа. Теперь подписывать платежи можно через API, не заходя в интернет-банк.

## 1.19.0
10.02.2021

- Добавили новый [метод](/doc/v2/redoc/tag/Servis-SBP:-Rabota-s-QR-kodami#get_qr_codes_payment_status_sbp__apiVersion__qr_codes__qrc_ids__payment_status_get) запроса статуса платежей для СБП, который работает быстрее.

## 1.13.0
10.11.2020

- Добавили [метод](/doc/v2/redoc/tag/Rabota-s-platezhami#create_payment_for_sign_payment__apiVersion__for_sign_post) создания платежа на подпись. Теперь можно формировать платежи и отправлять их напрямую в интернет-банк. Подписать такие платежи пока можно только в интернет-банке.

## 1.11.0
08.10.2020

- Добавили [методы](/doc/v2/redoc/tag/Rabota-s-balansami-schetov) работы с балансами: вы можете получить остаток по счетам на текущий день.
- Добавили [методы](/doc/v2/redoc/tag/Rabota-s-vypiskami)  работы с выписками: можно получать движения по счетам через API, необязательно заходить в интернет-банк.


## Path Table

| Method | Path | Description |
| --- | --- | --- |
| GET | [/acquiring/{apiVersion}/payments](#getacquiringapiversionpayments) | Get Payment Operation List |
| POST | [/acquiring/{apiVersion}/payments](#postacquiringapiversionpayments) | Create Payment Operation |
| GET | [/acquiring/{apiVersion}/payments/{operationId}](#getacquiringapiversionpaymentsoperationid) | Get Payment Operation Info |
| POST | [/acquiring/{apiVersion}/payments/{operationId}/capture](#postacquiringapiversionpaymentsoperationidcapture) | Capture Payment |
| POST | [/acquiring/{apiVersion}/payments/{operationId}/refund](#postacquiringapiversionpaymentsoperationidrefund) | Refund Payment Operation |
| POST | [/acquiring/{apiVersion}/payments_with_receipt](#postacquiringapiversionpayments_with_receipt) | Create Payment Operation With Receipt |
| GET | [/acquiring/{apiVersion}/retailers](#getacquiringapiversionretailers) | Get Retailers |
| GET | [/acquiring/{apiVersion}/subscriptions](#getacquiringapiversionsubscriptions) | Get Subscription List |
| POST | [/acquiring/{apiVersion}/subscriptions](#postacquiringapiversionsubscriptions) | Create Subscription |
| POST | [/acquiring/{apiVersion}/subscriptions/{operationId}/charge](#postacquiringapiversionsubscriptionsoperationidcharge) | Charge Subscription |
| GET | [/acquiring/{apiVersion}/subscriptions/{operationId}/status](#getacquiringapiversionsubscriptionsoperationidstatus) | Get Subscription Status |
| POST | [/acquiring/{apiVersion}/subscriptions/{operationId}/status](#postacquiringapiversionsubscriptionsoperationidstatus) | Set Subscription Status |
| POST | [/acquiring/{apiVersion}/subscriptions_with_receipt](#postacquiringapiversionsubscriptions_with_receipt) | Create Subscription With Receipt |
| GET | [/consent/{apiVersion}/consents](#getconsentapiversionconsents) | Get All Consents List |
| POST | [/consent/{apiVersion}/consents](#postconsentapiversionconsents) | Create New Consent |
| GET | [/consent/{apiVersion}/consents/{consentId}/child](#getconsentapiversionconsentsconsentidchild) | Get All Child Consents |
| GET | [/consent/{apiVersion}/{consentId}](#getconsentapiversionconsentid) | Get Consent Info |
| POST | [/invoice/{apiVersion}/bills](#postinvoiceapiversionbills) | Create Invoice |
| DELETE | [/invoice/{apiVersion}/bills/{customerCode}/{documentId}](#deleteinvoiceapiversionbillscustomercodedocumentid) | Delete Invoice |
| POST | [/invoice/{apiVersion}/bills/{customerCode}/{documentId}/email](#postinvoiceapiversionbillscustomercodedocumentidemail) | Send Invoice To Email |
| GET | [/invoice/{apiVersion}/bills/{customerCode}/{documentId}/file](#getinvoiceapiversionbillscustomercodedocumentidfile) | Get Invoice |
| GET | [/invoice/{apiVersion}/bills/{customerCode}/{documentId}/payment-status](#getinvoiceapiversionbillscustomercodedocumentidpayment-status) | Get Invoice Payment Status |
| POST | [/invoice/{apiVersion}/closing-documents](#postinvoiceapiversionclosing-documents) | Create Closing Document |
| DELETE | [/invoice/{apiVersion}/closing-documents/{customerCode}/{documentId}](#deleteinvoiceapiversionclosing-documentscustomercodedocumentid) | Delete Closing Documents |
| POST | [/invoice/{apiVersion}/closing-documents/{customerCode}/{documentId}/email](#postinvoiceapiversionclosing-documentscustomercodedocumentidemail) | Send Closing Documents To Email |
| GET | [/invoice/{apiVersion}/closing-documents/{customerCode}/{documentId}/file](#getinvoiceapiversionclosing-documentscustomercodedocumentidfile) | Get Closing Document |
| GET | [/open-banking/{apiVersion}/accounts](#getopen-bankingapiversionaccounts) | Get Accounts List |
| GET | [/open-banking/{apiVersion}/accounts/{accountId}](#getopen-bankingapiversionaccountsaccountid) | Get Account Info |
| GET | [/open-banking/{apiVersion}/accounts/{accountId}/authorized-card-transactions](#getopen-bankingapiversionaccountsaccountidauthorized-card-transactions) | Get Authorized Card Transactions |
| GET | [/open-banking/{apiVersion}/accounts/{accountId}/balances](#getopen-bankingapiversionaccountsaccountidbalances) | Get Balance Info |
| GET | [/open-banking/{apiVersion}/accounts/{accountId}/statements/{statementId}](#getopen-bankingapiversionaccountsaccountidstatementsstatementid) | Get Statement |
| GET | [/open-banking/{apiVersion}/balances](#getopen-bankingapiversionbalances) | Get Balances List |
| GET | [/open-banking/{apiVersion}/customers](#getopen-bankingapiversioncustomers) | Get Customers List |
| GET | [/open-banking/{apiVersion}/customers/{customerCode}](#getopen-bankingapiversioncustomerscustomercode) | Get Customer Info |
| GET | [/open-banking/{apiVersion}/statements](#getopen-bankingapiversionstatements) | Get Statements List |
| POST | [/open-banking/{apiVersion}/statements](#postopen-bankingapiversionstatements) | Init Statement |
| GET | [/payment/{apiVersion}/for-sign](#getpaymentapiversionfor-sign) | Get Payment For Sign List |
| POST | [/payment/{apiVersion}/for-sign](#postpaymentapiversionfor-sign) | Create Payment For Sign |
| POST | [/payment/{apiVersion}/order](#postpaymentapiversionorder) | Create Payment |
| GET | [/payment/{apiVersion}/status/{requestId}](#getpaymentapiversionstatusrequestid) | Get Payment Status |
| GET | [/sbp/{apiVersion}/account/{legalId}](#getsbpapiversionaccountlegalid) | Get Accounts List |
| POST | [/sbp/{apiVersion}/b2b-qr-code/merchant/{merchantId}/{accountId}](#postsbpapiversionb2b-qr-codemerchantmerchantidaccountid) | Register B2B Qr Code |
| GET | [/sbp/{apiVersion}/b2b-qr-code/{qrcId}](#getsbpapiversionb2b-qr-codeqrcid) | Get B2B Qr Code |
| POST | [/sbp/{apiVersion}/cashbox-qr-code](#postsbpapiversioncashbox-qr-code) | Register Cashbox Qrcode |
| GET | [/sbp/{apiVersion}/cashbox-qr-code/merchant/{merchantId}/{accountId}](#getsbpapiversioncashbox-qr-codemerchantmerchantidaccountid) | Get Cashbox Qrcode List |
| POST | [/sbp/{apiVersion}/cashbox-qr-code/{qrcId}](#postsbpapiversioncashbox-qr-codeqrcid) | Get Cashbox Qrcode |
| POST | [/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/account](#postsbpapiversioncashbox-qr-codeqrcidaccount) | Change Cashbox Qrcode Account |
| POST | [/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/activate](#postsbpapiversioncashbox-qr-codeqrcidactivate) | Activate Cashbox Qrcode |
| POST | [/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/deactivate](#postsbpapiversioncashbox-qr-codeqrciddeactivate) | Deactivate Cashbox Qrcode |
| GET | [/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/operation](#getsbpapiversioncashbox-qr-codeqrcidoperation) | Get Cashbox Qrcode Operation Info |
| GET | [/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/status](#getsbpapiversioncashbox-qr-codeqrcidstatus) | Get Cashbox Qrcode Status |
| GET | [/sbp/{apiVersion}/customer/{customerCode}/{bankCode}](#getsbpapiversioncustomercustomercodebankcode) | Get Customer Info |
| GET | [/sbp/{apiVersion}/get-sbp-payments](#getsbpapiversionget-sbp-payments) | Get Payments |
| GET | [/sbp/{apiVersion}/legal-entity/{legalId}](#getsbpapiversionlegal-entitylegalid) | Get Legal Entity |
| POST | [/sbp/{apiVersion}/legal-entity/{legalId}](#postsbpapiversionlegal-entitylegalid) | Set Legal Entity Status |
| GET | [/sbp/{apiVersion}/merchant/legal-entity/{legalId}](#getsbpapiversionmerchantlegal-entitylegalid) | Get Merchants List |
| POST | [/sbp/{apiVersion}/merchant/legal-entity/{legalId}](#postsbpapiversionmerchantlegal-entitylegalid) | Register Merchant |
| GET | [/sbp/{apiVersion}/merchant/{merchantId}](#getsbpapiversionmerchantmerchantid) | Get Merchant |
| PUT | [/sbp/{apiVersion}/merchant/{merchantId}](#putsbpapiversionmerchantmerchantid) | Set Merchant Status |
| GET | [/sbp/{apiVersion}/qr-code/legal-entity/{legalId}](#getsbpapiversionqr-codelegal-entitylegalid) | Get Qr Codes List |
| POST | [/sbp/{apiVersion}/qr-code/merchant/{merchantId}/{accountId}](#postsbpapiversionqr-codemerchantmerchantidaccountid) | Register Qr Code |
| GET | [/sbp/{apiVersion}/qr-code/{qrcId}](#getsbpapiversionqr-codeqrcid) | Get Qr Code |
| GET | [/sbp/{apiVersion}/qr-codes/{qrc_ids}/payment-status](#getsbpapiversionqr-codesqrc_idspayment-status) | Get Qr Codes Payment Status |
| POST | [/sbp/{apiVersion}/refund](#postsbpapiversionrefund) | Start Refund |
| GET | [/sbp/{apiVersion}/refund/{request_id}](#getsbpapiversionrefundrequest_id) | Get Refund Data |
| POST | [/sbp/{apiVersion}/register-sbp-legal-entity](#postsbpapiversionregister-sbp-legal-entity) | Register Legal Entity |
| DELETE | [/webhook/{apiVersion}/{client_id}](#deletewebhookapiversionclient_id) | Delete Webhook |
| GET | [/webhook/{apiVersion}/{client_id}](#getwebhookapiversionclient_id) | Get Webhooks |
| POST | [/webhook/{apiVersion}/{client_id}](#postwebhookapiversionclient_id) | Edit Webhook |
| PUT | [/webhook/{apiVersion}/{client_id}](#putwebhookapiversionclient_id) | Create Webhook |
| POST | [/webhook/{apiVersion}/{client_id}/test_send](#postwebhookapiversionclient_idtest_send) | Send Webhook |

## Reference Table

| Name | Path | Description |
| --- | --- | --- |
| Account | [#/components/schemas/Account](#componentsschemasaccount) |  |
| AccountDetailModel | [#/components/schemas/AccountDetailModel](#componentsschemasaccountdetailmodel) |  |
| AccountIdentificationEnum | [#/components/schemas/AccountIdentificationEnum](#componentsschemasaccountidentificationenum) |  |
| AccountListModel | [#/components/schemas/AccountListModel](#componentsschemasaccountlistmodel) |  |
| AccountListResponse | [#/components/schemas/AccountListResponse](#componentsschemasaccountlistresponse) |  |
| AccountModel | [#/components/schemas/AccountModel](#componentsschemasaccountmodel) |  |
| AccountResponseModel | [#/components/schemas/AccountResponseModel](#componentsschemasaccountresponsemodel) | Метод получения информации по счёту |
| AcquiringCapturePaymentResponseModel | [#/components/schemas/AcquiringCapturePaymentResponseModel](#componentsschemasacquiringcapturepaymentresponsemodel) |  |
| AcquiringChargeSubscriptionRequestDataModel | [#/components/schemas/AcquiringChargeSubscriptionRequestDataModel](#componentsschemasacquiringchargesubscriptionrequestdatamodel) |  |
| AcquiringChargeSubscriptionRequestModel | [#/components/schemas/AcquiringChargeSubscriptionRequestModel](#componentsschemasacquiringchargesubscriptionrequestmodel) |  |
| AcquiringChargeSubscriptionResponseDataModel | [#/components/schemas/AcquiringChargeSubscriptionResponseDataModel](#componentsschemasacquiringchargesubscriptionresponsedatamodel) |  |
| AcquiringCreatePaymentOperationRequestDataModel | [#/components/schemas/AcquiringCreatePaymentOperationRequestDataModel](#componentsschemasacquiringcreatepaymentoperationrequestdatamodel) |  |
| AcquiringCreatePaymentOperationRequestModel | [#/components/schemas/AcquiringCreatePaymentOperationRequestModel](#componentsschemasacquiringcreatepaymentoperationrequestmodel) |  |
| AcquiringCreatePaymentOperationResponseDataModel | [#/components/schemas/AcquiringCreatePaymentOperationResponseDataModel](#componentsschemasacquiringcreatepaymentoperationresponsedatamodel) |  |
| AcquiringCreatePaymentOperationResponseModel | [#/components/schemas/AcquiringCreatePaymentOperationResponseModel](#componentsschemasacquiringcreatepaymentoperationresponsemodel) |  |
| AcquiringCreatePaymentOperationWithReceiptRequestDataModel | [#/components/schemas/AcquiringCreatePaymentOperationWithReceiptRequestDataModel](#componentsschemasacquiringcreatepaymentoperationwithreceiptrequestdatamodel) |  |
| AcquiringCreatePaymentOperationWithReceiptRequestModel | [#/components/schemas/AcquiringCreatePaymentOperationWithReceiptRequestModel](#componentsschemasacquiringcreatepaymentoperationwithreceiptrequestmodel) |  |
| AcquiringCreatePaymentOperationWithReceiptResponseDataModel | [#/components/schemas/AcquiringCreatePaymentOperationWithReceiptResponseDataModel](#componentsschemasacquiringcreatepaymentoperationwithreceiptresponsedatamodel) |  |
| AcquiringCreatePaymentOperationWithReceiptResponseModel | [#/components/schemas/AcquiringCreatePaymentOperationWithReceiptResponseModel](#componentsschemasacquiringcreatepaymentoperationwithreceiptresponsemodel) |  |
| AcquiringCreateSubscriptionRequestDataModel | [#/components/schemas/AcquiringCreateSubscriptionRequestDataModel](#componentsschemasacquiringcreatesubscriptionrequestdatamodel) |  |
| AcquiringCreateSubscriptionRequestModel | [#/components/schemas/AcquiringCreateSubscriptionRequestModel](#componentsschemasacquiringcreatesubscriptionrequestmodel) |  |
| AcquiringCreateSubscriptionResponseDataModel | [#/components/schemas/AcquiringCreateSubscriptionResponseDataModel](#componentsschemasacquiringcreatesubscriptionresponsedatamodel) |  |
| AcquiringCreateSubscriptionResponseModel | [#/components/schemas/AcquiringCreateSubscriptionResponseModel](#componentsschemasacquiringcreatesubscriptionresponsemodel) |  |
| AcquiringCreateSubscriptionWithReceiptRequestDataModel | [#/components/schemas/AcquiringCreateSubscriptionWithReceiptRequestDataModel](#componentsschemasacquiringcreatesubscriptionwithreceiptrequestdatamodel) |  |
| AcquiringCreateSubscriptionWithReceiptRequestModel | [#/components/schemas/AcquiringCreateSubscriptionWithReceiptRequestModel](#componentsschemasacquiringcreatesubscriptionwithreceiptrequestmodel) |  |
| AcquiringCreateSubscriptionWithReceiptResponseDataModel | [#/components/schemas/AcquiringCreateSubscriptionWithReceiptResponseDataModel](#componentsschemasacquiringcreatesubscriptionwithreceiptresponsedatamodel) |  |
| AcquiringCreateSubscriptionWithReceiptResponseModel | [#/components/schemas/AcquiringCreateSubscriptionWithReceiptResponseModel](#componentsschemasacquiringcreatesubscriptionwithreceiptresponsemodel) |  |
| AcquiringCreatedStatus | [#/components/schemas/AcquiringCreatedStatus](#componentsschemasacquiringcreatedstatus) |  |
| AcquiringGetPaymentOperationInfoResponseDataModel | [#/components/schemas/AcquiringGetPaymentOperationInfoResponseDataModel](#componentsschemasacquiringgetpaymentoperationinforesponsedatamodel) |  |
| AcquiringGetPaymentOperationListItemModel | [#/components/schemas/AcquiringGetPaymentOperationListItemModel](#componentsschemasacquiringgetpaymentoperationlistitemmodel) |  |
| AcquiringGetPaymentOperationListResponseDataModel | [#/components/schemas/AcquiringGetPaymentOperationListResponseDataModel](#componentsschemasacquiringgetpaymentoperationlistresponsedatamodel) |  |
| AcquiringGetPaymentOperationListResponseModel | [#/components/schemas/AcquiringGetPaymentOperationListResponseModel](#componentsschemasacquiringgetpaymentoperationlistresponsemodel) |  |
| AcquiringGetSubscriptionStatusResponseDataModel | [#/components/schemas/AcquiringGetSubscriptionStatusResponseDataModel](#componentsschemasacquiringgetsubscriptionstatusresponsedatamodel) |  |
| AcquiringGetSubscriptionStatusResponseModel | [#/components/schemas/AcquiringGetSubscriptionStatusResponseModel](#componentsschemasacquiringgetsubscriptionstatusresponsemodel) |  |
| AcquiringPaymentMode | [#/components/schemas/AcquiringPaymentMode](#componentsschemasacquiringpaymentmode) |  |
| AcquiringPaymentOperationRefundModel | [#/components/schemas/AcquiringPaymentOperationRefundModel](#componentsschemasacquiringpaymentoperationrefundmodel) |  |
| AcquiringPaymentOperationRefundResponseModel | [#/components/schemas/AcquiringPaymentOperationRefundResponseModel](#componentsschemasacquiringpaymentoperationrefundresponsemodel) |  |
| AcquiringPaymentOrderRefundRequestDataModel | [#/components/schemas/AcquiringPaymentOrderRefundRequestDataModel](#componentsschemasacquiringpaymentorderrefundrequestdatamodel) |  |
| AcquiringPaymentOrderRefundRequestModel | [#/components/schemas/AcquiringPaymentOrderRefundRequestModel](#componentsschemasacquiringpaymentorderrefundrequestmodel) |  |
| AcquiringPaymentStatus | [#/components/schemas/AcquiringPaymentStatus](#componentsschemasacquiringpaymentstatus) |  |
| AcquiringRetailerListModel | [#/components/schemas/AcquiringRetailerListModel](#componentsschemasacquiringretailerlistmodel) |  |
| AcquiringRetailerListResponseModel | [#/components/schemas/AcquiringRetailerListResponseModel](#componentsschemasacquiringretailerlistresponsemodel) |  |
| AcquiringRetailerModel | [#/components/schemas/AcquiringRetailerModel](#componentsschemasacquiringretailermodel) |  |
| AcquiringRetailerStatus | [#/components/schemas/AcquiringRetailerStatus](#componentsschemasacquiringretailerstatus) |  |
| AcquiringSetSubscriptionStatusRequestDataModel | [#/components/schemas/AcquiringSetSubscriptionStatusRequestDataModel](#componentsschemasacquiringsetsubscriptionstatusrequestdatamodel) |  |
| AcquiringSetSubscriptionStatusRequestModel | [#/components/schemas/AcquiringSetSubscriptionStatusRequestModel](#componentsschemasacquiringsetsubscriptionstatusrequestmodel) |  |
| AcquiringSetSubscriptionStatusResponseDataModel | [#/components/schemas/AcquiringSetSubscriptionStatusResponseDataModel](#componentsschemasacquiringsetsubscriptionstatusresponsedatamodel) |  |
| AcquiringSubscriptionInputOptions | [#/components/schemas/AcquiringSubscriptionInputOptions](#componentsschemasacquiringsubscriptioninputoptions) |  |
| AcquiringSubscriptionListItemModel | [#/components/schemas/AcquiringSubscriptionListItemModel](#componentsschemasacquiringsubscriptionlistitemmodel) |  |
| AcquiringSubscriptionListResponseDataModel | [#/components/schemas/AcquiringSubscriptionListResponseDataModel](#componentsschemasacquiringsubscriptionlistresponsedatamodel) |  |
| AcquiringSubscriptionListResponseModel | [#/components/schemas/AcquiringSubscriptionListResponseModel](#componentsschemasacquiringsubscriptionlistresponsemodel) |  |
| AcquiringSubscriptionOutputOptions | [#/components/schemas/AcquiringSubscriptionOutputOptions](#componentsschemasacquiringsubscriptionoutputoptions) |  |
| AcquiringSubscriptionPeriodInput | [#/components/schemas/AcquiringSubscriptionPeriodInput](#componentsschemasacquiringsubscriptionperiodinput) |  |
| AcquiringSubscriptionPeriodOutput | [#/components/schemas/AcquiringSubscriptionPeriodOutput](#componentsschemasacquiringsubscriptionperiodoutput) |  |
| AcquiringSubscriptionStatus | [#/components/schemas/AcquiringSubscriptionStatus](#componentsschemasacquiringsubscriptionstatus) |  |
| AcquiringSubscriptionStatusInput | [#/components/schemas/AcquiringSubscriptionStatusInput](#componentsschemasacquiringsubscriptionstatusinput) |  |
| ActModel | [#/components/schemas/ActModel](#componentsschemasactmodel) |  |
| ActivateCashboxQrCodeRequestDataModel | [#/components/schemas/ActivateCashboxQrCodeRequestDataModel](#componentsschemasactivatecashboxqrcoderequestdatamodel) |  |
| ActivateCashboxQrCodeRequestModel | [#/components/schemas/ActivateCashboxQrCodeRequestModel](#componentsschemasactivatecashboxqrcoderequestmodel) |  |
| ActivateCashboxQrCodeResponseDataModel | [#/components/schemas/ActivateCashboxQrCodeResponseDataModel](#componentsschemasactivatecashboxqrcoderesponsedatamodel) |  |
| ActivateCashboxQrCodeResponseModel | [#/components/schemas/ActivateCashboxQrCodeResponseModel](#componentsschemasactivatecashboxqrcoderesponsemodel) |  |
| ApiVersion | [#/components/schemas/ApiVersion](#componentsschemasapiversion) |  |
| B2BQRCodeModelResponseModel | [#/components/schemas/B2BQRCodeModelResponseModel](#componentsschemasb2bqrcodemodelresponsemodel) |  |
| B2BQrCode | [#/components/schemas/B2BQrCode](#componentsschemasb2bqrcode) |  |
| B2BQrCodeResponseModel | [#/components/schemas/B2BQrCodeResponseModel](#componentsschemasb2bqrcoderesponsemodel) |  |
| BadRequestError | [#/components/schemas/BadRequestError](#componentsschemasbadrequesterror) |  |
| BadRequestErrorResponse | [#/components/schemas/BadRequestErrorResponse](#componentsschemasbadrequesterrorresponse) |  |
| BalanceAmountModel | [#/components/schemas/BalanceAmountModel](#componentsschemasbalanceamountmodel) |  |
| BalanceListModel | [#/components/schemas/BalanceListModel](#componentsschemasbalancelistmodel) |  |
| BalanceListResponseModel | [#/components/schemas/BalanceListResponseModel](#componentsschemasbalancelistresponsemodel) | Метод получения баланса |
| BalanceModel | [#/components/schemas/BalanceModel](#componentsschemasbalancemodel) |  |
| BooleanResponse | [#/components/schemas/BooleanResponse](#componentsschemasbooleanresponse) |  |
| BooleanResponseModel | [#/components/schemas/BooleanResponseModel](#componentsschemasbooleanresponsemodel) |  |
| CapabilitiesEnum | [#/components/schemas/CapabilitiesEnum](#componentsschemascapabilitiesenum) | `001` - только QR Static
`010` - только QR Dynamic
`011` - QR Static и QR Dynamic
`100` - Только QR Subscription
`101` - QR Subscription и QR Static
`110` - QR Subscription и QR Dynamic
`111` - QR Static, QR Dynamic и QR Subscription |
| CardTransactionAmountModel | [#/components/schemas/CardTransactionAmountModel](#componentsschemascardtransactionamountmodel) |  |
| CardTransactionListModel | [#/components/schemas/CardTransactionListModel](#componentsschemascardtransactionlistmodel) |  |
| CardTransactionModel | [#/components/schemas/CardTransactionModel](#componentsschemascardtransactionmodel) |  |
| CardTransactionTerminalData | [#/components/schemas/CardTransactionTerminalData](#componentsschemascardtransactionterminaldata) |  |
| CardTransactionsListResponseModel | [#/components/schemas/CardTransactionsListResponseModel](#componentsschemascardtransactionslistresponsemodel) |  |
| CashAccountInfoModel | [#/components/schemas/CashAccountInfoModel](#componentsschemascashaccountinfomodel) |  |
| CashboxQrCodeOutputCommission | [#/components/schemas/CashboxQrCodeOutputCommission](#componentsschemascashboxqrcodeoutputcommission) |  |
| CashboxQrCodeResponseModel | [#/components/schemas/CashboxQrCodeResponseModel](#componentsschemascashboxqrcoderesponsemodel) |  |
| ChangeCashboxQRCodeAccountRequestDataModel | [#/components/schemas/ChangeCashboxQRCodeAccountRequestDataModel](#componentsschemaschangecashboxqrcodeaccountrequestdatamodel) |  |
| ChangeCashboxQRCodeAccountRequestModel | [#/components/schemas/ChangeCashboxQRCodeAccountRequestModel](#componentsschemaschangecashboxqrcodeaccountrequestmodel) |  |
| ChangeCashboxQRCodeAccountResponseDataModel | [#/components/schemas/ChangeCashboxQRCodeAccountResponseDataModel](#componentsschemaschangecashboxqrcodeaccountresponsedatamodel) |  |
| ChangeCashboxQRCodeAccountResponseModel | [#/components/schemas/ChangeCashboxQRCodeAccountResponseModel](#componentsschemaschangecashboxqrcodeaccountresponsemodel) |  |
| ClosingDocumentCreateRequestDataModel | [#/components/schemas/ClosingDocumentCreateRequestDataModel](#componentsschemasclosingdocumentcreaterequestdatamodel) |  |
| ClosingDocumentCreateRequestModel | [#/components/schemas/ClosingDocumentCreateRequestModel](#componentsschemasclosingdocumentcreaterequestmodel) |  |
| ConsentCreateRequest | [#/components/schemas/ConsentCreateRequest](#componentsschemasconsentcreaterequest) |  |
| ConsentCreateRequestModel | [#/components/schemas/ConsentCreateRequestModel](#componentsschemasconsentcreaterequestmodel) |  |
| ConsentListModel | [#/components/schemas/ConsentListModel](#componentsschemasconsentlistmodel) |  |
| ConsentListResponse | [#/components/schemas/ConsentListResponse](#componentsschemasconsentlistresponse) |  |
| ConsentModel | [#/components/schemas/ConsentModel](#componentsschemasconsentmodel) |  |
| ConsentResponse | [#/components/schemas/ConsentResponse](#componentsschemasconsentresponse) |  |
| ConsentResponseModel | [#/components/schemas/ConsentResponseModel](#componentsschemasconsentresponsemodel) |  |
| ConsentStatusEnum | [#/components/schemas/ConsentStatusEnum](#componentsschemasconsentstatusenum) |  |
| ConsentTypeEnum | [#/components/schemas/ConsentTypeEnum](#componentsschemasconsenttypeenum) |  |
| ContentAct | [#/components/schemas/ContentAct](#componentsschemascontentact) |  |
| ContentInvoice | [#/components/schemas/ContentInvoice](#componentsschemascontentinvoice) |  |
| ContentInvoicef | [#/components/schemas/ContentInvoicef](#componentsschemascontentinvoicef) |  |
| ContentPackingList | [#/components/schemas/ContentPackingList](#componentsschemascontentpackinglist) |  |
| ContractorBankInfoModel | [#/components/schemas/ContractorBankInfoModel](#componentsschemascontractorbankinfomodel) |  |
| ContractorInfoModel | [#/components/schemas/ContractorInfoModel](#componentsschemascontractorinfomodel) |  |
| CounterpartTypeEnum | [#/components/schemas/CounterpartTypeEnum](#componentsschemascounterparttypeenum) |  |
| CustomerCodeAndBankCode | [#/components/schemas/CustomerCodeAndBankCode](#componentsschemascustomercodeandbankcode) |  |
| CustomerCodeAndBankCodeRequest | [#/components/schemas/CustomerCodeAndBankCodeRequest](#componentsschemascustomercodeandbankcoderequest) |  |
| CustomerInfoResponseV3 | [#/components/schemas/CustomerInfoResponseV3](#componentsschemascustomerinforesponsev3) |  |
| CustomerListModel | [#/components/schemas/CustomerListModel](#componentsschemascustomerlistmodel) |  |
| CustomerListResponseModel | [#/components/schemas/CustomerListResponseModel](#componentsschemascustomerlistresponsemodel) | Метод получения списка доступных клиентов |
| CustomerModel | [#/components/schemas/CustomerModel](#componentsschemascustomermodel) |  |
| CustomerResponseModel | [#/components/schemas/CustomerResponseModel](#componentsschemascustomerresponsemodel) | Метод получения информации по клиенту |
| DeactivateCashboxQrCodeResponseDataModel | [#/components/schemas/DeactivateCashboxQrCodeResponseDataModel](#componentsschemasdeactivatecashboxqrcoderesponsedatamodel) |  |
| DocumentCreateResponse | [#/components/schemas/DocumentCreateResponse](#componentsschemasdocumentcreateresponse) |  |
| DocumentCreateResponseModel | [#/components/schemas/DocumentCreateResponseModel](#componentsschemasdocumentcreateresponsemodel) |  |
| ErrorModel | [#/components/schemas/ErrorModel](#componentsschemaserrormodel) |  |
| ErrorResponse | [#/components/schemas/ErrorResponse](#componentsschemaserrorresponse) |  |
| ExternalAccountStatusEnum | [#/components/schemas/ExternalAccountStatusEnum](#componentsschemasexternalaccountstatusenum) |  |
| ExternalAccountSubTypeEnum | [#/components/schemas/ExternalAccountSubTypeEnum](#componentsschemasexternalaccountsubtypeenum) |  |
| ExternalAcquiringPaymentTypeEnum | [#/components/schemas/ExternalAcquiringPaymentTypeEnum](#componentsschemasexternalacquiringpaymenttypeenum) |  |
| ExternalBalanceStaticTypeEnum | [#/components/schemas/ExternalBalanceStaticTypeEnum](#componentsschemasexternalbalancestatictypeenum) | **Описание типов балансов**

  - `OpeningAvailable` - Начальный остаток
  - `ClosingAvailable` - Доступный баланс
  - `Expected` - Сумма заблокированных средств
  - `OverdraftAvailable` - Доступный лимит по овердрафту |
| ExternalBalanceTypeEnum | [#/components/schemas/ExternalBalanceTypeEnum](#componentsschemasexternalbalancetypeenum) |  |
| ExternalConsentTypeEnum | [#/components/schemas/ExternalConsentTypeEnum](#componentsschemasexternalconsenttypeenum) |  |
| ExternalCreditDebitIndicatorEnum | [#/components/schemas/ExternalCreditDebitIndicatorEnum](#componentsschemasexternalcreditdebitindicatorenum) |  |
| ExternalTransactionStatusEnum | [#/components/schemas/ExternalTransactionStatusEnum](#componentsschemasexternaltransactionstatusenum) |  |
| ExternalTransationTypeEnum | [#/components/schemas/ExternalTransationTypeEnum](#componentsschemasexternaltransationtypeenum) |  |
| ExternalTypeEnum | [#/components/schemas/ExternalTypeEnum](#componentsschemasexternaltypeenum) |  |
| FinancialInstitutionIdentificationEnum | [#/components/schemas/FinancialInstitutionIdentificationEnum](#componentsschemasfinancialinstitutionidentificationenum) |  |
| ForbiddenError | [#/components/schemas/ForbiddenError](#componentsschemasforbiddenerror) |  |
| ForbiddenErrorResponse | [#/components/schemas/ForbiddenErrorResponse](#componentsschemasforbiddenerrorresponse) |  |
| GetCashboxQRCodeListResponseDataModel | [#/components/schemas/GetCashboxQRCodeListResponseDataModel](#componentsschemasgetcashboxqrcodelistresponsedatamodel) |  |
| GetCashboxQRCodeListResponseItemModel | [#/components/schemas/GetCashboxQRCodeListResponseItemModel](#componentsschemasgetcashboxqrcodelistresponseitemmodel) |  |
| GetCashboxQRCodeListResponseModel | [#/components/schemas/GetCashboxQRCodeListResponseModel](#componentsschemasgetcashboxqrcodelistresponsemodel) |  |
| GetCashboxQRCodeRequestDataModel | [#/components/schemas/GetCashboxQRCodeRequestDataModel](#componentsschemasgetcashboxqrcoderequestdatamodel) |  |
| GetCashboxQRCodeRequestModel | [#/components/schemas/GetCashboxQRCodeRequestModel](#componentsschemasgetcashboxqrcoderequestmodel) |  |
| GetCashboxQrCodeOperationInfoResponseDataModel | [#/components/schemas/GetCashboxQrCodeOperationInfoResponseDataModel](#componentsschemasgetcashboxqrcodeoperationinforesponsedatamodel) |  |
| GetCashboxQrCodeOperationInfoResponseModel | [#/components/schemas/GetCashboxQrCodeOperationInfoResponseModel](#componentsschemasgetcashboxqrcodeoperationinforesponsemodel) |  |
| GetCashboxQrCodeResponseDataModel | [#/components/schemas/GetCashboxQrCodeResponseDataModel](#componentsschemasgetcashboxqrcoderesponsedatamodel) |  |
| GetCashboxQrCodeStatusResponseDataModel | [#/components/schemas/GetCashboxQrCodeStatusResponseDataModel](#componentsschemasgetcashboxqrcodestatusresponsedatamodel) |  |
| GetCashboxQrCodeStatusResponseModel | [#/components/schemas/GetCashboxQrCodeStatusResponseModel](#componentsschemasgetcashboxqrcodestatusresponsemodel) |  |
| GetCustomerInfoResponseModelV3 | [#/components/schemas/GetCustomerInfoResponseModelV3](#componentsschemasgetcustomerinforesponsemodelv3) |  |
| InitStatementModel | [#/components/schemas/InitStatementModel](#componentsschemasinitstatementmodel) |  |
| InvoiceCreateRequestDataModel | [#/components/schemas/InvoiceCreateRequestDataModel](#componentsschemasinvoicecreaterequestdatamodel) |  |
| InvoiceCreateRequestModel | [#/components/schemas/InvoiceCreateRequestModel](#componentsschemasinvoicecreaterequestmodel) |  |
| InvoiceModel | [#/components/schemas/InvoiceModel](#componentsschemasinvoicemodel) |  |
| InvoicePaymentStatusResponse | [#/components/schemas/InvoicePaymentStatusResponse](#componentsschemasinvoicepaymentstatusresponse) |  |
| InvoicePaymentStatusResponseModel | [#/components/schemas/InvoicePaymentStatusResponseModel](#componentsschemasinvoicepaymentstatusresponsemodel) |  |
| InvoicefModel | [#/components/schemas/InvoicefModel](#componentsschemasinvoicefmodel) |  |
| LegalEntity | [#/components/schemas/LegalEntity](#componentsschemaslegalentity) |  |
| LegalEntityResponseModel | [#/components/schemas/LegalEntityResponseModel](#componentsschemaslegalentityresponsemodel) |  |
| LinkModel | [#/components/schemas/LinkModel](#componentsschemaslinkmodel) |  |
| Measure | [#/components/schemas/Measure](#componentsschemasmeasure) |  |
| MediaTypeEnum | [#/components/schemas/MediaTypeEnum](#componentsschemasmediatypeenum) |  |
| Merchant | [#/components/schemas/Merchant](#componentsschemasmerchant) |  |
| MerchantId | [#/components/schemas/MerchantId](#componentsschemasmerchantid) |  |
| MerchantIdResponseModel | [#/components/schemas/MerchantIdResponseModel](#componentsschemasmerchantidresponsemodel) |  |
| MerchantListResponse | [#/components/schemas/MerchantListResponse](#componentsschemasmerchantlistresponse) |  |
| MerchantListResponseModel | [#/components/schemas/MerchantListResponseModel](#componentsschemasmerchantlistresponsemodel) |  |
| MerchantResponseModel | [#/components/schemas/MerchantResponseModel](#componentsschemasmerchantresponsemodel) |  |
| MetaModel | [#/components/schemas/MetaModel](#componentsschemasmetamodel) |  |
| NdsKindEnum | [#/components/schemas/NdsKindEnum](#componentsschemasndskindenum) |  |
| NotFoundError | [#/components/schemas/NotFoundError](#componentsschemasnotfounderror) |  |
| NotFoundErrorResponse | [#/components/schemas/NotFoundErrorResponse](#componentsschemasnotfounderrorresponse) |  |
| OptionalLegalId | [#/components/schemas/OptionalLegalId](#componentsschemasoptionallegalid) |  |
| OrderModel | [#/components/schemas/OrderModel](#componentsschemasordermodel) |  |
| OrderType | [#/components/schemas/OrderType](#componentsschemasordertype) |  |
| PackingListModel | [#/components/schemas/PackingListModel](#componentsschemaspackinglistmodel) |  |
| PaginatedLinkModel | [#/components/schemas/PaginatedLinkModel](#componentsschemaspaginatedlinkmodel) |  |
| PaymentForSignListItemModel | [#/components/schemas/PaymentForSignListItemModel](#componentsschemaspaymentforsignlistitemmodel) |  |
| PaymentForSignListResponseDataModel | [#/components/schemas/PaymentForSignListResponseDataModel](#componentsschemaspaymentforsignlistresponsedatamodel) |  |
| PaymentForSignListResponseModel | [#/components/schemas/PaymentForSignListResponseModel](#componentsschemaspaymentforsignlistresponsemodel) |  |
| PaymentForSignRequestDataModel | [#/components/schemas/PaymentForSignRequestDataModel](#componentsschemaspaymentforsignrequestdatamodel) |  |
| PaymentForSignRequestModel | [#/components/schemas/PaymentForSignRequestModel](#componentsschemaspaymentforsignrequestmodel) |  |
| PaymentForSignResponseDataModel | [#/components/schemas/PaymentForSignResponseDataModel](#componentsschemaspaymentforsignresponsedatamodel) |  |
| PaymentForSignResponseModel | [#/components/schemas/PaymentForSignResponseModel](#componentsschemaspaymentforsignresponsemodel) |  |
| PaymentMethod | [#/components/schemas/PaymentMethod](#componentsschemaspaymentmethod) |  |
| PaymentObject | [#/components/schemas/PaymentObject](#componentsschemaspaymentobject) |  |
| PaymentOrderRequestDataModel | [#/components/schemas/PaymentOrderRequestDataModel](#componentsschemaspaymentorderrequestdatamodel) |  |
| PaymentOrderRequestModel | [#/components/schemas/PaymentOrderRequestModel](#componentsschemaspaymentorderrequestmodel) |  |
| PaymentOrderResponseDataModel | [#/components/schemas/PaymentOrderResponseDataModel](#componentsschemaspaymentorderresponsedatamodel) |  |
| PaymentOrderResponseModel | [#/components/schemas/PaymentOrderResponseModel](#componentsschemaspaymentorderresponsemodel) |  |
| PaymentOrderStatusEnum | [#/components/schemas/PaymentOrderStatusEnum](#componentsschemaspaymentorderstatusenum) | **Описание статусов платежа**

  - `Initiated` - Все необходимые реквизиты для платежа получены, платёж готов к проверке на возможность проведения
  - `Wait For Owner Requisites` - Часть реквизитов для платежа получена, кроме реквизитов плательщика
  - `NotAllowed` - Платёж нельзя провести: либо у пользователя нет прав для подписи, либо платёж заблокирован
  комплаенсом
  - `Allowed` - Платёж готов к подписанию, все проверки пройдены
  - `WaitingForSign` - Платёж ждёт подписи
  - `WaitingForCreate` - Платёж подписан, ждёт создания внутри систем банка
  - `Created` - Платёж создан
  - `Paid` - Платёж оплачен
  - `Canceled` - Платёж отменен
  - `Rejected` - Платёж отменён |
| PaymentStatusResponseDataModel | [#/components/schemas/PaymentStatusResponseDataModel](#componentsschemaspaymentstatusresponsedatamodel) |  |
| PaymentStatusResponseModel | [#/components/schemas/PaymentStatusResponseModel](#componentsschemaspaymentstatusresponsemodel) |  |
| PositionModel | [#/components/schemas/PositionModel](#componentsschemaspositionmodel) |  |
| QRCodeListResponse | [#/components/schemas/QRCodeListResponse](#componentsschemasqrcodelistresponse) |  |
| QRCodeListResponseModel | [#/components/schemas/QRCodeListResponseModel](#componentsschemasqrcodelistresponsemodel) |  |
| QRCodeModelResponseModel | [#/components/schemas/QRCodeModelResponseModel](#componentsschemasqrcodemodelresponsemodel) |  |
| QRCodePaymentStatusExternal | [#/components/schemas/QRCodePaymentStatusExternal](#componentsschemasqrcodepaymentstatusexternal) | **Описание возможных статусов платежа**

  - `NotStarted` - операции по QR-коду не существует
  - `Received` - операция в обработке
  - `InProgress` - операция в обработке
  - `Accepted` - операция завершена успешно
  - `Rejected` - операция отклонена |
| QRCodePaymentStatusListResponse | [#/components/schemas/QRCodePaymentStatusListResponse](#componentsschemasqrcodepaymentstatuslistresponse) |  |
| QRCodePaymentStatusListResponseModel | [#/components/schemas/QRCodePaymentStatusListResponseModel](#componentsschemasqrcodepaymentstatuslistresponsemodel) |  |
| QRCodeRequestParams | [#/components/schemas/QRCodeRequestParams](#componentsschemasqrcoderequestparams) |  |
| QRCodeResponseModel | [#/components/schemas/QRCodeResponseModel](#componentsschemasqrcoderesponsemodel) |  |
| QrCode | [#/components/schemas/QrCode](#componentsschemasqrcode) |  |
| QrCodeContent | [#/components/schemas/QrCodeContent](#componentsschemasqrcodecontent) |  |
| QrCodeImageParams | [#/components/schemas/QrCodeImageParams](#componentsschemasqrcodeimageparams) |  |
| QrCodePaymentStatus | [#/components/schemas/QrCodePaymentStatus](#componentsschemasqrcodepaymentstatus) |  |
| QrTypeEnum | [#/components/schemas/QrTypeEnum](#componentsschemasqrtypeenum) | 01 - QR-Static (QR наклейка)
02 - QR-Dynamic (QR на кассе) |
| ReceiptClientModel | [#/components/schemas/ReceiptClientModel](#componentsschemasreceiptclientmodel) |  |
| ReceiptItemModel-Input | [#/components/schemas/ReceiptItemModel-Input](#componentsschemasreceiptitemmodel-input) |  |
| ReceiptItemModel-Output | [#/components/schemas/ReceiptItemModel-Output](#componentsschemasreceiptitemmodel-output) |  |
| ReceiptItemResponseModel | [#/components/schemas/ReceiptItemResponseModel](#componentsschemasreceiptitemresponsemodel) |  |
| RegisterB2BQRCode | [#/components/schemas/RegisterB2BQRCode](#componentsschemasregisterb2bqrcode) |  |
| RegisterB2BQRCodeRequest | [#/components/schemas/RegisterB2BQRCodeRequest](#componentsschemasregisterb2bqrcoderequest) |  |
| RegisterCashboxQrCodeRequestDataModel | [#/components/schemas/RegisterCashboxQrCodeRequestDataModel](#componentsschemasregistercashboxqrcoderequestdatamodel) |  |
| RegisterCashboxQrCodeRequestModel | [#/components/schemas/RegisterCashboxQrCodeRequestModel](#componentsschemasregistercashboxqrcoderequestmodel) |  |
| RegisterCashboxQrCodeResponseDataModel | [#/components/schemas/RegisterCashboxQrCodeResponseDataModel](#componentsschemasregistercashboxqrcoderesponsedatamodel) |  |
| RegisterCashboxQrCodeResponseModel | [#/components/schemas/RegisterCashboxQrCodeResponseModel](#componentsschemasregistercashboxqrcoderesponsemodel) |  |
| RegisterLegalEntityResponseModel | [#/components/schemas/RegisterLegalEntityResponseModel](#componentsschemasregisterlegalentityresponsemodel) |  |
| RegisterMerchant | [#/components/schemas/RegisterMerchant](#componentsschemasregistermerchant) |  |
| RegisterMerchantRequest | [#/components/schemas/RegisterMerchantRequest](#componentsschemasregistermerchantrequest) |  |
| RegisterQRCode | [#/components/schemas/RegisterQRCode](#componentsschemasregisterqrcode) |  |
| RegisterQRCodeRequest | [#/components/schemas/RegisterQRCodeRequest](#componentsschemasregisterqrcoderequest) |  |
| RegisteredB2BQrCode | [#/components/schemas/RegisteredB2BQrCode](#componentsschemasregisteredb2bqrcode) |  |
| RegisteredQrCode | [#/components/schemas/RegisteredQrCode](#componentsschemasregisteredqrcode) |  |
| SBPCashboxOperationQrCodeStatus | [#/components/schemas/SBPCashboxOperationQrCodeStatus](#componentsschemassbpcashboxoperationqrcodestatus) |  |
| SBPCashboxQrCodeStatus | [#/components/schemas/SBPCashboxQrCodeStatus](#componentsschemassbpcashboxqrcodestatus) |  |
| SBPCashboxTrxStatus | [#/components/schemas/SBPCashboxTrxStatus](#componentsschemassbpcashboxtrxstatus) | * ACWP - Операция завершена успешно
* RJCT - Операция отклонена
* RCVD - Операция в обработке
* NTST - Операции по QR-коду не существует |
| SBPPayment | [#/components/schemas/SBPPayment](#componentsschemassbppayment) |  |
| SBPPaymentList | [#/components/schemas/SBPPaymentList](#componentsschemassbppaymentlist) |  |
| SBPPaymentStatus | [#/components/schemas/SBPPaymentStatus](#componentsschemassbppaymentstatus) |  |
| SBPPaymentsResponse | [#/components/schemas/SBPPaymentsResponse](#componentsschemassbppaymentsresponse) |  |
| SBPRefund | [#/components/schemas/SBPRefund](#componentsschemassbprefund) |  |
| SBPRefundRequest | [#/components/schemas/SBPRefundRequest](#componentsschemassbprefundrequest) |  |
| SBPRefundRequestResponse | [#/components/schemas/SBPRefundRequestResponse](#componentsschemassbprefundrequestresponse) |  |
| SBPRefundRequestResponseModel | [#/components/schemas/SBPRefundRequestResponseModel](#componentsschemassbprefundrequestresponsemodel) |  |
| SBPRefundStatus | [#/components/schemas/SBPRefundStatus](#componentsschemassbprefundstatus) |  |
| SBPRefundStatusModel | [#/components/schemas/SBPRefundStatusModel](#componentsschemassbprefundstatusmodel) |  |
| SecondSideModel | [#/components/schemas/SecondSideModel](#componentsschemassecondsidemodel) |  |
| SendDocumentToEmailRequestDataModel | [#/components/schemas/SendDocumentToEmailRequestDataModel](#componentsschemassenddocumenttoemailrequestdatamodel) |  |
| SendDocumentToEmailRequestModel | [#/components/schemas/SendDocumentToEmailRequestModel](#componentsschemassenddocumenttoemailrequestmodel) |  |
| ShipmentDocumentModel | [#/components/schemas/ShipmentDocumentModel](#componentsschemasshipmentdocumentmodel) |  |
| StatementInitReqModel | [#/components/schemas/StatementInitReqModel](#componentsschemasstatementinitreqmodel) |  |
| StatementInitRequestDataModel | [#/components/schemas/StatementInitRequestDataModel](#componentsschemasstatementinitrequestdatamodel) | Метод создания выписки по счету |
| StatementInitRequestModel | [#/components/schemas/StatementInitRequestModel](#componentsschemasstatementinitrequestmodel) |  |
| StatementInitResponseDataModel | [#/components/schemas/StatementInitResponseDataModel](#componentsschemasstatementinitresponsedatamodel) | Метод создания выписки по счету |
| StatementInitResponseModel | [#/components/schemas/StatementInitResponseModel](#componentsschemasstatementinitresponsemodel) |  |
| StatementListModel | [#/components/schemas/StatementListModel](#componentsschemasstatementlistmodel) |  |
| StatementModel | [#/components/schemas/StatementModel](#componentsschemasstatementmodel) |  |
| StatementResponseDataModel | [#/components/schemas/StatementResponseDataModel](#componentsschemasstatementresponsedatamodel) | Метод получения выписки по счету |
| StatementStatus | [#/components/schemas/StatementStatus](#componentsschemasstatementstatus) |  |
| Status | [#/components/schemas/Status](#componentsschemasstatus) |  |
| StatusEnum | [#/components/schemas/StatusEnum](#componentsschemasstatusenum) |  |
| StatusRequest | [#/components/schemas/StatusRequest](#componentsschemasstatusrequest) |  |
| SupplierModel | [#/components/schemas/SupplierModel](#componentsschemassuppliermodel) |  |
| TaxFieldsModel | [#/components/schemas/TaxFieldsModel](#componentsschemastaxfieldsmodel) |  |
| TaxSystemCodeInput | [#/components/schemas/TaxSystemCodeInput](#componentsschemastaxsystemcodeinput) |  |
| TaxSystemCodeOutput | [#/components/schemas/TaxSystemCodeOutput](#componentsschemastaxsystemcodeoutput) | Перечисление для выдачи результатов из openapi. |
| TransactionAmountModel | [#/components/schemas/TransactionAmountModel](#componentsschemastransactionamountmodel) |  |
| TransactionModel | [#/components/schemas/TransactionModel](#componentsschemastransactionmodel) |  |
| UnauthorizedError | [#/components/schemas/UnauthorizedError](#componentsschemasunauthorizederror) |  |
| UnauthorizedErrorResponse | [#/components/schemas/UnauthorizedErrorResponse](#componentsschemasunauthorizederrorresponse) |  |
| UnitCodeEnum | [#/components/schemas/UnitCodeEnum](#componentsschemasunitcodeenum) |  |
| VatType | [#/components/schemas/VatType](#componentsschemasvattype) |  |
| Webhook | [#/components/schemas/Webhook](#componentsschemaswebhook) |  |
| WebhookEditRequest | [#/components/schemas/WebhookEditRequest](#componentsschemaswebhookeditrequest) |  |
| WebhookResponseModel | [#/components/schemas/WebhookResponseModel](#componentsschemaswebhookresponsemodel) |  |
| WebhookTestSendRequest | [#/components/schemas/WebhookTestSendRequest](#componentsschemaswebhooktestsendrequest) |  |
| WebhookTypeEnum | [#/components/schemas/WebhookTypeEnum](#componentsschemaswebhooktypeenum) |  |
| application__invoice__models__enums__PaymentStatusEnum | [#/components/schemas/application__invoice__models__enums__PaymentStatusEnum](#componentsschemasapplication__invoice__models__enums__paymentstatusenum) |  |
| application__open_banking__models__external_models__AccountListResponseModel | [#/components/schemas/application__open_banking__models__external_models__AccountListResponseModel](#componentsschemasapplication__open_banking__models__external_models__accountlistresponsemodel) | Метод получения списка доступных счетов |
| application__sbp__models__enums__PaymentStatusEnum | [#/components/schemas/application__sbp__models__enums__PaymentStatusEnum](#componentsschemasapplication__sbp__models__enums__paymentstatusenum) | **Описание возможных статусов платежа**
  - `Confirming` - операция в процессе подтверждения ОПКЦ СБП
  - `Confirmed` - операция подтверждена
  - `Initiated` - операция отправлена на обработку
  - `Accepting` - операция в обработке ОПКЦ СБП
  - `Accepted` - операция успешно завершена
  - `InProgress` - операция в обработке РЦ СБП
  - `Rejected` - операция отклонена
  - `Error` - ошибка выполнения операции
  - `Timeout` - тайм-аут выполнения операции |
| application__sbp__models__response_models__sbp__AccountListResponseModel | [#/components/schemas/application__sbp__models__response_models__sbp__AccountListResponseModel](#componentsschemasapplication__sbp__models__response_models__sbp__accountlistresponsemodel) |  |
| Необходимые разрешения | [#/components/securitySchemes/Необходимые разрешения](#componentssecurityschemesнеобходимые-разрешения) |  |

## Path Details

***

### [GET]/acquiring/{apiVersion}/payments

- Summary  
Get Payment Operation List

- Description  
Метод для получения списка операций  
  
- _CREATED_ - Операция создана  
- _APPROVED_ - Операция одобрена (оплата прошла успешно)  
- _ON-REFUND_ - Операция заблокирована на время выполнения возврата  
- _REFUNDED_ - Осуществлен возврат  
- _EXPIRED_ - Истек срок действия  


- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Уникальный код клиента
customerCode: string
```

```ts
// Начало периода создания операций
fromDate?: string
```

```ts
// Конец периода создания операций
toDate?: string
```

```ts
// Номер страницы
page?: integer //default: 1
```

```ts
// Количество записей на странице
perPage?: integer //default: 1000
```

```ts
status?: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Operation: {
      customerCode: string
      // Перечисление для выдачи результатов из openapi.
      taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
      paymentType?: enum[sbp, card, tinkoff]
      paymentId?: string
      // Используется для возврата при оплате по СБП
      transactionId?: string
      createdAt: string
      paymentMode?: enum[sbp, card, tinkoff][]
      redirectUrl?: string
      failRedirectUrl?: string
      Client: {
        name?: string
        email: string
        phone?: string
      }
      Items: {
        vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
        name: string
        amount: number
        quantity: number
        paymentMethod?: enum[full_payment, full_prepayment]
        paymentObject?: enum[goods, service, work]
        measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
      }[]
      // Отсутствует, если при создании платежа назначение не было указано
      purpose?: string
      amount: number
      status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
      operationId: string
      paymentLink: string
      merchantId?: string
      consumerId?: string
      Order: {
        orderId: string
        type: enum[refund, approval, authorized]
        amount: number
        time: string
      }[]
      Supplier: {
        phone: string
        name: string
        taxCode: string
      }
      preAuthorization?: boolean
      paidAt?: string
    }[]
  }
  Links: {
    self: string
    first?: string
    prev?: string
    next?: string
    last?: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/payments

- Summary  
Create Payment Operation

- Description  
Метод для создания ссылки на оплату

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    paymentMode?: enum[sbp, card, tinkoff][]
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    purpose: string
    status?: enum[CREATED]
    amount: number
    operationId: string
    paymentLink: string
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/acquiring/{apiVersion}/payments/{operationId}

- Summary  
Get Payment Operation Info

- Description  
Метод для получения информации о конкретной операции  
  
- _CREATED_ - Операция создана  
- _APPROVED_ - Операция одобрена (оплата прошла успешно)  
- _ON-REFUND_ - Операция заблокирована на время выполнения возврата  
- _REFUNDED_ - Осуществлен возврат  
- _EXPIRED_ - Истек срок действия  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Operation: {
      customerCode: string
      // Перечисление для выдачи результатов из openapi.
      taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
      paymentType?: enum[sbp, card, tinkoff]
      paymentId?: string
      // Используется для возврата при оплате по СБП
      transactionId?: string
      createdAt: string
      paymentMode?: enum[sbp, card, tinkoff][]
      redirectUrl?: string
      failRedirectUrl?: string
      Client: {
        name?: string
        email: string
        phone?: string
      }
      Items: {
        vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
        name: string
        amount: number
        quantity: number
        paymentMethod?: enum[full_payment, full_prepayment]
        paymentObject?: enum[goods, service, work]
        measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
      }[]
      // Отсутствует, если при создании платежа назначение не было указано
      purpose?: string
      amount: number
      status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
      operationId: string
      paymentLink: string
      merchantId?: string
      consumerId?: string
      Order: {
        orderId: string
        type: enum[refund, approval, authorized]
        amount: number
        time: string
      }[]
      Supplier: {
        phone: string
        name: string
        taxCode: string
      }
      preAuthorization?: boolean
      paidAt?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/payments/{operationId}/capture

- Summary  
Capture Payment

- Description  
Метод для списания средств при двухэтапной оплате

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/payments/{operationId}/refund

- Summary  
Refund Payment Operation

- Description  
Метод для возврата платежей, созданных через платёжную ссылку    
Возврат возможен только для платежа со статусом _APPROVED_

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    // Не больше суммы оплаты
    amount: number
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    isRefund: boolean
    operationId: string
    amount: number
    date: string
    orderId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/payments_with_receipt

- Summary  
Create Payment Operation With Receipt

- Description  
Метод для создания ссылки на оплату и отправки чека

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    paymentMode?: enum[sbp, card, tinkoff][]
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    purpose: string
    status?: enum[CREATED]
    amount: number
    operationId: string
    paymentLink: string
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
    customerCode: string
    redirectUrl?: string
    failRedirectUrl?: string
    paymentMode?: enum[sbp, card, tinkoff][]
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/acquiring/{apiVersion}/retailers

- Summary  
Get Retailers

- Description  
Метод для получения информации о ретейлере  
  
- _NEW_ - Ретейлер создан  
- _ADDRESS_DADATA_ - Адрес уточнен  
- _OPEN_ACCOUNT_ - Счёт открыт  
- _TWPG_SENDED_ - Данные мерчанта и терминала отправлены в процессинг  
- _RETAILER_CREATED_ - Мерчант создан в процессинге  
- _TERMINAL_CREATED_ - Терминал создан в процессинге  
- _FILE_SENT_ - файл отправлен в НСПК  
- _REG_ - Зарегистрирован  
- _CLOSE_ - Закрыт  


- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Уникальный код клиента
customerCode: string
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Retailer: {
      status: enum[NEW, ADDRESS_DADATA, OPEN_ACCOUNT, TWPG_SENDED, RETAILER_CREATED, TERMINAL_CREATED, FILE_SENT, REG, CLOSE]
      isActive: boolean
      mcc: string
      rate: number
      name: string
      url?: string
      // Может отсутствовать при значениях поля status: NEW, ADDRESS_DADATA и OPEN_ACCOUNT
      merchantId?: string
      // Будет заполнен при значениях поля status: TERMINAL_CREATED, FILE_SENT, REG
      terminalId?: string
      paymentModes?: enum[sbp, card, tinkoff][]
      cashbox?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/acquiring/{apiVersion}/subscriptions

- Summary  
Get Subscription List

- Description  
Метод для получения всех подписок

- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Уникальный код клиента
customerCode: string
```

```ts
// Номер страницы
page?: integer //default: 1
```

```ts
// Количество записей на странице
perPage?: integer //default: 1000
```

```ts
// Получить платежи по рекуррентным подпискам
recurring?: boolean
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Subscription: {
      customerCode: string
      // Перечисление для выдачи результатов из openapi.
      taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
      paymentId?: string
      // Используется для возврата при оплате по СБП
      transactionId?: string
      createdAt: string
      redirectUrl?: string
      failRedirectUrl?: string
      Client: {
        name?: string
        email: string
        phone?: string
      }
      Items: {
        vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
        name: string
        amount: number
        quantity: number
        paymentMethod?: enum[full_payment, full_prepayment]
        paymentObject?: enum[goods, service, work]
        measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
      }[]
      // Отсутствует, если при создании платежа назначение не было указано
      purpose?: string
      amount: number
      status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
      operationId: string
      paymentLink: string
      merchantId?: string
      consumerId?: string
      Options: {
        trancheCount?: integer
        period?: enum[Day, Month, Year, Quarter]
        daysInPeriod?: integer
      }
      Supplier: {
        phone: string
        name: string
        taxCode: string
      }
      recurring?: boolean
    }[]
  }
  Links: {
    self: string
    first?: string
    prev?: string
    next?: string
    last?: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/subscriptions

- Summary  
Create Subscription

- Description  
Метод для создания подписки по карте

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer //default: 12
      period?: enum[Day, Month, Year]
      daysInPeriod?: integer
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    purpose: string
    amount: number
    status?: enum[CREATED]
    operationId: string
    paymentLink: string
    consumerId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer
      period?: enum[Day, Month, Year, Quarter]
      daysInPeriod?: integer
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/subscriptions/{operationId}/charge

- Summary  
Charge Subscription

- Description  
Метод для списания средств по рекуррентной подписке

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    amount: number
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/acquiring/{apiVersion}/subscriptions/{operationId}/status

- Summary  
Get Subscription Status

- Description  
Метод для получения актуального статуса подписки

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status: enum[Active, Cancelled, Completed, Expired, Failed, PastDue, Preparing, Refused, Rejected, Suspended, Trial]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/subscriptions/{operationId}/status

- Summary  
Set Subscription Status

- Description  
Метод для установки статуса подписки

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    status: enum[Cancelled]
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/acquiring/{apiVersion}/subscriptions_with_receipt

- Summary  
Create Subscription With Receipt

- Description  
Метод для создания подписки по карте и отправке чека

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer //default: 12
      period?: enum[Day, Month, Year]
      daysInPeriod?: integer
    }
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    purpose: string
    amount: number
    status?: enum[CREATED]
    operationId: string
    paymentLink: string
    consumerId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer
      period?: enum[Day, Month, Year, Quarter]
      daysInPeriod?: integer
    }
    customerCode: string
    redirectUrl?: string
    failRedirectUrl?: string
    merchantId?: string
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/consent/{apiVersion}/consents

- Summary  
Get All Consents List

- Description  
Метод для получения списка разрешений  


#### Headers

```ts
// Уникальный код клиента
customer-code?: string
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Consent: {
      status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
      creationDateTime?: string
      statusUpdateDateTime?: string
      permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, ReadSBPData1C, EditSBPData1C, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
      expirationDateTime?: string
      consentId: string
      customerCode?: string
      applicationName?: string
      clientId?: string
      // Показывает истек срок разрешения или нет
      isValid?: boolean
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/consent/{apiVersion}/consents

- Summary  
Create New Consent

- Description  
Метод для создания разрешения  


#### RequestBody

- application/json

```ts
{
  Data: {
    status?: string //default: AwaitingAuthorisation
    creationDateTime?: string
    statusUpdateDateTime?: string
    permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
    expirationDateTime?: string
  }
  Risks: {
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Consent: {
      status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
      creationDateTime?: string
      statusUpdateDateTime?: string
      permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, ReadSBPData1C, EditSBPData1C, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
      expirationDateTime?: string
      consentId: string
      customerCode?: string
      applicationName?: string
      clientId?: string
      // Показывает истек срок разрешения или нет
      isValid?: boolean
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/consent/{apiVersion}/consents/{consentId}/child

- Summary  
Get All Child Consents

- Description  
Метод для получения всех дочерних разрешений  


#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Consent: {
      status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
      creationDateTime?: string
      statusUpdateDateTime?: string
      permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, ReadSBPData1C, EditSBPData1C, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
      expirationDateTime?: string
      consentId: string
      customerCode?: string
      applicationName?: string
      clientId?: string
      // Показывает истек срок разрешения или нет
      isValid?: boolean
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/consent/{apiVersion}/{consentId}

- Summary  
Get Consent Info

- Description  
Метод для получения информации о списке разрешений  


#### Headers

```ts
// Уникальный код клиента
customer-code?: string
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
    creationDateTime?: string
    statusUpdateDateTime?: string
    permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
    expirationDateTime?: string
    consentId: string
    customerCode?: string
    applicationName?: string
    clientId?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/invoice/{apiVersion}/bills

- Summary  
Create Invoice

- Description  
Метод для создания счёта на оплату  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    accountId: string
    customerCode: string
    SecondSide: {
      accountId?: string
      legalAddress?: string
      kpp?: string
      bankName?: string
      bankCorrAccount?: string
      taxCode: string
      type: enum[ip, company]
      secondSideName?: string
    }
    Content: {
      Invoice: {
        Positions: {
          positionName: string
          unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
          ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
          price: number
          quantity: number
          totalAmount: number
          totalNds?: number
        }[]
        date?: string
        totalAmount: number
        totalNds?: number
        number: string
        basedOn?: string
        comment?: string
        paymentExpiryDate?: string
      }
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    documentId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [DELETE]/invoice/{apiVersion}/bills/{customerCode}/{documentId}

- Summary  
Delete Invoice

- Description  
Метод для удаления счёта на оплату  
  
  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/invoice/{apiVersion}/bills/{customerCode}/{documentId}/email

- Summary  
Send Invoice To Email

- Description  
Метод для отправки счёта на почту

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    email: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/invoice/{apiVersion}/bills/{customerCode}/{documentId}/file

- Summary  
Get Invoice

- Description  
Метод для получения файла выставленного счёта

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{}
```

`application/pdf`

```ts
{
  "type": "string",
  "format": "binary"
}
```

- Examples

  - Invoice

```json
{
  "summary": "Invoice file",
  "X-Content-Url": "https://enter.tochka.com/doc/openapi/static/file_examples/invoice_example.pdf"
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/invoice/{apiVersion}/bills/{customerCode}/{documentId}/payment-status

- Summary  
Get Invoice Payment Status

- Description  
Метод для получения статуса счёта

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    paymentStatus: enum[payment_waiting, payment_expired, payment_paid]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/invoice/{apiVersion}/closing-documents

- Summary  
Create Closing Document

- Description  
Метод для создания закрывающего документа

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    accountId: string
    customerCode: string
    SecondSide: {
      accountId?: string
      legalAddress?: string
      kpp?: string
      bankName?: string
      bankCorrAccount?: string
      taxCode: string
      type: enum[ip, company]
      secondSideName?: string
    }
    documentId?: string
    Content: Partial(#/components/schemas/ContentAct) & Partial(#/components/schemas/ContentPackingList) & Partial(#/components/schemas/ContentInvoicef)
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    documentId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [DELETE]/invoice/{apiVersion}/closing-documents/{customerCode}/{documentId}

- Summary  
Delete Closing Documents

- Description  
Метод для удаления закрывающего документа

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/invoice/{apiVersion}/closing-documents/{customerCode}/{documentId}/email

- Summary  
Send Closing Documents To Email

- Description  
Метод для отправки закрывающего документа на почту

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    email: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/invoice/{apiVersion}/closing-documents/{customerCode}/{documentId}/file

- Summary  
Get Closing Document

- Description  
Метод для получения файла закрывающего документа

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{}
```

`application/pdf`

```ts
{
  "type": "string",
  "format": "binary"
}
```

- Examples

  - Act

```json
{
  "summary": "Invoice file",
  "X-Content-Url": "https://enter.tochka.com/doc/openapi/static/file_examples/act_example.pdf"
}
```

  - Invoicef

```json
{
  "summary": "Invoice file",
  "X-Content-Url": "https://enter.tochka.com/doc/openapi/static/file_examples/invoicef_example.pdf"
}
```

  - PackingList

```json
{
  "summary": "Invoice file",
  "X-Content-Url": "https://enter.tochka.com/doc/openapi/static/file_examples/packing_list_example.pdf"
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/accounts

- Summary  
Get Accounts List

- Description  
Метод для получения списка доступных счетов  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения списка доступных счетов
{
  Data: {
    Account: {
      customerCode: string
      accountId: string
      transitAccount?: string
      status: enum[Enabled, Disabled, Deleted, ProForma, Pending]
      statusUpdateDateTime: string
      currency: string
      accountType: enum[Business, Personal]
      accountSubType: enum[CreditCard, CurrentAccount, Loan, Mortgage, PrePaidCard, Savings, Special]
      registrationDate: string
      accountDetails: {
        schemeName: string
        identification: string
        name: string
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/accounts/{accountId}

- Summary  
Get Account Info

- Description  
Метод для получения информации по конкретному счёту  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения информации по счёту
{
  Data: {
    customerCode: string
    accountId: string
    transitAccount?: string
    status: enum[Enabled, Disabled, Deleted, ProForma, Pending]
    statusUpdateDateTime: string
    currency: string
    accountType: enum[Business, Personal]
    accountSubType: enum[CreditCard, CurrentAccount, Loan, Mortgage, PrePaidCard, Savings, Special]
    registrationDate: string
    accountDetails: {
      schemeName: string
      identification: string
      name: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/accounts/{accountId}/authorized-card-transactions

- Summary  
Get Authorized Card Transactions

- Description  
Метод для получения авторизованных карточных транзакций конкретного счёта

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Transactions: {
      accountId: string
      // Маскированный номер карты транзакции
      pan: string
      dateTime: string
      Amount: {
        // Сумма транзакции
        amount: number
        // Валюта транзакции, используется ISO 4217
        currency: string
      }
      AccountAmount:#/components/schemas/CardTransactionAmountModel
      TerminalData: {
        // Город терминала
        city?: string
        // Адрес терминала
        location?: string
        // Название торговой точки
        owner?: string
      }
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/accounts/{accountId}/balances

- Summary  
Get Balance Info

- Description  
Метод для получения информации о балансе конкретного счёта  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения баланса
{
  Data: {
    Balance: {
      accountId: string
      creditDebitIndicator: enum[Credit, Debit]
      // **Описание типов балансов**
      // 
      //   - `OpeningAvailable` - Начальный остаток
      //   - `ClosingAvailable` - Доступный баланс
      //   - `Expected` - Сумма заблокированных средств
      //   - `OverdraftAvailable` - Доступный лимит по овердрафту
      type: enum[OpeningAvailable, ClosingAvailable, Expected, OverdraftAvailable]
      dateTime: string
      Amount: {
        amount: number
        currency: string
      }
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/accounts/{accountId}/statements/{statementId}

- Summary  
Get Statement

- Description  
Метод для получения конкретной выписки    
  
После вызова метода `Init Statement` с помощью `statementId` можно отследить, на каком этапе создание определённой выписки.    
  
Есть три статуса:  
- _Created_ — создан запрос на выписку;    
- _Processing_ — запрос в обработке;    
- _Ready_ — выписка готова.  
  
**Особенности:**    
Метод _Init Statement_ отрабатывает асинхронно. Отражаются только операции, находящиеся в финальном статусе — _Ready_.  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      statementId?: string
      status: enum[Created, Processing, Error, Ready]
      startDateTime: string
      endDateTime: string
      creationDateTime: string
      startDateBalance?: number
      endDateBalance?: number
      Transaction: {
        transactionId?: string
        paymentId?: string
        creditDebitIndicator: enum[Credit, Debit]
        status: enum[Booked, Pending]
        documentNumber?: string
        transactionTypeCode?: enum[Неопределенное значение, Платежное поручение, Платежное требование, Денежный чек, РКО, Объявление на взнос наличными, ПКО, Требование-поручение, Инкассовое поручение, Расчетный чек, Аккредитив, Мемориальный ордер, Погашение кредита, Выдача кредита, Авизо, Банковские карты, Платежный ордер, Банковский ордер, Ордер по передаче ценностей, Программный ордер, Импортированная запись]
        documentProcessDate?: string
        description?: string
        Amount: {
          amount: number
          amountNat?: number
          currency: string
        }
        DebtorParty: {
          inn?: string
          name?: string
          kpp?: string
        }
        DebtorAccount: {
          schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
          identification?: string
        }
        DebtorAgent: {
          schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
          identification?: string
          accountIdentification?: string
          name?: string
        }
        CreditorParty:#/components/schemas/ContractorInfoModel
        CreditorAccount:#/components/schemas/CashAccountInfoModel
        CreditorAgent:#/components/schemas/ContractorBankInfoModel
        TaxFields: {
          originatorStatus?: string
          kbk?: string
          oktmo?: string
          base?: string
          documentNumber?: string
          documentDate?: Partial(integer) & Partial(string)
          type?: string
          field107?: string
        }
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/balances

- Summary  
Get Balances List

- Description  
Метод для получения баланса по нескольким счетам  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения баланса
{
  Data: {
    Balance: {
      accountId: string
      creditDebitIndicator: enum[Credit, Debit]
      // **Описание типов балансов**
      // 
      //   - `OpeningAvailable` - Начальный остаток
      //   - `ClosingAvailable` - Доступный баланс
      //   - `Expected` - Сумма заблокированных средств
      //   - `OverdraftAvailable` - Доступный лимит по овердрафту
      type: enum[OpeningAvailable, ClosingAvailable, Expected, OverdraftAvailable]
      dateTime: string
      Amount: {
        amount: number
        currency: string
      }
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/customers

- Summary  
Get Customers List

- Description  
Метод для получения списка доступных клиентов  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения списка доступных клиентов
{
  Data: {
    Customer: {
      customerCode: string
      customerType: enum[Business, Personal]
      isResident: boolean
      taxCode?: string
      fullName: string
      shortName?: string
      kpp?: string
      customerOgrn?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/customers/{customerCode}

- Summary  
Get Customer Info

- Description  
Метод для получения информации по конкретному клиенту  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения информации по клиенту
{
  Data: {
    customerCode: string
    customerType: enum[Business, Personal]
    isResident: boolean
    taxCode?: string
    fullName: string
    shortName?: string
    kpp?: string
    customerOgrn?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/open-banking/{apiVersion}/statements

- Summary  
Get Statements List

- Description  
Метод для получения списка доступных выписок    
  
После вызова метода `Init Statement` можно отследить, в каком статусе готовящаяся выписка:    
- _Created_ — только создан запрос на выписку;  
- _Processing_ — запрос в обработке;  
- _Ready_ — выписка готова.  
  
**Особенности:**    
Отражаются только операции, находящиеся в финальном статусе — _Ready_.  


- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Максимальное количество выписок в ответе
limit?: integer //default: 5
```

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод получения выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      statementId?: string
      status: enum[Created, Processing, Error, Ready]
      startDateTime: string
      endDateTime: string
      creationDateTime: string
      startDateBalance?: number
      endDateBalance?: number
      Transaction: {
        transactionId?: string
        paymentId?: string
        creditDebitIndicator: enum[Credit, Debit]
        status: enum[Booked, Pending]
        documentNumber?: string
        transactionTypeCode?: enum[Неопределенное значение, Платежное поручение, Платежное требование, Денежный чек, РКО, Объявление на взнос наличными, ПКО, Требование-поручение, Инкассовое поручение, Расчетный чек, Аккредитив, Мемориальный ордер, Погашение кредита, Выдача кредита, Авизо, Банковские карты, Платежный ордер, Банковский ордер, Ордер по передаче ценностей, Программный ордер, Импортированная запись]
        documentProcessDate?: string
        description?: string
        Amount: {
          amount: number
          amountNat?: number
          currency: string
        }
        DebtorParty: {
          inn?: string
          name?: string
          kpp?: string
        }
        DebtorAccount: {
          schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
          identification?: string
        }
        DebtorAgent: {
          schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
          identification?: string
          accountIdentification?: string
          name?: string
        }
        CreditorParty:#/components/schemas/ContractorInfoModel
        CreditorAccount:#/components/schemas/CashAccountInfoModel
        CreditorAgent:#/components/schemas/ContractorBankInfoModel
        TaxFields: {
          originatorStatus?: string
          kbk?: string
          oktmo?: string
          base?: string
          documentNumber?: string
          documentDate?: Partial(integer) & Partial(string)
          type?: string
          field107?: string
        }
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/open-banking/{apiVersion}/statements

- Summary  
Init Statement

- Description  
Метод для создания выписки по конкретному счёту  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
// Метод создания выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      startDateTime: string
      endDateTime: string
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
// Метод создания выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      statementId?: string
      status: enum[Created, Processing, Error, Ready]
      startDateTime: string
      endDateTime: string
      creationDateTime: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/payment/{apiVersion}/for-sign

- Summary  
Get Payment For Sign List

- Description  
Метод для получения списка платежей находящихся в разделе «На подпись» интернет-банка

- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Уникальный код клиента
customerCode: string
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Payment: {
      accountCode?: string
      bankCode?: string
      counterpartyBankBic: string
      counterpartyAccountNumber: string
      // Допустимые значения "0", 10 или 12 значное число
      counterpartyINN?: string
      // Допустимые значения "0" или 9 значное число
      counterpartyKPP?: string
      counterpartyName: string
      paymentAmount: number
      paymentDate: string
      paymentNumber?: string
      paymentPurpose: string
      // Допустимые значения "0", 10 или 12 значное число
      payerINN?: string
      // Допустимые значения "0" или 9 значное число
      payerKPP?: string
      counterpartyBankCorrAccount?: string
      paymentPriority?: string //default: 5
      codePurpose?: string
      supplierBillId?: string
      budgetPaymentCode?: string
      email?: string
      taxInfoDocumentDate?: string
      taxInfoDocumentNumber?: string
      taxInfoKBK?: string
      taxInfoOKATO?: string
      taxInfoPeriod?: string
      taxInfoReasonCode?: string
      taxInfoStatus?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/payment/{apiVersion}/for-sign

- Summary  
Create Payment For Sign

- Description  
Метод для создания платежа.    
  
Если вы создаёте платёж за третье лицо в бюджет, заполните поля `payerINN` и `payerKPP`.  
  
**Обратите внимание:**    
_paymentDate_ нужно заполнить по часовому поясу Москвы.  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    accountCode: string
    bankCode: string
    // Заполняется только при платеже за 3 лицо. Допустимые значения "0", 10 или 12 значное число
    payerINN?: string
    // Заполняется только за 3 лицо при платеже в бюджет. Допустимые значение "0" или 9 значное число
    payerKPP?: string
    counterpartyBankBic: string
    counterpartyAccountNumber: string
    counterpartyINN?: string
    // Допустимые значения "0" или 9 значное число
    counterpartyKPP?: string
    counterpartyName: string
    counterpartyBankCorrAccount?: string
    paymentAmount: number
    // Дата платежа, приведенная к часовому поясу Москвы
    paymentDate: string
    paymentNumber?: integer
    paymentPriority?: string //default: 5
    paymentPurpose: string
    // Заполняется только при платеже физ лицам на счета:('40810', '40817', '40823', '40824', '40826', '423', '30232', '40803', '40813', '40820', '426'). Допустимые значения 1,2,3,4,5 и пусто
    codePurpose?: string
    supplierBillId?: string
    taxInfoDocumentDate?: string
    taxInfoDocumentNumber?: string
    taxInfoKBK?: string
    taxInfoOKATO?: string
    taxInfoPeriod?: string
    taxInfoReasonCode?: string
    taxInfoStatus?: string
    budgetPaymentCode?: string
    email?: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    requestId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/payment/{apiVersion}/order

- Summary  
Create Payment

- Description  
Метод для создания и подписания платежа.    
  
- Если вы создаёте платеж в бюджет, то необходимо заполнить поле `payerKPP`  
- Если вы создаёте платёж _за третье лицо в бюджет_, заполните поля `payerINN` и `payerKPP`.  
  
**Обратите внимание:**    
_paymentDate_ нужно заполнить по часовому поясу Москвы.  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    accountCode?: string
    bankCode?: string
    // Заполняется только при платеже за 3 лицо. Допустимые значения "0", 10 или 12 значное число
    payerINN?: string
    // Заполняется при платеже в бюджет (за себя или 3 лицо). Допустимые значение "0" или 9 значное число
    payerKPP?: string
    counterpartyBankBic: string
    counterpartyAccountNumber: string
    counterpartyINN?: string
    // Допустимые значения "0" или 9 значное число
    counterpartyKPP?: string
    counterpartyName: string
    counterpartyBankCorrAccount?: string
    paymentAmount: number
    // Дата платежа, приведенная к часовому поясу Москвы
    paymentDate: string
    paymentNumber?: integer
    paymentPriority?: string //default: 5
    paymentPurpose: string
    // Заполняется только при платеже физ лицам на счета:('40810', '40817', '40823', '40824', '40826', '423', '30232', '40803', '40813', '40820', '426'). Допустимые значения 1,2,3,4,5 и пусто
    codePurpose?: string
    supplierBillId?: string
    taxInfoDocumentDate?: string
    taxInfoDocumentNumber?: string
    taxInfoKBK?: string
    taxInfoOKATO?: string
    taxInfoPeriod?: string
    taxInfoReasonCode?: string
    taxInfoStatus?: string
    budgetPaymentCode?: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    requestId: string
    redirectURL: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/payment/{apiVersion}/status/{requestId}

- Summary  
Get Payment Status

- Description  
Метод для получения статуса платежа  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    requestId: string
    // **Описание статусов платежа**
    // 
    //   - `Initiated` - Все необходимые реквизиты для платежа получены, платёж готов к проверке на возможность проведения
    //   - `Wait For Owner Requisites` - Часть реквизитов для платежа получена, кроме реквизитов плательщика
    //   - `NotAllowed` - Платёж нельзя провести: либо у пользователя нет прав для подписи, либо платёж заблокирован
    //   комплаенсом
    //   - `Allowed` - Платёж готов к подписанию, все проверки пройдены
    //   - `WaitingForSign` - Платёж ждёт подписи
    //   - `WaitingForCreate` - Платёж подписан, ждёт создания внутри систем банка
    //   - `Created` - Платёж создан
    //   - `Paid` - Платёж оплачен
    //   - `Canceled` - Платёж отменен
    //   - `Rejected` - Платёж отменён
    status: enum[Initiated, Wait For Owner Requisites, NotAllowed, Allowed, WaitingForSign, WaitingForCreate, Created, Paid, Canceled, Rejected]
[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/account/{legalId}

- Summary  
Get Accounts List

- Description  
Метод для получения списка счетов юрлица в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    AccountList: {
      accountId: string
      status: enum[Active, Suspended]
      createdAt: string
      legalId: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/b2b-qr-code/merchant/{merchantId}/{accountId}

- Summary  
Register B2B Qr Code

- Description  
Метод для регистрации B2B QR-кода в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    amount: integer
    paymentPurpose: string
    // Система, создавшая QR-код
    sourceName: string
    // Наличие НДС
    takeTax: boolean
    // Сумма НДС в копейках
    totalTaxAmount?: integer
    ttl?: integer //default: 4320
    // Ссылка для автоматического возврата плательщика из приложения банка в приложение или на сайт ТСП
    redirectUrl?: string
    uip?: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    payload: string
    qrcId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/b2b-qr-code/{qrcId}

- Summary  
Get B2B Qr Code

- Description  
Метод для получения информации о B2B QR-коде  


- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Ширина изображения (по умолчанию: 300)
width?: integer
```

```ts
// Высота изображения (по умолчанию: 300)
height?: integer
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    payload: string
    accountId: string
    createdAt: string
    merchantId: string
    legalId: string
    qrcId: string
    amount?: integer
    ttl?: string
    paymentPurpose?: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    redirectUrl?: string
    // Наличие НДС
    takeTax: boolean
    // Сумма НДС в копейках
    totalTaxAmount?: integer
    uip?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/cashbox-qr-code

- Summary  
Register Cashbox Qrcode

- Description  
Метод для регистрации кассового QR-кода

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    merchantId: string
    accountId: string
    redirectUrl?: string
    imageParams: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    payload: string
    qrcId: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/cashbox-qr-code/merchant/{merchantId}/{accountId}

- Summary  
Get Cashbox Qrcode List

- Description  
Метод для получения списка кассовых QR-кодов

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    qrCodes: {
      payload: string
      accountId: string
      merchantId: string
      legalId: string
      createdAt: string
      qrcId: string
      amount?: integer
      currency?: string
      paymentPurpose?: string
      paramsId?: string
      ttl?: integer
      commission: {
        mcc?: string
        percent?: number
        description?: string
      }
      redirectUrl?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/cashbox-qr-code/{qrcId}

- Summary  
Get Cashbox Qrcode

- Description  
Метод для получения информации о кассовом QR-коде

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    imageParams: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
    }
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    payload: string
    accountId: string
    merchantId: string
    legalId: string
    createdAt: string
    qrcId: string
    amount?: integer
    currency?: string //default: RUB
    paymentPurpose?: string
    paramsId?: string
    ttl?: integer
    commission: {
      mcc?: string
      percent?: number
      description?: string
    }
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    redirectUrl?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/account

- Summary  
Change Cashbox Qrcode Account

- Description  
Метод для смены счёта зачисления кассового QR-кода

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    accountId: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    accountId: string
    qrcId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/activate

- Summary  
Activate Cashbox Qrcode

- Description  
Метод для активации кассового QR-кода

- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    amount: integer
    // Валюта операции
    currency?: string //default: RUB
    paymentPurpose?: string
    ttl?: integer //default: 5
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    qrcId: string
    amount: integer
    currency?: string //default: RUB
    paramsId: string
    paymentPurpose?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/deactivate

- Summary  
Deactivate Cashbox Qrcode

- Description  
Метод для деактивации кассового QR-кода

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/operation

- Summary  
Get Cashbox Qrcode Operation Info

- Description  
Метод для получения статуса кассового QR-кода.

- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Идентификатор активных значений параметров QR-кода
paramsId: string
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    qrCodeStatus: enum[DEACTIVATED, WAITING_PAYMENT, IN_PROGRESS]
    // * ACWP - Операция завершена успешно
    // * RJCT - Операция отклонена
    // * RCVD - Операция в обработке
    // * NTST - Операции по QR-коду не существует
    trxStatus: enum[ACWP, RJCT, RCVD, NTST]
    trxId?: string
    // Целое, положительное число. Валюта операции – рубли РФ
    amount?: integer
    dateTime?: string
    payerId?: string
    kzo?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/cashbox-qr-code/{qrcId}/status

- Summary  
Get Cashbox Qrcode Status

- Description  
Метод для получения статуса кассового QR-кода.

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status: enum[INACTIVATED, WAITING_PAYMENT, IN_PROGRESS]
    paramsId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/customer/{customerCode}/{bankCode}

- Summary  
Get Customer Info

- Description  
Метод для получения информации о клиенте в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status: enum[Active, Suspended]
    legalId: string
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    customerCode: string
    entityType?: string
    inn: string
    kpp?: string
    name: string
    ogrn: string
    bankCode: string
    MerchantList: {
      status:#/components/schemas/StatusEnum
      createdAt: string
      address?: string
      city?: string
      countryCode: string
      countrySubDivisionCode?: string
      zipCode?: string
      merchantId: string
      legalId: string
      brandName: string
      // `001` - только QR Static
      // `010` - только QR Dynamic
      // `011` - QR Static и QR Dynamic
      // `100` - Только QR Subscription
      // `101` - QR Subscription и QR Static
      // `110` - QR Subscription и QR Dynamic
      // `111` - QR Static, QR Dynamic и QR Subscription
      capabilities: enum[001, 010, 011, 100, 101, 110, 111]
      contactPhoneNumber?: string
      mcc: string
      additionalContacts: {
      }[]
    }[]
    AccountList: {
      accountId: string
      status:#/components/schemas/StatusEnum
      createdAt: string
      legalId: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/get-sbp-payments

- Summary  
Get Payments

- Description  
Метод для получения списка платежей в Системе быстрых платежей  
  
Обратите внимание: при поиске платежей за прошедшие дни обязательно передавать атрибут `fromDate` с указанием начальной даты периода. Если этот атрибут не передать, поиск выдаст результаты только за вчерашний и сегодняшний дни.  


- Security  
Необходимые разрешения  

#### Parameters(Query)

```ts
// Уникальный код клиента
customerCode: string
```

```ts
// ID qr-кода для фильтрации
qrcId?: string
```

```ts
// Начало периода для запроса статусов платежей. При отсутствии параметра в запросе будет задано дефолтное значение
fromDate?: string //default: Вчера и сегодня
```

```ts
// Конец периода для запроса статусов платежей
toDate?: string
```

```ts
// Номер страницы
page?: integer //default: 1
```

```ts
// Количество записей на странице
perPage?: integer //default: 1000
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    Payments: {
      qrcId: string
      // **Описание возможных статусов платежа**
      //   - `Confirming` - операция в процессе подтверждения ОПКЦ СБП
      //   - `Confirmed` - операция подтверждена
      //   - `Initiated` - операция отправлена на обработку
      //   - `Accepting` - операция в обработке ОПКЦ СБП
      //   - `Accepted` - операция успешно завершена
      //   - `InProgress` - операция в обработке РЦ СБП
      //   - `Rejected` - операция отклонена
      //   - `Error` - ошибка выполнения операции
      //   - `Timeout` - тайм-аут выполнения операции
      status: enum[Confirming, Confirmed, Initiated, Accepting, Accepted, InProgress, Rejected, Error, Timeout]
      message: string
      refTransactionId: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/legal-entity/{legalId}

- Summary  
Get Legal Entity

- Description  
Метод для получения данных юрлица в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status: enum[Active, Suspended]
    legalId: string
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    customerCode: string
    entityType?: string
    inn: string
    kpp?: string
    name: string
    ogrn: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/legal-entity/{legalId}

- Summary  
Set Legal Entity Status

- Description  
Метод устанавливает статус юрлица в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    status: enum[Active, Suspended]
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/merchant/legal-entity/{legalId}

- Summary  
Get Merchants List

- Description  
Метод для получения списка ТСП юрлица  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    MerchantList: {
      status: enum[Active, Suspended]
      createdAt: string
      address?: string
      city?: string
      countryCode: string
      countrySubDivisionCode?: string
      zipCode?: string
      merchantId: string
      legalId: string
      brandName: string
      // `001` - только QR Static
      // `010` - только QR Dynamic
      // `011` - QR Static и QR Dynamic
      // `100` - Только QR Subscription
      // `101` - QR Subscription и QR Static
      // `110` - QR Subscription и QR Dynamic
      // `111` - QR Static, QR Dynamic и QR Subscription
      capabilities: enum[001, 010, 011, 100, 101, 110, 111]
      contactPhoneNumber?: string
      mcc: string
      additionalContacts: {
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/merchant/legal-entity/{legalId}

- Summary  
Register Merchant

- Description  
Метод для регистрации ТСП в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    address: string
    city: string
    countryCode: string
    countrySubDivisionCode: string
    zipCode: string
    brandName: string
    // `001` - только QR Static
    // `010` - только QR Dynamic
    // `011` - QR Static и QR Dynamic
    // `100` - Только QR Subscription
    // `101` - QR Subscription и QR Static
    // `110` - QR Subscription и QR Dynamic
    // `111` - QR Static, QR Dynamic и QR Subscription
    capabilities: enum[001, 010, 011, 100, 101, 110, 111]
    contactPhoneNumber?: string
    mcc: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    merchantId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/merchant/{merchantId}

- Summary  
Get Merchant

- Description  
Метод для получения информации о ТСП  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status: enum[Active, Suspended]
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    merchantId: string
    legalId: string
    brandName: string
    // `001` - только QR Static
    // `010` - только QR Dynamic
    // `011` - QR Static и QR Dynamic
    // `100` - Только QR Subscription
    // `101` - QR Subscription и QR Static
    // `110` - QR Subscription и QR Dynamic
    // `111` - QR Static, QR Dynamic и QR Subscription
    capabilities: enum[001, 010, 011, 100, 101, 110, 111]
    contactPhoneNumber?: string
    mcc: string
    additionalContacts: {
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [PUT]/sbp/{apiVersion}/merchant/{merchantId}

- Summary  
Set Merchant Status

- Description  
Метод устанавливает статус ТСП  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    status: enum[Active, Suspended]
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/qr-code/legal-entity/{legalId}

- Summary  
Get Qr Codes List

- Description  
Метод для получения списка QR-кодов  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    qrCodeList: {
      status: enum[Active, Suspended]
      payload: string
      accountId: string
      createdAt: string
      merchantId: string
      legalId: string
      qrcId: string
      amount?: integer
      ttl?: string
      paymentPurpose?: string
      image: {
        width: integer
        height: integer
        mediaType?: enum[image/png, image/svg+xml]
        content: string
      }
      commissionPercent: number
      currency?: string
      // 01 - QR-Static (QR наклейка)
      // 02 - QR-Dynamic (QR на кассе)
      qrcType: enum[01, 02]
      templateVersion: string
      sourceName?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/qr-code/merchant/{merchantId}/{accountId}

- Summary  
Register Qr Code

- Description  
Метод для регистрации статического или динамического QR-кода в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    // Поле обязательно для заполнения, если тип QR = QR-Dynamic
    amount?: integer
    currency?: string
    paymentPurpose: string
    // 01 - QR-Static (QR наклейка)
    // 02 - QR-Dynamic (QR на кассе)
    qrcType: enum[01, 02]
    imageParams: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
    }
    // Cистема, создавшая QR-код
    sourceName?: string
    // Задается, только если тип QR = QR-Dynamic
    ttl?: integer
    // Ссылка для автоматического возврата плательщика из приложения банка в приложение или на сайт ТСП
    redirectUrl?: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    payload: string
    qrcId: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/qr-code/{qrcId}

- Summary  
Get Qr Code

- Description  
Метод для получения информации о QR-коде  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    status: enum[Active, Suspended]
    payload: string
    accountId: string
    createdAt: string
    merchantId: string
    legalId: string
    qrcId: string
    amount?: integer
    ttl?: string
    paymentPurpose?: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    commissionPercent: number
    currency?: string
    // 01 - QR-Static (QR наклейка)
    // 02 - QR-Dynamic (QR на кассе)
    qrcType: enum[01, 02]
    templateVersion: string
    sourceName?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/qr-codes/{qrc_ids}/payment-status

- Summary  
Get Qr Codes Payment Status

- Description  
Метод для получения статусов операций по динамическим QR-кодам

- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    paymentList: {
      qrcId: string
      code: string
      // **Описание возможных статусов платежа**
      // 
      //   - `NotStarted` - операции по QR-коду не существует
      //   - `Received` - операция в обработке
      //   - `InProgress` - операция в обработке
      //   - `Accepted` - операция завершена успешно
      //   - `Rejected` - операция отклонена
      status?: enum[NotStarted, Received, InProgress, Accepted, Rejected]
      message: string
      trxId?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/refund

- Summary  
Start Refund

- Description  
Метод запрашивает возврат платежа через Систему быстрых платежей  
  
Если нужно вернуть деньги нерезиденту, назначение платежа должно начинаться с _«{VO99020} Возврат ошибочно полученной суммы transactionId»,_ где `transactionId` — это идентификатор оригинальной операции.  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    // БИК отправителя
    bankCode: string
    // Уникальный и неизменный идентификатор счёта
    accountCode: string
    // Cумма операции в рублях
    amount: string
    // Валюта операции
    currency?: string //default: RUB
    // ID qr-кода, по которому был сделан платеж
    qrcId: string
    // Назначение платежа
    purpose?: string
    // Идентификатор транзакции, по которой осуществляется возврат
    refTransactionId?: string
    // Идентификатор операции в НСПК, по которой можно осуществить возврат
    trxId?: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    requestId: string
    status: enum[WaitingForClientConfirm, Initiated, WaitingForConfirm, Confirmed, WaitingForAccept, Accepted, Rejected]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/sbp/{apiVersion}/refund/{request_id}

- Summary  
Get Refund Data

- Description  
Метод для получения информация о платеже-возврате по Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    requestId: string
    status: enum[WaitingForClientConfirm, Initiated, WaitingForConfirm, Confirmed, WaitingForAccept, Accepted, Rejected]
    // Описание статуса (причина ошибки или сообщение об успехе)
    statusDescription?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/sbp/{apiVersion}/register-sbp-legal-entity

- Summary  
Register Legal Entity

- Description  
Метод для регистрации юрлица в Системе быстрых платежей  


- Security  
Необходимые разрешения  

#### RequestBody

- application/json

```ts
{
  Data: {
    customerCode: string
    bankCode: string
  }
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    legalId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [DELETE]/webhook/{apiVersion}/{client_id}

- Summary  
Delete Webhook

- Description  
Метод для удаления вебхука  


#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [GET]/webhook/{apiVersion}/{client_id}

- Summary  
Get Webhooks

- Description  
Метод для получения списка вебхуков приложения  


#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
    url: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/webhook/{apiVersion}/{client_id}

- Summary  
Edit Webhook

- Description  
Метод для изменения _URL_ и типа вебхука

#### RequestBody

- application/json

```ts
{
  webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
  url: string
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
    url: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [PUT]/webhook/{apiVersion}/{client_id}

- Summary  
Create Webhook

- Description  
Метод для создания вебхуков  


#### RequestBody

- application/json

```ts
{
  webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
  url: string
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
    url: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

***

### [POST]/webhook/{apiVersion}/{client_id}/test_send

- Summary  
Send Webhook

- Description  
Метод для проверки отправки вебхука

#### RequestBody

- application/json

```ts
{
  webhookType: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment]
}
```

#### Responses

- 200 Successful Response

`application/json`

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

- 400 Bad Request

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 401 Unauthorized

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 403 Forbidden

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 404 Not Found

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

- 500 Server error

`application/json`

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

## References

### #/components/schemas/Account

```ts
{
  accountId: string
  status: enum[Active, Suspended]
  createdAt: string
  legalId: string
}
```

### #/components/schemas/AccountDetailModel

```ts
{
  schemeName: string
  identification: string
  name: string
}
```

### #/components/schemas/AccountIdentificationEnum

```ts
{
  "type": "string",
  "enum": [
    "RU.CBR.PAN",
    "RU.CBR.CellphoneNumber",
    "RU.CBR.BBAN"
  ],
  "title": "AccountIdentificationEnum"
}
```

### #/components/schemas/AccountListModel

```ts
{
  Account: {
    customerCode: string
    accountId: string
    transitAccount?: string
    status: enum[Enabled, Disabled, Deleted, ProForma, Pending]
    statusUpdateDateTime: string
    currency: string
    accountType: enum[Business, Personal]
    accountSubType: enum[CreditCard, CurrentAccount, Loan, Mortgage, PrePaidCard, Savings, Special]
    registrationDate: string
    accountDetails: {
      schemeName: string
      identification: string
      name: string
    }[]
  }[]
}
```

### #/components/schemas/AccountListResponse

```ts
{
  AccountList: {
    accountId: string
    status: enum[Active, Suspended]
    createdAt: string
    legalId: string
  }[]
}
```

### #/components/schemas/AccountModel

```ts
{
  customerCode: string
  accountId: string
  transitAccount?: string
  status: enum[Enabled, Disabled, Deleted, ProForma, Pending]
  statusUpdateDateTime: string
  currency: string
  accountType: enum[Business, Personal]
  accountSubType: enum[CreditCard, CurrentAccount, Loan, Mortgage, PrePaidCard, Savings, Special]
  registrationDate: string
  accountDetails: {
    schemeName: string
    identification: string
    name: string
  }[]
}
```

### #/components/schemas/AccountResponseModel

```ts
// Метод получения информации по счёту
{
  Data: {
    customerCode: string
    accountId: string
    transitAccount?: string
    status: enum[Enabled, Disabled, Deleted, ProForma, Pending]
    statusUpdateDateTime: string
    currency: string
    accountType: enum[Business, Personal]
    accountSubType: enum[CreditCard, CurrentAccount, Loan, Mortgage, PrePaidCard, Savings, Special]
    registrationDate: string
    accountDetails: {
      schemeName: string
      identification: string
      name: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringCapturePaymentResponseModel

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringChargeSubscriptionRequestDataModel

```ts
{
  Data: {
    amount: number
  }
}
```

### #/components/schemas/AcquiringChargeSubscriptionRequestModel

```ts
{
  amount: number
}
```

### #/components/schemas/AcquiringChargeSubscriptionResponseDataModel

```ts
{
  Data: {
    result: boolean
  }
}
```

### #/components/schemas/AcquiringCreatePaymentOperationRequestDataModel

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    paymentMode?: enum[sbp, card, tinkoff][]
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
  }
}
```

### #/components/schemas/AcquiringCreatePaymentOperationRequestModel

```ts
{
  customerCode: string
  amount: number
  purpose: string
  redirectUrl?: string
  failRedirectUrl?: string
  paymentMode?: enum[sbp, card, tinkoff][]
  saveCard?: boolean
  consumerId?: string
  merchantId?: string
  preAuthorization?: boolean
  ttl?: integer //default: 10080
}
```

### #/components/schemas/AcquiringCreatePaymentOperationResponseDataModel

```ts
{
  Data: {
    purpose: string
    status?: enum[CREATED]
    amount: number
    operationId: string
    paymentLink: string
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringCreatePaymentOperationResponseModel

```ts
{
  purpose: string
  status?: enum[CREATED]
  amount: number
  operationId: string
  paymentLink: string
  consumerId?: string
  merchantId?: string
  preAuthorization?: boolean
  ttl?: integer //default: 10080
}
```

### #/components/schemas/AcquiringCreatePaymentOperationWithReceiptRequestDataModel

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    paymentMode?: enum[sbp, card, tinkoff][]
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
}
```

### #/components/schemas/AcquiringCreatePaymentOperationWithReceiptRequestModel

```ts
{
  customerCode: string
  amount: number
  purpose: string
  redirectUrl?: string
  failRedirectUrl?: string
  paymentMode?: enum[sbp, card, tinkoff][]
  saveCard?: boolean
  consumerId?: string
  merchantId?: string
  preAuthorization?: boolean
  ttl?: integer //default: 10080
  taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
  Client: {
    name?: string
    email: string
    phone?: string
  }
  Items: {
    vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
    name: string
    amount: number
    quantity: number
    paymentMethod?: enum[full_payment, full_prepayment]
    paymentObject?: enum[goods, service, work]
    measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
  }[]
  Supplier: {
    phone: string
    name: string
    taxCode: string
  }
}
```

### #/components/schemas/AcquiringCreatePaymentOperationWithReceiptResponseDataModel

```ts
{
  Data: {
    purpose: string
    status?: enum[CREATED]
    amount: number
    operationId: string
    paymentLink: string
    consumerId?: string
    merchantId?: string
    preAuthorization?: boolean
    ttl?: integer //default: 10080
    customerCode: string
    redirectUrl?: string
    failRedirectUrl?: string
    paymentMode?: enum[sbp, card, tinkoff][]
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringCreatePaymentOperationWithReceiptResponseModel

```ts
{
  purpose: string
  status?: enum[CREATED]
  amount: number
  operationId: string
  paymentLink: string
  consumerId?: string
  merchantId?: string
  preAuthorization?: boolean
  ttl?: integer //default: 10080
  customerCode: string
  redirectUrl?: string
  failRedirectUrl?: string
  paymentMode?: enum[sbp, card, tinkoff][]
  taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
  Client: {
    name?: string
    email: string
    phone?: string
  }
  Items: {
    vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
    name: string
    amount: number
    quantity: number
    paymentMethod?: enum[full_payment, full_prepayment]
    paymentObject?: enum[goods, service, work]
    measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
  }[]
  Supplier: {
    phone: string
    name: string
    taxCode: string
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionRequestDataModel

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer //default: 12
      period?: enum[Day, Month, Year]
      daysInPeriod?: integer
    }
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionRequestModel

```ts
{
  customerCode: string
  amount: number
  purpose: string
  redirectUrl?: string
  failRedirectUrl?: string
  saveCard?: boolean
  consumerId?: string
  merchantId?: string
  recurring?: boolean
  Options: {
    trancheCount?: integer //default: 12
    period?: enum[Day, Month, Year]
    daysInPeriod?: integer
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionResponseDataModel

```ts
{
  Data: {
    purpose: string
    amount: number
    status?: enum[CREATED]
    operationId: string
    paymentLink: string
    consumerId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer
      period?: enum[Day, Month, Year, Quarter]
      daysInPeriod?: integer
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionResponseModel

```ts
{
  purpose: string
  amount: number
  status?: enum[CREATED]
  operationId: string
  paymentLink: string
  consumerId?: string
  recurring?: boolean
  Options: {
    trancheCount?: integer
    period?: enum[Day, Month, Year, Quarter]
    daysInPeriod?: integer
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionWithReceiptRequestDataModel

```ts
{
  Data: {
    customerCode: string
    amount: number
    purpose: string
    redirectUrl?: string
    failRedirectUrl?: string
    saveCard?: boolean
    consumerId?: string
    merchantId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer //default: 12
      period?: enum[Day, Month, Year]
      daysInPeriod?: integer
    }
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionWithReceiptRequestModel

```ts
{
  customerCode: string
  amount: number
  purpose: string
  redirectUrl?: string
  failRedirectUrl?: string
  saveCard?: boolean
  consumerId?: string
  merchantId?: string
  recurring?: boolean
  Options: {
    trancheCount?: integer //default: 12
    period?: enum[Day, Month, Year]
    daysInPeriod?: integer
  }
  taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
  Client: {
    name?: string
    email: string
    phone?: string
  }
  Items: {
    vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
    name: string
    amount: number
    quantity: number
    paymentMethod?: enum[full_payment, full_prepayment]
    paymentObject?: enum[goods, service, work]
    measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
  }[]
  Supplier: {
    phone: string
    name: string
    taxCode: string
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionWithReceiptResponseDataModel

```ts
{
  Data: {
    purpose: string
    amount: number
    status?: enum[CREATED]
    operationId: string
    paymentLink: string
    consumerId?: string
    recurring?: boolean
    Options: {
      trancheCount?: integer
      period?: enum[Day, Month, Year, Quarter]
      daysInPeriod?: integer
    }
    customerCode: string
    redirectUrl?: string
    failRedirectUrl?: string
    merchantId?: string
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringCreateSubscriptionWithReceiptResponseModel

```ts
{
  purpose: string
  amount: number
  status?: enum[CREATED]
  operationId: string
  paymentLink: string
  consumerId?: string
  recurring?: boolean
  Options: {
    trancheCount?: integer
    period?: enum[Day, Month, Year, Quarter]
    daysInPeriod?: integer
  }
  customerCode: string
  redirectUrl?: string
  failRedirectUrl?: string
  merchantId?: string
  taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent]
  Client: {
    name?: string
    email: string
    phone?: string
  }
  Items: {
    vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
    name: string
    amount: number
    quantity: number
    paymentMethod?: enum[full_payment, full_prepayment]
    paymentObject?: enum[goods, service, work]
    measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
  }[]
  Supplier: {
    phone: string
    name: string
    taxCode: string
  }
}
```

### #/components/schemas/AcquiringCreatedStatus

```ts
{
  "type": "string",
  "enum": [
    "CREATED"
  ],
  "title": "AcquiringCreatedStatus"
}
```

### #/components/schemas/AcquiringGetPaymentOperationInfoResponseDataModel

```ts
{
  Data: {
    Operation: {
      customerCode: string
      // Перечисление для выдачи результатов из openapi.
      taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
      paymentType?: enum[sbp, card, tinkoff]
      paymentId?: string
      // Используется для возврата при оплате по СБП
      transactionId?: string
      createdAt: string
      paymentMode?: enum[sbp, card, tinkoff][]
      redirectUrl?: string
      failRedirectUrl?: string
      Client: {
        name?: string
        email: string
        phone?: string
      }
      Items: {
        vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
        name: string
        amount: number
        quantity: number
        paymentMethod?: enum[full_payment, full_prepayment]
        paymentObject?: enum[goods, service, work]
        measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
      }[]
      // Отсутствует, если при создании платежа назначение не было указано
      purpose?: string
      amount: number
      status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
      operationId: string
      paymentLink: string
      merchantId?: string
      consumerId?: string
      Order: {
        orderId: string
        type: enum[refund, approval, authorized]
        amount: number
        time: string
      }[]
      Supplier: {
        phone: string
        name: string
        taxCode: string
      }
      preAuthorization?: boolean
      paidAt?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringGetPaymentOperationListItemModel

```ts
{
  customerCode: string
  // Перечисление для выдачи результатов из openapi.
  taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
  paymentType?: enum[sbp, card, tinkoff]
  paymentId?: string
  // Используется для возврата при оплате по СБП
  transactionId?: string
  createdAt: string
  paymentMode?: enum[sbp, card, tinkoff][]
  redirectUrl?: string
  failRedirectUrl?: string
  Client: {
    name?: string
    email: string
    phone?: string
  }
  Items: {
    vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
    name: string
    amount: number
    quantity: number
    paymentMethod?: enum[full_payment, full_prepayment]
    paymentObject?: enum[goods, service, work]
    measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
  }[]
  // Отсутствует, если при создании платежа назначение не было указано
  purpose?: string
  amount: number
  status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
  operationId: string
  paymentLink: string
  merchantId?: string
  consumerId?: string
  Order: {
    orderId: string
    type: enum[refund, approval, authorized]
    amount: number
    time: string
  }[]
  Supplier: {
    phone: string
    name: string
    taxCode: string
  }
  preAuthorization?: boolean
  paidAt?: string
}
```

### #/components/schemas/AcquiringGetPaymentOperationListResponseDataModel

```ts
{
  Data: {
    Operation: {
      customerCode: string
      // Перечисление для выдачи результатов из openapi.
      taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
      paymentType?: enum[sbp, card, tinkoff]
      paymentId?: string
      // Используется для возврата при оплате по СБП
      transactionId?: string
      createdAt: string
      paymentMode?: enum[sbp, card, tinkoff][]
      redirectUrl?: string
      failRedirectUrl?: string
      Client: {
        name?: string
        email: string
        phone?: string
      }
      Items: {
        vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
        name: string
        amount: number
        quantity: number
        paymentMethod?: enum[full_payment, full_prepayment]
        paymentObject?: enum[goods, service, work]
        measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
      }[]
      // Отсутствует, если при создании платежа назначение не было указано
      purpose?: string
      amount: number
      status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
      operationId: string
      paymentLink: string
      merchantId?: string
      consumerId?: string
      Order: {
        orderId: string
        type: enum[refund, approval, authorized]
        amount: number
        time: string
      }[]
      Supplier: {
        phone: string
        name: string
        taxCode: string
      }
      preAuthorization?: boolean
      paidAt?: string
    }[]
  }
  Links: {
    self: string
    first?: string
    prev?: string
    next?: string
    last?: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringGetPaymentOperationListResponseModel

```ts
{
  Operation: {
    customerCode: string
    // Перечисление для выдачи результатов из openapi.
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
    paymentType?: enum[sbp, card, tinkoff]
    paymentId?: string
    // Используется для возврата при оплате по СБП
    transactionId?: string
    createdAt: string
    paymentMode?: enum[sbp, card, tinkoff][]
    redirectUrl?: string
    failRedirectUrl?: string
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    // Отсутствует, если при создании платежа назначение не было указано
    purpose?: string
    amount: number
    status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
    operationId: string
    paymentLink: string
    merchantId?: string
    consumerId?: string
    Order: {
      orderId: string
      type: enum[refund, approval, authorized]
      amount: number
      time: string
    }[]
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
    preAuthorization?: boolean
    paidAt?: string
  }[]
}
```

### #/components/schemas/AcquiringGetSubscriptionStatusResponseDataModel

```ts
{
  Data: {
    status: enum[Active, Cancelled, Completed, Expired, Failed, PastDue, Preparing, Refused, Rejected, Suspended, Trial]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringGetSubscriptionStatusResponseModel

```ts
{
  status: enum[Active, Cancelled, Completed, Expired, Failed, PastDue, Preparing, Refused, Rejected, Suspended, Trial]
}
```

### #/components/schemas/AcquiringPaymentMode

```ts
{
  "type": "string",
  "enum": [
    "sbp",
    "card",
    "tinkoff"
  ],
  "title": "AcquiringPaymentMode"
}
```

### #/components/schemas/AcquiringPaymentOperationRefundModel

```ts
{
  isRefund: boolean
  operationId: string
  amount: number
  date: string
  orderId: string
}
```

### #/components/schemas/AcquiringPaymentOperationRefundResponseModel

```ts
{
  Data: {
    isRefund: boolean
    operationId: string
    amount: number
    date: string
    orderId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringPaymentOrderRefundRequestDataModel

```ts
{
  Data: {
    // Не больше суммы оплаты
    amount: number
  }
}
```

### #/components/schemas/AcquiringPaymentOrderRefundRequestModel

```ts
{
  // Не больше суммы оплаты
  amount: number
}
```

### #/components/schemas/AcquiringPaymentStatus

```ts
{
  "type": "string",
  "enum": [
    "CREATED",
    "APPROVED",
    "ON-REFUND",
    "REFUNDED",
    "EXPIRED",
    "REFUNDED_PARTIALLY",
    "AUTHORIZED"
  ],
  "title": "AcquiringPaymentStatus"
}
```

### #/components/schemas/AcquiringRetailerListModel

```ts
{
  Retailer: {
    status: enum[NEW, ADDRESS_DADATA, OPEN_ACCOUNT, TWPG_SENDED, RETAILER_CREATED, TERMINAL_CREATED, FILE_SENT, REG, CLOSE]
    isActive: boolean
    mcc: string
    rate: number
    name: string
    url?: string
    // Может отсутствовать при значениях поля status: NEW, ADDRESS_DADATA и OPEN_ACCOUNT
    merchantId?: string
    // Будет заполнен при значениях поля status: TERMINAL_CREATED, FILE_SENT, REG
    terminalId?: string
    paymentModes?: enum[sbp, card, tinkoff][]
    cashbox?: string
  }[]
}
```

### #/components/schemas/AcquiringRetailerListResponseModel

```ts
{
  Data: {
    Retailer: {
      status: enum[NEW, ADDRESS_DADATA, OPEN_ACCOUNT, TWPG_SENDED, RETAILER_CREATED, TERMINAL_CREATED, FILE_SENT, REG, CLOSE]
      isActive: boolean
      mcc: string
      rate: number
      name: string
      url?: string
      // Может отсутствовать при значениях поля status: NEW, ADDRESS_DADATA и OPEN_ACCOUNT
      merchantId?: string
      // Будет заполнен при значениях поля status: TERMINAL_CREATED, FILE_SENT, REG
      terminalId?: string
      paymentModes?: enum[sbp, card, tinkoff][]
      cashbox?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringRetailerModel

```ts
{
  status: enum[NEW, ADDRESS_DADATA, OPEN_ACCOUNT, TWPG_SENDED, RETAILER_CREATED, TERMINAL_CREATED, FILE_SENT, REG, CLOSE]
  isActive: boolean
  mcc: string
  rate: number
  name: string
  url?: string
  // Может отсутствовать при значениях поля status: NEW, ADDRESS_DADATA и OPEN_ACCOUNT
  merchantId?: string
  // Будет заполнен при значениях поля status: TERMINAL_CREATED, FILE_SENT, REG
  terminalId?: string
  paymentModes?: enum[sbp, card, tinkoff][]
  cashbox?: string
}
```

### #/components/schemas/AcquiringRetailerStatus

```ts
{
  "type": "string",
  "enum": [
    "NEW",
    "ADDRESS_DADATA",
    "OPEN_ACCOUNT",
    "TWPG_SENDED",
    "RETAILER_CREATED",
    "TERMINAL_CREATED",
    "FILE_SENT",
    "REG",
    "CLOSE"
  ],
  "title": "AcquiringRetailerStatus"
}
```

### #/components/schemas/AcquiringSetSubscriptionStatusRequestDataModel

```ts
{
  Data: {
    status: enum[Cancelled]
  }
}
```

### #/components/schemas/AcquiringSetSubscriptionStatusRequestModel

```ts
{
  status: enum[Cancelled]
}
```

### #/components/schemas/AcquiringSetSubscriptionStatusResponseDataModel

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringSubscriptionInputOptions

```ts
{
  trancheCount?: integer //default: 12
  period?: enum[Day, Month, Year]
  daysInPeriod?: integer
}
```

### #/components/schemas/AcquiringSubscriptionListItemModel

```ts
{
  customerCode: string
  // Перечисление для выдачи результатов из openapi.
  taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
  paymentId?: string
  // Используется для возврата при оплате по СБП
  transactionId?: string
  createdAt: string
  redirectUrl?: string
  failRedirectUrl?: string
  Client: {
    name?: string
    email: string
    phone?: string
  }
  Items: {
    vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
    name: string
    amount: number
    quantity: number
    paymentMethod?: enum[full_payment, full_prepayment]
    paymentObject?: enum[goods, service, work]
    measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
  }[]
  // Отсутствует, если при создании платежа назначение не было указано
  purpose?: string
  amount: number
  status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
  operationId: string
  paymentLink: string
  merchantId?: string
  consumerId?: string
  Options: {
    trancheCount?: integer
    period?: enum[Day, Month, Year, Quarter]
    daysInPeriod?: integer
  }
  Supplier: {
    phone: string
    name: string
    taxCode: string
  }
  recurring?: boolean
}
```

### #/components/schemas/AcquiringSubscriptionListResponseDataModel

```ts
{
  Data: {
    Subscription: {
      customerCode: string
      // Перечисление для выдачи результатов из openapi.
      taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
      paymentId?: string
      // Используется для возврата при оплате по СБП
      transactionId?: string
      createdAt: string
      redirectUrl?: string
      failRedirectUrl?: string
      Client: {
        name?: string
        email: string
        phone?: string
      }
      Items: {
        vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
        name: string
        amount: number
        quantity: number
        paymentMethod?: enum[full_payment, full_prepayment]
        paymentObject?: enum[goods, service, work]
        measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
      }[]
      // Отсутствует, если при создании платежа назначение не было указано
      purpose?: string
      amount: number
      status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
      operationId: string
      paymentLink: string
      merchantId?: string
      consumerId?: string
      Options: {
        trancheCount?: integer
        period?: enum[Day, Month, Year, Quarter]
        daysInPeriod?: integer
      }
      Supplier: {
        phone: string
        name: string
        taxCode: string
      }
      recurring?: boolean
    }[]
  }
  Links: {
    self: string
    first?: string
    prev?: string
    next?: string
    last?: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/AcquiringSubscriptionListResponseModel

```ts
{
  Subscription: {
    customerCode: string
    // Перечисление для выдачи результатов из openapi.
    taxSystemCode?: enum[osn, usn_income, usn_income_outcome, esn, patent, envd]
    paymentId?: string
    // Используется для возврата при оплате по СБП
    transactionId?: string
    createdAt: string
    redirectUrl?: string
    failRedirectUrl?: string
    Client: {
      name?: string
      email: string
      phone?: string
    }
    Items: {
      vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
      name: string
      amount: number
      quantity: number
      paymentMethod?: enum[full_payment, full_prepayment]
      paymentObject?: enum[goods, service, work]
      measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
    }[]
    // Отсутствует, если при создании платежа назначение не было указано
    purpose?: string
    amount: number
    status: enum[CREATED, APPROVED, ON-REFUND, REFUNDED, EXPIRED, REFUNDED_PARTIALLY, AUTHORIZED]
    operationId: string
    paymentLink: string
    merchantId?: string
    consumerId?: string
    Options: {
      trancheCount?: integer
      period?: enum[Day, Month, Year, Quarter]
      daysInPeriod?: integer
    }
    Supplier: {
      phone: string
      name: string
      taxCode: string
    }
    recurring?: boolean
  }[]
}
```

### #/components/schemas/AcquiringSubscriptionOutputOptions

```ts
{
  trancheCount?: integer
  period?: enum[Day, Month, Year, Quarter]
  daysInPeriod?: integer
}
```

### #/components/schemas/AcquiringSubscriptionPeriodInput

```ts
{
  "type": "string",
  "enum": [
    "Day",
    "Month",
    "Year"
  ],
  "title": "AcquiringSubscriptionPeriodInput"
}
```

### #/components/schemas/AcquiringSubscriptionPeriodOutput

```ts
{
  "type": "string",
  "enum": [
    "Day",
    "Month",
    "Year",
    "Quarter"
  ],
  "title": "AcquiringSubscriptionPeriodOutput"
}
```

### #/components/schemas/AcquiringSubscriptionStatus

```ts
{
  "type": "string",
  "enum": [
    "Active",
    "Cancelled",
    "Completed",
    "Expired",
    "Failed",
    "PastDue",
    "Preparing",
    "Refused",
    "Rejected",
    "Suspended",
    "Trial"
  ],
  "title": "AcquiringSubscriptionStatus"
}
```

### #/components/schemas/AcquiringSubscriptionStatusInput

```ts
{
  "type": "string",
  "enum": [
    "Cancelled"
  ],
  "title": "AcquiringSubscriptionStatusInput"
}
```

### #/components/schemas/ActModel

```ts
{
  Positions: {
    positionName: string
    unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
    ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
    price: number
    quantity: number
    totalAmount: number
    totalNds?: number
  }[]
  date?: string
  totalAmount: number
  totalNds?: number
  number: string
  basedOn?: string
}
```

### #/components/schemas/ActivateCashboxQrCodeRequestDataModel

```ts
{
  Data: {
    amount: integer
    // Валюта операции
    currency?: string //default: RUB
    paymentPurpose?: string
    ttl?: integer //default: 5
  }
}
```

### #/components/schemas/ActivateCashboxQrCodeRequestModel

```ts
{
  amount: integer
  // Валюта операции
  currency?: string //default: RUB
  paymentPurpose?: string
  ttl?: integer //default: 5
}
```

### #/components/schemas/ActivateCashboxQrCodeResponseDataModel

```ts
{
  Data: {
    qrcId: string
    amount: integer
    currency?: string //default: RUB
    paramsId: string
    paymentPurpose?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/ActivateCashboxQrCodeResponseModel

```ts
{
  qrcId: string
  amount: integer
  currency?: string //default: RUB
  paramsId: string
  paymentPurpose?: string
}
```

### #/components/schemas/ApiVersion

```ts
{
  "type": "string",
  "enum": [
    "v1.0"
  ],
  "title": "ApiVersion"
}
```

### #/components/schemas/B2BQRCodeModelResponseModel

```ts
{
  Data: {
    payload: string
    qrcId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/B2BQrCode

```ts
{
  payload: string
  accountId: string
  createdAt: string
  merchantId: string
  legalId: string
  qrcId: string
  amount?: integer
  ttl?: string
  paymentPurpose?: string
  image: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
    content: string
  }
  redirectUrl?: string
  // Наличие НДС
  takeTax: boolean
  // Сумма НДС в копейках
  totalTaxAmount?: integer
  uip?: string
}
```

### #/components/schemas/B2BQrCodeResponseModel

```ts
{
  Data: {
    payload: string
    accountId: string
    createdAt: string
    merchantId: string
    legalId: string
    qrcId: string
    amount?: integer
    ttl?: string
    paymentPurpose?: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    redirectUrl?: string
    // Наличие НДС
    takeTax: boolean
    // Сумма НДС в копейках
    totalTaxAmount?: integer
    uip?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/BadRequestError

```ts
{
  errorCode: string
  message: string
  url: string
}
```

### #/components/schemas/BadRequestErrorResponse

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

### #/components/schemas/BalanceAmountModel

```ts
{
  amount: number
  currency: string
}
```

### #/components/schemas/BalanceListModel

```ts
{
  Balance: {
    accountId: string
    creditDebitIndicator: enum[Credit, Debit]
    // **Описание типов балансов**
    // 
    //   - `OpeningAvailable` - Начальный остаток
    //   - `ClosingAvailable` - Доступный баланс
    //   - `Expected` - Сумма заблокированных средств
    //   - `OverdraftAvailable` - Доступный лимит по овердрафту
    type: enum[OpeningAvailable, ClosingAvailable, Expected, OverdraftAvailable]
    dateTime: string
    Amount: {
      amount: number
      currency: string
    }
  }[]
}
```

### #/components/schemas/BalanceListResponseModel

```ts
// Метод получения баланса
{
  Data: {
    Balance: {
      accountId: string
      creditDebitIndicator: enum[Credit, Debit]
      // **Описание типов балансов**
      // 
      //   - `OpeningAvailable` - Начальный остаток
      //   - `ClosingAvailable` - Доступный баланс
      //   - `Expected` - Сумма заблокированных средств
      //   - `OverdraftAvailable` - Доступный лимит по овердрафту
      type: enum[OpeningAvailable, ClosingAvailable, Expected, OverdraftAvailable]
      dateTime: string
      Amount: {
        amount: number
        currency: string
      }
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/BalanceModel

```ts
{
  accountId: string
  creditDebitIndicator: enum[Credit, Debit]
  // **Описание типов балансов**
  // 
  //   - `OpeningAvailable` - Начальный остаток
  //   - `ClosingAvailable` - Доступный баланс
  //   - `Expected` - Сумма заблокированных средств
  //   - `OverdraftAvailable` - Доступный лимит по овердрафту
  type: enum[OpeningAvailable, ClosingAvailable, Expected, OverdraftAvailable]
  dateTime: string
  Amount: {
    amount: number
    currency: string
  }
}
```

### #/components/schemas/BooleanResponse

```ts
{
  result: boolean
}
```

### #/components/schemas/BooleanResponseModel

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/CapabilitiesEnum

```ts
{
  "type": "string",
  "enum": [
    "001",
    "010",
    "011",
    "100",
    "101",
    "110",
    "111"
  ],
  "title": "CapabilitiesEnum",
  "description": "`001` - только QR Static\n`010` - только QR Dynamic\n`011` - QR Static и QR Dynamic\n`100` - Только QR Subscription\n`101` - QR Subscription и QR Static\n`110` - QR Subscription и QR Dynamic\n`111` - QR Static, QR Dynamic и QR Subscription"
}
```

### #/components/schemas/CardTransactionAmountModel

```ts
{
  // Сумма транзакции
  amount: number
  // Валюта транзакции, используется ISO 4217
  currency: string
}
```

### #/components/schemas/CardTransactionListModel

```ts
{
  Transactions: {
    accountId: string
    // Маскированный номер карты транзакции
    pan: string
    dateTime: string
    Amount: {
      // Сумма транзакции
      amount: number
      // Валюта транзакции, используется ISO 4217
      currency: string
    }
    AccountAmount:#/components/schemas/CardTransactionAmountModel
    TerminalData: {
      // Город терминала
      city?: string
      // Адрес терминала
      location?: string
      // Название торговой точки
      owner?: string
    }
  }[]
}
```

### #/components/schemas/CardTransactionModel

```ts
{
  accountId: string
  // Маскированный номер карты транзакции
  pan: string
  dateTime: string
  Amount: {
    // Сумма транзакции
    amount: number
    // Валюта транзакции, используется ISO 4217
    currency: string
  }
  AccountAmount:#/components/schemas/CardTransactionAmountModel
  TerminalData: {
    // Город терминала
    city?: string
    // Адрес терминала
    location?: string
    // Название торговой точки
    owner?: string
  }
}
```

### #/components/schemas/CardTransactionTerminalData

```ts
{
  // Город терминала
  city?: string
  // Адрес терминала
  location?: string
  // Название торговой точки
  owner?: string
}
```

### #/components/schemas/CardTransactionsListResponseModel

```ts
{
  Data: {
    Transactions: {
      accountId: string
      // Маскированный номер карты транзакции
      pan: string
      dateTime: string
      Amount: {
        // Сумма транзакции
        amount: number
        // Валюта транзакции, используется ISO 4217
        currency: string
      }
      AccountAmount:#/components/schemas/CardTransactionAmountModel
      TerminalData: {
        // Город терминала
        city?: string
        // Адрес терминала
        location?: string
        // Название торговой точки
        owner?: string
      }
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/CashAccountInfoModel

```ts
{
  schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
  identification?: string
}
```

### #/components/schemas/CashboxQrCodeOutputCommission

```ts
{
  mcc?: string
  percent?: number
  description?: string
}
```

### #/components/schemas/CashboxQrCodeResponseModel

```ts
{
  payload: string
  accountId: string
  merchantId: string
  legalId: string
  createdAt: string
  qrcId: string
  amount?: integer
  currency?: string //default: RUB
  paymentPurpose?: string
  paramsId?: string
  ttl?: integer
  commission: {
    mcc?: string
    percent?: number
    description?: string
  }
  image: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
    content: string
  }
  redirectUrl?: string
}
```

### #/components/schemas/ChangeCashboxQRCodeAccountRequestDataModel

```ts
{
  Data: {
    accountId: string
  }
}
```

### #/components/schemas/ChangeCashboxQRCodeAccountRequestModel

```ts
{
  accountId: string
}
```

### #/components/schemas/ChangeCashboxQRCodeAccountResponseDataModel

```ts
{
  Data: {
    accountId: string
    qrcId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/ChangeCashboxQRCodeAccountResponseModel

```ts
{
  accountId: string
  qrcId: string
}
```

### #/components/schemas/ClosingDocumentCreateRequestDataModel

```ts
{
  Data: {
    accountId: string
    customerCode: string
    SecondSide: {
      accountId?: string
      legalAddress?: string
      kpp?: string
      bankName?: string
      bankCorrAccount?: string
      taxCode: string
      type: enum[ip, company]
      secondSideName?: string
    }
    documentId?: string
    Content: Partial(#/components/schemas/ContentAct) & Partial(#/components/schemas/ContentPackingList) & Partial(#/components/schemas/ContentInvoicef)
  }
}
```

### #/components/schemas/ClosingDocumentCreateRequestModel

```ts
{
  accountId: string
  customerCode: string
  SecondSide: {
    accountId?: string
    legalAddress?: string
    kpp?: string
    bankName?: string
    bankCorrAccount?: string
    taxCode: string
    type: enum[ip, company]
    secondSideName?: string
  }
  documentId?: string
  Content: Partial(#/components/schemas/ContentAct) & Partial(#/components/schemas/ContentPackingList) & Partial(#/components/schemas/ContentInvoicef)
}
```

### #/components/schemas/ConsentCreateRequest

```ts
{
  Data: {
    status?: string //default: AwaitingAuthorisation
    creationDateTime?: string
    statusUpdateDateTime?: string
    permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
    expirationDateTime?: string
  }
  Risks: {
  }
}
```

### #/components/schemas/ConsentCreateRequestModel

```ts
{
  status?: string //default: AwaitingAuthorisation
  creationDateTime?: string
  statusUpdateDateTime?: string
  permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
  expirationDateTime?: string
}
```

### #/components/schemas/ConsentListModel

```ts
{
  Consent: {
    status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
    creationDateTime?: string
    statusUpdateDateTime?: string
    permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, ReadSBPData1C, EditSBPData1C, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
    expirationDateTime?: string
    consentId: string
    customerCode?: string
    applicationName?: string
    clientId?: string
    // Показывает истек срок разрешения или нет
    isValid?: boolean
  }[]
}
```

### #/components/schemas/ConsentListResponse

```ts
{
  Data: {
    Consent: {
      status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
      creationDateTime?: string
      statusUpdateDateTime?: string
      permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, ReadSBPData1C, EditSBPData1C, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
      expirationDateTime?: string
      consentId: string
      customerCode?: string
      applicationName?: string
      clientId?: string
      // Показывает истек срок разрешения или нет
      isValid?: boolean
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/ConsentModel

```ts
{
  status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
  creationDateTime?: string
  statusUpdateDateTime?: string
  permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
  expirationDateTime?: string
  consentId: string
  customerCode?: string
  applicationName?: string
  clientId?: string
}
```

### #/components/schemas/ConsentResponse

```ts
{
  Data: {
    status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
    creationDateTime?: string
    statusUpdateDateTime?: string
    permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
    expirationDateTime?: string
    consentId: string
    customerCode?: string
    applicationName?: string
    clientId?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/ConsentResponseModel

```ts
{
  status?: enum[AwaitingAuthorisation, Authorised, Rejected, Revoked]
  creationDateTime?: string
  statusUpdateDateTime?: string
  permissions?: enum[ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements, ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits, ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData, ReadSBPData1C, EditSBPData1C, CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData, MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData, MakeCustomer][]
  expirationDateTime?: string
  consentId: string
  customerCode?: string
  applicationName?: string
  clientId?: string
  // Показывает истек срок разрешения или нет
  isValid?: boolean
}
```

### #/components/schemas/ConsentStatusEnum

```ts
{
  "type": "string",
  "enum": [
    "AwaitingAuthorisation",
    "Authorised",
    "Rejected",
    "Revoked"
  ],
  "title": "ConsentStatusEnum"
}
```

### #/components/schemas/ConsentTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "ReadAccountsBasic",
    "ReadAccountsDetail",
    "ReadBalances",
    "ReadStatements",
    "ReadTransactionsBasic",
    "ReadTransactionsCredits",
    "ReadTransactionsDebits",
    "ReadTransactionsDetail",
    "ReadCustomerData",
    "ReadSBPData",
    "EditSBPData",
    "ReadSBPData1C",
    "EditSBPData1C",
    "CreatePaymentForSign",
    "CreatePaymentOrder",
    "ReadAcquiringData",
    "MakeAcquiringOperation",
    "ManageInvoiceData",
    "ManageWebhookData",
    "MakeCustomer"
  ],
  "title": "ConsentTypeEnum"
}
```

### #/components/schemas/ContentAct

```ts
{
  Act: {
    Positions: {
      positionName: string
      unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
      ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
      price: number
      quantity: number
      totalAmount: number
      totalNds?: number
    }[]
    date?: string
    totalAmount: number
    totalNds?: number
    number: string
    basedOn?: string
  }
}
```

### #/components/schemas/ContentInvoice

```ts
{
  Invoice: {
    Positions: {
      positionName: string
      unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
      ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
      price: number
      quantity: number
      totalAmount: number
      totalNds?: number
    }[]
    date?: string
    totalAmount: number
    totalNds?: number
    number: string
    basedOn?: string
    comment?: string
    paymentExpiryDate?: string
  }
}
```

### #/components/schemas/ContentInvoicef

```ts
{
  Invoicef: {
    Positions: {
      positionName: string
      unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
      ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
      price: number
      quantity: number
      totalAmount: number
      totalNds?: number
    }[]
    date?: string
    totalAmount: number
    totalNds?: number
    number: string
    basedOn?: string
    shipmentDocuments: {
      name: string
      date: string
      number: string
    }[]
  }
}
```

### #/components/schemas/ContentPackingList

```ts
{
  PackingList: {
    Positions: {
      positionName: string
      unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
      ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
      price: number
      quantity: number
      totalAmount: number
      totalNds?: number
    }[]
    date?: string
    totalAmount: number
    totalNds?: number
    number: string
    basedOn?: string
  }
}
```

### #/components/schemas/ContractorBankInfoModel

```ts
{
  schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
  identification?: string
  accountIdentification?: string
  name?: string
}
```

### #/components/schemas/ContractorInfoModel

```ts
{
  inn?: string
  name?: string
  kpp?: string
}
```

### #/components/schemas/CounterpartTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "ip",
    "company"
  ],
  "title": "CounterpartTypeEnum"
}
```

### #/components/schemas/CustomerCodeAndBankCode

```ts
{
  customerCode: string
  bankCode: string
}
```

### #/components/schemas/CustomerCodeAndBankCodeRequest

```ts
{
  Data: {
    customerCode: string
    bankCode: string
  }
}
```

### #/components/schemas/CustomerInfoResponseV3

```ts
{
  status: enum[Active, Suspended]
  legalId: string
  createdAt: string
  address?: string
  city?: string
  countryCode: string
  countrySubDivisionCode?: string
  zipCode?: string
  customerCode: string
  entityType?: string
  inn: string
  kpp?: string
  name: string
  ogrn: string
  bankCode: string
  MerchantList: {
    status:#/components/schemas/StatusEnum
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    merchantId: string
    legalId: string
    brandName: string
    // `001` - только QR Static
    // `010` - только QR Dynamic
    // `011` - QR Static и QR Dynamic
    // `100` - Только QR Subscription
    // `101` - QR Subscription и QR Static
    // `110` - QR Subscription и QR Dynamic
    // `111` - QR Static, QR Dynamic и QR Subscription
    capabilities: enum[001, 010, 011, 100, 101, 110, 111]
    contactPhoneNumber?: string
    mcc: string
    additionalContacts: {
    }[]
  }[]
  AccountList: {
    accountId: string
    status:#/components/schemas/StatusEnum
    createdAt: string
    legalId: string
  }[]
}
```

### #/components/schemas/CustomerListModel

```ts
{
  Customer: {
    customerCode: string
    customerType: enum[Business, Personal]
    isResident: boolean
    taxCode?: string
    fullName: string
    shortName?: string
    kpp?: string
    customerOgrn?: string
  }[]
}
```

### #/components/schemas/CustomerListResponseModel

```ts
// Метод получения списка доступных клиентов
{
  Data: {
    Customer: {
      customerCode: string
      customerType: enum[Business, Personal]
      isResident: boolean
      taxCode?: string
      fullName: string
      shortName?: string
      kpp?: string
      customerOgrn?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/CustomerModel

```ts
{
  customerCode: string
  customerType: enum[Business, Personal]
  isResident: boolean
  taxCode?: string
  fullName: string
  shortName?: string
  kpp?: string
  customerOgrn?: string
}
```

### #/components/schemas/CustomerResponseModel

```ts
// Метод получения информации по клиенту
{
  Data: {
    customerCode: string
    customerType: enum[Business, Personal]
    isResident: boolean
    taxCode?: string
    fullName: string
    shortName?: string
    kpp?: string
    customerOgrn?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/DeactivateCashboxQrCodeResponseDataModel

```ts
{
  Data: {
    result: boolean
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/DocumentCreateResponse

```ts
{
  documentId: string
}
```

### #/components/schemas/DocumentCreateResponseModel

```ts
{
  Data: {
    documentId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/ErrorModel

```ts
{
  errorCode: string
  message: string
  url: string
}
```

### #/components/schemas/ErrorResponse

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

### #/components/schemas/ExternalAccountStatusEnum

```ts
{
  "type": "string",
  "enum": [
    "Enabled",
    "Disabled",
    "Deleted",
    "ProForma",
    "Pending"
  ],
  "title": "ExternalAccountStatusEnum"
}
```

### #/components/schemas/ExternalAccountSubTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "CreditCard",
    "CurrentAccount",
    "Loan",
    "Mortgage",
    "PrePaidCard",
    "Savings",
    "Special"
  ],
  "title": "ExternalAccountSubTypeEnum"
}
```

### #/components/schemas/ExternalAcquiringPaymentTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "sbp",
    "card",
    "tinkoff"
  ],
  "title": "ExternalAcquiringPaymentTypeEnum"
}
```

### #/components/schemas/ExternalBalanceStaticTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "OpeningAvailable",
    "ClosingAvailable",
    "Expected",
    "OverdraftAvailable"
  ],
  "title": "ExternalBalanceStaticTypeEnum",
  "description": "**Описание типов балансов**\n\n  - `OpeningAvailable` - Начальный остаток\n  - `ClosingAvailable` - Доступный баланс\n  - `Expected` - Сумма заблокированных средств\n  - `OverdraftAvailable` - Доступный лимит по овердрафту"
}
```

### #/components/schemas/ExternalBalanceTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "Credit",
    "Debit"
  ],
  "title": "ExternalBalanceTypeEnum"
}
```

### #/components/schemas/ExternalConsentTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "ReadAccountsBasic",
    "ReadAccountsDetail",
    "ReadBalances",
    "ReadStatements",
    "ReadTransactionsBasic",
    "ReadTransactionsCredits",
    "ReadTransactionsDebits",
    "ReadTransactionsDetail",
    "ReadCustomerData",
    "ReadSBPData",
    "EditSBPData",
    "CreatePaymentForSign",
    "CreatePaymentOrder",
    "ReadAcquiringData",
    "MakeAcquiringOperation",
    "ManageInvoiceData",
    "ManageWebhookData",
    "MakeCustomer"
  ],
  "title": "ExternalConsentTypeEnum"
}
```

### #/components/schemas/ExternalCreditDebitIndicatorEnum

```ts
{
  "type": "string",
  "enum": [
    "Credit",
    "Debit"
  ],
  "title": "ExternalCreditDebitIndicatorEnum"
}
```

### #/components/schemas/ExternalTransactionStatusEnum

```ts
{
  "type": "string",
  "enum": [
    "Booked",
    "Pending"
  ],
  "title": "ExternalTransactionStatusEnum"
}
```

### #/components/schemas/ExternalTransationTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "Неопределенное значение",
    "Платежное поручение",
    "Платежное требование",
    "Денежный чек, РКО",
    "Объявление на взнос наличными, ПКО",
    "Требование-поручение",
    "Инкассовое поручение",
    "Расчетный чек",
    "Аккредитив",
    "Мемориальный ордер",
    "Погашение кредита",
    "Выдача кредита",
    "Авизо",
    "Банковские карты",
    "Платежный ордер",
    "Банковский ордер",
    "Ордер по передаче ценностей",
    "Программный ордер",
    "Импортированная запись"
  ],
  "title": "ExternalTransationTypeEnum"
}
```

### #/components/schemas/ExternalTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "Business",
    "Personal"
  ],
  "title": "ExternalTypeEnum"
}
```

### #/components/schemas/FinancialInstitutionIdentificationEnum

```ts
{
  "type": "string",
  "enum": [
    "RU.CBR.BICFI",
    "RU.CBR.BIK"
  ],
  "title": "FinancialInstitutionIdentificationEnum"
}
```

### #/components/schemas/ForbiddenError

```ts
{
  errorCode: string
  message: string
  url: string
}
```

### #/components/schemas/ForbiddenErrorResponse

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

### #/components/schemas/GetCashboxQRCodeListResponseDataModel

```ts
{
  Data: {
    qrCodes: {
      payload: string
      accountId: string
      merchantId: string
      legalId: string
      createdAt: string
      qrcId: string
      amount?: integer
      currency?: string
      paymentPurpose?: string
      paramsId?: string
      ttl?: integer
      commission: {
        mcc?: string
        percent?: number
        description?: string
      }
      redirectUrl?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/GetCashboxQRCodeListResponseItemModel

```ts
{
  payload: string
  accountId: string
  merchantId: string
  legalId: string
  createdAt: string
  qrcId: string
  amount?: integer
  currency?: string
  paymentPurpose?: string
  paramsId?: string
  ttl?: integer
  commission: {
    mcc?: string
    percent?: number
    description?: string
  }
  redirectUrl?: string
}
```

### #/components/schemas/GetCashboxQRCodeListResponseModel

```ts
{
  qrCodes: {
    payload: string
    accountId: string
    merchantId: string
    legalId: string
    createdAt: string
    qrcId: string
    amount?: integer
    currency?: string
    paymentPurpose?: string
    paramsId?: string
    ttl?: integer
    commission: {
      mcc?: string
      percent?: number
      description?: string
    }
    redirectUrl?: string
  }[]
}
```

### #/components/schemas/GetCashboxQRCodeRequestDataModel

```ts
{
  Data: {
    imageParams: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
    }
  }
}
```

### #/components/schemas/GetCashboxQRCodeRequestModel

```ts
{
  imageParams: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
  }
}
```

### #/components/schemas/GetCashboxQrCodeOperationInfoResponseDataModel

```ts
{
  Data: {
    qrCodeStatus: enum[DEACTIVATED, WAITING_PAYMENT, IN_PROGRESS]
    // * ACWP - Операция завершена успешно
    // * RJCT - Операция отклонена
    // * RCVD - Операция в обработке
    // * NTST - Операции по QR-коду не существует
    trxStatus: enum[ACWP, RJCT, RCVD, NTST]
    trxId?: string
    // Целое, положительное число. Валюта операции – рубли РФ
    amount?: integer
    dateTime?: string
    payerId?: string
    kzo?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/GetCashboxQrCodeOperationInfoResponseModel

```ts
{
  qrCodeStatus: enum[DEACTIVATED, WAITING_PAYMENT, IN_PROGRESS]
  // * ACWP - Операция завершена успешно
  // * RJCT - Операция отклонена
  // * RCVD - Операция в обработке
  // * NTST - Операции по QR-коду не существует
  trxStatus: enum[ACWP, RJCT, RCVD, NTST]
  trxId?: string
  // Целое, положительное число. Валюта операции – рубли РФ
  amount?: integer
  dateTime?: string
  payerId?: string
  kzo?: string
}
```

### #/components/schemas/GetCashboxQrCodeResponseDataModel

```ts
{
  Data: {
    payload: string
    accountId: string
    merchantId: string
    legalId: string
    createdAt: string
    qrcId: string
    amount?: integer
    currency?: string //default: RUB
    paymentPurpose?: string
    paramsId?: string
    ttl?: integer
    commission: {
      mcc?: string
      percent?: number
      description?: string
    }
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    redirectUrl?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/GetCashboxQrCodeStatusResponseDataModel

```ts
{
  Data: {
    status: enum[INACTIVATED, WAITING_PAYMENT, IN_PROGRESS]
    paramsId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/GetCashboxQrCodeStatusResponseModel

```ts
{
  status: enum[INACTIVATED, WAITING_PAYMENT, IN_PROGRESS]
  paramsId: string
}
```

### #/components/schemas/GetCustomerInfoResponseModelV3

```ts
{
  Data: {
    status: enum[Active, Suspended]
    legalId: string
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    customerCode: string
    entityType?: string
    inn: string
    kpp?: string
    name: string
    ogrn: string
    bankCode: string
    MerchantList: {
      status:#/components/schemas/StatusEnum
      createdAt: string
      address?: string
      city?: string
      countryCode: string
      countrySubDivisionCode?: string
      zipCode?: string
      merchantId: string
      legalId: string
      brandName: string
      // `001` - только QR Static
      // `010` - только QR Dynamic
      // `011` - QR Static и QR Dynamic
      // `100` - Только QR Subscription
      // `101` - QR Subscription и QR Static
      // `110` - QR Subscription и QR Dynamic
      // `111` - QR Static, QR Dynamic и QR Subscription
      capabilities: enum[001, 010, 011, 100, 101, 110, 111]
      contactPhoneNumber?: string
      mcc: string
      additionalContacts: {
      }[]
    }[]
    AccountList: {
      accountId: string
      status:#/components/schemas/StatusEnum
      createdAt: string
      legalId: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/InitStatementModel

```ts
{
  accountId: string
  statementId?: string
  status: enum[Created, Processing, Error, Ready]
  startDateTime: string
  endDateTime: string
  creationDateTime: string
}
```

### #/components/schemas/InvoiceCreateRequestDataModel

```ts
{
  Data: {
    accountId: string
    customerCode: string
    SecondSide: {
      accountId?: string
      legalAddress?: string
      kpp?: string
      bankName?: string
      bankCorrAccount?: string
      taxCode: string
      type: enum[ip, company]
      secondSideName?: string
    }
    Content: {
      Invoice: {
        Positions: {
          positionName: string
          unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
          ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
          price: number
          quantity: number
          totalAmount: number
          totalNds?: number
        }[]
        date?: string
        totalAmount: number
        totalNds?: number
        number: string
        basedOn?: string
        comment?: string
        paymentExpiryDate?: string
      }
    }
  }
}
```

### #/components/schemas/InvoiceCreateRequestModel

```ts
{
  accountId: string
  customerCode: string
  SecondSide: {
    accountId?: string
    legalAddress?: string
    kpp?: string
    bankName?: string
    bankCorrAccount?: string
    taxCode: string
    type: enum[ip, company]
    secondSideName?: string
  }
  Content: {
    Invoice: {
      Positions: {
        positionName: string
        unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
        ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
        price: number
        quantity: number
        totalAmount: number
        totalNds?: number
      }[]
      date?: string
      totalAmount: number
      totalNds?: number
      number: string
      basedOn?: string
      comment?: string
      paymentExpiryDate?: string
    }
  }
}
```

### #/components/schemas/InvoiceModel

```ts
{
  Positions: {
    positionName: string
    unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
    ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
    price: number
    quantity: number
    totalAmount: number
    totalNds?: number
  }[]
  date?: string
  totalAmount: number
  totalNds?: number
  number: string
  basedOn?: string
  comment?: string
  paymentExpiryDate?: string
}
```

### #/components/schemas/InvoicePaymentStatusResponse

```ts
{
  paymentStatus: enum[payment_waiting, payment_expired, payment_paid]
}
```

### #/components/schemas/InvoicePaymentStatusResponseModel

```ts
{
  Data: {
    paymentStatus: enum[payment_waiting, payment_expired, payment_paid]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/InvoicefModel

```ts
{
  Positions: {
    positionName: string
    unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
    ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
    price: number
    quantity: number
    totalAmount: number
    totalNds?: number
  }[]
  date?: string
  totalAmount: number
  totalNds?: number
  number: string
  basedOn?: string
  shipmentDocuments: {
    name: string
    date: string
    number: string
  }[]
}
```

### #/components/schemas/LegalEntity

```ts
{
  status: enum[Active, Suspended]
  legalId: string
  createdAt: string
  address?: string
  city?: string
  countryCode: string
  countrySubDivisionCode?: string
  zipCode?: string
  customerCode: string
  entityType?: string
  inn: string
  kpp?: string
  name: string
  ogrn: string
}
```

### #/components/schemas/LegalEntityResponseModel

```ts
{
  Data: {
    status: enum[Active, Suspended]
    legalId: string
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    customerCode: string
    entityType?: string
    inn: string
    kpp?: string
    name: string
    ogrn: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/LinkModel

```ts
{
  self: string
}
```

### #/components/schemas/Measure

```ts
{
  "type": "string",
  "enum": [
    "г.",
    "кг.",
    "т.",
    "см.",
    "дм.",
    "м.",
    "см2.",
    "дм2.",
    "м2.",
    "мл.",
    "л.",
    "м3",
    "кВт.ч.",
    "Гкал.",
    "дн.",
    "ч.",
    "мин.",
    "сек.",
    "Кб.",
    "Мб.",
    "Гб.",
    "Тб.",
    "шт."
  ],
  "title": "Measure"
}
```

### #/components/schemas/MediaTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "image/png",
    "image/svg+xml"
  ],
  "title": "MediaTypeEnum"
}
```

### #/components/schemas/Merchant

```ts
{
  status: enum[Active, Suspended]
  createdAt: string
  address?: string
  city?: string
  countryCode: string
  countrySubDivisionCode?: string
  zipCode?: string
  merchantId: string
  legalId: string
  brandName: string
  // `001` - только QR Static
  // `010` - только QR Dynamic
  // `011` - QR Static и QR Dynamic
  // `100` - Только QR Subscription
  // `101` - QR Subscription и QR Static
  // `110` - QR Subscription и QR Dynamic
  // `111` - QR Static, QR Dynamic и QR Subscription
  capabilities: enum[001, 010, 011, 100, 101, 110, 111]
  contactPhoneNumber?: string
  mcc: string
  additionalContacts: {
  }[]
}
```

### #/components/schemas/MerchantId

```ts
{
  merchantId: string
}
```

### #/components/schemas/MerchantIdResponseModel

```ts
{
  Data: {
    merchantId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/MerchantListResponse

```ts
{
  MerchantList: {
    status: enum[Active, Suspended]
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    merchantId: string
    legalId: string
    brandName: string
    // `001` - только QR Static
    // `010` - только QR Dynamic
    // `011` - QR Static и QR Dynamic
    // `100` - Только QR Subscription
    // `101` - QR Subscription и QR Static
    // `110` - QR Subscription и QR Dynamic
    // `111` - QR Static, QR Dynamic и QR Subscription
    capabilities: enum[001, 010, 011, 100, 101, 110, 111]
    contactPhoneNumber?: string
    mcc: string
    additionalContacts: {
    }[]
  }[]
}
```

### #/components/schemas/MerchantListResponseModel

```ts
{
  Data: {
    MerchantList: {
      status: enum[Active, Suspended]
      createdAt: string
      address?: string
      city?: string
      countryCode: string
      countrySubDivisionCode?: string
      zipCode?: string
      merchantId: string
      legalId: string
      brandName: string
      // `001` - только QR Static
      // `010` - только QR Dynamic
      // `011` - QR Static и QR Dynamic
      // `100` - Только QR Subscription
      // `101` - QR Subscription и QR Static
      // `110` - QR Subscription и QR Dynamic
      // `111` - QR Static, QR Dynamic и QR Subscription
      capabilities: enum[001, 010, 011, 100, 101, 110, 111]
      contactPhoneNumber?: string
      mcc: string
      additionalContacts: {
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/MerchantResponseModel

```ts
{
  Data: {
    status: enum[Active, Suspended]
    createdAt: string
    address?: string
    city?: string
    countryCode: string
    countrySubDivisionCode?: string
    zipCode?: string
    merchantId: string
    legalId: string
    brandName: string
    // `001` - только QR Static
    // `010` - только QR Dynamic
    // `011` - QR Static и QR Dynamic
    // `100` - Только QR Subscription
    // `101` - QR Subscription и QR Static
    // `110` - QR Subscription и QR Dynamic
    // `111` - QR Static, QR Dynamic и QR Subscription
    capabilities: enum[001, 010, 011, 100, 101, 110, 111]
    contactPhoneNumber?: string
    mcc: string
    additionalContacts: {
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/MetaModel

```ts
{
  totalPages: integer
}
```

### #/components/schemas/NdsKindEnum

```ts
{
  "type": "string",
  "enum": [
    "nds_0",
    "nds_5",
    "nds_7",
    "nds_10",
    "nds_20",
    "without_nds"
  ],
  "title": "NdsKindEnum"
}
```

### #/components/schemas/NotFoundError

```ts
{
  errorCode: string
  message: string
  url: string
}
```

### #/components/schemas/NotFoundErrorResponse

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

### #/components/schemas/OptionalLegalId

```ts
{
  legalId: string
}
```

### #/components/schemas/OrderModel

```ts
{
  orderId: string
  type: enum[refund, approval, authorized]
  amount: number
  time: string
}
```

### #/components/schemas/OrderType

```ts
{
  "type": "string",
  "enum": [
    "refund",
    "approval",
    "authorized"
  ],
  "title": "OrderType"
}
```

### #/components/schemas/PackingListModel

```ts
{
  Positions: {
    positionName: string
    unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
    ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
    price: number
    quantity: number
    totalAmount: number
    totalNds?: number
  }[]
  date?: string
  totalAmount: number
  totalNds?: number
  number: string
  basedOn?: string
}
```

### #/components/schemas/PaginatedLinkModel

```ts
{
  self: string
  first?: string
  prev?: string
  next?: string
  last?: string
}
```

### #/components/schemas/PaymentForSignListItemModel

```ts
{
  accountCode?: string
  bankCode?: string
  counterpartyBankBic: string
  counterpartyAccountNumber: string
  // Допустимые значения "0", 10 или 12 значное число
  counterpartyINN?: string
  // Допустимые значения "0" или 9 значное число
  counterpartyKPP?: string
  counterpartyName: string
  paymentAmount: number
  paymentDate: string
  paymentNumber?: string
  paymentPurpose: string
  // Допустимые значения "0", 10 или 12 значное число
  payerINN?: string
  // Допустимые значения "0" или 9 значное число
  payerKPP?: string
  counterpartyBankCorrAccount?: string
  paymentPriority?: string //default: 5
  codePurpose?: string
  supplierBillId?: string
  budgetPaymentCode?: string
  email?: string
  taxInfoDocumentDate?: string
  taxInfoDocumentNumber?: string
  taxInfoKBK?: string
  taxInfoOKATO?: string
  taxInfoPeriod?: string
  taxInfoReasonCode?: string
  taxInfoStatus?: string
}
```

### #/components/schemas/PaymentForSignListResponseDataModel

```ts
{
  Data: {
    Payment: {
      accountCode?: string
      bankCode?: string
      counterpartyBankBic: string
      counterpartyAccountNumber: string
      // Допустимые значения "0", 10 или 12 значное число
      counterpartyINN?: string
      // Допустимые значения "0" или 9 значное число
      counterpartyKPP?: string
      counterpartyName: string
      paymentAmount: number
      paymentDate: string
      paymentNumber?: string
      paymentPurpose: string
      // Допустимые значения "0", 10 или 12 значное число
      payerINN?: string
      // Допустимые значения "0" или 9 значное число
      payerKPP?: string
      counterpartyBankCorrAccount?: string
      paymentPriority?: string //default: 5
      codePurpose?: string
      supplierBillId?: string
      budgetPaymentCode?: string
      email?: string
      taxInfoDocumentDate?: string
      taxInfoDocumentNumber?: string
      taxInfoKBK?: string
      taxInfoOKATO?: string
      taxInfoPeriod?: string
      taxInfoReasonCode?: string
      taxInfoStatus?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/PaymentForSignListResponseModel

```ts
{
  Payment: {
    accountCode?: string
    bankCode?: string
    counterpartyBankBic: string
    counterpartyAccountNumber: string
    // Допустимые значения "0", 10 или 12 значное число
    counterpartyINN?: string
    // Допустимые значения "0" или 9 значное число
    counterpartyKPP?: string
    counterpartyName: string
    paymentAmount: number
    paymentDate: string
    paymentNumber?: string
    paymentPurpose: string
    // Допустимые значения "0", 10 или 12 значное число
    payerINN?: string
    // Допустимые значения "0" или 9 значное число
    payerKPP?: string
    counterpartyBankCorrAccount?: string
    paymentPriority?: string //default: 5
    codePurpose?: string
    supplierBillId?: string
    budgetPaymentCode?: string
    email?: string
    taxInfoDocumentDate?: string
    taxInfoDocumentNumber?: string
    taxInfoKBK?: string
    taxInfoOKATO?: string
    taxInfoPeriod?: string
    taxInfoReasonCode?: string
    taxInfoStatus?: string
  }[]
}
```

### #/components/schemas/PaymentForSignRequestDataModel

```ts
{
  Data: {
    accountCode: string
    bankCode: string
    // Заполняется только при платеже за 3 лицо. Допустимые значения "0", 10 или 12 значное число
    payerINN?: string
    // Заполняется только за 3 лицо при платеже в бюджет. Допустимые значение "0" или 9 значное число
    payerKPP?: string
    counterpartyBankBic: string
    counterpartyAccountNumber: string
    counterpartyINN?: string
    // Допустимые значения "0" или 9 значное число
    counterpartyKPP?: string
    counterpartyName: string
    counterpartyBankCorrAccount?: string
    paymentAmount: number
    // Дата платежа, приведенная к часовому поясу Москвы
    paymentDate: string
    paymentNumber?: integer
    paymentPriority?: string //default: 5
    paymentPurpose: string
    // Заполняется только при платеже физ лицам на счета:('40810', '40817', '40823', '40824', '40826', '423', '30232', '40803', '40813', '40820', '426'). Допустимые значения 1,2,3,4,5 и пусто
    codePurpose?: string
    supplierBillId?: string
    taxInfoDocumentDate?: string
    taxInfoDocumentNumber?: string
    taxInfoKBK?: string
    taxInfoOKATO?: string
    taxInfoPeriod?: string
    taxInfoReasonCode?: string
    taxInfoStatus?: string
    budgetPaymentCode?: string
    email?: string
  }
}
```

### #/components/schemas/PaymentForSignRequestModel

```ts
{
  accountCode: string
  bankCode: string
  // Заполняется только при платеже за 3 лицо. Допустимые значения "0", 10 или 12 значное число
  payerINN?: string
  // Заполняется только за 3 лицо при платеже в бюджет. Допустимые значение "0" или 9 значное число
  payerKPP?: string
  counterpartyBankBic: string
  counterpartyAccountNumber: string
  counterpartyINN?: string
  // Допустимые значения "0" или 9 значное число
  counterpartyKPP?: string
  counterpartyName: string
  counterpartyBankCorrAccount?: string
  paymentAmount: number
  // Дата платежа, приведенная к часовому поясу Москвы
  paymentDate: string
  paymentNumber?: integer
  paymentPriority?: string //default: 5
  paymentPurpose: string
  // Заполняется только при платеже физ лицам на счета:('40810', '40817', '40823', '40824', '40826', '423', '30232', '40803', '40813', '40820', '426'). Допустимые значения 1,2,3,4,5 и пусто
  codePurpose?: string
  supplierBillId?: string
  taxInfoDocumentDate?: string
  taxInfoDocumentNumber?: string
  taxInfoKBK?: string
  taxInfoOKATO?: string
  taxInfoPeriod?: string
  taxInfoReasonCode?: string
  taxInfoStatus?: string
  budgetPaymentCode?: string
  email?: string
}
```

### #/components/schemas/PaymentForSignResponseDataModel

```ts
{
  Data: {
    requestId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/PaymentForSignResponseModel

```ts
{
  requestId: string
}
```

### #/components/schemas/PaymentMethod

```ts
{
  "type": "string",
  "enum": [
    "full_payment",
    "full_prepayment"
  ],
  "title": "PaymentMethod"
}
```

### #/components/schemas/PaymentObject

```ts
{
  "type": "string",
  "enum": [
    "goods",
    "service",
    "work"
  ],
  "title": "PaymentObject"
}
```

### #/components/schemas/PaymentOrderRequestDataModel

```ts
{
  Data: {
    accountCode?: string
    bankCode?: string
    // Заполняется только при платеже за 3 лицо. Допустимые значения "0", 10 или 12 значное число
    payerINN?: string
    // Заполняется при платеже в бюджет (за себя или 3 лицо). Допустимые значение "0" или 9 значное число
    payerKPP?: string
    counterpartyBankBic: string
    counterpartyAccountNumber: string
    counterpartyINN?: string
    // Допустимые значения "0" или 9 значное число
    counterpartyKPP?: string
    counterpartyName: string
    counterpartyBankCorrAccount?: string
    paymentAmount: number
    // Дата платежа, приведенная к часовому поясу Москвы
    paymentDate: string
    paymentNumber?: integer
    paymentPriority?: string //default: 5
    paymentPurpose: string
    // Заполняется только при платеже физ лицам на счета:('40810', '40817', '40823', '40824', '40826', '423', '30232', '40803', '40813', '40820', '426'). Допустимые значения 1,2,3,4,5 и пусто
    codePurpose?: string
    supplierBillId?: string
    taxInfoDocumentDate?: string
    taxInfoDocumentNumber?: string
    taxInfoKBK?: string
    taxInfoOKATO?: string
    taxInfoPeriod?: string
    taxInfoReasonCode?: string
    taxInfoStatus?: string
    budgetPaymentCode?: string
  }
}
```

### #/components/schemas/PaymentOrderRequestModel

```ts
{
  accountCode?: string
  bankCode?: string
  // Заполняется только при платеже за 3 лицо. Допустимые значения "0", 10 или 12 значное число
  payerINN?: string
  // Заполняется при платеже в бюджет (за себя или 3 лицо). Допустимые значение "0" или 9 значное число
  payerKPP?: string
  counterpartyBankBic: string
  counterpartyAccountNumber: string
  counterpartyINN?: string
  // Допустимые значения "0" или 9 значное число
  counterpartyKPP?: string
  counterpartyName: string
  counterpartyBankCorrAccount?: string
  paymentAmount: number
  // Дата платежа, приведенная к часовому поясу Москвы
  paymentDate: string
  paymentNumber?: integer
  paymentPriority?: string //default: 5
  paymentPurpose: string
  // Заполняется только при платеже физ лицам на счета:('40810', '40817', '40823', '40824', '40826', '423', '30232', '40803', '40813', '40820', '426'). Допустимые значения 1,2,3,4,5 и пусто
  codePurpose?: string
  supplierBillId?: string
  taxInfoDocumentDate?: string
  taxInfoDocumentNumber?: string
  taxInfoKBK?: string
  taxInfoOKATO?: string
  taxInfoPeriod?: string
  taxInfoReasonCode?: string
  taxInfoStatus?: string
  budgetPaymentCode?: string
}
```

### #/components/schemas/PaymentOrderResponseDataModel

```ts
{
  Data: {
    requestId: string
    redirectURL: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/PaymentOrderResponseModel

```ts
{
  requestId: string
  redirectURL: string
}
```

### #/components/schemas/PaymentOrderStatusEnum

```ts
{
  "type": "string",
  "enum": [
    "Initiated",
    "Wait For Owner Requisites",
    "NotAllowed",
    "Allowed",
    "WaitingForSign",
    "WaitingForCreate",
    "Created",
    "Paid",
    "Canceled",
    "Rejected"
  ],
  "title": "PaymentOrderStatusEnum",
  "description": "**Описание статусов платежа**\n\n  - `Initiated` - Все необходимые реквизиты для платежа получены, платёж готов к проверке на возможность проведения\n  - `Wait For Owner Requisites` - Часть реквизитов для платежа получена, кроме реквизитов плательщика\n  - `NotAllowed` - Платёж нельзя провести: либо у пользователя нет прав для подписи, либо платёж заблокирован\n  комплаенсом\n  - `Allowed` - Платёж готов к подписанию, все проверки пройдены\n  - `WaitingForSign` - Платёж ждёт подписи\n  - `WaitingForCreate` - Платёж подписан, ждёт создания внутри систем банка\n  - `Created` - Платёж создан\n  - `Paid` - Платёж оплачен\n  - `Canceled` - Платёж отменен\n  - `Rejected` - Платёж отменён"
}
```

### #/components/schemas/PaymentStatusResponseDataModel

```ts
{
  Data: {
    requestId: string
    // **Описание статусов платежа**
    // 
    //   - `Initiated` - Все необходимые реквизиты для платежа получены, платёж готов к проверке на возможность проведения
    //   - `Wait For Owner Requisites` - Часть реквизитов для платежа получена, кроме реквизитов плательщика
    //   - `NotAllowed` - Платёж нельзя провести: либо у пользователя нет прав для подписи, либо платёж заблокирован
    //   комплаенсом
    //   - `Allowed` - Платёж готов к подписанию, все проверки пройдены
    //   - `WaitingForSign` - Платёж ждёт подписи
    //   - `WaitingForCreate` - Платёж подписан, ждёт создания внутри систем банка
    //   - `Created` - Платёж создан
    //   - `Paid` - Платёж оплачен
    //   - `Canceled` - Платёж отменен
    //   - `Rejected` - Платёж отменён
    status: enum[Initiated, Wait For Owner Requisites, NotAllowed, Allowed, WaitingForSign, WaitingForCreate, Created, Paid, Canceled, Rejected]
[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/PaymentStatusResponseModel

```ts
{
  requestId: string
  // **Описание статусов платежа**
  // 
  //   - `Initiated` - Все необходимые реквизиты для платежа получены, платёж готов к проверке на возможность проведения
  //   - `Wait For Owner Requisites` - Часть реквизитов для платежа получена, кроме реквизитов плательщика
  //   - `NotAllowed` - Платёж нельзя провести: либо у пользователя нет прав для подписи, либо платёж заблокирован
  //   комплаенсом
  //   - `Allowed` - Платёж готов к подписанию, все проверки пройдены
  //   - `WaitingForSign` - Платёж ждёт подписи
  //   - `WaitingForCreate` - Платёж подписан, ждёт создания внутри систем банка
  //   - `Created` - Платёж создан
  //   - `Paid` - Платёж оплачен
  //   - `Canceled` - Платёж отменен
  //   - `Rejected` - Платёж отменён
  status: enum[Initiated, Wait For Owner Requisites, NotAllowed, Allowed, WaitingForSign, WaitingForCreate, Created, Paid, Canceled, Rejected]
[]
}
```

### #/components/schemas/PositionModel

```ts
{
  positionName: string
  unitCode: enum[шт., тыс.шт., компл., пар., усл.ед., упак., услуга., пач., мин., ч., сут., г., кг., л., м., м2., м3., км., га., кВт., кВт.ч.]
  ndsKind: enum[nds_0, nds_5, nds_7, nds_10, nds_20, without_nds]
  price: number
  quantity: number
  totalAmount: number
  totalNds?: number
}
```

### #/components/schemas/QRCodeListResponse

```ts
{
  qrCodeList: {
    status: enum[Active, Suspended]
    payload: string
    accountId: string
    createdAt: string
    merchantId: string
    legalId: string
    qrcId: string
    amount?: integer
    ttl?: string
    paymentPurpose?: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    commissionPercent: number
    currency?: string
    // 01 - QR-Static (QR наклейка)
    // 02 - QR-Dynamic (QR на кассе)
    qrcType: enum[01, 02]
    templateVersion: string
    sourceName?: string
  }[]
}
```

### #/components/schemas/QRCodeListResponseModel

```ts
{
  Data: {
    qrCodeList: {
      status: enum[Active, Suspended]
      payload: string
      accountId: string
      createdAt: string
      merchantId: string
      legalId: string
      qrcId: string
      amount?: integer
      ttl?: string
      paymentPurpose?: string
      image: {
        width: integer
        height: integer
        mediaType?: enum[image/png, image/svg+xml]
        content: string
      }
      commissionPercent: number
      currency?: string
      // 01 - QR-Static (QR наклейка)
      // 02 - QR-Dynamic (QR на кассе)
      qrcType: enum[01, 02]
      templateVersion: string
      sourceName?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/QRCodeModelResponseModel

```ts
{
  Data: {
    payload: string
    qrcId: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/QRCodePaymentStatusExternal

```ts
{
  "type": "string",
  "enum": [
    "NotStarted",
    "Received",
    "InProgress",
    "Accepted",
    "Rejected"
  ],
  "title": "QRCodePaymentStatusExternal",
  "description": "**Описание возможных статусов платежа**\n\n  - `NotStarted` - операции по QR-коду не существует\n  - `Received` - операция в обработке\n  - `InProgress` - операция в обработке\n  - `Accepted` - операция завершена успешно\n  - `Rejected` - операция отклонена"
}
```

### #/components/schemas/QRCodePaymentStatusListResponse

```ts
{
  paymentList: {
    qrcId: string
    code: string
    // **Описание возможных статусов платежа**
    // 
    //   - `NotStarted` - операции по QR-коду не существует
    //   - `Received` - операция в обработке
    //   - `InProgress` - операция в обработке
    //   - `Accepted` - операция завершена успешно
    //   - `Rejected` - операция отклонена
    status?: enum[NotStarted, Received, InProgress, Accepted, Rejected]
    message: string
    trxId?: string
  }[]
}
```

### #/components/schemas/QRCodePaymentStatusListResponseModel

```ts
{
  Data: {
    paymentList: {
      qrcId: string
      code: string
      // **Описание возможных статусов платежа**
      // 
      //   - `NotStarted` - операции по QR-коду не существует
      //   - `Received` - операция в обработке
      //   - `InProgress` - операция в обработке
      //   - `Accepted` - операция завершена успешно
      //   - `Rejected` - операция отклонена
      status?: enum[NotStarted, Received, InProgress, Accepted, Rejected]
      message: string
      trxId?: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/QRCodeRequestParams

```ts
{
  width: integer
  height: integer
  mediaType?: enum[image/png, image/svg+xml]
}
```

### #/components/schemas/QRCodeResponseModel

```ts
{
  Data: {
    status: enum[Active, Suspended]
    payload: string
    accountId: string
    createdAt: string
    merchantId: string
    legalId: string
    qrcId: string
    amount?: integer
    ttl?: string
    paymentPurpose?: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
    commissionPercent: number
    currency?: string
    // 01 - QR-Static (QR наклейка)
    // 02 - QR-Dynamic (QR на кассе)
    qrcType: enum[01, 02]
    templateVersion: string
    sourceName?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/QrCode

```ts
{
  status: enum[Active, Suspended]
  payload: string
  accountId: string
  createdAt: string
  merchantId: string
  legalId: string
  qrcId: string
  amount?: integer
  ttl?: string
  paymentPurpose?: string
  image: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
    content: string
  }
  commissionPercent: number
  currency?: string
  // 01 - QR-Static (QR наклейка)
  // 02 - QR-Dynamic (QR на кассе)
  qrcType: enum[01, 02]
  templateVersion: string
  sourceName?: string
}
```

### #/components/schemas/QrCodeContent

```ts
{
  width: integer
  height: integer
  mediaType?: enum[image/png, image/svg+xml]
  content: string
}
```

### #/components/schemas/QrCodeImageParams

```ts
{
  width: integer
  height: integer
  mediaType?: enum[image/png, image/svg+xml]
}
```

### #/components/schemas/QrCodePaymentStatus

```ts
{
  qrcId: string
  code: string
  // **Описание возможных статусов платежа**
  // 
  //   - `NotStarted` - операции по QR-коду не существует
  //   - `Received` - операция в обработке
  //   - `InProgress` - операция в обработке
  //   - `Accepted` - операция завершена успешно
  //   - `Rejected` - операция отклонена
  status?: enum[NotStarted, Received, InProgress, Accepted, Rejected]
  message: string
  trxId?: string
}
```

### #/components/schemas/QrTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "01",
    "02"
  ],
  "title": "QrTypeEnum",
  "description": "01 - QR-Static (QR наклейка)\n02 - QR-Dynamic (QR на кассе)"
}
```

### #/components/schemas/ReceiptClientModel

```ts
{
  name?: string
  email: string
  phone?: string
}
```

### #/components/schemas/ReceiptItemModel-Input

```ts
{
  vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
  name: string
  amount: number
  quantity: number
  paymentMethod?: enum[full_payment, full_prepayment]
  paymentObject?: enum[goods, service, work]
  measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
}
```

### #/components/schemas/ReceiptItemModel-Output

```ts
{
  vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
  name: string
  amount: number
  quantity: number
  paymentMethod?: enum[full_payment, full_prepayment]
  paymentObject?: enum[goods, service, work]
  measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
}
```

### #/components/schemas/ReceiptItemResponseModel

```ts
{
  vatType?: enum[none, vat0, vat5, vat7, vat10, vat20, vat105, vat107, vat110, vat120]
  name: string
  amount: number
  quantity: number
  paymentMethod?: enum[full_payment, full_prepayment]
  paymentObject?: enum[goods, service, work]
  measure?: enum[г., кг., т., см., дм., м., см2., дм2., м2., мл., л., м3, кВт.ч., Гкал., дн., ч., мин., сек., Кб., Мб., Гб., Тб., шт.]
}
```

### #/components/schemas/RegisterB2BQRCode

```ts
{
  amount: integer
  paymentPurpose: string
  // Система, создавшая QR-код
  sourceName: string
  // Наличие НДС
  takeTax: boolean
  // Сумма НДС в копейках
  totalTaxAmount?: integer
  ttl?: integer //default: 4320
  // Ссылка для автоматического возврата плательщика из приложения банка в приложение или на сайт ТСП
  redirectUrl?: string
  uip?: string
}
```

### #/components/schemas/RegisterB2BQRCodeRequest

```ts
{
  Data: {
    amount: integer
    paymentPurpose: string
    // Система, создавшая QR-код
    sourceName: string
    // Наличие НДС
    takeTax: boolean
    // Сумма НДС в копейках
    totalTaxAmount?: integer
    ttl?: integer //default: 4320
    // Ссылка для автоматического возврата плательщика из приложения банка в приложение или на сайт ТСП
    redirectUrl?: string
    uip?: string
  }
}
```

### #/components/schemas/RegisterCashboxQrCodeRequestDataModel

```ts
{
  Data: {
    merchantId: string
    accountId: string
    redirectUrl?: string
    imageParams: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
    }
  }
}
```

### #/components/schemas/RegisterCashboxQrCodeRequestModel

```ts
{
  merchantId: string
  accountId: string
  redirectUrl?: string
  imageParams: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
  }
}
```

### #/components/schemas/RegisterCashboxQrCodeResponseDataModel

```ts
{
  Data: {
    payload: string
    qrcId: string
    image: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
      content: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/RegisterCashboxQrCodeResponseModel

```ts
{
  payload: string
  qrcId: string
  image: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
    content: string
  }
}
```

### #/components/schemas/RegisterLegalEntityResponseModel

```ts
{
  Data: {
    legalId: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/RegisterMerchant

```ts
{
  address: string
  city: string
  countryCode: string
  countrySubDivisionCode: string
  zipCode: string
  brandName: string
  // `001` - только QR Static
  // `010` - только QR Dynamic
  // `011` - QR Static и QR Dynamic
  // `100` - Только QR Subscription
  // `101` - QR Subscription и QR Static
  // `110` - QR Subscription и QR Dynamic
  // `111` - QR Static, QR Dynamic и QR Subscription
  capabilities: enum[001, 010, 011, 100, 101, 110, 111]
  contactPhoneNumber?: string
  mcc: string
}
```

### #/components/schemas/RegisterMerchantRequest

```ts
{
  Data: {
    address: string
    city: string
    countryCode: string
    countrySubDivisionCode: string
    zipCode: string
    brandName: string
    // `001` - только QR Static
    // `010` - только QR Dynamic
    // `011` - QR Static и QR Dynamic
    // `100` - Только QR Subscription
    // `101` - QR Subscription и QR Static
    // `110` - QR Subscription и QR Dynamic
    // `111` - QR Static, QR Dynamic и QR Subscription
    capabilities: enum[001, 010, 011, 100, 101, 110, 111]
    contactPhoneNumber?: string
    mcc: string
  }
}
```

### #/components/schemas/RegisterQRCode

```ts
{
  // Поле обязательно для заполнения, если тип QR = QR-Dynamic
  amount?: integer
  currency?: string
  paymentPurpose: string
  // 01 - QR-Static (QR наклейка)
  // 02 - QR-Dynamic (QR на кассе)
  qrcType: enum[01, 02]
  imageParams: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
  }
  // Cистема, создавшая QR-код
  sourceName?: string
  // Задается, только если тип QR = QR-Dynamic
  ttl?: integer
  // Ссылка для автоматического возврата плательщика из приложения банка в приложение или на сайт ТСП
  redirectUrl?: string
}
```

### #/components/schemas/RegisterQRCodeRequest

```ts
{
  Data: {
    // Поле обязательно для заполнения, если тип QR = QR-Dynamic
    amount?: integer
    currency?: string
    paymentPurpose: string
    // 01 - QR-Static (QR наклейка)
    // 02 - QR-Dynamic (QR на кассе)
    qrcType: enum[01, 02]
    imageParams: {
      width: integer
      height: integer
      mediaType?: enum[image/png, image/svg+xml]
    }
    // Cистема, создавшая QR-код
    sourceName?: string
    // Задается, только если тип QR = QR-Dynamic
    ttl?: integer
    // Ссылка для автоматического возврата плательщика из приложения банка в приложение или на сайт ТСП
    redirectUrl?: string
  }
}
```

### #/components/schemas/RegisteredB2BQrCode

```ts
{
  payload: string
  qrcId: string
}
```

### #/components/schemas/RegisteredQrCode

```ts
{
  payload: string
  qrcId: string
  image: {
    width: integer
    height: integer
    mediaType?: enum[image/png, image/svg+xml]
    content: string
  }
}
```

### #/components/schemas/SBPCashboxOperationQrCodeStatus

```ts
{
  "type": "string",
  "enum": [
    "DEACTIVATED",
    "WAITING_PAYMENT",
    "IN_PROGRESS"
  ],
  "title": "SBPCashboxOperationQrCodeStatus"
}
```

### #/components/schemas/SBPCashboxQrCodeStatus

```ts
{
  "type": "string",
  "enum": [
    "INACTIVATED",
    "WAITING_PAYMENT",
    "IN_PROGRESS"
  ],
  "title": "SBPCashboxQrCodeStatus"
}
```

### #/components/schemas/SBPCashboxTrxStatus

```ts
{
  "type": "string",
  "enum": [
    "ACWP",
    "RJCT",
    "RCVD",
    "NTST"
  ],
  "title": "SBPCashboxTrxStatus",
  "description": "* ACWP - Операция завершена успешно\n* RJCT - Операция отклонена\n* RCVD - Операция в обработке\n* NTST - Операции по QR-коду не существует"
}
```

### #/components/schemas/SBPPayment

```ts
{
  qrcId: string
  // **Описание возможных статусов платежа**
  //   - `Confirming` - операция в процессе подтверждения ОПКЦ СБП
  //   - `Confirmed` - операция подтверждена
  //   - `Initiated` - операция отправлена на обработку
  //   - `Accepting` - операция в обработке ОПКЦ СБП
  //   - `Accepted` - операция успешно завершена
  //   - `InProgress` - операция в обработке РЦ СБП
  //   - `Rejected` - операция отклонена
  //   - `Error` - ошибка выполнения операции
  //   - `Timeout` - тайм-аут выполнения операции
  status: enum[Confirming, Confirmed, Initiated, Accepting, Accepted, InProgress, Rejected, Error, Timeout]
  message: string
  refTransactionId: string
}
```

### #/components/schemas/SBPPaymentList

```ts
{
  Payments: {
    qrcId: string
    // **Описание возможных статусов платежа**
    //   - `Confirming` - операция в процессе подтверждения ОПКЦ СБП
    //   - `Confirmed` - операция подтверждена
    //   - `Initiated` - операция отправлена на обработку
    //   - `Accepting` - операция в обработке ОПКЦ СБП
    //   - `Accepted` - операция успешно завершена
    //   - `InProgress` - операция в обработке РЦ СБП
    //   - `Rejected` - операция отклонена
    //   - `Error` - ошибка выполнения операции
    //   - `Timeout` - тайм-аут выполнения операции
    status: enum[Confirming, Confirmed, Initiated, Accepting, Accepted, InProgress, Rejected, Error, Timeout]
    message: string
    refTransactionId: string
  }[]
}
```

### #/components/schemas/SBPPaymentStatus

```ts
{
  "type": "string",
  "enum": [
    "WaitingForClientConfirm",
    "Initiated",
    "WaitingForConfirm",
    "Confirmed",
    "WaitingForAccept",
    "Accepted",
    "Rejected"
  ],
  "title": "SBPPaymentStatus"
}
```

### #/components/schemas/SBPPaymentsResponse

```ts
{
  Data: {
    Payments: {
      qrcId: string
      // **Описание возможных статусов платежа**
      //   - `Confirming` - операция в процессе подтверждения ОПКЦ СБП
      //   - `Confirmed` - операция подтверждена
      //   - `Initiated` - операция отправлена на обработку
      //   - `Accepting` - операция в обработке ОПКЦ СБП
      //   - `Accepted` - операция успешно завершена
      //   - `InProgress` - операция в обработке РЦ СБП
      //   - `Rejected` - операция отклонена
      //   - `Error` - ошибка выполнения операции
      //   - `Timeout` - тайм-аут выполнения операции
      status: enum[Confirming, Confirmed, Initiated, Accepting, Accepted, InProgress, Rejected, Error, Timeout]
      message: string
      refTransactionId: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/SBPRefund

```ts
{
  // БИК отправителя
  bankCode: string
  // Уникальный и неизменный идентификатор счёта
  accountCode: string
  // Cумма операции в рублях
  amount: string
  // Валюта операции
  currency?: string //default: RUB
  // ID qr-кода, по которому был сделан платеж
  qrcId: string
  // Назначение платежа
  purpose?: string
  // Идентификатор транзакции, по которой осуществляется возврат
  refTransactionId?: string
  // Идентификатор операции в НСПК, по которой можно осуществить возврат
  trxId?: string
}
```

### #/components/schemas/SBPRefundRequest

```ts
{
  Data: {
    // БИК отправителя
    bankCode: string
    // Уникальный и неизменный идентификатор счёта
    accountCode: string
    // Cумма операции в рублях
    amount: string
    // Валюта операции
    currency?: string //default: RUB
    // ID qr-кода, по которому был сделан платеж
    qrcId: string
    // Назначение платежа
    purpose?: string
    // Идентификатор транзакции, по которой осуществляется возврат
    refTransactionId?: string
    // Идентификатор операции в НСПК, по которой можно осуществить возврат
    trxId?: string
  }
}
```

### #/components/schemas/SBPRefundRequestResponse

```ts
{
  requestId: string
  status: enum[WaitingForClientConfirm, Initiated, WaitingForConfirm, Confirmed, WaitingForAccept, Accepted, Rejected]
}
```

### #/components/schemas/SBPRefundRequestResponseModel

```ts
{
  Data: {
    requestId: string
    status: enum[WaitingForClientConfirm, Initiated, WaitingForConfirm, Confirmed, WaitingForAccept, Accepted, Rejected]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/SBPRefundStatus

```ts
{
  requestId: string
  status: enum[WaitingForClientConfirm, Initiated, WaitingForConfirm, Confirmed, WaitingForAccept, Accepted, Rejected]
  // Описание статуса (причина ошибки или сообщение об успехе)
  statusDescription?: string
}
```

### #/components/schemas/SBPRefundStatusModel

```ts
{
  Data: {
    requestId: string
    status: enum[WaitingForClientConfirm, Initiated, WaitingForConfirm, Confirmed, WaitingForAccept, Accepted, Rejected]
    // Описание статуса (причина ошибки или сообщение об успехе)
    statusDescription?: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/SecondSideModel

```ts
{
  accountId?: string
  legalAddress?: string
  kpp?: string
  bankName?: string
  bankCorrAccount?: string
  taxCode: string
  type: enum[ip, company]
  secondSideName?: string
}
```

### #/components/schemas/SendDocumentToEmailRequestDataModel

```ts
{
  Data: {
    email: string
  }
}
```

### #/components/schemas/SendDocumentToEmailRequestModel

```ts
{
  email: string
}
```

### #/components/schemas/ShipmentDocumentModel

```ts
{
  name: string
  date: string
  number: string
}
```

### #/components/schemas/StatementInitReqModel

```ts
{
  accountId: string
  startDateTime: string
  endDateTime: string
}
```

### #/components/schemas/StatementInitRequestDataModel

```ts
// Метод создания выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      startDateTime: string
      endDateTime: string
    }
  }
}
```

### #/components/schemas/StatementInitRequestModel

```ts
{
  Statement: {
    accountId: string
    startDateTime: string
    endDateTime: string
  }
}
```

### #/components/schemas/StatementInitResponseDataModel

```ts
// Метод создания выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      statementId?: string
      status: enum[Created, Processing, Error, Ready]
      startDateTime: string
      endDateTime: string
      creationDateTime: string
    }
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/StatementInitResponseModel

```ts
{
  Statement: {
    accountId: string
    statementId?: string
    status: enum[Created, Processing, Error, Ready]
    startDateTime: string
    endDateTime: string
    creationDateTime: string
  }
}
```

### #/components/schemas/StatementListModel

```ts
{
  Statement: {
    accountId: string
    statementId?: string
    status: enum[Created, Processing, Error, Ready]
    startDateTime: string
    endDateTime: string
    creationDateTime: string
    startDateBalance?: number
    endDateBalance?: number
    Transaction: {
      transactionId?: string
      paymentId?: string
      creditDebitIndicator: enum[Credit, Debit]
      status: enum[Booked, Pending]
      documentNumber?: string
      transactionTypeCode?: enum[Неопределенное значение, Платежное поручение, Платежное требование, Денежный чек, РКО, Объявление на взнос наличными, ПКО, Требование-поручение, Инкассовое поручение, Расчетный чек, Аккредитив, Мемориальный ордер, Погашение кредита, Выдача кредита, Авизо, Банковские карты, Платежный ордер, Банковский ордер, Ордер по передаче ценностей, Программный ордер, Импортированная запись]
      documentProcessDate?: string
      description?: string
      Amount: {
        amount: number
        amountNat?: number
        currency: string
      }
      DebtorParty: {
        inn?: string
        name?: string
        kpp?: string
      }
      DebtorAccount: {
        schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
        identification?: string
      }
      DebtorAgent: {
        schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
        identification?: string
        accountIdentification?: string
        name?: string
      }
      CreditorParty:#/components/schemas/ContractorInfoModel
      CreditorAccount:#/components/schemas/CashAccountInfoModel
      CreditorAgent:#/components/schemas/ContractorBankInfoModel
      TaxFields: {
        originatorStatus?: string
        kbk?: string
        oktmo?: string
        base?: string
        documentNumber?: string
        documentDate?: Partial(integer) & Partial(string)
        type?: string
        field107?: string
      }
    }[]
  }[]
}
```

### #/components/schemas/StatementModel

```ts
{
  accountId: string
  statementId?: string
  status: enum[Created, Processing, Error, Ready]
  startDateTime: string
  endDateTime: string
  creationDateTime: string
  startDateBalance?: number
  endDateBalance?: number
  Transaction: {
    transactionId?: string
    paymentId?: string
    creditDebitIndicator: enum[Credit, Debit]
    status: enum[Booked, Pending]
    documentNumber?: string
    transactionTypeCode?: enum[Неопределенное значение, Платежное поручение, Платежное требование, Денежный чек, РКО, Объявление на взнос наличными, ПКО, Требование-поручение, Инкассовое поручение, Расчетный чек, Аккредитив, Мемориальный ордер, Погашение кредита, Выдача кредита, Авизо, Банковские карты, Платежный ордер, Банковский ордер, Ордер по передаче ценностей, Программный ордер, Импортированная запись]
    documentProcessDate?: string
    description?: string
    Amount: {
      amount: number
      amountNat?: number
      currency: string
    }
    DebtorParty: {
      inn?: string
      name?: string
      kpp?: string
    }
    DebtorAccount: {
      schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
      identification?: string
    }
    DebtorAgent: {
      schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
      identification?: string
      accountIdentification?: string
      name?: string
    }
    CreditorParty:#/components/schemas/ContractorInfoModel
    CreditorAccount:#/components/schemas/CashAccountInfoModel
    CreditorAgent:#/components/schemas/ContractorBankInfoModel
    TaxFields: {
      originatorStatus?: string
      kbk?: string
      oktmo?: string
      base?: string
      documentNumber?: string
      documentDate?: Partial(integer) & Partial(string)
      type?: string
      field107?: string
    }
  }[]
}
```

### #/components/schemas/StatementResponseDataModel

```ts
// Метод получения выписки по счету
{
  Data: {
    Statement: {
      accountId: string
      statementId?: string
      status: enum[Created, Processing, Error, Ready]
      startDateTime: string
      endDateTime: string
      creationDateTime: string
      startDateBalance?: number
      endDateBalance?: number
      Transaction: {
        transactionId?: string
        paymentId?: string
        creditDebitIndicator: enum[Credit, Debit]
        status: enum[Booked, Pending]
        documentNumber?: string
        transactionTypeCode?: enum[Неопределенное значение, Платежное поручение, Платежное требование, Денежный чек, РКО, Объявление на взнос наличными, ПКО, Требование-поручение, Инкассовое поручение, Расчетный чек, Аккредитив, Мемориальный ордер, Погашение кредита, Выдача кредита, Авизо, Банковские карты, Платежный ордер, Банковский ордер, Ордер по передаче ценностей, Программный ордер, Импортированная запись]
        documentProcessDate?: string
        description?: string
        Amount: {
          amount: number
          amountNat?: number
          currency: string
        }
        DebtorParty: {
          inn?: string
          name?: string
          kpp?: string
        }
        DebtorAccount: {
          schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
          identification?: string
        }
        DebtorAgent: {
          schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
          identification?: string
          accountIdentification?: string
          name?: string
        }
        CreditorParty:#/components/schemas/ContractorInfoModel
        CreditorAccount:#/components/schemas/CashAccountInfoModel
        CreditorAgent:#/components/schemas/ContractorBankInfoModel
        TaxFields: {
          originatorStatus?: string
          kbk?: string
          oktmo?: string
          base?: string
          documentNumber?: string
          documentDate?: Partial(integer) & Partial(string)
          type?: string
          field107?: string
        }
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/StatementStatus

```ts
{
  "type": "string",
  "enum": [
    "Created",
    "Processing",
    "Error",
    "Ready"
  ],
  "title": "StatementStatus"
}
```

### #/components/schemas/Status

```ts
{
  status: enum[Active, Suspended]
}
```

### #/components/schemas/StatusEnum

```ts
{
  "type": "string",
  "enum": [
    "Active",
    "Suspended"
  ],
  "title": "StatusEnum"
}
```

### #/components/schemas/StatusRequest

```ts
{
  Data: {
    status: enum[Active, Suspended]
  }
}
```

### #/components/schemas/SupplierModel

```ts
{
  phone: string
  name: string
  taxCode: string
}
```

### #/components/schemas/TaxFieldsModel

```ts
{
  originatorStatus?: string
  kbk?: string
  oktmo?: string
  base?: string
  documentNumber?: string
  documentDate?: Partial(integer) & Partial(string)
  type?: string
  field107?: string
}
```

### #/components/schemas/TaxSystemCodeInput

```ts
{
  "type": "string",
  "enum": [
    "osn",
    "usn_income",
    "usn_income_outcome",
    "esn",
    "patent"
  ],
  "title": "TaxSystemCodeInput"
}
```

### #/components/schemas/TaxSystemCodeOutput

```ts
{
  "type": "string",
  "enum": [
    "osn",
    "usn_income",
    "usn_income_outcome",
    "esn",
    "patent",
    "envd"
  ],
  "title": "TaxSystemCodeOutput",
  "description": "Перечисление для выдачи результатов из openapi."
}
```

### #/components/schemas/TransactionAmountModel

```ts
{
  amount: number
  amountNat?: number
  currency: string
}
```

### #/components/schemas/TransactionModel

```ts
{
  transactionId?: string
  paymentId?: string
  creditDebitIndicator: enum[Credit, Debit]
  status: enum[Booked, Pending]
  documentNumber?: string
  transactionTypeCode?: enum[Неопределенное значение, Платежное поручение, Платежное требование, Денежный чек, РКО, Объявление на взнос наличными, ПКО, Требование-поручение, Инкассовое поручение, Расчетный чек, Аккредитив, Мемориальный ордер, Погашение кредита, Выдача кредита, Авизо, Банковские карты, Платежный ордер, Банковский ордер, Ордер по передаче ценностей, Программный ордер, Импортированная запись]
  documentProcessDate?: string
  description?: string
  Amount: {
    amount: number
    amountNat?: number
    currency: string
  }
  DebtorParty: {
    inn?: string
    name?: string
    kpp?: string
  }
  DebtorAccount: {
    schemeName: enum[RU.CBR.PAN, RU.CBR.CellphoneNumber, RU.CBR.BBAN]
    identification?: string
  }
  DebtorAgent: {
    schemeName?: enum[RU.CBR.BICFI, RU.CBR.BIK]
    identification?: string
    accountIdentification?: string
    name?: string
  }
  CreditorParty:#/components/schemas/ContractorInfoModel
  CreditorAccount:#/components/schemas/CashAccountInfoModel
  CreditorAgent:#/components/schemas/ContractorBankInfoModel
  TaxFields: {
    originatorStatus?: string
    kbk?: string
    oktmo?: string
    base?: string
    documentNumber?: string
    documentDate?: Partial(integer) & Partial(string)
    type?: string
    field107?: string
  }
}
```

### #/components/schemas/UnauthorizedError

```ts
{
  errorCode: string
  message: string
  url: string
}
```

### #/components/schemas/UnauthorizedErrorResponse

```ts
{
  code: string
  id: string
  message: string
  Errors: {
    errorCode: string
    message: string
    url: string
  }[]
}
```

### #/components/schemas/UnitCodeEnum

```ts
{
  "type": "string",
  "enum": [
    "шт.",
    "тыс.шт.",
    "компл.",
    "пар.",
    "усл.ед.",
    "упак.",
    "услуга.",
    "пач.",
    "мин.",
    "ч.",
    "сут.",
    "г.",
    "кг.",
    "л.",
    "м.",
    "м2.",
    "м3.",
    "км.",
    "га.",
    "кВт.",
    "кВт.ч."
  ],
  "title": "UnitCodeEnum"
}
```

### #/components/schemas/VatType

```ts
{
  "type": "string",
  "enum": [
    "none",
    "vat0",
    "vat5",
    "vat7",
    "vat10",
    "vat20",
    "vat105",
    "vat107",
    "vat110",
    "vat120"
  ],
  "title": "VatType"
}
```

### #/components/schemas/Webhook

```ts
{
  webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
  url: string
}
```

### #/components/schemas/WebhookEditRequest

```ts
{
  webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
  url: string
}
```

### #/components/schemas/WebhookResponseModel

```ts
{
  Data: {
    webhooksList?: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment][]
    url: string
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/WebhookTestSendRequest

```ts
{
  webhookType: enum[incomingPayment, outgoingPayment, incomingSbpPayment, acquiringInternetPayment, incomingSbpB2BPayment]
}
```

### #/components/schemas/WebhookTypeEnum

```ts
{
  "type": "string",
  "enum": [
    "incomingPayment",
    "outgoingPayment",
    "incomingSbpPayment",
    "acquiringInternetPayment",
    "incomingSbpB2BPayment"
  ],
  "title": "WebhookTypeEnum"
}
```

### #/components/schemas/application__invoice__models__enums__PaymentStatusEnum

```ts
{
  "type": "string",
  "enum": [
    "payment_waiting",
    "payment_expired",
    "payment_paid"
  ],
  "title": "PaymentStatusEnum"
}
```

### #/components/schemas/application__open_banking__models__external_models__AccountListResponseModel

```ts
// Метод получения списка доступных счетов
{
  Data: {
    Account: {
      customerCode: string
      accountId: string
      transitAccount?: string
      status: enum[Enabled, Disabled, Deleted, ProForma, Pending]
      statusUpdateDateTime: string
      currency: string
      accountType: enum[Business, Personal]
      accountSubType: enum[CreditCard, CurrentAccount, Loan, Mortgage, PrePaidCard, Savings, Special]
      registrationDate: string
      accountDetails: {
        schemeName: string
        identification: string
        name: string
      }[]
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/schemas/application__sbp__models__enums__PaymentStatusEnum

```ts
{
  "type": "string",
  "enum": [
    "Confirming",
    "Confirmed",
    "Initiated",
    "Accepting",
    "Accepted",
    "InProgress",
    "Rejected",
    "Error",
    "Timeout"
  ],
  "title": "PaymentStatusEnum",
  "description": "**Описание возможных статусов платежа**\n  - `Confirming` - операция в процессе подтверждения ОПКЦ СБП\n  - `Confirmed` - операция подтверждена\n  - `Initiated` - операция отправлена на обработку\n  - `Accepting` - операция в обработке ОПКЦ СБП\n  - `Accepted` - операция успешно завершена\n  - `InProgress` - операция в обработке РЦ СБП\n  - `Rejected` - операция отклонена\n  - `Error` - ошибка выполнения операции\n  - `Timeout` - тайм-аут выполнения операции"
}
```

### #/components/schemas/application__sbp__models__response_models__sbp__AccountListResponseModel

```ts
{
  Data: {
    AccountList: {
      accountId: string
      status: enum[Active, Suspended]
      createdAt: string
      legalId: string
    }[]
  }
  Links: {
    self: string
  }
  Meta: {
    totalPages: integer
  }
}
```

### #/components/securitySchemes/Необходимые разрешения

```ts
{
  "type": "oauth2",
  "flows": {
    "password": {
      "scopes": {
        "ReadAccountsBasic": "Получение базовой информации о счёте",
        "ReadAccountsDetail": "Получение детальной информации о счёте",
        "ReadBalances": "Получение баланса",
        "ReadStatements": "Получение выписок по счету",
        "ReadCustomerData": "Получение информации о клиенте",
        "ReadSBPData": "Получение информации в сервисе СБП",
        "EditSBPData": "Изменение информации в сервисе СБП",
        "CreatePaymentForSign": "Создание платежа на подпись",
        "CreatePaymentOrder": "Подписание платежа",
        "ReadAcquiringData": "Получение информации о платёжной ссылке",
        "MakeAcquiringOperation": "Совершение операций через платёжные ссылки",
        "ManageInvoiceData": "Выставление счетов и создание закрывающих документов",
        "ManageWebhookData": "Получение вебхуков"
      },
      "tokenUrl": "https://enter.tochka.com/connect/token"
    }
  }
}
```