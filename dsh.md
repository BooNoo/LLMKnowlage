# DSH — API и интеграция

> Собрано по исходникам DSH `0.2.1-rc.5` и проверено на живом инстансе `dsh --profile web`.

## 1. Что такое DSH

DeepSeek Harness — агентский рантайм:

- **Cordis** — плагинная архитектура (сервисы, эффекты, HMR).
- **Typert RPC** — типизированные Remote-методы `<namespace>/<method>` поверх транспорта.
- **Профили** — упорядоченные стеки *bundle-слоёв* (`web`, `headless`, `sdk`, `sdk-minimal`, `acp`, `desktop`).

Всё, что видно наружу, — это Remote-методы.

## 2. Три поверхности интеграции

| | Headless | SDK | Web API |
|---|---|---|---|
| Команда | `dsh --profile headless "задача"` | `dsh --profile sdk` | `dsh --profile web` |
| Транспорт | CLI, один запуск | JSON-RPC 2.0 поверх stdio | HTTP + WebSocket |
| Ответ | печатает финальный текст | `session/wait` + нотификации | `session/follow` / `session/page` |
| Когда | скрипты, CI | встраивание в код | работа с уже запущенным GUI |

## 3. Запуск Web

```sh
dsh --profile web --no-open
dsh --profile web --no-open --port 8080
dsh --profile web --host 192.168.1.10 --trusted-host 192.168.1.10 --no-open
```

- Печатает `dsh web: http://127.0.0.1:3080/?token=<RANDOM>`.
- По умолчанию открывает браузер (`--no-open` отключает).
- `--host 0.0.0.0` запрещён (только конкретный IP).
- Порт по умолчанию — **3080**.

## 4. Аутентификация Web API

- **launch-token** — случайный, **per-process**, только в памяти. Не персистится, из env/файла не достать.
- В **desktop-приложении** токен уходит в Electron по IPC и **никуда не печатается** (в URL браузера он тут же редиректится на чистый).
- Обмен: `GET /connect?token=<TOKEN>&interactions=none` → `302` + signed cookie.
- Cookie: `dsh-auth-<authority>=v2...`, authority-bound, `HttpOnly`, `SameSite=Strict`, `Path=/`, 30 дней.
- **Host/Origin fence**: loopback / bind-IP / `trustedHosts`. Иначе `403`.
- `401` = токен не тот; `403` = Host/Origin.

> Один токен = один процесс. После рестарта `dsh web` токен меняется.

## 5. Wire-протокол

### 5.1 Unary (HTTP)

```
POST /api/<namespace>/<method>
Content-Type: application/json
Cookie: <cookie из /connect>

{
  "type": "client-request",
  "rpcId": "<любая строка>",
  "method": "<namespace>/<method>",
  "payload": { "args": { ... } }
}
```

Ответ:

```json
{ "type": "server-response", "rpcId": "<echo>", "result": { "ok": true, "value": { ... } } }
```

Ошибка — **внутри** конверта (HTTP остаётся 200):

```json
{ "result": { "ok": false, "error": { "code": "...", "message": "...", "details": { ... } } } }
```

Бинарные результаты → `multipart/form-data` (`metadata` + `bytes-N`).

### 5.2 Streams (WebSocket)

```
ws://<host>/api/remote.mux      (тот же cookie)
```

**client → host:**

```json
{"type":"open","streamId":"s1","endpoint":"session/follow","payload":{"args":{...}}}
{"type":"item","streamId":"s1","value":{...}}
{"type":"end","streamId":"s1"}
{"type":"cancel","streamId":"s1"}
```

**host → client:**

```json
{"type":"item","streamId":"s1","value":{...}}
{"type":"error","streamId":"s1","error":{"code":"...","message":"...","details":{}}}
{"type":"end","streamId":"s1"}
```

## 6. Формат `args`

- метод с одним параметром `request` → `{"args":{"request":{...}}}`
- `session/list` → `{"args":{"_request":{}}}` (поле называется именно `_request`)
- без параметров → `{"args":{}}`

## 7. Session API

| Метод | args | Назначение |
|---|---|---|
| `session/list` | `{_request:{}}` | список сессий |
| `session/create` | `{request:{}}` (опц. `sessionId`,`cwd`,`workspaceId`,`agentPreset`) | создать |
| `session/prompt` | `{request:{requestId,sessionId,mode,content}}` | поставить сообщение |
| `session/page` | `{request:{address,throughSeq,beforeSeq?,maxMessages?}}` | **история** |
| `session/projections` | `{request:{sessionId}}` | свёрнутое состояние |
| `session/follow` | stream `{request:{address,assistantStream?,maxMessages?}}` | baseline + live |
| `session/search` | `{request:{query}}` | поиск |
| `session/rename` | `{request:{sessionId,title}}` | переименовать |
| `session/fork` | `{request:{sessionId,atSeq?}}` | форк |
| `session/selectModel` | `{request:{sessionId,provider,model,reasoningEffort?}}` | модель |
| `session/modelCatalog` | `{}` | каталог моделей |

### 7.1 `prompt` — это квитанция

`session/prompt` возвращает `{accepted:true}` — сообщение поставлено в очередь. **Ответа в нём нет.**

### 7.2 Как читать данные

- `session/projections` — только **проекции** (title, todo, cwd, tokenUsage, inbox…), не лог.
- `session/page` — постраничная **история**. `throughSeq` обязателен.
  - Источник `throughSeq`: `projections.asOfSeq` **или** `follow` → `snapshot.cursor` (это один и тот же watermark).
  - Пагинация назад: `beforeSeq` = seq первого события предыдущей страницы.
  - Возвращает `{records:[{type:"event",event:{type,seq,time,data}}], hasMore}`.
