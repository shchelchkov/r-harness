# Создание нового plugin-модуля

## Общие сведения

Плагин — нативная динамическая библиотека (`cdylib`, файл `.so`) на Rust. r-function загружает её в свой процесс через `dlopen` и вызывает в отдельном потоке (`spawn_blocking`). От WASM-модуля плагин отличается следующим:

| | WASM-модуль | Плагин |
|---|---|---|
| Изоляция | песочница wasmtime | **нет**: код работает в процессе r-function |
| Лимиты памяти и времени | есть (`wasm.*`) | нет |
| Паника | временная ошибка, повтор группы | **аварийное завершение всего r-function** (`panic = "abort"`) |
| Ввод-вывод | stdin/stdout | буфер через ABI `r-plugin-api` |
| Исходящий HTTP | `http_request` | недоступен |
| Встроенная база r-db | нет | `insert_value`, `append_value`, `db_get_value`, `db_get_history`, `db_remove_value` |
| Состояние на дату (`get_value_at`) и полигоны | нет | есть |

Плагины выбирают для ресурсоёмких расчётов с состоянием, где важна производительность.

**Контракт ABI** (версия `ABI_VERSION = 3` в `r-plugin-api/src/lib.rs`). Библиотека экспортирует:
- `abi_version` — через макрос `export_abi_version!()`. Хост отказывается загружать плагин, собранный под более новую версию ABI, чем у хоста;
- `process(api, input_ptr, input_len) -> Buffer` — обработка с возвратом кадров следующему плагину цепочки;
- `process_discard(...)` — необязательный вариант без сборки выхода; хост вызывает его для последнего плагина цепочки;
- `free_buffer(buffer)` — освобождение буфера, который вернул плагин.

**Вход** — JSON-массив кадров одной группы (одинаковые `setting_code` и ключ). Кадры одного ключа обрабатывает один воркер по порядку.

**Принятая структура кода:**
- крейт плагина (`functions/plugin/f-<имя>`) — тонкая обёртка с экспортами ABI;
- логика, состояние и тесты — в `functions/plugin/plugin-common`;
- общий цикл `plugin_common::batch::run` разбирает вход, вызывает обработчик для каждого кадра, в конце пачки сохраняет состояние и отправляет накопленные результаты.

## Шаг 1. Описать состояние (если нужно)

Тип состояния реализует трейт `plugin_common::value::FnState`. `CODE` — код настройки, под которым состояние хранится у хоста. Файл `plugin-common/src/example_value.rs`:

```rust
use crate::value::FnState;
use sonic_rs::{JsonValueTrait, Value, json};

#[derive(Clone, Default, Debug, PartialEq)]
pub struct ExampleState {
    pub count: i64,
}

impl FnState for ExampleState {
    const CODE: &'static str = "example";

    fn from_value(values: &[Value]) -> Self {
        let count = values.first().and_then(|v| v.get("count")).and_then(|v| v.as_i64());
        count.map(|count| Self { count }).unwrap_or_default()
    }

    fn into_value(self) -> Value {
        json!({ "count": self.count })
    }
}
```

Подключите кэш состояния к пачке в `plugin-common/src/batch.rs`, в три места:
- поле структуры `Batch`: `pub examples: StateCache<'a, ExampleState>`;
- `Batch::new`: `examples: StateCache::new(api, Rc::clone(&at))`;
- `Batch::finish`: `self.examples.flush()`.

Если код состояния определяется во время работы, используйте `batch.state_cache::<S>(code)`.

Методы кэша:
- `load` — значение или состояние по умолчанию;
- `try_load` — `None`, если ключа нет;
- `peek` — чтение без копии;
- `save` — запись;
- `load_from` — если ключа нет ни в памяти, ни в снимке базы, хост запрашивает состояние у state-service на время текущего кадра.

Состояние читается у хоста один раз за пачку и записывается при `finish`.

## Шаг 2. Реализовать обработчик

Файл `plugin-common/src/example.rs`. Обработчик вызывается для каждого кадра; результаты складываются в `batch.outbox`:

