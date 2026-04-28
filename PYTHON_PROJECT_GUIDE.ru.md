# changedetection.io как учебный Python-проект

Этот файл - практический путеводитель по проекту для разработчика с опытом C#/JavaScript/TypeScript/HTML/CSS, но без большого опыта Python.

Короткий вердикт: проект хорошо подходит для изучения Python на реальном коде, если идти по нему выборочно. Это не учебный минимальный пример, а живое Flask-приложение с API, фоновыми воркерами, очередями, парсингом веб-страниц, уведомлениями, тестами, Docker и упаковкой в pip-пакет.

## Что это за приложение

changedetection.io отслеживает изменения на веб-страницах и отправляет уведомления. Пользователь добавляет URL, приложение периодически получает содержимое, фильтрует его, сравнивает с предыдущим снимком, сохраняет историю и при необходимости отправляет уведомление.

Упрощенный поток:

1. Пользователь добавляет watch через UI или API.
2. Watch сохраняется в datastore.
3. Планировщик/очередь ставит watch на проверку.
4. Worker получает страницу через fetcher.
5. Processor извлекает и нормализует содержимое.
6. Diff-логика сравнивает новое состояние со старым.
7. Результат сохраняется.
8. Notification layer отправляет уведомления.

## Точки входа

Главные точки входа:

- `changedetection.py` - маленький CLI-wrapper для прямого запуска из репозитория.
- `changedetectionio/__init__.py` - содержит `main()`, разбор CLI-аргументов, подготовку datastore, запуск приложения.
- `setup.py` - объявляет console script: `changedetection.io=changedetectionio:main`.
- `changedetectionio/flask_app.py` - создание Flask-приложения, регистрация routes/API, настройка Jinja, CSRF, CORS, Socket.IO, очередей.

Если смотреть глазами C#/ASP.NET или Node/Express:

- `changedetectionio/flask_app.py` похож на смесь `Program.cs`, DI/bootstrap и route registration.
- `changedetectionio/api/*.py` похожи на API controllers/resources.
- `changedetectionio/blueprint/*` похожи на feature modules/controllers для UI.
- `changedetectionio/model/*` похожи на domain models, но с заметным legacy: часть моделей наследуется от `dict`.
- `changedetectionio/store/*` похож на persistence/repository layer.
- `changedetectionio/worker.py` и `worker_pool.py` похожи на background services.

## Основная структура

### Корень проекта

- `README.md`, `README-pip.md` - описание продукта и пользовательский запуск.
- `requirements.txt` - runtime и test-зависимости.
- `setup.py`, `setup.cfg`, `MANIFEST.in` - упаковка Python-пакета.
- `Dockerfile`, `docker-compose.yml`, `docker-entrypoint.sh` - контейнерный запуск.
- `.ruff.toml`, `.pre-commit-config.yaml` - lint/format/check tooling.
- `docs/api-spec.yaml` - OpenAPI-спецификация.

### `changedetectionio/api`

REST API-слой. Здесь удобно изучать Flask-RESTful-подход: request parsing, auth, resource classes, responses, ошибки.

Хорошие стартовые файлы:

- `api/SystemInfo.py`
- `api/Search.py`
- `api/Spec.py`
- `api/Watch.py` позже, когда будет понятнее модель watch.

### `changedetectionio/blueprint`

UI-oriented feature modules. Flask Blueprint - это способ разнести routes по функциональным областям.

Примеры:

- `blueprint/ui/*` - основные UI-экраны.
- `blueprint/rss/*` - RSS endpoints.
- `blueprint/settings/*` - настройки.
- `blueprint/tags/*` - группы/теги.
- `blueprint/imports/*` - импорт.

### `changedetectionio/templates` и `changedetectionio/static`

Server-rendered UI на Jinja2 + статические ассеты. Для человека с HTML/CSS/JS это понятная зона, но стоит помнить: Jinja-шаблоны рендерятся на сервере, ближе к Razor/EJS/Nunjucks, чем к React/Vue.

### `changedetectionio/model`

