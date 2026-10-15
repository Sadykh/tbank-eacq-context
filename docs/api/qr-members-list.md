# QrMembersList — Получить список банков-пользователей QR для возврата

Метод возвращает список участников СБП (банков), в которые может быть сделан возврат платежа, совершенного
по QR. Используется перед возвратом, чтобы понять, возможен ли он и в какой банк.

Источник: <https://developer.tbank.ru/eacq/api/qr-members-list>

```http
POST https://securepay.tinkoff.ru/v2/QrMembersList
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `PaymentId` | string | Да | Идентификатор платежа по QR |
| `Token` | string | Да | Подпись запроса |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/QrMembersList \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "PaymentId": "700031849",
    "Token": "7241ac8307f349afb7bb9dda760717721bbb45950b97c67289f23d8c69cc7b96"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `Members` | array | Нет | Список участников. Возвращается, только если возврат возможен |
| `Members[].MemberId` | string | Да | Идентификатор участника (банка) |
| `Members[].MemberName` | string | Да | Наименование участника |
| `Members[].IsPayee` | boolean | Да | Участник был получателем указанного платежа |
| `OrderId` | string, ≤ 50 | Да | Идентификатор заказа в системе мерчанта |
| `Success` | boolean | Да | Успешность запроса |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Message` | string | Нет | Краткое описание ошибки |

## Пример успешного ответа

```json
{
  "Members": [
    { "MemberId": "1000000", "MemberName": "T-Банк", "IsPayee": true }
  ],
  "OrderId": "21050",
  "Success": true,
  "ErrorCode": "0",
  "Message": "OK"
}
```

## Пример: проверка возможности возврата (Python)

```python
resp = call("QrMembersList", {"PaymentId": payment_id})
members = resp.get("Members") or []
if not members:
    raise RuntimeError("Возврат по этому платежу через СБП невозможен")
```

Сам возврат выполняется методом `Cancel`, статус — `GetQrState`. См. [scenario-refund.md](../scenario-refund.md).