```rust
use crate::batch::Batch;
use f_common::fun;
use r_error::plugin::error::PluginError;
use sonic_rs::Object;

pub const SETTING_CODE: &str = "example_out";

pub fn process(batch: &mut Batch, obj: &Object) -> Result<(), PluginError> {
    let key = fun::f_topic(obj);
    let mut state = batch.examples.load(key);
    state.count += 1;

    let mut value = Object::with_capacity(2);
    value.insert("key", key);
    value.insert("count", state.count);
    batch.examples.save(key, state);

    let mut envelope = Object::with_capacity(3);
    envelope.insert("setting_code", SETTING_CODE);
    envelope.insert("key", key);
    envelope.insert("value", vec![value.into_value()]);
    batch.outbox.push(envelope);
    Ok(())
}
```

Добавьте в `plugin-common/src/lib.rs` строки `pub mod example;` и `pub mod example_value;`.

**Правила реализации:**
- **Никаких паник.** Не используйте `unwrap`, `expect`, индексацию без проверки, операции с риском переполнения. Паника аварийно завершает весь экземпляр r-function вместе со всеми группами, которые он обрабатывает.
- **Некорректный кадр** — `Err(PluginError::Processing(..))` и предупреждение в журнал (`tracing::warn!`). Такой кадр не попадает в выход для следующего плагина, а обработка пачки продолжается.
- **Глобальное изменяемое состояние запрещено.** Плагин вызывается из нескольких потоков, поэтому всё состояние хранится у хоста через `StateCache`.
- **Передать данные следующему плагину цепочки** можно через `batch.annotate(key, value)`: поле добавится в кадр на выходе.
- **Журнал.** В плагине работает своя копия `tracing`; её инициализирует `batch::run`. Уровень задаётся той же переменной `RUST_LOG`, что и у хоста.

## Шаг 3. Создать крейт плагина

1. Создайте каталог `functions/plugin/f-example` с `Cargo.toml`:

```toml
[package]
name = "f-example"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
r-plugin-api = { path = "../../../r-plugin-api" }
f-common = { path = "../../fun/f-common" }
plugin-common = { path = "../plugin-common" }
sonic-rs = "0.5.8"
```

2. Опишите экспорты ABI в `src/lib.rs` по образцу `f-test_key`:

```rust
use plugin_common::batch::{self, Output};
use plugin_common::example;
use r_plugin_api::{Buffer, HostApi};

r_plugin_api::export_abi_version!();

#[unsafe(no_mangle)]
pub unsafe extern "C" fn process(api: *const HostApi, input_ptr: *const u8, input_len: usize) -> Buffer {
    unsafe { batch::run(api, input_ptr, input_len, Output::Keep, example::process) }
}

#[unsafe(no_mangle)]
pub unsafe extern "C" fn process_discard(api: *const HostApi, input_ptr: *const u8, input_len: usize) -> Buffer {
    unsafe { batch::run(api, input_ptr, input_len, Output::Discard, example::process) }
}

#[unsafe(no_mangle)]
pub unsafe extern "C" fn free_buffer(buffer: Buffer) {
    unsafe { plugin_common::buffer::free(buffer) }
}
```

3. Добавьте `"f-example"` в `members` файла `functions/plugin/Cargo.toml`. Release-профиль общий: LTO, `strip`, `panic = "abort"`.

Имя артефакта образуется из имени пакета с заменой дефиса на подчёркивание: `libf_example.so`.

## Шаг 4. Протестировать

1. **Модульные тесты** пишутся в `plugin-common` рядом с логикой. Хост заменяется заглушкой `crate::batch::tests::api()`; содержимое `batch.outbox.envelopes()` и состояние проверяются напрямую:

```bash
cd functions/plugin
cargo test -p plugin-common
```

2. **Сборка и сквозная проверка загрузки.** Тест создаёт файл плагина, выполняет `dlopen` и проверяет экспортированные символы:

```bash
cargo build --release -p f-example
R_PLUGIN_SMOKE_LIB=/путь/к/libf_example.{so,dylib} cargo test -p r-plugin -- --ignored
```

Если в `~/.cargo/config.toml` задан общий `target-dir`, свежий артефакт ищите там, а не в `functions/plugin/target`. Сборка на macOS даёт `.dylib`, которая годится только для локальных тестов. Для r-json нужна Linux-сборка из CI.

