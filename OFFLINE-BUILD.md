# kaeru — сборка без интернета

Этот форк отличается от апстрима ровно одним: в него закоммичены **все**
зависимости. Стек не менялся — тот же cozo + RocksDB, тот же rmcp, тот же
rustls, те же версии. Исходники крейтов не правились.

| | |
|---|---|
| База | `LamantinAI/kaeru`, ветка `main`, коммит `77817f9`, версия `0.5.0` |
| Зависимостей в `vendor/` | 461 крейт, 511 МБ |
| Сеть при сборке | не нужна — `[net] offline = true` в `.cargo/config.toml` |
| Проверено | полная `cargo build --release -p kaeru-mcp` с пустым `CARGO_HOME`: 2 мин 48 с, из сети не скачано ни байта |

Что добавлено поверх апстрима:

* `vendor/` — исходники всех зависимостей, включая единственную git-зависимость
  (`graph_builder`, пропатченный форк `lawless-m/graph`);
* `.cargo/config.toml` — подмена crates.io и git-источника на `vendor/`
  плюс жёсткий офлайн-режим;
* `.gitattributes` — `* -text`, отключает конвертацию переводов строк.
  **Без этого на Windows сборка гарантированно падает** (см. «Checksum mismatch»);
* этот файл.

`Cargo.lock` закоммичен и соответствует `vendor/` байт-в-байт.

---

## 1. Что должно быть на машине

Одного Rust-а **недостаточно**: cozo собирает RocksDB из C++-исходников, а zstd
и lz4 — из C. Это свойство апстримного kaeru, а не форка.

### Обязательно

| Компонент | Зачем | Как проверить |
|---|---|---|
| `rustc` / `cargo` **1.88+** | edition 2024 + самый высокий `rust-version` среди зависимостей — 1.88.0. Проверялось на 1.95 | `rustc -vV` |
| **MSVC Build Tools**, workload «Desktop development with C++» | `cl.exe` для RocksDB / zstd / lz4. `link.exe` оттуда же нужен и самому Rust на таргете `*-msvc` | `where cl` |
| **LLVM / libclang** | `zstd-sys` генерирует биндинги через `bindgen` | `where clang`, либо переменная `LIBCLANG_PATH` |

### Не нужно, вопреки ожиданиям

* **cmake** — `aws-lc-sys` для `x86_64-pc-windows-msvc` идёт по `cc`-ветке,
  потому что для этого таргета у него есть pregenerated-биндинги.
* **nasm** — если выставить `AWS_LC_SYS_PREBUILT_NASM=1`. Готовые объектные
  файлы лежат в `vendor/aws-lc-sys-0.41.0/builder/prebuilt-nasm/`.
* **perl**, **OpenSSL** — не участвуют: TLS через rustls, без системного SSL.

### Если LLVM поставить нельзя

Требование libclang снимается без изменения версий зависимостей: у `zstd-sys`
есть собственные pregenerated-биндинги (`src/bindings_zstd.rs`), надо лишь
выключить его дефолтную фичу `bindgen`. Два шага, оба офлайн:

1. В `vendor/zstd-sys-2.0.16+zstd.1.5.7/Cargo.toml`, в блоке `[features]`,
   убрать строку `"bindgen",` из списка `default`.
2. В `vendor/zstd-sys-2.0.16+zstd.1.5.7/.cargo-checksum.json` заменить
   содержимое на `{"files":{},"package":"<тот же package-хеш, что был>"}` —
   иначе cargo увидит правку файла и упадёт на проверке контрольной суммы.
   Пустой `files` отключает пофайловую сверку только для этого крейта.

Версии крейтов при этом не меняются, `bindgen` просто уходит из сборки.

---

## 2. Клонирование

```bat
git -c core.autocrlf=false clone https://github.com/<аккаунт>/<репозиторий>.git
```

`.gitattributes` и так запрещает конвертацию EOL, но флаг снимает вопрос, если
клонируют старым git-ом или с навязанным системным `core.autocrlf=true`.

Если git ругается `Filename too long`:

```bat
git config --global core.longpaths true
```

