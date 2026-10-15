# GetQr — Сформировать QR

Метод регистрирует динамический QR-код СБП для ранее созданного платежа и возвращает информацию о нем:
функциональную платежную ссылку (payload), SVG-изображение QR или deeplink в приложение выбранного банка.
Вызывается после метода `Init` (Инициировать платеж).

Источник: <https://developer.tbank.ru/eacq/api/get-qr>

```http
POST https://securepay.tinkoff.ru/v2/GetQr
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала, выдается в Т-Бизнес |
| `PaymentId` | string | Да | Идентификатор платежа из ответа `Init` |
| `DataType` | string: `PAYLOAD` \| `IMAGE` | Нет | `PAYLOAD` (по умолчанию) — вернуть только payload-ссылку; `IMAGE` — вернуть SVG-изображение QR |
| `BankId` | string | Нет | Внутренний идентификатор банка, выбранного для оплаты (из `GetQrBankList`). Если передан, в `Data` вместо payload возвращается deeplink. Передается только при `DataType` = `PAYLOAD` или не задан |
| `PaymentMethod` | string: `SBP` \| `DR` | Нет | Способ оплаты: `SBP` — СБП, `DR` — цифровой рубль |
| `Token` | string | Да | Подпись запроса, см. [token.md](../token.md) |

## Пример запроса: payload-ссылка

```bash
curl -X POST https://securepay.tinkoff.ru/v2/GetQr \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "PaymentId": "10063",
    "DataType": "PAYLOAD",
    "Token": "871199b37f207f0c4f721a37cdcc71dfcea880b4a4b85e3cf852c5dc1e99a8d6"
  }'
```

## Пример запроса: deeplink в приложение конкретного банка

```json
{
  "TerminalKey": "TBankTest",
  "PaymentId": "10063",
  "DataType": "PAYLOAD",
  "BankId": "bank100000000004",
  "PaymentMethod": "SBP",
  "Token": "871199b37f207f0c4f721a37cdcc71dfcea880b4a4b85e3cf852c5dc1e99a8d6"
}
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `OrderId` | string, ≤ 50 | Да | Идентификатор заказа в системе мерчанта |
| `Success` | boolean | Да | Успешность запроса |
| `Data` | string | Да | При `DataType=PAYLOAD` — ссылка (payload), которую нужно закодировать в QR, или deeplink при переданном `BankId`; при `DataType=IMAGE` — SVG-изображение QR с закодированным payload |
| `PaymentId` | string | Да | Идентификатор платежа |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Message` | string, ≤ 255 | Нет | Краткое описание ошибки |
| `Details` | string | Нет | Подробное описание ошибки |
| `RequestKey` | string | Да | Идентификатор запроса на привязку счета. Передается при привязке счета после оплаты по СБП (`Init` с `Recurrent=Y`) |

## Пример успешного ответа

```json
{
  "TerminalKey": "TBankTest",
  "OrderId": "21057",
  "Success": true,
  "Data": "https://qr.nspk.ru/AD1000670LSS7DN18SJQDNP4B05KLJL2",
  "PaymentId": "10063",
  "ErrorCode": "0",
  "RequestKey": "ed989549-d3be-4758-95c7-22647e03f9ec"
}
```

## Пример ответа с ошибкой

```json
{
  "Success": false,
  "ErrorCode": "1051",
  "Message": "Неверные параметры",
  "Details": "Параметр \"PaymentId\" не может быть пустым"
}
```

## Пример: показать QR покупателю (Python)

```python
import base64

qr = call("GetQr", {"PaymentId": payment_id, "DataType": "IMAGE"})
svg = qr["Data"]  # готовая SVG-разметка QR
html = f'<img src="data:image/svg+xml;base64,{base64.b64encode(svg.encode()).decode()}" />'

# Для мобильного браузера удобнее payload-ссылка — она открывает выбор банка СБП
link = call("GetQr", {"PaymentId": payment_id, "DataType": "PAYLOAD"})["Data"]
```

`call()` — обертка, которая добавляет `TerminalKey` и `Token` (см. README).

## Примечания

- После успешной оплаты платеж переходит в `CONFIRMED`, покупатель перенаправляется на `SuccessURL`.
- При неуспехе платеж остается в `FORM_SHOWED`, после трех неудачных попыток — `REJECTED`.
- Срок жизни QR задается параметром `RedirectDueDate` в `Init` (1 минута — 90 дней, по умолчанию 24 часа).
- HTTP 500 возвращает модель внутренней ошибки, см. [errors.md](../errors.md).
