# Лабораторная работа №7: Логирование и обработка ошибок в Python

### Цель работы
* Освоить принципы разработки декораторов с параметрами.
* Научиться разделять ответственность функций (бизнес-логика) и декораторов (сквозная логика). 
* Научиться обрабатывать исключения, возникающие при работе с внешними API.
* Освоить логирование в разные типы потоков (sys.stdout, io.StringIO, logging).
* Научиться тестировать функцию и поведение логирования.

### Задание
1. Реализация декоратора. Написать универсальный декоратор `@logger`, поддерживающий вывод в консоль, файл и стандартный модуль `logging`.
2. Работа с API. Реализовать функцию `get_currencies` для получения актуальных курсов валют с сервера ЦБ РФ.
3. Обернуть функцию декоратором.
4. Создать файл-логирование.

### Реализация

#### Декоратор логирования
Декоратор фиксирует момент вызова функции, переданные аргументы, результат выполнения или возникшую ошибку (с выводом traceback).
 
```python
import sys
import traceback
import logging
import requests
import functools

def logger(func: object = None, *, handle: object = sys.stdout) -> object:
    """
    Декоратор ведет логирование тремя способами:
    1) sys.stdout (по умолчанию) → write()
    2) любой файловый поток, например io.StringIO → write()
    3) logging.Logger → .info() / .error()

    Логирует:
    - INFO: старт вызова + аргументы
    - INFO: успешное завершение + результат
    - ERROR: текст и тип исключения, затем повторно выбрасывает
    """

    def decorator(fn):

        @functools.wraps(fn)
        def inner(*args, **kwargs):
            # Определяем способ логирования
            is_logger = hasattr(handle, "info") and callable(getattr(handle, "info"))
            try:
                msg_start = f"Calling {fn.__name__} args={args}, kwargs={kwargs}"
                if is_logger:
                    handle.info(msg_start)
                else:
                    handle.write(f"INFO: {msg_start}\n")

                result = fn(*args, **kwargs)

                msg_end = f"{fn.__name__} returned {result}"
                if is_logger:
                    handle.info(msg_end)
                else:
                    handle.write(f"INFO: {msg_end}\n")

                return result
            except Exception as e:
                msg_err = f"Exception in {fn.__name__}: {type(e).__name__}: {e}"
                if is_logger:
                    handle.error(msg_err)
                    # Для отладки можно добавить traceback
                    handle.error(traceback.format_exc())
                else:
                    handle.write(f"ERROR: {msg_err}\n")
                    handle.write(traceback.format_exc() + "\n")
                raise  # повторно выбрасываем исключение

        return inner

    if func is None:
        return decorator
    else:
        return decorator(func)


@logger(handle=sys.stdout)
def get_currencies(currency_codes: list,
                   url: str = "https://www.cbr-xml-daily.ru/daily_json.js",
                   handle=sys.stdout) -> dict:
    """
    Получает курсы валют с API ЦБ РФ.

    Поднимает исключения:
    - API недоступен → ConnectionError
    - Некорректный JSON → ValueError
    - Нет ключа 'Valute' → KeyError
    - Валюта отсутствует → KeyError
    - Неверный тип курса → TypeError
    """

    try:
        response = requests.get(url)
        response.raise_for_status()
    except requests.exceptions.RequestException as e:
        handle.write(f"Ошибка при запросе к API: {e}\n")
        raise ConnectionError("API недоступен")

    # Ошибка структуры JSON
    try:
        data = response.json()
    except ValueError:
        raise ValueError("Некорректный JSON")

    if "Valute" not in data:
        raise KeyError("Отсутствует ключ 'Valute' в данных API")

    result = {}

    for code in currency_codes:
        if code not in data["Valute"]:
            result[code] = f"Код валюты '{code}' не найден."
            continue
        value = data["Valute"][code]["Value"]

        if not isinstance(value, (int, float)):
            raise TypeError(f"Неверный тип курса для валюты: {code}")
        result[code] = value

    return result

'''# 1
get_currencies(["USD", "EUR"])

# 2
stream = io.StringIO()
get_currencies(["USD"], handle=stream)

# 3
logger = logging.getLogger("TRACE")
logger.setLevel(logging.INFO)
logger.addHandler(logging.StreamHandler(sys.stdout))'''



currencies = ["USD", "EUR"]
get_currencies(currencies, handle=logger)


# запись в файл
if __name__ == "__main__":

    # Настраиваем логгер, который пишет в файл
    file_logger = logging.getLogger("currency_file")
    file_logger.setLevel(logging.INFO)

    # Создаем обработчик записи в файл
    file_handler = logging.FileHandler("currency.log", encoding="utf-8")
    formatter = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s")
    file_handler.setFormatter(formatter)
    file_logger.addHandler(file_handler)

    # Оборачиваем функцию в декоратор с file_logger
    get_currencies_file = logger(handle=file_logger)(get_currencies)

    # Вызов
    currency_list = currencies
    get_currencies_file(currency_list)

    print("Логи записаны в файл currency.log")
```

