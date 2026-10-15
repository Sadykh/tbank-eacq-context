# GetQrState — Получить статус возврата

Метод возвращает статус возврата платежа по СБП, а при ошибке возврата — код и описание ошибки от СБП.

Источник: <https://developer.tbank.ru/eacq/api/get-qr-state>

```http
POST https://securepay.tinkoff.ru/v2/GetQrState
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `PaymentId` | string | Да | Идентификатор платежа из ответа `Init` |
| `Token` | string | Да | Подпись запроса |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/GetQrState \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "PaymentId": "700031849",
    "Token": "l43kb4hyi6lknb23bv4gdfskjn238fsllsdf8"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `Success` | boolean | Да | Успешность запроса |
| `ErrorCode` | string | Да | Код ошибки, `"0"` — успех |
| `Status` | string | Условно | Статус платежа (`ASYNC_REFUNDING`, `REFUNDED`, `PARTIAL_REFUNDED`, `CONFIRMED` и др.). Обязателен, если статус получен без ошибки |
| `QrCancelCode` | string | Нет | Код ошибки возврата от СБП, например `I05043` |
| `QrCancelMessage` | string | Нет | Описание ошибки возврата по QR |
| `OrderId` | string, ≤ 50 | Нет | Идентификатор заказа в системе мерчанта |
| `Amount` | integer | Нет | Сумма отмены в копейках |
| `Message` | string | Нет | Краткое описание ошибки запроса |

## Пример ответа: ошибка возврата от СБП

```json
{
  "Success": true,
  "ErrorCode": "0",
  "Status": "CONFIRMED",
  "QrCancelCode": "I05043",
  "QrCancelMessage": "У покупателя нет расчетного счета в этом банке. Попробуйте вернуть деньги на счет этого покупателя в другом банке",
  "OrderId": "7830122",
  "Amount": 10000,
  "Message": "OK"
}
```

## Пример: опрос статуса возврата (Python)

```python
import time

for _ in range(30):
    state = call("GetQrState", {"PaymentId": payment_id})
    if state.get("QrCancelCode"):
        raise RuntimeError(f"Возврат отклонен СБП: {state['QrCancelCode']} {state.get('QrCancelMessage')}")
    if state.get("Status") in ("REFUNDED", "PARTIAL_REFUNDED"):
        break
    time.sleep(10)
```
