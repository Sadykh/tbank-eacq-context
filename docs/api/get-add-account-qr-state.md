# GetAddAccountQrState — Получить статус привязки счета к магазину

Метод возвращает статус привязки счета покупателя к магазину по идентификатору запроса `RequestKey`,
полученному из `AddAccountQr` или из ответа/уведомления `GetQr` при привязке после оплаты.

Источник: <https://developer.tbank.ru/eacq/api/get-add-account-qr-state>

```http
POST https://securepay.tinkoff.ru/v2/GetAddAccountQrState
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `RequestKey` | string | Да | Идентификатор запроса на привязку счета |
| `Token` | string | Да | Подпись запроса |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/GetAddAccountQrState \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "RequestKey": "13021e10-a3ed-4f14-bcd1-823b5ac37390",
    "Token": "7241ac8307f349afb7bb9dda760717721bbb45950b97c67289f23d8c69cc7b96"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `RequestKey` | string | Да | Идентификатор запроса на привязку счета |
| `Status` | string, ≤ 20 | Да | `NEW` — запрос получен; `PROCESSING` — в обработке; `ACTIVE` — привязка успешна; `INACTIVE` — привязка неуспешна или деактивирована |
| `AccountToken` | string | Нет | Идентификатор привязки счета, назначается банком плательщика. Используется в `ChargeQr` |
| `BankMemberId` | string | Нет | Идентификатор банка покупателя. Передается при статусе `ACTIVE` или `INACTIVE` |
| `BankMemberName` | string | Нет | Наименование банка-эмитента, заполнено если передан `BankMemberId` |
| `Success` | boolean | Да | Успешность запроса |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Message` | string, ≤ 255 | Нет | Краткое описание ошибки |

## Пример успешного ответа

```json
{
  "TerminalKey": "TBankTest",
  "RequestKey": "13021e10-a3ed-4f14-bcd1-823b5ac37390",
  "BankMemberId": "100000000004",
  "BankMemberName": "T-Банк",
  "AccountToken": "a022254a5c3c46a7327c8a12cb5c8389",
  "Success": true,
  "Status": "ACTIVE",
  "ErrorCode": "0",
  "Message": "OK"
}
```

## Пример: ожидание привязки (Python)

```python
import time


def wait_account_binding(request_key: str, timeout_s: int = 300) -> dict:
    deadline = time.monotonic() + timeout_s
    while time.monotonic() < deadline:
        state = call("GetAddAccountQrState", {"RequestKey": request_key})
        if state["Status"] in ("ACTIVE", "INACTIVE"):
            return state  # при ACTIVE сохраните state["AccountToken"] и state["BankMemberId"]
        time.sleep(5)
    raise TimeoutError(f"Привязка {request_key} не завершилась за {timeout_s} с")
```

Основной способ получить результат — уведомление `LINKACCOUNT`, см. [notifications.md](../notifications.md);
опрос статуса используйте как резервный.
