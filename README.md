# ClickHelper Alpha 0.1

Новый самостоятельный нативный мод для Minecraft Bedrock под LeviLauncher.
**Не является продолжением и не использует код ClickRepeater 2.13.**

Цель этой версии — подготовить проверенную техническую основу: архитектуру,
движки, модель настроек, структуру меню, сборку и упаковку. Внешний дизайн
меню будет подключён отдельно.

## Статус (важно читать первым)

| Что | Статус |
| --- | --- |
| Логика Click Repeater (5 профилей, интервал 20–1000 мс) | Реализована и покрыта host-тестами |
| Логика Click Combination (5 профилей, 2–5 действий, задержки 0–1000 мс, one-shot) | Реализована и покрыта host-тестами |
| Наблюдение клавиатуры/мыши через `pl::input` | Реализовано против реальных заголовков SDK |
| Меню: категории-вкладки, профили, переключатели, слайдеры, выбор ключа | Модель и регистрация реализованы против реальных заголовков SDK |
|Manifest / `.levipack` / GitHub Actions | Подготовлены |
| **Фактическая инъекция нажатий в игру** | **НЕ реализована. См. блокер ниже.** |
| Android `.so` собрана в этой сессии | **Нет** (локально нет Android NDK) |
| Работоспособность в игре | **Не подтверждена, не заявляется** |

## Блокер: инъекция нажатий

Публичный SDK LeviLauncher (`LiteLDev/preloader-android`, зафиксированная
ревизия `cc08b40a31471d3e2dedb267f5f17b28da2b0604`) в `pl/Input.hpp`
предоставляет только **наблюдение**:

* `registerTouchCallback(TouchCallback)`
* `registerKeyCallback(KeyCallback)`
* `registerTextInputCallback(TextInputCallback)`
* `registerMouseCallback(MouseCallback)`
* `showKeyboard()` / `hideKeyboard()` / `openDocument(...)`

`KeyEvent` содержит `keyCode`, `unicodeChar`, `isKeyDown`; `MouseEvent` —
`button`, `isDown`. Функции отправки нажатия в игру в публичном API
**отсутствуют**. Обратные вызовы возвращают `bool` только для *поглощения*
события, а не для его генерации.

Функции `pl::memory::hook()` / `resolveSignature()` существуют, но это низкоуровневые
detour-хуки: они требуют адреса или сигнатуры конкретной функции конкретной
сборки игры. Такая сигнатура в публичном SDK не определена и в этом проекте
**не выдумана**. Пока не будет проверенного бэкенда инъекции (или официального
API), движки работают с `UnavailableInputSink`: они корректно определяют
условия, интервалы и последовательности, но нажатия **не доходят до игры**.
В логе при включении печатается явное предупреждение.

Дополнительные неподтверждённые места:

1. Нумерация `MouseEvent::button` в публичном SDK не описана — значения 0–4 в
   меню являются только вариантами выбора и требуют проверки на устройстве.
2. Совместимость с конкретной версией игры не проверена; `minecraft_versions`
   в манифесте намеренно пустой (все версии), пока версия не подтверждена.
3. Схема настроек v2 (`setConfigSchemaJson`) доступна только в зафиксированной
   ревизии SDK; более старый preloader может не иметь этого символа.

Детали проверки API: [`docs/API_AUDIT.md`](docs/API_AUDIT.md).

## Структура