Ни LFS, ни submodules не используются — `vendor/` это обычные файлы в дереве.

---

## 3. Сборка

```bat
set AWS_LC_SYS_PREBUILT_NASM=1
cargo build --release -p kaeru-mcp
```

Результат — `target\release\kaeru-mcp.exe`.

Сборка обязана проходить при физически отключённой сети. Если чего-то не хватит,
cargo упадёт с внятным `no matching package`, а не зависнет на прокси.

Тесты (тоже офлайн):

```bat
cargo test --workspace
```

---

## 4. Если сборка упала

**`checksum mismatch for ...` / `failed to verify the checksum`**
Файл в `vendor/` изменился по дороге. Две обычные причины:

* git сконвертировал переводы строк — значит `.gitattributes` не применился.
  Лечится `git config core.autocrlf false`, затем `git checkout -- .`;
* файл вырезал сканер при передаче. Cargo назовёт конкретный файл — его надо
  донести отдельно.

**`no matching package named ... found`**
В `vendor/` нет крейта, который требует `Cargo.lock`. Cargo требует, чтобы в
vendor-дереве присутствовали **все** записи лока — даже те, что под текущий
таргет никогда не компилируются. Не правьте `Cargo.toml`; пересоберите `vendor/`
на машине с сетью: `cargo vendor --versioned-dirs vendor`.

**`Unable to find libclang`**
См. «Если LLVM поставить нельзя» в разделе 1.

**`could not find native static library` / ошибки линковки**
Не встали MSVC Build Tools либо активен не тот тулчейн. `rustc -vV` покажет
`host:` — под него всё и вендорилось.

---

## 5. Бинарные файлы внутри `vendor/`

`.exe`, `.dll` и `.so` в дереве отсутствуют полностью — три тестовые фикстуры
(`wit-component/libdl.so`, `libloading/tests/nagisa{32,64}.dll`) удалены, их
контрольные суммы обнулены. Ни один из этих файлов в сборке не участвует.

Остальное — объектные файлы и статические библиотеки:

| Что | Сколько | Нужно для `x86_64-pc-windows-msvc`? |
|---|---|---|
| `ring-0.17.14/pregenerated/*.o` | 17 | **да** — иначе понадобится NASM |
| `aws-lc-sys-0.41.0/builder/prebuilt-nasm/*.obj` | 26 | **да**, при `AWS_LC_SYS_PREBUILT_NASM=1` |
| `windows_x86_64_msvc-0.52.6/lib/windows.0.52.0.lib` | 1, 5.0 МБ | **да** — `ring` тянет `windows-sys 0.52` |
| `winapi-x86_64-pc-windows-gnu-0.4.0/lib/*.a` | 1416, 54 МБ | нет для msvc, **да** для `*-pc-windows-gnu` |
| `windows_x86_64_gnu-0.52.6/lib/libwindows.0.52.0.a` | 1, 12 МБ | нет для msvc, **да** для `*-pc-windows-gnu` |
| `windows_{i686,aarch64}_*` — `.lib` / `.a` | 5, ~31 МБ | нет — другие архитектуры |
| `.bin` / `.der` в `ring`, `rustls*`, `sct`, `webpki` | ~90 | нет — тестовые фикстуры |
| `*.wasm`, `wit-bindgen/*.a` | 5 | нет — тестовые фикстуры |

Уже вырезано: `winapi-i686-pc-windows-gnu` (1387 файлов `.a`, 51 МБ) — этот
крейт нужен только для 32-битного `-gnu` таргета. Каталог оставлен с манифестом,
потому что резолв cargo требует его наличия; удалены только библиотеки, а
`.cargo-checksum.json` обнулён.

Тем же приёмом можно вырезать любую строку из таблицы, помеченную «нет»:

```bash
find vendor/<крейт> -name '*.a' -delete
# в vendor/<крейт>/.cargo-checksum.json оставить {"files":{},"package":"<хеш>"}
```

После правки обязательно прогнать `cargo metadata --offline` — он падает сразу,
если vendor-дерево стало неконсистентным.
