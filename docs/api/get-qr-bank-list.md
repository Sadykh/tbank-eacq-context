# GetQrBankList — Получить список банков-участников СБП для платежа

Метод возвращает список банков-участников СБП (НСПК). Используется, когда мерчант сам показывает покупателю
список банков и формирует deeplink: выбранный `BankId` передается в `GetQr` или `AddAccountQr`.

Источник: <https://developer.tbank.ru/eacq/api/get-qr-bank-list>

```http
POST https://securepay.tinkoff.ru/v2/GetQrBankList
Content-Type: application/json
```

## Параметры запроса

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `TerminalKey` | string, ≤ 64 | Да | Идентификатор терминала |
| `ScenarioType` | string: `qr` \| `sub` | Нет | Сценарий: `qr` — оплата (по умолчанию), `sub` — привязка счета |
| `Device` | object | Да | Тип и ОС устройства покупателя |
| `Device.Type` | string: `desktop` \| `mobile` | Да | Тип устройства |
| `Device.Os` | string, ≤ 255 | Нет | ОС устройства, например `iOS` |
| `PaymentMethod` | string: `SBP` \| `DR` | Нет | Способ оплаты |
| `Token` | string, ≤ 255 | Да | Подпись запроса. Объект `Device` в подписи не участвует |

## Пример запроса

```bash
curl -X POST https://securepay.tinkoff.ru/v2/GetQrBankList \
  -H "Content-Type: application/json" \
  -d '{
    "TerminalKey": "TBankTest",
    "ScenarioType": "qr",
    "Device": { "Type": "mobile", "Os": "iOS" },
    "PaymentMethod": "SBP",
    "Token": "871199b37f207f0c4f721a37cdcc71dfcea880b4a4b85e3cf852c5dc1e99a8d6"
  }'
```

## Параметры ответа

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `Success` | boolean | Да | Успешность запроса |
| `ErrorCode` | string | Да | Код ошибки, `"0"` — успех |
| `Message` | string | Нет | Краткое описание ошибки |
| `BankList` | array | Да | Список банков от НСПК |
| `BankList[].BankId` | string, ≤ 200 | Да | Внутренний идентификатор банка — передается в `GetQr.BankId` |
| `BankList[].NspkBankId` | string, ≤ 200 | Да | Идентификатор банка в НСПК, например `100000000004` |
| `BankList[].BankName` | string, ≤ 200 | Да | Наименование банка |
| `BankList[].BankLogo` | string, ≤ 200 | Да | URL логотипа банка |
| `BankList[].BankOrder` | integer | Да | Порядок сортировки |

## Пример успешного ответа

```json
{
  "Success": true,
  "ErrorCode": "0",
  "Message": "OK",
  "BankList": [
    {
      "BankId": "bank100000000004",
      "NspkBankId": "100000000004",
      "BankName": "Т-Банк",
      "BankLogo": "https://qr.nspk.ru/proxyapp/logo/bank100000000004.png",
      "BankOrder": 1
    },
    {
      "BankId": "bank100000000111",
      "NspkBankId": "100000000111",
      "BankName": "Сбербанк",
      "BankLogo": "https://qr.nspk.ru/proxyapp/logo/bank100000000111.png",
      "BankOrder": 2
    }
  ]
}
```

## Пример: свой список банков и deeplink (Python)

```python
banks = call("GetQrBankList", {
    "ScenarioType": "qr",
    "Device": {"Type": "mobile", "Os": "Android"},
})["BankList"]
banks.sort(key=lambda b: b["BankOrder"])

# Покупатель выбрал банк — получаем deeplink в его приложение
chosen = banks[0]
deeplink = call("GetQr", {
    "PaymentId": payment_id,
    "DataType": "PAYLOAD",
    "BankId": chosen["BankId"],
})["Data"]
```

## Примечания

- Для привязки счета (`AddAccountQr`) запрашивайте список с `ScenarioType: "sub"`.
- В официальном примере ответа `BankId` показан в формате UUID, в примерах `GetQr` — `bank100000000004`;
  передавайте значение `BankId` из ответа без изменений.
