# Bruno — практическое руководство

> Собрано по итогам настройки коллекции `DSH` (Bruno v4 / OpenCollection YAML).
> Все факты проверены на живом инстансе.

## 1. Что такое Bruno

- Open-source, **offline-first** API-клиент (альтернатива Postman/Insomnia).
- Коллекции хранятся как **plain-text файлы в папке** → нормально живут в git.
- Есть CLI (`bru run`) для CI.
- Поддерживает HTTP/REST, GraphQL, gRPC, WebSocket, SOAP.
- Коллекция = дерево папок + запросов + окружений.

## 2. Два формата хранения

| | Классический `.bru` | **OpenCollection YAML** |
|---|---|---|
| Версии | Bruno v1–v3 | Bruno **v3.1+ / v4** |
| Корень | `bruno.json` + `collection.bru` | `opencollection.yml` |
| Запрос | `*.bru` | `*.yml` |
| Окружение | `environments/*.bru` | `environments/*.yml` |
| Папка | `folder.bru` | `folder.yml` |

**Как определить формат:** если в корне лежит `opencollection.yml` — это YAML-формат; если `bruno.json` — классический. **Не смешивай** оба в одной папке (Bruno запутается, часть файлов будет помечена ошибкой).

## 3. OpenCollection YAML — структура

### 3.1 Корень `opencollection.yml`

```yaml
opencollection: "1.0.0"     # обязательный заголовок версии
info:
  name: DSH
config:                      # collection-level (proxy и т.п.)
  proxy:
    inherit: true
docs:
  content: "Описание коллекции"
  type: text/markdown
bundled: false               # false = дерево файлов на диске
extensions:
  bruno:
    ignore: [node_modules, .git]
```

### 3.2 Запрос (`*.yml`)

```yaml
info:
  name: List sessions
  type: http                 # http | graphql | grpc | websocket | folder
  seq: 1                     # порядок в UI

http:
  method: POST
  url: "{{baseUrl}}/api/session/list"
  headers:
    - name: content-type
      value: application/json
  params:
    query:
      - name: token
        value: "{{token}}"
  body:
    type: json               # json | text | xml | form-urlencoded | multipart-form | file
    data: |-
      { "type": "client-request", ... }

runtime:
  scripts:
    - type: before-request   # before-request | after-response | tests
      code: |-
        const token = String(bru.getEnvVar("token") ?? "").trim();
        if (!token) throw new Error("token is empty");
    - type: tests
      code: |-
        test("ok", function() {
          expect(res.status).to.equal(200);
          expect(res.body.result.ok).to.equal(true);
        });

settings:
  followRedirects: false     # не идти по 302
  timeout: 0
  encodeUrl: true

docs: |-
  Markdown-описание запроса.
```

### 3.3 WebSocket-запрос

```yaml
info:
  name: Follow session (stream)
  type: websocket            # ← именно "websocket", НЕ "ws"
  seq: 1

websocket:                  # ← именно блок "websocket:", НЕ "ws:"
  url: "{{wsUrl}}/api/remote.mux"
  headers:
    - name: cookie
      value: "{{apiCookie}}"
  message:
    type: text               # text | json | xml | binary
    data: |-
      {"type":"open","streamId":"follow-1","endpoint":"session/follow","payload":{"args":{"request":{"address":{"kind":"session","sessionId":"{{sessionId}}"}}}}}
```

> ⚠️ Схема OpenCollection — `additionalProperties: false`. Ключи `type: ws` и блок `ws:` дают красную ошибку у файла. Правильно — `websocket`.

### 3.4 Папка `folder.yml`

```yaml
info:
  name: "2. Session"
  type: folder
```

### 3.5 Окружение `environments/<name>.yml`

```yaml
name: DSH Local
variables:
  - name: baseUrl
    value: http://127.0.0.1:3080
  - name: token
    value: ""
```

Secret-переменная:

```yaml
  - secret: true
    name: token
    type: string
```

