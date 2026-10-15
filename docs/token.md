# Подпись запроса (Token)

`Token` — обязательный параметр каждого метода СБП. Это SHA-256 хеш от значений параметров запроса и пароля
терминала. Пароль находится в [личном кабинете интернет-эквайринга](https://business.tbank.ru/oplata/main).

Источник: <https://developer.tbank.ru/eacq/intro/developer/token>

## Алгоритм

1. Соберите пары `ключ: значение` **только корневых** параметров запроса. Вложенные объекты и массивы
   (`DATA`, `Data`, `Receipt`, `Device`, `Shops`) в подписи не участвуют. Параметр `Token` не включается.
2. Добавьте пару `Password: <пароль терминала>`.
3. Отсортируйте пары по ключу в алфавитном порядке.
4. Конкатенируйте **только значения** в одну строку без разделителей.
5. Примените SHA-256 к строке в кодировке UTF-8, результат — hex в нижнем регистре.
6. Передайте результат в параметре `Token`.

## Пример расчета (тестовый вектор из документации)

Запрос `Init` с вложенными `DATA` и `Receipt` — они исключаются из подписи:

```json
[{"TerminalKey": "MerchantTerminalKey"}, {"Amount": "19200"}, {"OrderId": "00000"},
 {"Description": "Подарочная карта на 1000 рублей"}, {"Password": "11111111111111"}]
```

После сортировки по ключу и конкатенации значений:

```text
19200Подарочная карта на 1000 рублей0000011111111111111MerchantTerminalKey
```

SHA-256:

```text
72dd466f8ace0a37a1f740ce5fb78101712bc0665d91a8108c7c8a0ccd426db2
```

## Python

```python
import hashlib


def make_token(params: dict, password: str) -> str:
    """Подпись запроса T-Bank EACQ: SHA-256 от значений корневых параметров + Password."""
    data = {
        k: v for k, v in params.items()
        if k != "Token" and v is not None and not isinstance(v, (dict, list))
    }
    data["Password"] = password

    def fmt(v):
        # bool -> "true"/"false", иначе str(True) даст "True"
        return ("true" if v else "false") if isinstance(v, bool) else str(v)

    raw = "".join(fmt(data[k]) for k in sorted(data))
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()


body = {
    "TerminalKey": "MerchantTerminalKey",
    "Amount": 19200,
    "OrderId": "00000",
    "Description": "Подарочная карта на 1000 рублей",
    "DATA": {"Phone": "+71234567890"},
}
assert make_token(body, "11111111111111") == (
    "72dd466f8ace0a37a1f740ce5fb78101712bc0665d91a8108c7c8a0ccd426db2"
)
```

## Node.js / TypeScript

```typescript
import { createHash } from "node:crypto";

export function makeToken(params: Record<string, unknown>, password: string): string {
  const data: Record<string, string> = { Password: password };
  for (const [key, value] of Object.entries(params)) {
    if (key === "Token" || value === null || value === undefined) continue;
    if (typeof value === "object") continue; // вложенные объекты и массивы не подписываются
    data[key] = String(value); // true -> "true"
  }
  const raw = Object.keys(data).sort().map((k) => data[k]).join("");
  return createHash("sha256").update(raw, "utf8").digest("hex");
}

const body = { TerminalKey: "TBankTest", PaymentId: "10063", DataType: "PAYLOAD" };
const signed = { ...body, Token: makeToken(body, process.env.TBANK_PASSWORD!) };
```

## PHP

```php
<?php
function makeToken(array $params, string $password): string
{
    $data = ['Password' => $password];
    foreach ($params as $key => $value) {
        if ($key === 'Token' || $value === null || is_array($value) || is_object($value)) {
            continue;
        }
        // (string)true === "1" в PHP, поэтому bool приводим явно
        $data[$key] = is_bool($value) ? ($value ? 'true' : 'false') : (string)$value;
    }
    ksort($data, SORT_STRING);
    return hash('sha256', implode('', $data));
}

$body = ['TerminalKey' => 'TBankTest', 'PaymentId' => '10063'];
$body['Token'] = makeToken($body, getenv('TBANK_PASSWORD'));
```

## Частые ошибки

- В подпись попали вложенные объекты (`DATA`, `Receipt`, `Device` у `GetQrBankList`) — их нужно исключать.
- Булево значение превратилось в `True` (Python) или `1` (PHP) вместо `true`.
- Значения в подписи и в теле запроса отличаются (например, другой формат даты `RedirectDueDate`).
- Сортировка не по ключу, а по значению.

Проверить корректность токена для конкретного заказа можно в личном кабинете: «Операции» → заказ →
«Дополнительная информация о заказе» → поле `inittokenisvalid` (`true` — токен валиден).

## Проверка токена во входящем уведомлении

Для HTTP-уведомлений токен считается так же: берутся все корневые параметры уведомления, кроме `Token`
и вложенных объектов (`Data`, `Receipt`), добавляется `Password`, пары сортируются по ключу, значения
конкатенируются и хешируются SHA-256. Результат сравните с `Token` из уведомления.

```text
Строка: 111100000001111000000200000******0000111111111110000000000000AUTHORIZEDtrue1234567890DEMO
SHA-256: 1c0964277d0213349243065a0d5b838b8e90d2d25f740d0f2767836e710e80c8
```

```python
import hmac


def verify_notification(payload: dict, password: str) -> bool:
    expected = make_token(payload, password)  # make_token исключает Token и вложенные объекты
    return hmac.compare_digest(expected, str(payload.get("Token", "")))
```
