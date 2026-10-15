# Тестирование СБП

Для тестирования оплат через СБП используйте тестовый терминал с суффиксом `DEMO`. Запросы с него
отправляются на **боевой** адрес `https://securepay.tinkoff.ru/v2`. Результат оплаты эмулируется методом
`SbpPayTest`.

Источник: <https://developer.tbank.ru/eacq/intro/errors/test-sbp>

- Сценарии тестирования СБП не обязательны для активации приема платежей.
- При тестовых сценариях по СБП уведомления на электронную почту не приходят.

## Сценарии

| Сценарий | Параметры `SbpPayTest` | Ожидаемый статус в `GetState` |
|---|---|---|
| Платеж — успех | `PaymentId` | `CONFIRMED` |
| Платеж — отказ по таймауту | `PaymentId`, `IsDeadlineExpired: true` | `DEADLINE_EXPIRED` |
| Платеж — отклонен Т-Бизнесом | `PaymentId`, `IsRejected: true` | `REJECTED` |
| Возврат — успех | после успешного платежа вызвать `Cancel` | `REFUNDED` |

Шаги каждого платежного сценария:

1. `Init` — создать платеж.
2. `GetQr` — сгенерировать динамический QR-код.
3. Отобразить QR на странице.
4. `SbpPayTest` с `PaymentId` и флагом сценария.
5. `GetState` — проверить статус.

## Пример автотеста (Python, pytest)

```python
import pytest

client = TBankSbp("1234567890DEMO", "demo-password")  # см. scenario-dynamic-qr.md


@pytest.mark.parametrize("flags, expected", [
    ({}, "CONFIRMED"),
    ({"IsDeadlineExpired": True}, "DEADLINE_EXPIRED"),
    ({"IsRejected": True}, "REJECTED"),
])
def test_sbp_payment(flags, expected, order_id):
    created = client.create_qr_payment(order_id, 1000, "Тест СБП")
    client.call("SbpPayTest", {"PaymentId": created["payment_id"], **flags})
    assert client.get_status(created["payment_id"]) == expected


def test_sbp_refund(order_id):
    created = client.create_qr_payment(order_id, 1000, "Тест возврата СБП")
    client.call("SbpPayTest", {"PaymentId": created["payment_id"]})
    client.call("Cancel", {"PaymentId": created["payment_id"]})
    assert client.get_status(created["payment_id"]) == "REFUNDED"
```

Статус может меняться асинхронно — при необходимости опрашивайте `GetState` с паузой.
