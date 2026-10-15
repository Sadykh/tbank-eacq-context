# ChargeQr — Автоплатеж по QR

Метод проводит платеж по привязанному счету покупателя через СБП без участия покупателя (автоплатеж).
Требует `PaymentId` из `Init` (с `Recurrent=Y` и `DATA={"QR":"true"}`) и `AccountToken` из привязки счета.

Метод относится к разделу «Проведение платежа по сохраненным реквизитам», но используется только вместе
с методами СБП.

Источник: <https://developer.tbank.ru/eacq/api/charge-qr>

```http
POST https://securepay.tinkoff.ru/v2/ChargeQr
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `PaymentId` | string | Да | Идентификатор платежа из `Init` |
| `AccountToken` | string | Да | Идентификатор привязки счета, назначается банком-эмитентом |
| `Token` | string | Да | Подпись запроса |
| `IP` | string | Нет | IP-адрес покупателя |
| `SendEmail` | boolean | Нет | Отправлять уведомление об оплате на почту покупателя |
| `InfoEmail` | string, email | Условно | Email покупателя. Обязателен, если `SendEmail=true` |
| `BankMemberId` | string, ≤ 200 | Условно | Идентификатор банка плательщика. Обязателен, если привязан счет в другом банке |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/ChargeQr \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "PaymentId": "700001702044",
    "AccountToken": "70LSS7DN18SJQRS10006DNPKLJL24B05",
    "BankMemberId": "100000000004",
    "SendEmail": true,
    "InfoEmail": "customer@test.com",
    "Token": "f5a3be479324a6d3a4d9efa0d02880b77d04a91758deddcbd9e752a6df97cab5"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `Amount` | integer | Да | Сумма в копейках |
| `OrderId` | string, ≤ 50 | Да | Идентификатор заказа в системе мерчанта |
| `Success` | boolean | Да | Успешность запроса |
| `Status` | string | Нет | `CONFIRMED` — платеж выполнен; `REJECTED` — не выполнен; `FORM_SHOWED` — платеж в обработке, ожидается подтверждение от НСПК |
| `PaymentId` | string, ≤ 20 | Да | Идентификатор платежа |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Message` | string, ≤ 255 | Нет | Краткое описание ошибки |
| `Details` | string | Нет | Подробное описание ошибки |
| `Currency` | string | Нет | Код валюты ISO 4217, `643` — рубль |

## Пример успешного ответа

```json
{
  "TerminalKey": "TBankTest",
  "Amount": 100000,
  "OrderId": "21050",
  "Success": true,
  "Status": "CONFIRMED",
  "PaymentId": "13660",
  "ErrorCode": "0",
  "Currency": "643"
}
```

## Пример: ежемесячное списание (Python)

```python
def charge_subscription(account_token: str, bank_member_id: str, amount_kop: int, order_id: str) -> dict:
    payment = call("Init", {
        "Amount": amount_kop,
        "OrderId": order_id,
        "Description": "Ежемесячная подписка",
        "Recurrent": "Y",
        "DATA": {"QR": "true"},
    })
    result = call("ChargeQr", {
        "PaymentId": payment["PaymentId"],
        "AccountToken": account_token,
        "BankMemberId": bank_member_id,
    })
    # FORM_SHOWED — финальный результат придет в уведомлении или через GetState
    return result
```

## Примечания

- В схеме ответа статус ожидания указан как `FORM SHOWED`, в справочнике статусов — `FORM_SHOWED`.
  Обрабатывайте оба варианта.
- Если счет привязан к одному из терминалов мерчанта, автоплатеж можно провести на любом другом его
  терминале: банк ищет привязку по идентификатору магазина.