Domain model layer.

Важные файлы:

- `model/Watch.py` - центральная модель отслеживаемой страницы. Большой и сложный файл.
- `model/App.py` - модель настроек приложения.
- `model/Tag.py`, `model/Tags.py` - теги/группы.
- `model/persistence.py` - mixin для сохранения.
- `model/schema_utils.py` - вспомогательная логика схем.

Важно: `Watch.py` сам документирует технический долг. Модель наследуется от `dict`, а в комментариях описано, что в идеале это можно было бы заменить на Pydantic-like модель. Это полезно для обучения реальному Python, но не стоит воспринимать как образцовый современный domain modeling.

### `changedetectionio/store`

Persistence layer. Проект использует file-based datastore, то есть данные хранятся в файловой структуре, а не в PostgreSQL/MySQL.

Файлы:

- `store/__init__.py` - основная логика datastore.
- `store/file_saving_datastore.py` - файловое сохранение.
- `store/updates.py` - миграции/обновления структуры данных.
- `store/base.py` - базовые элементы слоя хранения.

### `changedetectionio/content_fetchers`

Слой получения контента.

Файлы:

- `content_fetchers/requests.py` - обычные HTTP-запросы.
- `content_fetchers/playwright.py` - browser-based fetching через Playwright.
- `content_fetchers/puppeteer.py` - browser-based fetching через Puppeteer.
- `content_fetchers/webdriver_selenium.py` - Selenium.
- `content_fetchers/base.py` - базовая логика/контракт.

Это хороший блок для изучения абстракций, наследования, обработки ошибок, интеграции с внешними библиотеками и разницы между simple HTTP и полноценным браузерным рендерингом.

### `changedetectionio/processors`

Слой обработки полученного контента.

Основные направления:

- `processors/text_json_diff/*` - текст/HTML/JSON processing.
- `processors/restock_diff/*` - логика товаров, цен, наличия.
- `processors/image_ssim_diff/*` - сравнение изображений/screenshot.
- `processors/base.py`, `processors/magic.py`, `processors/extract.py` - общие механизмы.

Это одна из самых полезных зон для изучения прикладного Python: строки, HTML parsing, JSON, регулярные выражения, исключения, композиция функций.

### `changedetectionio/diff`

Diff/tokenizer logic.

Хорошее место для старта:

- `diff/tokenizers/natural_text.py`
- `diff/tokenizers/words_and_html.py`

Тут меньше инфраструктуры и больше чистой логики.

### `changedetectionio/notification`

Слой уведомлений. Проект использует `apprise`, чтобы поддерживать много каналов уведомлений.

Смотреть после того, как понятен основной поток watch -> worker -> processor -> changed/not changed.

### `changedetectionio/conditions`

Условия срабатывания. Полезно для изучения plugin-like архитектуры и бизнес-правил.

### `changedetectionio/llm`

AI/LLM-related функциональность: построение prompt, parsing response, evaluator, trimming больших текстов.

Хорошо изучать позже, когда уже понятна основная архитектура. Иначе LLM-часть будет отвлекать от базового Python/Flask.

### `changedetectionio/tests`

Большая тестовая база. Это один из лучших способов изучать проект.

Полезные стартовые тесты:

- `tests/unit/test_time_handler.py`
- `tests/test_html_to_text.py`
- `tests/test_jsonpath_jq_selector.py`
- `tests/test_api_search.py`
- `tests/llm/test_response_parser.py`

## Основные зависимости

Минимальная версия Python указана в `setup.py`: `>=3.10`.

Ключевые зависимости из `requirements.txt`:

