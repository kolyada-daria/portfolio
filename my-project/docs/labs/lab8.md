# Лабораторная работа №8: Клиент-серверное приложение на Python с использованием Jinja2

### Цель работы
Создание клиент-серверного приложения на Python с использованием архитектуры MVC, стандартной библиотеки http.server, шаблонизатора Jinja2 и базы данных SQLite.

### Архитектура проекта (MVC)
Проект разделен на независимые слои, что обеспечивает легкость масштабирования:

**Models**:

* Сущности User, Currency, UserCurrency, App и Author. Все атрибуты инкапсулированы и снабжены геттерами/сеттерами с валидацией типов.

**Views**: 

* Шаблоны Jinja2 в папке templates/, обеспечивающие динамический рендеринг HTML-страниц.

**Controller**: 

* Класс на базе BaseHTTPRequestHandler в myapp.py, отвечающий за маршрутизацию и обработку параметров запроса (query-params).

### Ключевой функционал

1. **Интеграция с API ЦБ РФ**. Автоматическое получение актуальных котировок и их приведение к единому номиналу через вспомогательный модуль currencies_api.py. 
2. **Система подписок**. Реализовано взаимодействие с базой данных SQLite для хранения выбора пользователей. Подписка/отписка происходит через обработку POST-запросов. 
3. **Визуализация**. На странице профиля пользователя реализованы интерактивные графики Chart.js, отображающие динамику курсов за последние 30 дней на основе архивных данных API.

### Реализация
```python
from jinja2 import Environment, PackageLoader, select_autoescape
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs

from utils.currencies_api import get_currencies
from utils.currencies_history import get_currency_history
from models.currency import Currency
from models.user import User
from models.user_currency import UserCurrency
from models import Author
import requests


# Получение JSON с ЦБ РФ
url = "https://www.cbr-xml-daily.ru/daily_json.js"
response = requests.get(url)
full_data = response.json()["Valute"]

data = get_currencies()

# Список валют
currency_list = []
for code, info in full_data.items():
    c = Currency(
        currency_id=info["ID"],
        num_code=info["NumCode"],
        char_code=code,
        name=info["Name"],
        value=info["Value"],
        nominal=1
    )
    currency_list.append(c)

users = [
    User(1, "Александр"),
    User(2, "Мария"),
    User(3, "Даниил")
]
user_currencies = []  # список подписок пользователей
main_author = Author('Коляда Дарья', 'P3124')
env = Environment(
    loader=PackageLoader("myapp"),
    autoescape=select_autoescape()
)


class MyServer(BaseHTTPRequestHandler):
    def _nav(self):
        return [
            #{'caption': 'Основная страница', 'href': "https://nickzhukov.ru"},
            #{'caption': 'Пользователь', 'href': '/user'},
            {'caption': 'Все пользователи', 'href': '/users'},
            {'caption': 'Валюты', 'href': '/currencies'},
            {'caption': 'Подписки', 'href': '/subscriptions'},
            {'caption': 'Об авторе', 'href': '/author'}
        ]

    def _send_response(self, html: str):
        self.send_response(200)
        self.send_header("Content-type", "text/html; charset=utf-8")
        self.end_headers()
        self.wfile.write(html.encode("utf-8"))

    def do_GET(self):
        global currency_list
        parsed = urlparse(self.path)

        if parsed.path == "/":
            currencies = get_currencies()
            usd = next((c for c in currencies if c.char_code == "USD"), None)
            eur = next((c for c in currencies if c.char_code == "EUR"), None)

            template = env.get_template("index.html")
            html = template.render(
                myapp="Отслеживание курсов валют",
                navigation=self._nav(),
                author_name=main_author.name,
                author_group=main_author.group,
                usd=usd,
                eur=eur
            )
            self._send_response(html)

        elif parsed.path == "/author":
            template = env.get_template("author.html")
            html = template.render(
                title="Информация об авторе",
                navigation=self._nav(),
                author_name=main_author.name,
                author_group=main_author.group
            )
            self._send_response(html)

        elif parsed.path == "/users":
            template = env.get_template("users.html")
            html = template.render(
                title="Пользователи",
                navigation=self._nav(),
                users=users
            )
            self._send_response(html)

        elif parsed.path == "/user":
            params = parse_qs(parsed.query)
            if "id" not in params:
                self.send_error(400, "User ID required")
                return
            try:
                user_id = int(params["id"][0])
            except ValueError:
                self.send_error(400, "Invalid user id")
                return

            user_ = next((u for u in users if u.user_id == user_id), None)
            if not user_:
                self.send_error(404, "User not found")
                return

            # Получаем подписки
            subs_raw = UserCurrency.get_all()
            user_subs = []

            for uid, code in subs_raw:
                if uid != user_id:
                    continue
                currency = next((c for c in currency_list if c.char_code == code), None)
                if currency:
                    user_subs.append(UserCurrency(user_, currency))

            subscribed_codes = [sub.currency.char_code for sub in user_subs]

            # Формируем историю валют
            history = {}
            for sub in user_subs:
                char_code = sub.currency.char_code
                history[char_code] = get_currency_history(char_code, days=90)

            user_template = env.get_template("user.html")
            html = user_template.render(
                title=f"Профиль: {user_.name}",
                navigation=self._nav(),
                user=user_,
                subscriptions=user_subs,
                currencies=currency_list,
                subscribed_codes=subscribed_codes,
                history=history
            )
            self._send_response(html)

        elif parsed.path == "/currencies":
            currency_list = get_currencies()

            cur_template = env.get_template("currencies.html")
            html = cur_template.render(
                title="Курсы валют",
                navigation=self._nav(),
                currencies=currency_list
            )
            self._send_response(html)


        elif self.path == "/subscriptions":
            user_map = {u.user_id: u for u in users}
            currency_map = {c.char_code: c for c in currency_list}
            subs_raw = UserCurrency.get_all()
            user_currencies = []
            for uid, code in subs_raw:
                user_obj = user_map.get(uid)
                curr_obj = currency_map.get(code)
                if user_obj and curr_obj:
                    user_currencies.append(UserCurrency(user_obj, curr_obj))
            subscriptions_template = env.get_template("subscriptions.html")

            html = subscriptions_template.render(
                title="Подписки пользователей",
                navigation=self._nav(),
                users=users,
                user_currencies=user_currencies
            )
            self._send_response(html)

    def do_POST(self):
        length = int(self.headers["Content-Length"])
        data = self.rfile.read(length).decode()
        params = parse_qs(data)

        if self.path == "/subscribe":
            user_id = int(params["user_id"][0])
            currency_code = params["currency_code"][0]

            UserCurrency.add(user_id, currency_code)

            self.send_response(302)
            self.send_header("Location", f"/user?id={user_id}")
            self.end_headers()

        elif self.path.startswith("/unsubscribe"):
            query = parse_qs(urlparse(self.path).query)
            user_id = int(query["id"][0])
            length = int(self.headers.get("Content-Length", 0))
            body = self.rfile.read(length).decode()
            params = parse_qs(body)
            currency_code = params["currency"][0]
            try:
                UserCurrency.delete(user_id, currency_code)
            except Exception as e:
                print(f"[ERROR] Ошибка при отписке: {e}")  # Логирование ошибки
                self.send_response(500)
                self.end_headers()
                return
            self.send_response(302)
            self.send_header("Location", f"/user?id={user_id}")
            self.end_headers()

if __name__ == '__main__':
    httpd = HTTPServer(('localhost', 8080), MyServer)
    print('server is running at http://localhost:8080')
    httpd.serve_forever()
```