```
clickhelper-alpha/
├── CMakeLists.txt                 # общая сборка, Android target, упаковка
├── manifest.json.in               # manifest LeviLauncher (type: preload-native)
├── config/
│   ├── config.json                # значения по умолчанию (сгенерированы)
│   └── config.schema.json         # информационная схема файла конфигурации
├── include/clickhelper/
│   ├── core/                      # типы, KeyId, константы, IInputSink
│   ├── input/                     # InputState (удержание и фронты нажатий)
│   ├── repeater/                  # RepeaterEngine
│   ├── combination/               # CombinationEngine
│   ├── settings/                  # SettingsStore (JSON, климпы, apply_value)
│   ├── ui/                        # MenuModel (SDK-независимое описание меню)
│   └── platform/                  # UnavailableInputSink
├── src/                           # реализации перечисленных модулей
│   └── platform/                  # LeviAdapter — единственный слой, который
│                                  # включает заголовки LeviLauncher SDK
├── src/main.cpp                   # PL_REGISTER_MOD
├── tests/test_core.cpp            # host-тесты движков и настроек
├── tools/dump_config.cpp          # генератор config.json / config.schema.json
└── .github/workflows/build.yml    # host-тесты + Android arm64-v8a + .levipack
```

Модули CMake: `ch_core`, `ch_input`, `ch_repeater`, `ch_combination`,
`ch_settings`, `ch_ui`, `ch_platform`, `clickhelper_alpha`. Движки не включают
заголовков LeviLauncher и тестируются на хосте.

## Поведение

### Click Repeater

* 5 независимых профилей.
* `trigger1` — необязательный (`none` считается выполненным).
* `trigger2` — обязательный для автоматической активации.
* Активно, когда `masterEnabled`, профиль включён, `trigger2` удерживается и
  `trigger1` удерживается (если задан).
* При активации целевая клавиша/кнопка нажимается сразу, затем каждые
  `intervalMs` (20–1000 мс). Отпускание любого обязательного триггера останавливает
  повтор.

### Click Combination

* 5 независимых профилей.
* Оба триггера могут быть `none`. Основной триггер — `trigger2`, иначе `trigger1`.
* Если основной триггер не задан, последовательность можно запустить вручную
  кнопкой `Run once (manual)` в меню.
* 2–5 действий, у каждого своя клавиша/кнопка и задержка **после** него
  (0–1000 мс) перед следующим действием.
* Запуск только по фронту нажатия; удержание триггера не вызывает повтор.

## Сборка

### Host-тесты (без Android SDK)

```bash
cmake -S . -B build-host -G Ninja -DCLICKHELPER_BUILD_TESTS=ON
cmake --build build-host
ctest --test-dir build-host --output-on-failure
```

В этой сессии выполнено: **1/1 тест пройден** (8 групп проверок).

### Android ARM64

Нужны Android SDK/NDK и CMake 3.22+.

```bash
cmake -S . -B build-android -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24 \
  -DCLICKHELPER_BUILD_TESTS=OFF
cmake --build build-android --target levi_package
```

Результат: `build-android/clickhelper_alpha-0.1.0-arm64-v8a.levipack`.

Внутри архива:

```
clickhelper_alpha/
├── manifest.json
├── libclickhelper_alpha.so
└── config/config.json, config/config.schema.json
```

CMake автоматически получает `nlohmann/json` и зафиксированную ревизию
`preloader-android` через `FetchContent`. Для воспроизводимости ревизию можно
переопределить через `-DLEVI_SDK_REVISION=...`, но тогда публичные API должны
быть совместимы с кодом адаптера.

### GitHub Actions

`.github/workflows/build.yml` запускает host-тесты и отдельную сборку
`arm64-v8a`, проверяет состав `.levipack` и загружает его артефактом.
Сам workflow в этой сессии не выполнялся (нет remote и runners).

## Что делать дальше

1. Найти или получить официальный/проверенный способ отправки key/mouse events
   в игру для конкретной версии Minecraft Bedrock; заменить
   `UnavailableInputSink` на проверенный бэкенд и добавить интеграционные тесты
   на устройстве.
2. Зафиксировать поддерживаемые версии игры и заполнить `minecraft_versions`.
3. Проверить нумерацию кнопок мыши на устройстве.
4. Подключить отдельно разработанный дизайн меню, не меняя движки: меню
   описано данными в `ch_ui`, адаптер переводит их в `pl::modmenu`.