- `flask`, `flask-login`, `flask-restful`, `flask-wtf`, `flask-cors`, `flask-socketio` - web/API/UI/session stack.
- `jinja2` - server-side templates.
- `requests`, `requests-file`, `requests[socks]` - HTTP fetching.
- `beautifulsoup4`, `lxml`, `inscriptis` - HTML parsing/extraction.
- `jsonpath-ng`, `jq` - JSON extraction/filtering.
- `selenium`, `pyppeteer-ng`, `greenlet` - browser automation/fetching.
- `apprise` - уведомления.
- `diff_match_patch`, `levenshtein` - сравнение текста.
- `arrow`, `pytz`, `tzdata`, `timeago` - даты/время/timezones.
- `pytest`, `pytest-flask`, `pytest-mock`, `pytest-xdist` - тесты.
- `ruff`, `pre_commit` - code quality tooling.
- `litellm`, `rank-bm25` - LLM-related функциональность.

## Что изучать в каком порядке

### 1. Python-синтаксис на маленьких модулях

Начать с файлов, где мало framework magic:

- `changedetectionio/time_handler.py`
- `changedetectionio/strtobool.py`
- `changedetectionio/favicon_utils.py`
- `changedetectionio/diff/tokenizers/natural_text.py`
- `changedetectionio/llm/response_parser.py`

Что смотреть:

- imports;
- функции;
- type hints;
- exceptions;
- enum;
- работа со строками и dict/list;
- отличие Python truthy/falsy от JS/C# привычек.

### 2. Тесты рядом с логикой

После каждого маленького модуля открывать соответствующие тесты.

Пример:

- код: `changedetectionio/time_handler.py`
- тесты: `changedetectionio/tests/unit/test_time_handler.py`

Цель: привыкнуть к Python testing workflow. Для тебя это будет аналогично чтению Jest/xUnit/NUnit tests как спецификации поведения.

### 3. Flask basics

Перейти к небольшим API-файлам:

- `changedetectionio/api/SystemInfo.py`
- `changedetectionio/api/Search.py`
- `changedetectionio/api/Spec.py`

Что понять:

- как Flask получает request;
- как возвращается response;
- как устроены resource classes;
- где подключается auth/CSRF;
- как API регистрируется в `flask_app.py`.

### 4. UI через Flask Blueprint и Jinja

Смотреть:

- `changedetectionio/blueprint/ui/views.py`
- `changedetectionio/blueprint/rss/blueprint.py`
- `changedetectionio/templates/base.html`
- `changedetectionio/templates/menu.html`

Цель: связать route -> Python handler -> Jinja template -> HTML.

### 5. Domain model и datastore

Смотреть осторожно, потому что файлы крупные:

- `changedetectionio/model/App.py`
- `changedetectionio/model/Tag.py`
- `changedetectionio/model/Watch.py`
- `changedetectionio/store/base.py`
- `changedetectionio/store/file_saving_datastore.py`
- `changedetectionio/store/__init__.py`

Цель: понять, где живут данные и как watch превращается из UI/API input в сохраненную сущность.

### 6. Fetchers и processors

Смотреть:

- `changedetectionio/content_fetchers/base.py`
- `changedetectionio/content_fetchers/requests.py`
- `changedetectionio/processors/base.py`
- `changedetectionio/processors/text_json_diff/processor.py`
- `changedetectionio/processors/extract.py`

Цель: понять центральный бизнес-процесс: получить страницу, извлечь полезное содержимое, подготовить к сравнению.

### 7. Worker/queue lifecycle

Смотреть:

- `changedetectionio/custom_queue.py`
- `changedetectionio/queue_handlers.py`
- `changedetectionio/worker.py`
- `changedetectionio/worker_pool.py`

Цель: понять background execution, concurrency, shutdown, error handling.

### 8. Большие фичи

Когда база понятна:

- notifications;
- restock/price detection;
- image diff;
- browser steps;
- LLM evaluation;
- plugins.

## Как запускать в dev-режиме

Ниже команды для Windows PowerShell из корня репозитория.

### 1. Создать виртуальное окружение

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

Если Python 3.12 не установлен, можно использовать Python 3.10 или 3.11:

```powershell
py -3.11 -m venv .venv
```

### 2. Установить зависимости

```powershell
pip install -r requirements.txt
pip install -e .
```

`pip install -e .` ставит проект в editable mode: изменения в исходниках сразу используются без переустановки пакета.