### Скриншоты страниц сайта

* Главная страница
![lab8_index.png](..%2Fassets%2Fimages%2Flab8_index.png)
* Список пользователей
![lab8_users.png](..%2Fassets%2Fimages%2Flab8_users.png)
* Страница пользователя
![lab8_user1.png](..%2Fassets%2Fimages%2Flab8_user1.png)
![lab8_user2.png](..%2Fassets%2Fimages%2Flab8_user2.png)
![lab8_user3.png](..%2Fassets%2Fimages%2Flab8_user3.png)
* Список валют
![lab8_currencies.png](..%2Fassets%2Fimages%2Flab8_currencies.png)
* Подписки пользователей
![lab8_subscriptions.png](..%2Fassets%2Fimages%2Flab8_subscriptions.png)
* Страница об авторе
![lab8_about_author.png](..%2Fassets%2Fimages%2Flab8_about_author.png)

### Тестирование
Для верификации надежности системы использован pytest:

* Модели: тестирование корректности установки значений и обработки ошибок валидации. 

* Контроллер: проверка доступности маршрутов /users, /currencies, /author и корректности статус-кодов.

* Шаблоны: проверка правильности отображения переданных данных в отрендеренном HTML-коде.

### Выводы
В ходе работы была реализована архитектура веб-приложения с нуля. 
Это дало глубокое понимание того, как данные проходят путь от API или базы данных до конечного пользователя в браузере через контроллеры и шаблоны. 
Особое внимание было уделено обработке граничных случаев, таких как пустые списки подписок или ошибки доступа к API.

### Ссылка на репозиторий
[Репозиторий проекта](https://github.com/kolyada-daria/py_itmo/tree/master/lab8)