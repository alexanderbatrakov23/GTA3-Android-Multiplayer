# GTA III Android — Multiplayer Mod

Модификация для Android-версии **Grand Theft Auto III** (Rockstar Games),
добавляющая **мультиплеер** с сетевой синхронизацией через **RakNet**.

Проект создан как инструмент для **реверс-инжиниринга игровой механики**
и как **база для мультиплеера** на Android — в частности, для порта
[re3](https://github.com/halpz/re3).

---

## ✨ Возможности

- **Сетевая синхронизация** через RakNet
- **Спавн игроков** и nametags
- **Чат** между игроками
- **Синхронизация позиции**, здоровья, брони
- **Обработка диалогов**, спавн-скрина
- **Кнопка MULTIPLAYER** в главном меню (ImGui)
- **Прямой вызов нативных функций** игры через Dobby

---

## 🏗️ Архитектура

```
JNI_OnLoad
   │
   ├── Находит базовый адрес libR1.so через dl_iterate_phdr
   ├── Запускает InitThread
   │
InitThread
   ├── Создаёт CGui (контекст ImGui)
   ├── Ставит хуки:
   │    ├── Render2dStuff         (рендер в игре)
   │    ├── eglSwapBuffers        (рендер в меню)
   │    ├── GetIconForOption      (детектор главного меню)
   │    ├── CText::Get            (подмена текста)
   │    └── ProcessButtonPresses  (прокси)
   │
Render2dStuffHook (в игре)
   └── рисует чат, nametags, spawn screen

eglSwapBuffersHook (в меню)
   └── рисует кнопку MULTIPLAYER

sendTouchEvent (JNI)
   └── передаёт координаты тача в ImGui
```

**Ключевые компоненты:**
- **Dobby** — inline-хукинг нативных функций
- **Dear ImGui** — UI поверх игры через OpenGL ES 3
- **RakNet** — сетевая библиотека
- **dl_iterate_phdr** — корректное получение базы `.so`
- **JNI** — проброс тач-событий из Java в native

---

## 📁 Структура проекта

```
multiplayer_android/
├── jni/
│   ├── Android.mk
│   ├── Application.mk
│   ├── main.cpp / main.h        # JNI_OnLoad, InitThread, touch handler
│   ├── game/
│   │   ├── game.h               # Оффсеты + прототипы
│   │   └── hooks.cpp            # Хуки + обёртки
│   ├── gui/
│   │   ├── gui.cpp / gui.h      # CGui — ImGui
│   │   ├── CChatWindow.cpp      # Чат
│   │   ├── CDialogWindow.cpp    # Диалоги
│   │   └── CSpawnScreen.cpp     # Спавн-скрин
│   ├── client/
│   │   ├── Multiplayer.cpp/h    # Инициализация MP
│   │   ├── CNetClient.cpp/h     # Сеть (RakNet)
│   │   ├── CLocalPlayer.cpp/h   # Локальный игрок
│   │   ├── CRemotePlayer.cpp/h  # Удалённые игроки
│   │   ├── CPlayerManager.cpp/h # Менеджер игроков
│   │   └── NetProtocol.h        # Протокол пакетов
│   ├── util/
│   │   ├── util.cpp / util.h    # find_library, gta_log
│   │   └── arm.cpp / arm.h      # UnFuck, PatchMemory
│   └── vendor/
│       ├── Dobby/               # Inline hooking
│       ├── imgui/               # Dear ImGui
│       └── raknet/              # RakNet
```

---

## 🔧 Сборка

### Требования
- **Android NDK r21e** (или новее)
- **arm64-v8a** устройство или эмулятор
- **GTA III** установлена на устройстве

### Команды

```bash
# Windows
cd C:\path\to\project\jni
C:\android-ndk-r21e\ndk-build.cmd clean
C:\android-ndk-r21e\ndk-build.cmd NDK_DEBUG=1

# Linux / macOS
cd /path/to/project/jni
$ANDROID_NDK_HOME/ndk-build clean
$ANDROID_NDK_HOME/ndk-build NDK_DEBUG=1
```

После сборки `.so` появится в:
```
libs/arm64-v8a/libmultiplayer.so
```

### Установка

1. Собери `libmultiplayer.so`.
2. Закинь `.so` в `lib/arm64-v8a/` внутри APK игры (через APKTool / MT Manager).
3. В `.smali` найди `GTA3.smali` → `<clinit>` и добавь:
   ```smali
   const-string v0, "multiplayer"
   invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
   ```
4. Пересобери APK, подпиши, установи.
5. Запусти игру — в главном меню появится кнопка **MULTIPLAYER**.

---

## 🎮 Использование

1. Открой главное меню игры.
2. Нажми **MULTIPLAYER** (ImGui-кнопка).
3. Начнётся подключение к серверу (`178.250.156.53:5555`).
4. После подключения — чат, nametags, синхронизация.

**Настройка сервера:** IP и порт задаются в `client/Multiplayer.cpp`:
```cpp
char CMultiplayer::szServerIP[128] = "178.250.156.53";
int CMultiplayer::iServerPort = 5555;
```

---

## 🚧 Что нужно доделать

### 🔴 Критично

- [ ] **Синхронизация игроков** — сейчас `CRemotePlayer` только сохраняет данные, но **не создаёт педов в мире**. Нужно:
  - `CPlayerPed::Create` для удалённых игроков
  - `CWorld::Add` для добавления в мир
  - `SetModelIndex` для скина
  - Обновление позиции каждый кадр
- [ ] **Спавн локального игрока** — `CLocalPlayer::Spawn()` пустой. Нужно:
  - Создать педа через `CPlayerPed::Create`
  - Телепортировать в точку спавна
  - Выдать оружие
- [ ] **Обработка `ID_PLAYER_SYNC_VMP`** — сейчас данные сохраняются, но **не применяются** к педу. Нужно:
  - Обновлять позицию, rotation, health, armour
  - Применять анимации
- [ ] **Синхронизация выстрелов** — нет пакета `ID_PLAYER_SHOT_VMP`. Нужно:
  - Отправлять при выстреле
  - Воспроизводить у удалённых игроков

### 🟡 Важно

- [ ] **Синхронизация транспорта** — `ID_VEHICLE_SYNC_VMP` **не реализован**. Нужно:
  - Создавать машины
  - Синхронизировать позицию, скорость, водителя
- [ ] **Обработка входа/выхода из машины** — нет пакетов
- [ ] **Синхронизация оружия** — `ID_GIVE_PLAYER_WEAPON` не обрабатывается
- [ ] **Синхронизация здоровья/брони** — `ID_SET_PLAYER_HEALTH`, `ID_SET_PLAYER_ARMOUR` не обрабатываются
- [ ] **Синхронизация погоды и времени** — `ID_SET_WORLD_WEATHER`, `ID_SET_WORLD_TIME`
- [ ] **Nametags** — `DrawNametags()` пустой. Нужно:
  - Проецировать 3D-позицию игрока в 2D
  - Рисовать ник через ImGui
  - Учитывать дистанцию
- [ ] **Чат** — `CChatWindow::Draw()` пустой. Нужно:
  - Рисовать историю сообщений
  - Поле ввода
  - Отправка через `ID_CHAT_MESSAGE_VMP`

### 🟢 Желательно

- [ ] **Античит** — проверка скорости, телепортации
- [ ] **Интерполяция** — плавное движение удалённых игроков между пакетами
- [ ] **Лаговый компенсатор** — учёт пинга
- [ ] **Клиентская валидация** — проверка данных от сервера
- [ ] **Логирование** — запись в файл для отладки
- [ ] **Реконнект** — автоматическое переподключение при потере связи
- [ ] **Меню настроек MP** — IP сервера, ник, порт
- [ ] **Список игроков** — таблица с никами и пингом

### 🐛 Известные баги

- [ ] **`ProcessButtonPresses`** — проксируется, но `screen`/`item` читаются с неправильных смещений. **Решение:** не читать `this`, использовать патч `aScreens`.
- [ ] **`DoSettingsBeforeStartingAGame`** — хук встаёт, но функция не вызывается при Start Game. **Решение:** найти правильную точку входа.
- [ ] **Кнопка MULTIPLAYER** — ImGui-кнопка не появляется. **Решение:** проверить `g_bInMainMenu` через логи `GetIconForOptionHook`.
- [ ] **Вылет при New Game** — если применён `PatchMainMenu`. **Решение:** временно отключить патч.

---

## 🔍 Как найти оффсеты

Если игра обновится и оффсеты сломаются:

1. Открой `libR1.so` в **IDA Pro** (ARM64).
2. `Shift+F12` — окно строк.
3. Найди строку `FEP_STA` (Start Game).
4. `X` — cross-references.
5. Рядом — функция, которая использует эту строку.
6. Адрес функции в `.text` — оффсет.
7. Обнови `game/game.h`.

**Символы:** в `libR1.so` **есть экспортируемые символы** `CMenuManager::*` — используй **Symbols window** (`Shift+F4`) для поиска.

---

## ⚠️ Дисклеймер

Проект создан **исключительно в образовательных целях** — для изучения
внутренней механики GTA III и разработки открытого порта **re3**.

- Не используй для получения преимущества в онлайн-играх.
- Не распространяй модифицированные APK с игрой.
- Все права на GTA III принадлежат **Rockstar Games / Take-Two Interactive**.

Автор не несёт ответственности за любое использование данного кода.

---

## 📜 Лицензия

MIT License — используй, форкай, изменяй свободно.

```
Copyright (c) 2025 ТВОЙ_НИК

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 🙏 Благодарности

- [Dobby](https://github.com/jmpews/Dobby) — inline hooking framework
- [Dear ImGui](https://github.com/ocornut/imgui) — UI framework
- [RakNet](https://github.com/facebookarchive/RakNet) — сетевая библиотека
- [re3](https://github.com/halpz/re3) — open-source реимплементация GTA III
- **Rockstar Games** — за оригинальную игру