На Windows часть опциональных возможностей может отличаться от Linux/macOS. Например, `jq` в requirements включен только для Linux/macOS, а browser-based fetching может потребовать отдельной установки браузеров/раннеров. Для первого изучения лучше начать с обычного requests-based режима.

### 3. Создать директорию для данных

```powershell
New-Item -ItemType Directory -Force .\.dev-data
```

### 4. Запустить приложение

Вариант через console script:

```powershell
changedetection.io -d .\.dev-data -p 5000 -l DEBUG
```

Вариант напрямую через wrapper:

```powershell
python .\changedetection.py -d .\.dev-data -p 5000 -l DEBUG
```

После запуска открыть:

```text
http://127.0.0.1:5000
```

### 5. Полезные CLI-флаги

Смотреть help:

```powershell
python .\changedetection.py --help
```

Часто полезные флаги:

- `-d PATH` - путь к datastore.
- `-p PORT` - порт, по умолчанию 5000.
- `-h HOST` - host, по умолчанию `0.0.0.0`.
- `-l LEVEL` - уровень логирования: `DEBUG`, `INFO`, `WARNING` и т.д.
- `-u URL` - добавить URL при запуске.
- `-r all` - поставить все watches на перепроверку при запуске.
- `-b` - batch mode: обработать очередь и выйти.

Пример:

```powershell
python .\changedetection.py -d .\.dev-data -p 5000 -l DEBUG -u https://example.com
```

## Как дебажить в VS Code

Да, этот проект можно нормально отлаживать в VS Code: ставить breakpoint в Python-файлах, запускать приложение под debugger, смотреть переменные, call stack и шагать по коду.

### 1. Подготовить VS Code

Установить расширение Microsoft Python:

- `Python`
- `Pylance`

Затем выбрать интерпретатор из виртуального окружения:

```text
Ctrl+Shift+P -> Python: Select Interpreter -> .\.venv\Scripts\python.exe
```

Если проект запускался без `.venv`, можно выбрать тот Python, в который были установлены зависимости через `pip install -r requirements.txt`. Но для обучения и отладки лучше использовать отдельное виртуальное окружение.

### 2. Создать `.vscode/launch.json`

В корне проекта создать файл `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug changedetection.io",
      "type": "debugpy",
      "request": "launch",
      "program": "${workspaceFolder}/changedetection.py",
      "args": [
        "-d",
        "${workspaceFolder}/.dev-data",
        "-p",
        "5000",
        "-l",
        "DEBUG"
      ],
      "console": "integratedTerminal",
      "justMyCode": false
    }
  ]
}
```

`justMyCode: false` полезен для изучения проекта: debugger сможет заходить не только в твой код, но и в код зависимостей/фреймворка, если это нужно. Если хочется меньше шума, можно поставить `true`.

### 3. Запустить отладку

1. Поставить breakpoint, например в `changedetectionio/__init__.py`, `changedetectionio/flask_app.py` или `changedetectionio/blueprint/ui/views.py`.
2. Открыть вкладку Run and Debug в VS Code.
3. Выбрать `Debug changedetection.io`.
4. Нажать Start Debugging.
5. Открыть в браузере:

```text
http://127.0.0.1:5000
```

Когда запрос попадет в код с breakpoint, VS Code остановит выполнение.

### 4. Где удобно ставить breakpoint

- `changedetection.py` - самый внешний wrapper запуска.
- `changedetectionio/__init__.py` - разбор аргументов и запуск приложения.
- `changedetectionio/flask_app.py` - создание Flask app и регистрация routes.
- `changedetectionio/blueprint/ui/views.py` - UI routes.
- `changedetectionio/api/*.py` - API endpoints.
- `changedetectionio/worker.py` - background processing.
- `changedetectionio/processors/*` - обработка полученного контента.

Практичный первый сценарий: поставить breakpoint в `changedetectionio/blueprint/ui/views.py`, открыть главную страницу и посмотреть, как Flask route превращается в HTML response.

## Как запускать тесты

Все тесты:

```powershell
pytest
```

