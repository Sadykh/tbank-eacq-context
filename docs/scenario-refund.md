# Сценарий: возврат платежа по СБП

СБП поддерживает частичные и полные возвраты. Возврат выполняется общим методом `Cancel` (Отменить платеж),
а для проверки возможности возврата и его статуса есть специальные методы СБП.

Источники:
- <https://developer.tbank.ru/eacq/api/qr-members-list>
- <https://developer.tbank.ru/eacq/api/get-qr-state>

## Последовательность

1. (Опционально) `QrMembersList` — проверить, что возврат возможен и в какой банк. Если `Members` пустой
   или отсутствует, возврат через СБП невозможен.
2. `Cancel` с `PaymentId` (и `Amount` для частичного возврата) — инициировать возврат.
3. Платеж переходит в `ASYNC_REFUNDING` — возврат обрабатывается асинхронно.
4. `GetQrState` — получить статус возврата. Итог: `REFUNDED` / `PARTIAL_REFUNDED`, либо ошибка СБП в
   `QrCancelCode` и `QrCancelMessage`.

## Пример запросов

1. QrMembersList:

```json
{ "TerminalKey": "TBankTest", "PaymentId": "700031849", "Token": "<sha256>" }
```

2. Cancel — частичный возврат 50 ₽:

```json
{ "TerminalKey": "TBankTest", "PaymentId": "700031849", "Amount": 5000, "Token": "<sha256>" }
```

4. GetQrState:

```json
{ "TerminalKey": "TBankTest", "PaymentId": "700031849", "Token": "<sha256>" }
```

## Пример (Python)

```python
import time


def refund_sbp(payment_id: str, amount_kop: int | None = None) -> dict:
    members = call("QrMembersList", {"PaymentId": payment_id}).get("Members") or []
    if not members:
        raise RuntimeError("Возврат по СБП недоступен для этого платежа")

    params = {"PaymentId": payment_id}
    if amount_kop is not None:
        params["Amount"] = amount_kop  # частичный возврат
    call("Cancel", params)

    for _ in range(30):
        state = call("GetQrState", {"PaymentId": payment_id})
        if state.get("QrCancelCode"):
            raise RuntimeError(f"СБП отклонил возврат: {state['QrCancelCode']} — {state.get('QrCancelMessage')}")
        if state.get("Status") in ("REFUNDED", "PARTIAL_REFUNDED"):
            return state
        time.sleep(10)
    raise TimeoutError("Возврат не завершился")
```

## Типичная ошибка возврата

```json
{
  "Success": true,
  "ErrorCode": "0",
  "Status": "CONFIRMED",
  "QrCancelCode": "I05043",
  "QrCancelMessage": "У покупателя нет расчетного счета в этом банке. Попробуйте вернуть деньги на счет этого покупателя в другом банке",
  "OrderId": "7830122",
  "Amount": 10000
}
```

`Success: true` здесь означает, что запрос статуса выполнен, а не что возврат прошел. Проверяйте
`QrCancelCode` и `Status`.