- `session/follow` — открывающий `snapshot` (`records` + `cursor` + `hasMore`) и затем live-события.

### 7.3 События хода

```
user/message → turn/start → step/start → assistant/message
            → tool/call → tool/result → step/end → … → turn/end
```

- `turn/end.data.reason`: `completed` / `error` / `aborted`.
- Финальный ответ = последний непустой `assistant/message`.
- Служебные: `system/message`, `permission/preset`, `model/selection`, `agent/inbox/spliced`.

## 8. Plugin API

### 8.1 Инвентарь (read-only)

`pluginInventory/list` → `{ entries, agentPresets, managementAvailable }`

```json
{ "entryId": "include:session", "moduleName": "@deepseek-ai/dsh-session",
  "enabled": true, "fiberPhase": "active",
  "meta": { "title": {"en":"session","zh":"…"}, "description": {...} } }
```

### 8.2 Управление

| Метод | args | Назначение |
|---|---|---|
| `pluginManager/listPlugins` | `{}` | плагины + `patchId`/`readOnlyReason` (массив) |
| `pluginManager/listBundles` | `{}` | bundle'ы (массив) |
| `pluginManager/registries` | `{}` | реестры |
| `pluginManager/setPluginEnabled` | `{id, enabled}` | вкл/выкл плагин |
| `pluginManager/setBundleEnabled` | `{name, enabled}` | вкл/выкл bundle |
| `pluginManager/installBundle` | `{spec, options?}` | установка через pnpm |
| `pluginManager/removeBundle` | `{name}` | удаление |
| `pluginManager/inspect` | `{spec, options?}` | проверка spec до установки |
| `pluginManager/waitForInstall` | `{requestId}` | дождаться установки |
| `pluginManager/cancelInstall` | `{requestId}` | отменить установку |

**Важно:** `setPluginEnabled.id` — это **`entryId`** (напр. `include:tool-plugin-manager`), не `patchId`. Строки с `readOnlyReason` менять нельзя.

## 9. Bundles vs Plugins

```
пустой root
  ← patch bundle #1  (@deepseek-ai/dsh-base)
  ← patch bundle #2  (@deepseek-ai/dsh-web-app: host-строки + пресеты)
  ← профильный cordis.patch.yml
  ← $DSH_HOME/cordis.patch.yml
  ← --patch
        ↓
   итоговое дерево plugin-строк (например 194)
```

- **Bundle** — пакет с `dsh.bundle.patch` в `package.json` → один/несколько `cordis.patch.yml`. Это *слой композиции*: объявляет/переопределяет строки плагинов. Вкл/выкл = правка `dsh.profile.bundles` в `package.json`. Выключение сохраняет зависимость; включение добавляет в конец (меняет приоритет).
- **Plugin** — одна *строка* в рантайм-дереве. Вкл/выкл = правка `disabled` в `cordis.patch.yml` профиля.

## 10. Headless и SDK

### Headless

```sh
dsh --profile headless "запусти тесты"
dsh --profile headless --json "..."             # NDJSON-события
dsh --profile headless --session-id <id> "..."  # продолжить
```

- Блокируется, печатает финальный ответ, exit 0/1.
- `--json`: `session`, `status`, `text`, `thinking`, `tool_call`, `tool_result`, `final`, `error`.
- Креды: `DEEPSEEK_API_KEY` из env / `.env` / `$DSH_HOME/.env`.

### SDK (stdio, JSON-RPC 2.0, одна строка = одно сообщение)

Запросы: `initialize`, `session/prompt`, `session/resume`, `session/wait`, `session/working-directory/get|set`, `shutdown`.
Нотификации: `session.event`, `session.status`, `subagent.started`, `subagent.finished`.

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"provider":"deepseek-official","model":"deepseek-flash"}}
{"jsonrpc":"2.0","id":2,"method":"session/prompt","params":{"sessionId":"s1","content":[{"type":"text","text":"hi"}]}}
```

`session/wait` — явное «дождись idle».

## 11. Рецепты (curl)

```sh
# 1) обмен токена на cookie
curl -i -c /tmp/dsh.jar "http://127.0.0.1:3080/connect?token=<TOKEN>&interactions=none"

# 2) список сессий
curl -b /tmp/dsh.jar -H 'Content-Type: application/json' \
  -d '{"type":"client-request","rpcId":"1","method":"session/list","payload":{"args":{"_request":{}}}}' \
  http://127.0.0.1:3080/api/session/list

# 3) история
curl -b /tmp/dsh.jar -H 'Content-Type: application/json' \
  -d '{"type":"client-request","rpcId":"2","method":"session/page","payload":{"args":{"request":{"address":{"kind":"session","sessionId":"<SID>"},"throughSeq":81}}}}' \
  http://127.0.0.1:3080/api/session/page
```

## 12. Подводные камни

1. **Токен per-process** — после рестарта `dsh web` старый токен невалиден (401).
2. **Desktop не печатает токен** — для API удобнее свой `dsh --profile web --no-open`.
3. **`prompt` ≠ ответ** — это квитанция; читай `page`/`follow`.
4. **`projections` ≠ история** — только свёрнутое состояние.
5. **`throughSeq` обязателен** в `session/page`.
6. **`session/list` → `_request`**, остальные → `request`.
7. **Мутации `pluginManager` persist'ятся** в профиль; часть требует рестарта.
8. **`installBundle` идёт через pnpm** — сеть, может быть долго.