Один файл:

```powershell
pytest changedetectionio\tests\unit\test_time_handler.py -v
```

Один тест:

```powershell
pytest changedetectionio\tests\unit\test_time_handler.py::TestAmIInsideTime::test_invalid_day_of_week -v
```

Если полный набор тестов тяжелый или требует внешних компонентов, начинай с unit/small tests и постепенно расширяй область.

## Как читать Python-код в этом проекте

Практичный порядок чтения файла:

1. Imports: какие внешние зависимости и внутренние модули участвуют.
2. Module-level constants/globals: в этом проекте их много, особенно в app/bootstrap слоях.
3. Маленькие функции: они обычно дают словарь терминов проекта.
4. Public classes/functions: что используется снаружи.
5. Tests: какие сценарии авторы считают важными.
6. Call sites: где функция вызывается. Для этого удобно использовать `rg "function_name"`.

## На что обратить внимание как C#/TS-разработчику

- Python module import выполняет код верхнего уровня файла. Поэтому side effects при import важнее, чем в C#/TS.
- В проекте есть глобальные объекты Flask app/queues/datastore. Это типично для Flask-приложений, но требует внимательности в тестах и concurrency.
- `dict` используется как модель данных чаще, чем хотелось бы в строгом TypeScript/C# стиле.
- Type hints есть не везде. Python runtime не проверяет типы сам по себе.
- Exceptions являются нормальной частью flow при работе с I/O, network, parsing.
- `with open(...)` - аналог `using`/`IDisposable` для ресурсов.
- `pytest` fixtures концептуально похожи на test setup/DI, но синтаксис другой.
- Jinja templates похожи на Razor/EJS/Nunjucks: server-side rendering, template inheritance, filters/helpers.

## Хорошие учебные задания на этом проекте

1. Добавить unit-test на небольшую функцию и сломать/починить его.
2. Найти route в UI, пройти путь до template и обратно.
3. Добавить маленький helper в `time_handler.py` или tokenizer и покрыть тестом.
4. Добавить новый API response field в небольшой endpoint.
5. Разобрать один processor на входы/выходы и нарисовать для себя flow.
6. Найти место, где создается watch, и проследить его до сохранения.
7. Запустить приложение с пустым datastore, добавить watch, посмотреть какие файлы появились в `.dev-data`.

## Что пока не трогать первым делом

- Полный `flask_app.py` целиком. Лучше использовать как карту подключений.
- `model/Watch.py` целиком. Читать по конкретным методам и тестам.
- Browser automation через Playwright/Puppeteer/Selenium, пока не понятен requests-based path.
- Image diff и LLM-фичи, пока не понятен основной pipeline.
- Docker production setup, если цель именно Python-код.

## Рекомендуемый маршрут на 2 недели

День 1-2:

- Запустить проект.
- Прочитать `changedetection.py`, `changedetectionio/__init__.py` до `main()`.
- Запустить один маленький тест.

День 3-4:

- Разобрать `time_handler.py` и его тесты.
- Написать один дополнительный тест.

День 5-6:

- Разобрать `api/SystemInfo.py`, `api/Search.py`.
- Найти регистрацию API в `flask_app.py`.

День 7-8:

- Разобрать один Blueprint и один Jinja template.
- Понять route -> handler -> template.

День 9-10:

- Разобрать `content_fetchers/requests.py` и базовый fetch flow.
- Найти, где fetcher вызывается из worker/processor.

День 11-12:

- Разобрать `processors/text_json_diff`.
- Понять, как из HTML/JSON получается сравниваемый текст.

День 13-14:

- Разобрать сохранение watch в datastore.
- Добавить маленькое изменение с тестом.

## Итог

Для абсолютного новичка этот проект был бы слишком большим. Для разработчика C#/JS/TS он подходит хорошо: знакомые архитектурные идеи помогут не утонуть, а Python будет изучаться на реальных задачах. Лучший подход - не читать проект линейно, а идти по вертикальным срезам: endpoint, test, model, persistence, worker, processor.
