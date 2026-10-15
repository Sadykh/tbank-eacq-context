# Ошибки и модели ответов методов СБП

Все методы СБП возвращают HTTP 200 как при успехе, так и при бизнес-ошибке. Результат определяется полями
`Success` и `ErrorCode` (`"0"` — успех). HTTP 500 возвращается при внутренней ошибке сервера.

## Стандартная модель ошибки (HTTP 200)

Ответ 200 каждого метода описан как `oneOf`: модель успешного ответа или `ErrorResponse`.

| Параметр | Тип | Описание |
|---|---|---|
| `Success` | boolean | `false` |
| `ErrorCode` | string | Код ошибки |
| `Message` | string | Краткое описание ошибки |
| `Details` | string | Подробное описание ошибки |

```json
{
  "Success": false,
  "ErrorCode": "1051",
  "Message": "Неверные параметры",
  "Details": "Параметр \"PaymentId\" не может быть пустым"
}
```

## Внутренняя ошибка сервера (HTTP 500)

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `Success` | boolean | Нет | `false` |
| `ErrorCode` | string | Да | Код ошибки, например `9999` |
| `Message` | string | Да | Описание ошибки |
| `Causes` | array of string | Да | Подробное описание причины |
| `StackTrace` | string | Нет | Отчет об ошибке |

```json
{
  "Success": false,
  "ErrorCode": "9999",
  "Message": "Внутренняя ошибка системы.",
  "Causes": ["Internal Service Error."]
}
```

## Ошибки возврата от СБП

`GetQrState` возвращает ошибку возврата от СБП отдельно от `ErrorCode`:

| Параметр | Пример | Описание |
|---|---|---|
| `QrCancelCode` | `I05043` | Код ошибки возврата от СБП |
| `QrCancelMessage` | `У покупателя нет расчетного счета в этом банке...` | Описание ошибки |

## Обработка ответа (Python)

```python
import requests


class TBankError(Exception):
    def __init__(self, method: str, data: dict):
        self.code = data.get("ErrorCode")
        self.message = data.get("Message")
        self.details = data.get("Details") or data.get("Causes")
        super().__init__(f"{method}: [{self.code}] {self.message} — {self.details}")


def post(method: str, body: dict) -> dict:
    resp = requests.post(f"https://securepay.tinkoff.ru/v2/{method}", json=body, timeout=30)
    data = resp.json()  # и 200, и 500 возвращают JSON
    if resp.status_code >= 500 or not data.get("Success") or data.get("ErrorCode") != "0":
        raise TBankError(method, data)
    return data
```

Полный справочник кодов ошибок: <https://developer.tbank.ru/eacq/intro/errors/error-codes>.
