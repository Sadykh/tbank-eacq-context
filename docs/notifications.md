# HTTP-уведомления по СБП

Т-Банк отправляет POST-запрос с JSON на `NotificationURL` при изменении статуса платежа и при привязке счета.

Источники:
- <https://developer.tbank.ru/eacq/intro/developer/notification>
- <https://developer.tbank.ru/eacq/intro/developer/notification/notify-samples>

## Правила

- URL для уведомлений о платежах: параметр `NotificationURL` в `Init`, иначе — значение из настроек терминала.
- Уведомления о **привязке счета по QR** (`LINKACCOUNT`) отправляются только на `NotificationURL` из
  настроек терминала.
- Ответ на успешно обработанное уведомление: HTTP 200 с телом `OK` (без тегов, заглавными латинскими буквами).
- Без ответа `OK` уведомление повторяется раз в час в течение 24 часов, затем раз в сутки в течение месяца,
  после чего попадает в архив (хранится 90 дней, можно запросить повторную отправку).
- Для HTTPS допустим порт 443.
- Параметры со значением `null` в уведомление не попадают.
- Перед обработкой проверяйте `Token`, см. [token.md](token.md).

## Уведомление о платеже по СБП

`Pan` содержит маскированный номер телефона покупателя. Объект `Data` содержит дополнительные данные СБП
(`Source: "qrsbp"`, `bankMemberId`, `extQrId`, `AccountToken` и др.) и в подписи не участвует.

```json
{
  "TerminalKey": "1*******4",
  "OrderId": "0*******-*********e",
  "Success": true,
  "Status": "CONFIRMED",
  "PaymentId": 8000000004,
  "ErrorCode": "0",
  "Amount": 15000,
  "Pan": "+7(98*)***-**-*4",
  "Token": "c**********************************************************4",
  "Data": {
    "AccountToken": "c**********************************************************4",
    "sourceId": "A*****************3",
    "operationDate": "2026-06-30",
    "connection_type": "API",
    "bankMemberId": "1*********1",
    "bankName": "bankname",
    "PayType": "O",
    "settlementDate": "2026-06-30",
    "Source": "qrsbp",
    "extOperationId": "A*****************3",
    "umit_bpay": "044525225",
    "interBankOperation": false,
    "extQrId": "A*********************E",
    "paymentDate": "2026-06-30T14:52:09",
    "REDIRECT": false
  }
}
```

Аналогичные уведомления приходят со статусами `AUTHORIZED`, `REJECTED`, `REFUNDED`.

## Уведомление о неуспешном автоплатеже

```json
{
  "TerminalKey": "1***********7",
  "OrderId": "1************7",
  "Success": false,
  "Status": "REJECTED",
  "PaymentId": 9000000007,
  "ErrorCode": "5060",
  "Amount": 29900,
  "Token": "****************************************************************",
  "Data": {
    "QR": "true",
    "CUSTOMER_KEY": "4*****5",
    "OperationInitiatorType": "I",
    "connection_type": "API",
    "AccountToken": "********************************",
    "REDIRECT": "false"
  }
}
```

## Уведомление о привязке счета (LINKACCOUNT)

Приходит только по статусам `ACTIVE` и `INACTIVE`.

```json
{
  "TerminalKey": "1**********5",
  "RequestKey": "1******6",
  "Status": "ACTIVE",
  "Success": "true",
  "ErrorCode": "0",
  "Message": "Плательщик подтвердил привязку счета",
  "AccountToken": "********************************",
  "BankMemberId": "1**********1",
  "BankMemberName": "bankname",
  "Token": "****************************************************************",
  "NotificationType": "LINKACCOUNT"
}
```

Отказ покупателя: `"Status": "INACTIVE"`, `"Success": "false"`,
`"Message": "Плательщик отказался от привязки счета"`.

## Обработчик уведомлений (Python, Flask)

```python
from flask import Flask, request

app = Flask(__name__)
PASSWORD = "secret"


@app.post("/tbank/notify")
def tbank_notify():
    payload = request.get_json(force=True)
    if not verify_notification(payload, PASSWORD):  # см. token.md
        return "invalid token", 400

    if payload.get("NotificationType") == "LINKACCOUNT":
        if payload["Status"] == "ACTIVE":
            save_account(payload["RequestKey"], payload["AccountToken"], payload["BankMemberId"])
    else:
        update_order(payload["OrderId"], payload["PaymentId"], payload["Status"])

    return "OK", 200
```

## Способы получить статус без уведомлений

- `GetState` — статус конкретного платежа.
- `CheckOrder` — история платежей по заказу.
- `GetQrState` — статус возврата по СБП.
- `GetAddAccountQrState` — статус привязки счета.