## Шаг 5. Опубликовать артефакт

1. Отправьте изменения в ветку из `.gitea/workflows/build-plugin.yaml` (сейчас `start_signal`). Сборку запускают изменения в `functions/plugin/**` и `functions/fun/**`, а также в коде хоста (`src/**`, `r-*/**`, `Cargo.toml`), потому что плагины зависят от `r-plugin-api` и `r-error`.
2. CI собирает весь workspace плагинов в образе `rust:1-bookworm`, совместимом с образом r-function. Флаги CPU (`x86-64-v2`, `avx2`, `pclmulqdq`) берутся из `docker/cargo/config.plugin.toml` и подходят всем серверам, где запускается r-function.
3. Затем CI коммитит `*.so` в r-json, в каталог `plugin/function_settings` (параметр `function_config.git_plugin_path`). Изменение `plugin-common` может обновить и остальные плагины.
4. Соблюдайте порядок:
   - **`.so` должна попасть в r-json раньше, чем настройка, которая на неё ссылается.** Ошибка загрузки плагина временная: группа будет обрабатываться повторно, пока файл не появится, и обработка партиции встанет;
   - если плагину нужна новая версия ABI, сначала выкладывается хост r-function с этой версией, затем плагины.

## Шаг 6. Подключить плагин в r-json

1. Определите экземпляр r-function и каталог его настроек, как для WASM-модуля: `kafka_consumer.topics` и `function_config` в YAML экземпляра.
2. В файле `json/<контур>/function_settings/<setting_code>.json` у элемента с `"code": "code"` укажите плагин. Остальные поля скопируйте из существующей настройки:

```json
{ "code": "code", "key": "code", "module": [""], "plugin": ["libf_example.so"], "isActive": true }
```

   - В `plugin` перечисляется цепочка плагинов по порядку.
   - Если одновременно заданы `module` и `plugin`, сначала выполняется WASM-цепочка, затем плагинная. Обе получают один и тот же вход.
   - Группировка задаётся атрибутом с `"isKey": true`, как у WASM-модуля.
3. Для кода из `outbox` (`example_out`) создайте маршрут `json/<контур>/stream_setting/example_out.json`: `isActive`, `channel` (топик Kafka) и при необходимости `settingCodeStream`. Если маршрута нет, публикация идёт в статические `kafka_producer.topics`.
4. Отправьте изменения в ветку r-json, которую читает экземпляр (`git_revision`).

## Шаг 7. Применить и проверить

1. **Применение.** r-function подхватывает изменения в пределах `git_fetch_interval_secs` или сразу по запросу `POST /api/functions/git/refresh`. Новая версия плагина сохраняется на диск под именем, равным хешу содержимого, и загружается без перезапуска.
2. **Проверка через HTTP API экземпляра:**
   - `GET /api/functions/setting/functions/<setting_code>` — настройка загружена;
   - `GET /api/functions/setting/plugin/cache/<setting_code>` — плагин загружен;
   - `GET /api/functions/values/<setting_code>` — текущее состояние.
3. **Результат** — в выходном топике (`rpk topic consume`, Redpanda Console); проверьте и отставание группы потребителей.
4. **Журнал:**
   - `plugin invoke failed` с признаком `transient`;
   - `group plugin poison; routing to DLQ`;
   - сообщения самого плагина (`plugin::…`);
   - `plugin ABI … is newer than host ABI`.

## Типичные проблемы

| Признак | Причина и действие |
|---|---|
| Экземпляр r-function перезапускается | паника в плагине; найдите место по журналу и уберите `unwrap` и подобные вызовы |
| Группа обрабатывается снова и снова | `.so` ещё нет в r-json, имя в `plugin` не совпадает с файлом, или библиотека не загружается |
| `plugin ABI … is newer than host ABI` | сначала выложите хост с новой версией `r-plugin-api` |
| Обработка идёт, а выхода нет | нет `outbox.push`, или маршрут `stream_setting` неактивен, или у него нет `channel` |
| Следующий плагин цепочки не получает кадры | обработчик вернул `Err` для этих кадров |
