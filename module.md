# Создание нового WASM-модуля

## Общие сведения

WASM-модуль — программа на Rust, собранная под платформу `wasm32-wasip1`. r-function исполняет её в изолированной среде.

- **Вход.** JSON в stdin. При вызове из потока Kafka модуль получает массив сообщений одной группы: у всех одинаковые код настройки (`setting_code`) и ключ. При вызове по таймеру — объект `{setting_code, key, now_ns}`.
- **Выход.** JSON в stdout передаётся следующему модулю цепочки. **Выход последнего модуля хост отбрасывает**, поэтому результаты публикуются явно — функцией хоста `send_value`.
- **Ограничения.** Доступа к файловой системе и сети нет. Память, объём stdout и время вызова ограничены параметрами `wasm.memory_mib`, `wasm.stdout_mib` и `wasm.timeout_ms` в конфигурации экземпляра. По умолчанию это 64 MiB, 4 MiB и 5 с.

Функции хоста (обёртки в `run_common::host`):

| Функция | Назначение |
|---|---|
| `get_function_setting(code)` | настройка функции по коду |
| `get_function_value({setting_code, key})` | значения-параметры из r-json (только чтение) |
| `get_stream_setting(code)` | маршруты потока |
| `get_value` / `put_value` / `remove_value` | состояние модуля между вызовами: `{setting_code, key[, value]}` |
| `send_value({setting_code, key, value: [...]})` | публикация результата по маршрутам `setting_code` |
| `http_request({method, url, body, content_type, auth})` | исходящий HTTP-запрос через хост; авторизация — по имени профиля |

## Шаг 1. Создать крейт модуля

1. Создайте каталог `functions/runtime/f-<имя>`, например `f-example`.
2. Опишите `Cargo.toml`. Имя файла артефакта задаётся полем `[[bin]] name`, а не именем пакета:

```toml
[package]
name = "f-example"
version = "0.1.0"
edition = "2024"

[dependencies]
sonic-rs = "0.5"
run-common = { path = "../run-common" }
f-common = { path = "../../fun/f-common" }

[[bin]]
name = "f-example"
path = "src/main.rs"
```

3. Добавьте `"f-example"` в `members` файла `functions/runtime/Cargo.toml`. Release-профиль (LTO, `strip`, `panic = "abort"`) общий для всего workspace.

## Шаг 2. Реализовать логику

Принятый в проекте шаблон:
- `main.rs` только читает вход, обходит сообщения и пишет выход;
- прикладная логика лежит в `functions/runtime/run-common/src/<имя>.rs` и подключается через `pub mod <имя>;` в `run-common/src/lib.rs`.

`src/main.rs`:

```rust
use f_common::fun;
use run_common::example;
use sonic_rs::{JsonContainerTrait, Value};
use std::io::{self, Read, Write};

fn main() {
    let mut buf = Vec::new();
    io::stdin().read_to_end(&mut buf).expect("read stdin");
    let mut v: Value = fun::from_slice(&buf);

    if let Some(values) = v.as_array() {
        for value in values.iter() {
            if let Some(obj) = value.as_object() {
                example::process(obj);
            }
        }
    } else if let Some(obj) = v.as_object() {
        example::process(obj);
    }

    let out = fun::to_vec(&mut v);
    if let Err(e) = io::stdout().write_all(&out) {
        eprintln!("write stdout failed ({} bytes): {e}", out.len());
        std::process::exit(1);
    }
}
```

`run-common/src/example.rs` — публикация результата через `send_value`:

```rust
use crate::host;
use f_common::fun;
use sonic_rs::Object;

const SETTING_CODE: &str = "example_out";

pub fn process(obj: &Object) {
    let mut value = Object::with_capacity(obj.len() + 1);
    for (k, val) in obj.iter() {
        if k != "setting_code" {
            value.insert(k, val.clone());
        }
    }
    value.insert("setting_code", SETTING_CODE);
    // расчёт и дополнение value

    let mut envelope = Object::with_capacity(3);
    envelope.insert("setting_code", SETTING_CODE);
    envelope.insert("key", fun::f_topic(obj));
    envelope.insert("value", vec![value.into_value()]);
    if let Err(rc) = host::send_value(&fun::to_vec(&mut envelope)) {
        eprintln!("example: send_value rc={rc}");
    }
}
```

Если модулю нужно состояние между вызовами, используйте `run_common::value::load` и `save` или напрямую `host::get_value` и `put_value`.

**Правила реализации:**
- **Не допускайте паники на данных.** Паника и выход за лимиты считаются временной ошибкой: группа не подтверждается и обрабатывается снова, и обработка партиции встаёт. Некорректное сообщение пропускайте с диагностикой.
- **Неустранимая ошибка входа** — завершение с ненулевым кодом (`std::process::exit(1)`). Такая ошибка не временная, и сообщения группы уходят в DLQ.
- **stderr только для диагностики.** Любой непустой stderr пишется в журнал хоста на уровне `warn` при каждом вызове, поэтому не выводите в stderr ничего на штатном пути.
- **Сеть только через `http_request`.** Ключи доступа в модуль не передаются: указывается имя профиля `auth` из конфигурации r-function.

