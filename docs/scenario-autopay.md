# Сценарий: автоплатежи через СБП (привязка счета)

Автоплатеж через СБП — повторный платеж, который списывается с привязанного счета покупателя без его участия.
В отличие от автоплатежей по картам и T-Pay, у автоплатежа по СБП нет родительского платежа: вместо него
привязывается счет, а банк покупателя выдает `AccountToken`.

Источник: <https://developer.tbank.ru/eacq/scenarios/payments/PCI_DSS/autopay/>

## Условия

- Доступно только по API. Включите в личном кабинете: «Магазины» → «Прием оплаты» → «Своя платежная форма» →
  «Настроить» → включить СБП.
- Только одностадийная оплата.
- Два этапа: (1) привязка счета, (2) автоплатежи `ChargeQr`.
- Уведомление с `AccountToken` приходит на `NotificationURL` из **настроек терминала**, а не на
  `NotificationURL`, переданный в `Init`.
- Привязка, сделанная на одном терминале мерчанта, действует для всех его терминалов.

## Вариант A. Привязка счета до оплаты

1. `AddAccountQr` — получить QR привязки и `RequestKey`.
2. Покупатель подтверждает привязку в своем банке.
3. Уведомление `NotificationType: "LINKACCOUNT"`, `Status: "ACTIVE"` → сохраните `AccountToken` и `BankMemberId`.
   Резервный способ — `GetAddAccountQrState` по `RequestKey`.
4. Для каждого списания: `Init` с `Recurrent: "Y"` и `DATA: {"QR": "true"}` → `PaymentId`.
5. `ChargeQr` с `PaymentId` и `AccountToken`.

1. AddAccountQr:

```json
{
  "TerminalKey": "TBankTest",
  "Description": "Подписка «Премиум», списание 299 ₽ ежемесячно",
  "DataType": "PAYLOAD",
  "Token": "<sha256>"
}
```

4. Init для автоплатежа:

```json
{
  "TerminalKey": "TBankTest",
  "Amount": 29900,
  "OrderId": "sub-2026-10",
  "Description": "Подписка «Премиум», октябрь",
  "Recurrent": "Y",
  "DATA": { "QR": "true" },
  "Token": "<sha256>"
}
```

5. ChargeQr:

```json
{
  "TerminalKey": "TBankTest",
  "PaymentId": "700001702044",
  "AccountToken": "70LSS7DN18SJQRS10006DNPKLJL24B05",
  "Token": "<sha256>"
}
```

## Вариант B. Привязка счета при первой оплате

1. `Init` с `Recurrent: "Y"`, `DATA: {"QR": "true"}` и обязательным `Description`.
2. `GetQr` — покупатель оплачивает по QR и одновременно привязывает счет.
3. После оплаты в уведомлении или ответе `GetQr` придет `RequestKey`.
4. `GetAddAccountQrState` с `RequestKey` — проверить статус привязки.
5. Уведомление `LINKACCOUNT` с `AccountToken` — сохраните его.
6. Последующие списания: `Init` (`Recurrent: "Y"`, `DATA: {"QR": "true"}`) → `ChargeQr` с `AccountToken`.

## Пример (Python)

```python
# Этап 1: привязка (вариант A)
binding = call("AddAccountQr", {
    "Description": "Подписка «Премиум», списание 299 ₽ ежемесячно",
    "DataType": "PAYLOAD",
})
show_to_customer(binding["Data"])
save_request_key(user_id, binding["RequestKey"])


# Обработчик уведомления о привязке (см. notifications.md)
def on_link_account(notification: dict) -> None:
    if notification["Status"] == "ACTIVE":
        save_account(
            request_key=notification["RequestKey"],
            account_token=notification["AccountToken"],
            bank_member_id=notification["BankMemberId"],
        )


# Этап 2: списание
def charge(account_token: str, bank_member_id: str, amount_kop: int, order_id: str) -> dict:
    payment = call("Init", {
        "Amount": amount_kop,
        "OrderId": order_id,
        "Description": "Подписка «Премиум»",
        "Recurrent": "Y",
        "DATA": {"QR": "true"},
    })
    return call("ChargeQr", {
        "PaymentId": payment["PaymentId"],
        "AccountToken": account_token,
        "BankMemberId": bank_member_id,
    })
```

## Статусы

Привязка: `NEW` → `PROCESSING` → `ACTIVE` | `INACTIVE`.

Автоплатеж: `NEW` → `FORM_SHOWED` (банк принял запрос, ждет НСПК) → `AUTHORIZING` → `CONFIRMED` | `REJECTED`.

## Управление привязками

- `GetAccountQrList` — все привязки терминала со статусами и `AccountToken`.
- `GetAddAccountQrState` — статус конкретной привязки по `RequestKey`.