## 4. Переменные и подстановки

- `{{name}}` — обычная подстановка.
- Динамические: `{{$guid}}`, `{{$timestamp}}`, `{{$randomInt}}`, `{{$randomUUID}}` и т.д.
- Приоритет (выше → ниже):
  1. runtime (`bru.setVar`)
  2. request
  3. folder
  4. collection
  5. environment
  6. global environment
  7. process env (`process.env.*`)

## 5. Secrets vs Variables — главный подводный камень

| | Variables | Secrets (`secret: true`) |
|---|---|---|
| Где значение | в `.yml` (`value:`) | в локальной БД `bruno.db` |
| В `.yml` | есть `value` | **нет** `value` |
| Прозрачность | видно и диффится | скрыто, не диффится |

**Практика:** для локального токена лучше **обычная переменная** — значение видно, легко проверить на опечатку/перенос строки. Secret удобен для гита, но если он не сохранился/не активен, `{{token}}` подставится **пустым**, и запрос уйдёт с пустым токеном (получишь 401).

Путь к БД Bruno на macOS: `~/Library/Application Support/bruno/bruno.db`.

## 6. Cookies

- Bruno держит **cookie jar** и шлёт cookie между запросами на тот же хост.
- Ответ `/connect` в DSH — `302` + `set-cookie`; cookie подхватывается автоматически для HTTP.
- Для **WebSocket** надёжнее прокинуть cookie вручную: захватить `set-cookie` в `after-response` и положить в переменную `apiCookie`, затем `headers: [{name: cookie, value: "{{apiCookie}}"}]`.
- Достать заголовок: `res.getHeader("set-cookie")`.

## 7. CLI

```sh
bru run --env "DSH Local"                       # прогнать коллекцию
bru run --env "DSH Local" --folder "2. Session"
bru run --env ... --tests                        # только тесты
bru run --env ... --reporter-html out.html
bru run --env ... --bail
```

CI: `npm install -g @usebruno/cli`, затем `working-directory` = корень коллекции.

## 8. Git

- `.gitignore` → `environments/` (там токен) и `.DS_Store`.
- `.bru` / `.yml` — обычный текст, диффится построчно.
- Не коммить токен: держи в обычной переменной локально + gitignore, либо в secret.

## 9. Грабли, которые реально встретились

1. **`type: ws` / блок `ws:`** → файл с красной ошибкой. Нужно `type: websocket` + блок `websocket:`.
2. **Токен в Secrets не сохранился/не активен** → `{{token}}` пустой → 401 `dsh web authentication required`.
3. **Токен введён руками с опечаткой** — в base64url легко спутать `I`/`l` и `0`/`O`. Или скопирован **с переносом строки** → 401. Копировать только саму строку после `?token=`.
4. **`/connect` отвечает 302** → нужен `settings.followRedirects: false`, иначе Bruno может уйти на `/api/remote.mux` (это WebSocket-путь).
5. **Смешение форматов** `.bru` и `opencollection.yml` в одной папке.
6. **Порт меняется** при каждом старте `dsh web` — обновляй `baseUrl`/`wsUrl`.

## 10. Карта нашей коллекции `DSH`

```
DSH/
├── opencollection.yml
├── environments/DSH Local.yml
├── 1. Auth/connect                     token → cookie
├── 2. Session/
│   ├── list                            session/list
│   ├── model-catalog                   session/modelCatalog (smoke)
│   ├── create                          session/create (+ автозахват sessionId)
│   ├── prompt                          session/prompt
│   ├── projections                     session/projections (+ автозахват throughSeq)
│   ├── page                            session/page (история)
│   ├── search / rename / fork / select-model
├── 3. Streams/follow                   websocket session/follow
├── 4. System/plugin-inventory          pluginInventory/list
└── 5. Plugins/
    ├── list-plugins / list-bundles / registries
    ├── set-plugin-enabled / set-bundle-enabled
    └── install-bundle / remove-bundle
```