#### Демонстрационный пример: `solve_quadratic`
Для демонстрации уровней логирования реализована функция решения квадратного уравнения:
* **INFO:** При нахождении корней.
* **WARNING:** Если дискриминант < 0.
* **ERROR:** При передаче некорректных типов данных.
* **CRITICAL:** В ситуации $a=0, b=0$.

```python
import logging
import math

logging.basicConfig(
    filename="quadratic.log",
    level=logging.DEBUG,
    format="%(levelname)s: %(message)s"
)

def solve_quadratic(a, b, c):
    logging.info(f"Solving equation: {a}x^2 + {b}x + {c} = 0")

    # Ошибка типов
    for name, value in zip(("a", "b", "c"), (a, b, c)):
        if not isinstance(value, (int, float)):
            logging.critical(f"Parameter '{name}' must be a number, got: {value}")
            raise TypeError(f"Coefficient '{name}' must be numeric")

    # Ошибка: a == 0
    if a == 0:
        logging.error("Coefficient 'a' cannot be zero")
        raise ValueError("a cannot be zero")

    d = b*b - 4*a*c
    logging.debug(f"Discriminant: {d}")

    if d < 0:
        logging.warning("Discriminant < 0: no real roots")
        return None

    if d == 0:
        x = -b / (2*a)
        logging.info("One real root")
        return (x,)

    root1 = (-b + math.sqrt(d)) / (2*a)
    root2 = (-b - math.sqrt(d)) / (2*a)
    logging.info("Two real roots computed")
    return root1, root2


if __name__ == "__main__":
    # Два корня
    print(solve_quadratic(1, -5, 6))   # x^2 -5x +6 =0 → roots 2,3

    # Один корень
    print(solve_quadratic(1, -2, 1))   # x^2 -2x +1 =0 → root 1

    # Дискриминант <0
    print(solve_quadratic(1, 0, 1))    # x^2 +1 =0 → no real roots

    # Ошибка типов
    try:
        solve_quadratic("a", 2, 3)
    except TypeError:
        pass

    # Невозможная ситуация a=b=0
    try:
        solve_quadratic(0, 0, 5)
    except ValueError:
        pass
```

#### Тестирование

Для верификации работы использован модуль `unittest`. Процесс разделен на два блока:

1. Тесты функции: проверка корректности возвращаемых значений и корректного выброса исключений при сетевых ошибках или неверном JSON.
2. Тесты декоратора: использование `io.StringIO` для перехвата потока вывода и проверки наличия строк "INFO" и "ERROR" в логах.

### Вывод
В результате выполнения работы была создана гибкая система логирования, которая отделена от основной логики приложения. Это позволяет легко менять способ отслеживания работы программы (консоль/файл/логгер) без изменения самого кода функций.