# GetAccountQrList — Получить список счетов, привязанных к магазину

Метод возвращает список счетов покупателей, привязанных к магазину (терминалу) через СБП, со статусами
привязок и `AccountToken`.

Источник: <https://developer.tbank.ru/eacq/api/get-account-qr-list>

```http
POST https://securepay.tinkoff.ru/v2/GetAccountQrList
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `Token` | string | Да | Подпись запроса |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/GetAccountQrList \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "Token": "7241ac8307f349afb7bb9dda760717721bbb45950b97c67289f23d8c69cc7b96"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `Success` | boolean | Да | Успешность запроса |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Message` | string, ≤ 255 | Нет | Краткое описание ошибки |
| `AccountTokens` | array | Нет | Список привязанных счетов |
| `AccountTokens[].RequestKey` | string | Да | Идентификатор запроса на привязку счета |
| `AccountTokens[].Status` | string: `NEW` \| `PROCESSING` \| `ACTIVE` \| `INACTIVE` | Да | Статус привязки |
| `AccountTokens[].AccountToken` | string | Нет | Идентификатор привязки счета, назначается банком плательщика |
| `AccountTokens[].BankMemberId` | string | Нет | Идентификатор банка покупателя. Передается при `ACTIVE` или `INACTIVE` |
| `AccountTokens[].BankMemberName` | string | Нет | Наименование банка-эмитента |

## Пример успешного ответа

```json
{
  "TerminalKey": "TBankTest",
  "Success": true,
  "ErrorCode": "0",
  "Message": "OK",
  "AccountTokens": [
    {
      "RequestKey": "8de92934-26c9-474c-a4ce-424f2021d24d",
      "Status": "ACTIVE",
      "AccountToken": "0b67f2cae19b41809f85d5674de30a1a",
      "BankMemberId": "100000000004",
      "BankMemberName": "T-Банк"
    },
    {
      "RequestKey": "13021e10-a3ed-4f14-bcd1-823b5ac37390",
      "Status": "PROCESSING"
    }
  ]
}
```

## Пример: активные привязки (Python)

```python
accounts = call("GetAccountQrList", {}).get("AccountTokens", [])
active = {a["AccountToken"]: a["BankMemberId"] for a in accounts if a["Status"] == "ACTIVE"}
```

`AccountToken` и `BankMemberId` отсутствуют у привязок в статусах `NEW` и `PROCESSING`.
