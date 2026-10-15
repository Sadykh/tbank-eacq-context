# Сценарий: оплата по динамическому QR-коду СБП

Динамический QR-код одноразовый и действует для оплаты одной покупки. Сумму задает магазин — покупатель
сканирует QR-код (или переходит по ссылке на мобильном) и подтверждает оплату в приложении своего банка.

Источник: <https://developer.tbank.ru/eacq/scenarios/payments/PCI_DSS/sbp/>

## Последовательность вызовов

1. `POST /v2/Init` — создать платеж, получить `PaymentId`.
2. (Опционально) `POST /v2/GetQrBankList` — получить список банков, если вы сами показываете выбор банка.
3. `POST /v2/GetQr` — получить QR: payload-ссылку, SVG-изображение или deeplink (с `BankId`).
4. Показать QR покупателю (desktop) или перенаправить по ссылке (mobile).
5. Получить результат: HTTP-уведомление на `NotificationURL` или `POST /v2/GetState`.
   - Успех — статус `CONFIRMED`, покупатель перенаправляется на `SuccessURL`.
   - Неуспех — платеж остается в `FORM_SHOWED`, после трех неудачных попыток — `REJECTED`.
   - Истек срок `RedirectDueDate` — `DEADLINE_EXPIRED`.

## Шаг 1. Инициировать платеж

```json
{
  "TerminalKey": "TBankTest",
  "Amount": 140000,
  "OrderId": "21050",
  "Description": "Подарочная карта на 1400.00 рублей",
  "PayType": "O",
  "RedirectDueDate": "2026-10-16T12:00:00+03:00",
  "NotificationURL": "https://shop.example.com/tbank/notify",
  "SuccessURL": "https://shop.example.com/success",
  "FailURL": "https://shop.example.com/fail",
  "Token": "<sha256>"
}
```

Ответ:

```json
{
  "Success": true,
  "ErrorCode": "0",
  "TerminalKey": "TBankTest",
  "Status": "NEW",
  "PaymentId": "3093639567",
  "OrderId": "21050",
  "Amount": 140000,
  "PaymentURL": "https://pay.tbank.ru/new/fU1ppgqa"
}
```

Требования `Init` для СБП:
- `Amount` — в копейках, не меньше 1000 (10 рублей).
- `Description` — текст отображается в мобильном банке покупателя, максимум 140 символов для СБП.
- СБП поддерживает только одностадийную оплату (`PayType: "O"`).
- `RedirectDueDate` — срок жизни QR, от 1 минуты до 90 дней; по умолчанию 24 часа.

## Шаг 2. Сформировать QR

```json
{
  "TerminalKey": "TBankTest",
  "PaymentId": "3093639567",
  "DataType": "PAYLOAD",
  "Token": "<sha256>"
}
```

```json
{
  "TerminalKey": "TBankTest",
  "OrderId": "21050",
  "Success": true,
  "Data": "https://qr.nspk.ru/AD1000670LSS7DN18SJQDNP4B05KLJL2",
  "PaymentId": "3093639567",
  "ErrorCode": "0"
}
```

## Полный пример (Python)

```python
import hashlib
import requests

BASE_URL = "https://securepay.tinkoff.ru/v2"


class TBankSbp:
    def __init__(self, terminal_key: str, password: str):
        self.terminal_key = terminal_key
        self.password = password

    def _token(self, params: dict) -> str:
        data = {k: v for k, v in params.items()
                if k != "Token" and v is not None and not isinstance(v, (dict, list))}
        data["Password"] = self.password
        fmt = lambda v: ("true" if v else "false") if isinstance(v, bool) else str(v)
        return hashlib.sha256("".join(fmt(data[k]) for k in sorted(data)).encode()).hexdigest()

    def call(self, method: str, params: dict) -> dict:
        body = {"TerminalKey": self.terminal_key, **params}
        body["Token"] = self._token(body)
        data = requests.post(f"{BASE_URL}/{method}", json=body, timeout=30).json()
        if not data.get("Success") or data.get("ErrorCode") != "0":
            raise RuntimeError(f"{method}: {data.get('ErrorCode')} {data.get('Message')} {data.get('Details')}")
        return data

    def create_qr_payment(self, order_id: str, amount_kop: int, description: str,
                          as_image: bool = False, bank_id: str | None = None) -> dict:
        payment = self.call("Init", {
            "Amount": amount_kop,
            "OrderId": order_id,
            "Description": description,
            "PayType": "O",
        })
        qr_params = {"PaymentId": payment["PaymentId"], "DataType": "IMAGE" if as_image else "PAYLOAD"}
        if bank_id and not as_image:
            qr_params["BankId"] = bank_id  # Data вернется как deeplink в приложение банка
        qr = self.call("GetQr", qr_params)
        return {"payment_id": payment["PaymentId"], "qr": qr["Data"]}

    def get_status(self, payment_id: str) -> str:
        return self.call("GetState", {"PaymentId": payment_id})["Status"]


client = TBankSbp("TBankTest", "secret")
result = client.create_qr_payment("order-1001", 15000, "Заказ №1001")
```

## Desktop или mobile

- **Desktop**: запросите `DataType: "IMAGE"` и вставьте SVG на страницу, либо закодируйте payload-ссылку в
  QR на своей стороне.
- **Mobile**: используйте payload-ссылку (`https://qr.nspk.ru/...`) — она открывает выбор банка, — или
  получите список банков `GetQrBankList` с `Device.Type: "mobile"` и запросите deeplink через `GetQr` с `BankId`.

## Статический QR-код

Многоразовый QR, сумму вводит покупатель. Создается без API: магазин с выставлением счета → личный кабинет →
«Магазины» → «Прием оплаты» → «Статический QR-код». Не интегрируется с онлайн-кассой.
