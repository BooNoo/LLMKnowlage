# DSH как внешний harness: «harness as code»

> **Статус:** концепция + анализ реализуемости (по исходникам DSH `0.2.1-rc.5`). PoC ещё не собран.
> Связанные документы: [dsh.md](./dsh.md), [bruno.md](./bruno.md).

## 1. Идея

```
(DSH + мои bundles/plugins)  <--API-->  моя система
```

- Мои bundle'ы и плагины живут **отдельно** (каждый — свой npm-пакет/репозиторий).
- Есть **один файл описания harness** (аналог `docker-compose.yml`).
- По нему поднимается **настроенный инстанс DSH** — с моими слоями, плагинами и конфигом.
- Снаружи система общается с ним по API (HTTP/WS или stdio).

## 2. Вердикт

**Реализуемо и довольно чисто**, но с важной оговоркой:

- Нативно DSH описывает harness как **директорию профиля** (2 файла + `node_modules`), а **не** как один файл.
- «Один файл» достигается **тонким генератором**: `harness.yml` → материализация в профиль → `pnpm install` → `dsh --profile …`.

Аналогия с compose верна на уровне **слоёв**: bundle'ы ≈ `services:`, профиль ≈ `docker-compose.yml`, `--patch` ≈ override-файлы.

## 3. Механика DSH

### 3.1 Профиль — это директория

`$DSH_HOME/profiles/<name>/`:

```
package.json          # { dsh: { profile: { bundles: [...] } }, dependencies: {...} }
cordis.patch.yml      # твой patch-слой (top-level YAML array)
cordis.yml            # [] — авто-генерируемый пустой root, ПЕРЕЗАПИСЫВАЕТСЯ каждый boot
pnpm-workspace.yaml   # packages: ['.'], nodeLinker: hoisted, autoInstallPeers: false
compatibility.json    # опционально: exemptions по версиям
node_modules/         # установленные bundle'ы
```

Точный манифест (из `initProfile`):

```json
{
  "name": "dsh-profile-my-harness",
  "private": true,
  "dependencies": {},
  "dsh": { "profile": { "bundles": ["@deepseek-ai/dsh-base", "@my-org/dsh-bundle-core"] } }
}
```

### 3.2 Порядок композиции (ниже → выше приоритет)

```
1. bundle-слои в порядке dsh.profile.bundles
2. профильный cordis.patch.yml
3. $DSH_HOME/cordis.patch.yml          (машинные настройки, перекрывает профиль)
4. --patch <file> (repeatable, по порядку argv)
5. telemetry switch
```

### 3.3 Patch-язык — это и есть «compose»

`cordis.patch.yml` — top-level YAML-массив:

```yaml
# таргетная правка существующей строки по id (last write wins)
- id: system-prompt
  config:
    personaPrefix: "You are an agent for My System."

# выключить строку
- id: hmr
  disabled: true

# вставить новые строки (плагины)
- insert:
    - id: my-tools
      name: '@my-org/dsh-tools'
      inject: [tools]
      config:
        endpoint: !!js process.env.MY_SYSTEM_URL
```

Правила:

- `!!js` — JS-выражения, вычисляются при композиции (напр. `disabled: !!js "!ctx.get('profileContext')"`).
- Патч **заменяет весь `config` строки, а не мержит** — перечисляй все нужные поля.
- Ссылка на несуществующий `id` → warning в stderr.
- Пустой/только-комментарии файл **ломает boot** → пиши `[]`.

### 3.4 Bundle — пакет с патчем

```json
// @my-org/dsh-bundle-core/package.json
{
  "name": "@my-org/dsh-bundle-core",
  "version": "1.0.0",
  "dsh": { "manifestVersion": 1, "bundle": { "patch": "./cordis.patch.yml" } }
}
```

`patch` может быть списком файлов — применяются по порядку как **один слой**.

### 3.5 Инструменты

