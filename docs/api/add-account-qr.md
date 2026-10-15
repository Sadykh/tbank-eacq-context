# AddAccountQr — Привязать счет к магазину

Метод инициирует привязку счета покупателя к магазину через СБП и возвращает QR (payload, SVG или deeplink),
по которому покупатель подтверждает привязку в приложении своего банка. После привязки мерчант получает
`AccountToken` и может проводить автоплатежи методом `ChargeQr`.

Источник: <https://developer.tbank.ru/eacq/api/add-account-qr>

```http
POST https://securepay.tinkoff.ru/v2/AddAccountQr
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `Description` | string | Да | Подробное описание деталей заказа / подписки, отображается покупателю |
| `DataType` | string: `PAYLOAD` \| `IMAGE` | Нет | `PAYLOAD` (по умолчанию) — ссылка; `IMAGE` — SVG-изображение QR |
| `BankId` | string | Нет | Банк покупателя из `GetQrBankList` (`ScenarioType=sub`). Если передан, в `Data` возвращается deeplink. Только для `DataType` = `PAYLOAD` или не задан |
| `Data` | object | Нет | Дополнительные параметры `ключ: значение` для кастомизированной страницы: ключ ≤ 20 символов, значение ≤ 100, не более 20 пар; спецсимволы кодируются `urlencode`. В подписи не участвует |
| `RedirectDueDate` | string, date-time | Нет | Срок жизни ссылки/QR: от 1 минуты до 90 дней от текущей даты, формат `YYYY-MM-DDTHH24:MI:SS+GMT`, например `2016-08-31T12:28:00+03:00`. Если не передан — берется настройка терминала `REDIRECT_TIMEOUT` (в часах); если она меньше нуля — 1440 минут (1 сутки) |
| `Token` | string | Да | Подпись запроса |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/AddAccountQr \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "Description": "Подписка на сервис, ежемесячное списание",
    "DataType": "PAYLOAD",
    "RedirectDueDate": "2016-08-31T12:28:00+03:00",
    "Token": "871199b37f207f0c4f721a37cdcc71dfcea880b4a4b85e3cf852c5dc1e99a8d6"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `Data` | string | Да | Payload-ссылка (`https://sub.nspk.ru/...`), deeplink или SVG-изображение QR — в зависимости от `DataType`/`BankId` |
| `RequestKey` | string | Да | Идентификатор запроса на привязку счета — сохраните для `GetAddAccountQrState` |
| `ErrorCode` | string, ≤ 20 | Да | Код ошибки, `"0"` — успех |
| `Success` | boolean | Да | Успешность запроса |
| `Message` | string, ≤ 255 | Нет | Краткое описание ошибки |

## Пример успешного ответа

```json
{
  "TerminalKey": "TBankTest",
  "Data": "https://sub.nspk.ru/AB50803R2RH0LJ2A9RTU038L6NT5RU1M?type=03&bank=100000000004&crc=F3D0",
  "RequestKey": "ed989549-d3be-4758-95c7-22647e03f9ec",
  "ErrorCode": "0",
  "Success": true,
  "Message": "OK"
}
```

## Что дальше

1. Покажите покупателю `Data` (QR или ссылку) — он подтверждает привязку в своем банке.
2. Дождитесь уведомления `NotificationType: "LINKACCOUNT"` со статусом `ACTIVE` и сохраните `AccountToken`.
   Уведомление приходит на `NotificationURL` из **настроек терминала**, а не из `Init`.
3. Или опрашивайте статус методом `GetAddAccountQrState` по `RequestKey`.
4. Проводите списания через `Init` (`Recurrent=Y`, `DATA={"QR":"true"}`) + `ChargeQr` с `AccountToken`.

Подробнее — [scenario-autopay.md](../scenario-autopay.md).
