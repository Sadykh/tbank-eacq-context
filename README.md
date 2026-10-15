# T-Bank Internet Acquiring API — СБП

Документация методов API интернет-эквайринга Т-Банка (T-Bank EACQ) для приема платежей через
Систему быстрых платежей (СБП), подготовленная для индексации в [Context7](https://context7.com).

Источник: <https://developer.tbank.ru/eacq/api/sbp>

## Базовые сведения

| Параметр | Значение |
|---|---|
| Базовый URL | `https://securepay.tinkoff.ru/v2` |
| HTTP-метод | `POST` для всех методов |
| Формат тела | `application/json` |
| Подпись | параметр `Token` в теле запроса — SHA-256, см. [docs/token.md](docs/token.md) |
| Тестовая среда | терминал с суффиксом `DEMO` на боевом адресе, см. [docs/testing.md](docs/testing.md) |

## Методы СБП

| Метод | Endpoint | Назначение |
|---|---|---|
| Сформировать QR | `POST /v2/GetQr` | Регистрирует динамический QR для платежа и возвращает payload, SVG или deeplink |
| Получить список банков-участников СБП для платежа | `POST /v2/GetQrBankList` | Список банков НСПК с `BankId` для своего выбора банка и deeplink |
| Привязать счет к магазину | `POST /v2/AddAccountQr` | Инициирует привязку счета покупателя для автоплатежей |
| Получить статус привязки счета к магазину | `POST /v2/GetAddAccountQrState` | Статус привязки по `RequestKey` |
| Получить список счетов, привязанных к магазину | `POST /v2/GetAccountQrList` | Все привязки счетов терминала |
| Получить список банков-пользователей QR для возврата | `POST /v2/QrMembersList` | Банки, в которые можно вернуть платеж по QR |
| Получить статус возврата | `POST /v2/GetQrState` | Статус возврата платежа по СБП |
| Создать тестовую платежную сессию | `POST /v2/SbpPayTest` | Эмуляция оплаты по QR на DEMO-терминале |
| Автоплатеж по QR (связанный метод) | `POST /v2/ChargeQr` | Списание по привязанному счету (`AccountToken`) |

## Содержание

- [Обзор СБП: особенности, ограничения, статусы](docs/overview.md)
- [Подпись запроса (Token)](docs/token.md)
- [Сценарий: оплата по динамическому QR](docs/scenario-dynamic-qr.md)
- [Сценарий: автоплатежи через СБП (привязка счета)](docs/scenario-autopay.md)
- [Сценарий: возвраты по СБП](docs/scenario-refund.md)
- [HTTP-уведомления по СБП](docs/notifications.md)
- [Тестирование СБП](docs/testing.md)
- [Ошибки и модели ответов](docs/errors.md)
- Справочник методов:
  - [GetQr](docs/api/get-qr.md)
  - [GetQrBankList](docs/api/get-qr-bank-list.md)
  - [AddAccountQr](docs/api/add-account-qr.md)
  - [GetAddAccountQrState](docs/api/get-add-account-qr-state.md)
  - [GetAccountQrList](docs/api/get-account-qr-list.md)
  - [ChargeQr](docs/api/charge-qr.md)
  - [QrMembersList](docs/api/qr-members-list.md)
  - [GetQrState](docs/api/get-qr-state.md)
  - [SbpPayTest](docs/api/sbp-pay-test.md)

## Быстрый старт (Python)

```python
import hashlib
import requests

BASE_URL = "https://securepay.tinkoff.ru/v2"
TERMINAL_KEY = "TBankTest"
PASSWORD = "secret"  # пароль терминала из личного кабинета


def make_token(params: dict, password: str) -> str:
    data = {
        k: v for k, v in params.items()
        if k != "Token" and v is not None and not isinstance(v, (dict, list))
    }
    data["Password"] = password

    def fmt(v):
        return ("true" if v else "false") if isinstance(v, bool) else str(v)

    raw = "".join(fmt(data[k]) for k in sorted(data))
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()


def call(method: str, params: dict) -> dict:
    body = {"TerminalKey": TERMINAL_KEY, **params}
    body["Token"] = make_token(body, PASSWORD)
    resp = requests.post(f"{BASE_URL}/{method}", json=body, timeout=30)
    data = resp.json()
    if not data.get("Success") or data.get("ErrorCode") != "0":
        raise RuntimeError(f"{method} failed: {data}")
    return data


# 1. Создать платеж на 150 руб.
payment = call("Init", {"Amount": 15000, "OrderId": "order-42", "Description": "Заказ №42"})

# 2. Получить ссылку для QR (payload) по этому платежу
qr = call("GetQr", {"PaymentId": payment["PaymentId"], "DataType": "PAYLOAD"})
print(qr["Data"])  # https://qr.nspk.ru/... — закодируйте в QR или откройте на мобильном
```