| Инструмент | Зачем |
|---|---|
| `dsh --profile X --from-default-profile headless` | создать профиль из шаблона |
| `dsh plugin --profile X add <spec>` | установка (pnpm) + обновление `dsh.profile.bundles` |
| `--patch <file>` | внешний overlay без правки профиля |
| `--dump-config` | **превью итогового дерева** (с `!!js` verbatim + какими слоями изменено) |
| `--dump-config-schema` | JSON Schema скомпонованных конфигов |
| `pluginManager/installBundle` | то же из API (мутации persist'ятся в профиль) |

Спецификация пакета для установки: имя из реестра, **абсолютный путь**, **git-адрес**, **tarball**.
В `insert.name` — ещё и file URL / `./` относительно патча.

## 4. «Один файл» на практике

```
my-harness/
├── harness.yml        # ← единый источник истины
├── generate.mjs       # тонкий материализатор
└── bundles/           # каждый — свой пакет/репо
    ├── dsh-bundle-core/
    └── dsh-bundle-tools/
```

```yaml
# harness.yml
dsh: "0.2.1-rc.5"                # pin версии рантайма
profile: my-harness
mode: headless                    # headless | web | sdk
bundles:
  - "@deepseek-ai/dsh-base"
  - "@deepseek-ai/dsh-headless"
  - "@my-org/dsh-bundle-core"
  - "@my-org/dsh-bundle-tools"
patch: |
  - id: llm
    config: { provider: deepseek-official, model: deepseek-flash }
```

Генератор повторяет `initProfile`:

```js
// generate.mjs (псевдо)
const dir = `${process.env.DSH_HOME}/profiles/${cfg.profile}`;
mkdirSync(dir, { recursive: true });
writeFileSync(`${dir}/package.json`, JSON.stringify({
  name: `dsh-profile-${cfg.profile}`,
  private: true,
  dependencies: {},                          // сюда — file:/git: на твои bundle'ы
  dsh: { profile: { bundles: cfg.bundles } }
}, null, 2));
writeFileSync(`${dir}/cordis.patch.yml`, cfg.patch);
writeFileSync(`${dir}/cordis.yml`, "[]\n");
```

Запуск:

```sh
node generate.mjs && pnpm install --dir "$DSH_HOME/profiles/my-harness"
dsh --profile my-harness --patch ./env-prod.patch.yml
```

## 5. Главные сложности

1. **Нет single-file режима.** Профиль — директория; `cordis.yml` принудительно перезаписывается в `[]` на каждом boot. Единственные persistent-поверхности — `cordis.patch.yml` (+ home + `--patch`).
2. **Patch = replace, не merge.** Меняешь одно поле — перечисляй весь конфиг строки; легко «потерять» настройки из bundle.
3. **`!!js` — исполняемый код.** Композиция вычисляет выражения, а `--dump-config-schema` **импортирует модули**. Не применять к недоверенным bundle'ам.
4. **Compatibility gate + тихий skip.** Проверяется `engines.dsh` (или peer-зависимости на `@deepseek-ai/dsh*`). Несовместимый bundle **пропускается** (`skippedBundles`, warning) — профиль может «успешно» подняться **без твоих фич**. Проверять в CI.
5. **Required entries.** Обязательные id: `agent-loop`, `webserver`, `web-runtime`, `modules`, `connection`, `headless-runner`, `acp`, `sdk-jsonrpc-server`. Нельзя собрать «голый» harness без `dsh-base` + mode-bundle.
6. **HMR и детерминизм.** `web`-шаблон включает live reload, остальные — startup-only. Для деплоя фиксировать startup-only.
7. **Drift: generated vs mutated.** Если генерируешь профиль из `harness.yml`, а потом дёргаешь `installBundle` (пишет в `package.json` + патч) — источник истины расходится. Выбрать одно: **build-time (иммутабельный профиль)** или **runtime-управление**.
8. **pnpm-специфика.** `nodeLinker: hoisted`, `autoInstallPeers: false` — peer-зависимости объявлять явно.
9. **Версии.** `saveExact`, `minimumReleaseAge` (pnpm может «придержать» релиз — версия из `inspect` ≠ установленная). Плюс `engines.node >= 24`.
10. **Безопасность.** Код bundle'а исполняется **in-process** вне sandbox воркспейса — только свои проверенные пакеты.
11. **Имя `desktop` зарезервировано** Electron — свой профиль так не назвать.

## 6. Рекомендуемая архитектура

```
┌──────────────────────────────────────────────┐
│  MY SYSTEM                                   │
│      │ API (HTTP/WS или stdio)               │
│      ▼                                       │
│  DSH runtime ── profile: my-harness          │
│      ├─ dsh-base             (ядро)          │
│      ├─ dsh-headless|sdk|web (транспорт)     │
│      ├─ my-bundle-core       (patch: inserts)│
│      └─ my-bundle-tools      (patch: inserts)│
└──────────────────────────────────────────────┘
```

- **Транспорт по связи:** свой процесс → `sdk` (stdio, `session/wait`); сеть → `web` (HTTP+WS); batch → `headless`.
- **Мои bundle'ы** = отдельные npm/git-пакеты, каждый со своим `cordis.patch.yml`, вставляющим мои плагины (tools/services).
- **`harness.yml`** — единый источник; генератор + `pnpm install` при деплое; `--patch` для per-env отличий.

## 7. CI-гейт

```sh
dsh --profile my-harness --dump-config > composed.yml
# проверить: нет skippedBundles, присутствуют ожидаемые id
```

`--dump-config` показывает итоговое дерево с `!!js` verbatim и слоями, которые его изменили; патчи, не попавшие ни в одну строку, репортятся с меткой слоя.

## 8. Следующие шаги (PoC)

1. `harness.yml` + `generate.mjs` + один демо-bundle (`@my-org/dsh-bundle-*`).
2. Собрать профиль в `$DSH_HOME/profiles/poc-harness`, прогнать `--dump-config`.
3. Запустить `dsh --profile poc-harness` (headless/sdk) и дернуть API снаружи.
