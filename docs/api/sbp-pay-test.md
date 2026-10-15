# SbpPayTest — Создать тестовую платежную сессию

Метод создает тестовую платежную сессию по СБП с предопределенным результатом: успешная оплата, отказ по
таймауту или отказ банка. Работает только с тестовым терминалом (суффикс `DEMO`) на боевом адресе.

Источник: <https://developer.tbank.ru/eacq/api/sbp-pay-test>

```http
POST https://securepay.tinkoff.ru/v2/SbpPayTest
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор тестового терминала (`...DEMO`) |
| `PaymentId` | string, ≤ 20 | Да | Идентификатор платежа, для которого уже вызван `GetQr` |
| `Token` | string | Да | Подпись запроса |
| `IsDeadlineExpired` | boolean | Нет | `true` — эмулировать отказ по таймауту (статус `DEADLINE_EXPIRED`). Нельзя вместе с `IsRejected=true` |
| `IsRejected` | boolean | Нет | `true` — эмулировать отказ банка (статус `REJECTED`). Нельзя вместе с `IsDeadlineExpired=true` |

Без флагов эмулируется успешная оплата (статус `CONFIRMED`).

## Пример запроса: успешная оплата

```bash
curl -X POST https://securepay.tinkoff.ru/v2/SbpPayTest \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "1234567890DEMO",
    "PaymentId": "13660",
    "Token": "7241ac8307f349afb7bb9dda760717721bbb45950b97c67289f23d8c69cc7b96"
  }'
```

## Пример запроса: отказ по таймауту

```json
{
  "TerminalKey": "1234567890DEMO",
  "PaymentId": "13660",
  "IsDeadlineExpired": true,
  "IsRejected": false,
  "Token": "7241ac8307f349afb7bb9dda760717721bbb45950b97c67289f23d8c69cc7b96"
}
```

Булевы `IsDeadlineExpired`/`IsRejected` участвуют в подписи как строки `true`/`false`.

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `Success` | boolean | Да | Успешность запроса |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Message` | string, ≤ 255 | Да | Краткое описание ошибки |
| `Details` | string | Да | Подробное описание ошибки |

## Пример успешного ответа

```json
{
  "Success": true,
  "ErrorCode": "0",
  "Message": "OK",
  "Details": "0"
}
```

Сценарии тестирования — [testing.md](../testing.md).