## Шаг 3. Собрать модуль локально

```bash
rustup target add wasm32-wasip1
cd functions/runtime
cargo build --release --target wasm32-wasip1 -p f-example
```

Артефакт `f-example.wasm` появится в `target/wasm32-wasip1/release/` или в каталоге из `CARGO_TARGET_DIR`. Модуль, который импортирует функции хоста, вне r-function не запускается. Проверять его нужно в окружении r-function (`docker/local`) или на тестовом контуре.

## Шаг 4. Опубликовать артефакт

1. Закоммитьте изменения и отправьте их в ветку, указанную в `.gitea/workflows/build-wasm.yaml` (сейчас `start_signal`). Сборку запускают изменения в `functions/runtime/**` или `functions/fun/**`; вручную её можно запустить через `workflow_dispatch`.
2. Workflow соберёт все модули и закоммитит `*.wasm` в r-json, в каталог `wasm/function_settings`.
3. Порядок важен: **артефакт должен оказаться в r-json раньше, чем настройка, которая на него ссылается.** Иначе ошибка загрузки модуля будет временной, и сообщения будут обрабатываться повторно, пока артефакт не появится.

## Шаг 5. Подключить модуль в r-json

1. **Определите контур.** Нужен экземпляр r-function, который читает входной топик (`kafka_consumer.topics` в его YAML). Его каталоги настроек заданы в `function_config` (например, `json/processor/function_settings`).
2. **Создайте настройку функции** `json/<контур>/function_settings/<setting_code>.json`. Удобнее всего скопировать существующий файл контура и изменить ключевые поля; остальные оставить как в исходном файле:

```json
[
  { "code": "code",  "key": "code",  "module": ["f-example.wasm"], "isActive": true },
  { "code": "topic", "key": "topic", "isKey": true, "isActive": true }
]
```

- `module` — цепочка модулей по порядку. Имя должно совпадать с именем файла, включая `.wasm`.
- Атрибут с `isKey: true` задаёт поле сообщения, по которому сообщения группируются.
3. **Проверьте `setting_code` входных сообщений.** Он должен совпадать с именем файла настройки. Код задаётся конфигурацией источника или полем `settingCodeStream` маршрута предыдущей ступени.
4. **Задайте маршрут публикации** для кода из `send_value`: файл `json/<контур>/stream_setting/example_out.json`.

```json
[
  {
    "isActive": true,
    "key": "example_out",
    "code": "example.example_out",
    "settingCode": "example_out",
    "settingCodeStream": "example_out",
    "channel": "example_channel"
  }
]
```

- `channel` — топик Kafka.
- `settingCodeStream` — код настройки, который получит сообщение на следующей ступени.
- Если маршрута нет, публикация идёт в статические топики `kafka_producer.topics`.
5. **Параметры модуля** при необходимости разместите в `function_values`; модуль читает их через `get_function_value`.
6. Отправьте изменения в ветку r-json, которую читает экземпляр (`function_config.git_revision`, например `origin/main`).

## Шаг 6. Запуск по расписанию (при необходимости)

Если модуль должен работать по таймеру, а не по сообщениям Kafka, добавьте в YAML экземпляра раздел `watchdog`:

```yaml
watchdog:
  - module_name: f-example.wasm
    setting_code: example
    key: example0
    interval_secs: 30
```

YAML экземпляра на лету не перечитывается. Изменение применяется при выкладке через `build-deploy`: workflow запускается при изменении `*.yaml`.

## Шаг 7. Применить и проверить

1. **Применение.** r-function подхватывает изменения r-json в пределах `git_fetch_interval_secs` (обычно 30 с). Сразу применить их можно запросом `POST /api/functions/git/refresh`.
2. **Проверка через HTTP API экземпляра:**
    - `GET /api/functions/setting/functions/<setting_code>` — настройка загружена;
    - `GET /api/functions/setting/module/cache/<setting_code>` — модуль скомпилирован и есть в кэше;
    - `GET /api/functions/values/<setting_code>` — текущее состояние.
3. **Результат** — сообщения в выходном топике (Redpanda Console, `rpk topic consume`) и отставание группы потребителей (`rpk group describe`).
4. **Журнал экземпляра:**
    - `wasm invoke failed` с признаком `transient`;
    - `group poison; routing to DLQ`;
    - `wasm guest stderr`;
    - `send_value … (dropped)`.

## Типичные проблемы

| Признак | Причина и действие |
|---|---|
| Модуля нет в кэше, ошибка загрузки | имя в `module` не совпадает с файлом в `wasm/function_settings`, или артефакт ещё не опубликован |
| Обработка идёт, а выхода нет | выход последнего модуля отбрасывается — нужен `send_value`; проверьте `isActive` и `channel` маршрута (в журнале `no active route`) |
| Одна и та же группа обрабатывается снова и снова | паника, нехватка памяти или превышение времени; исправьте модуль или увеличьте `wasm.*` с учётом `pipeline.chunk_max` |
| Сообщения попадают в DLQ | модуль завершился с ненулевым кодом или вернул некорректный JSON |

