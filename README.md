<div align="center">

# Home Console

**Распределённая микросервисная платформа управления умным домом**
с архитектурными механизмами безопасного расширения функциональности

[![Python](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![Tauri](https://img.shields.io/badge/Tauri-2-FFC131?logo=tauri&logoColor=black)](https://tauri.app/)
[![License](https://img.shields.io/badge/License-Proprietary-lightgrey)](#)

</div>

---

## 📖 О проекте

**Home Console** — это open-source платформа управления умным домом с двухуровневой архитектурой: **встроенные системные модули** (RuntimeModule) и **пользовательские расширения** (Plugin), изолированные через специализированный набор прокси-классов.

Платформа решает фундаментальный разрыв на рынке систем IoT:

| Категория | Примеры | Проблема |
|---|---|---|
| **Открытые** | Home Assistant, openHAB | Высокая расширяемость — нулевая изоляция расширений |
| **Закрытые** | Apple Home, Яндекс УД | Безопасность достигается запретом независимых разработчиков |
| **Home Console** | этот проект | Открытость + архитектурная изоляция через прокси-сэт |

Каждое расширение получает доступ к платформе только через **PluginRuntimeFacade** — frozen-dataclass с проксированными интерфейсами (NamespacedStorageProxy, ServiceRegistryProxy, EventBusProxy и ещё четыре). Управляющие компоненты ядра — plugin_manager, module_manager, orchestration_service — в фасад не передаются. Это переводит ограничение доступа из плоскости договорённостей разработчиков в плоскость структуры передаваемого объекта.

---

## 🏗️ Архитектура организации

```
                    ┌─────────────────────────────────────────────┐
                    │              Клиентский слой                │
                    │   ┌───────────────────────────────────────┐ │
                    │   │       platform-home-console           │ │
                    │   │   web │ desktop (Tauri) │ mobile      │ │
                    │   └───────────────────────────────────────┘ │
                    │                                             │
                    │   ┌───────────────────────────────────────┐ │
                    │   │           home-console-cli            │ │
                    │   │              (hc) CLI                 │ │
                    │   └───────────────────────────────────────┘ │
                    └─────────────────┬───────────────────────────┘
                                      │ HTTP API (FastAPI)
                                      ▼
                    ┌─────────────────────────────────────────────┐
                    │           core-runtime-service              │
                    │                                             │
                    │   Ядро платформы: 17 системных модулей,     │
                    │   PluginSandbox, 7 прокси-классов,          │
                    │   TrustEngine, ExecutionController          │
                    └─────────┬─────────────────────┬─────────────┘
                              │                     │
                  ┌───────────┘                     └────────────┐
                  │ установка                                     │ разработка
                  ▼                                               ▼
        ┌──────────────────┐                            ┌──────────────────┐
        │  marketplace-api │                            │    python-sdk    │
        │                  │                            │                  │
        │ Публичный реестр │                            │ SDK разработчика │
        │ плагинов с semver│                            │     плагинов     │
        └──────────────────┘                            └──────────────────┘
```

---

## 📦 Репозитории

### 🔧 [`core-runtime-service`](https://github.com/home-console/core-runtime-service)

**Ядро платформы.** Распределённый runtime с двухуровневой системой расширений и архитектурной изоляцией плагинов.

- **Стек:** Python 3.13 · FastAPI · asyncio · SQLite/PostgreSQL · Docker (single-stage)
- **Объём:** 17 системных модулей · 7 прокси-классов изоляции · 3 backend исполнения (in_process, process, container)
- **Тестирование:** 1 573 автоматизированных теста · 134 файла · ~33 600 строк · бенчмарки производительности (storage p99 < 1 мс, API p99 < 200 мс)
- **Ключевые подсистемы:** CoreRuntime · ServiceRegistry · EventBus (at-least-once + dedup) · CapabilityRegistry · OperationRegistry · PluginSandbox · TrustEngine + MFA · SecureStorageWrapper (Merkle) · AuditBinder · PolicyEngine · DependencyResolver · ExecutionController

### 🛒 [`marketplace-api`](https://github.com/home-console/marketplace-api)

**Публичный реестр плагинов** с поддержкой semver-версионирования, подписей и rate-limiting.

- **Стек:** Python 3.13 · FastAPI 0.115 · SQLAlchemy 2.0 · Alembic · PostgreSQL · cryptography · Docker
- **Возможности:** публикация плагинов через Bearer-аутентификацию (роли publisher/admin), хранение архивов с подписями, контроль скачиваний по IP, Swagger UI
- **Интеграция:** реализует контракт `RegistryClient` из `core-runtime-service` (`GET /registry/index.json`)
- **Endpoints:** `POST /plugins`, `POST /plugins/{name}/releases`, `GET /packages/{filename}`, `POST /admin/keys`

### 🖥️ [`platform-home-console`](https://github.com/home-console/platform-home-console)

**Monorepo с клиентскими приложениями.** Web, desktop и mobile интерфейсы поверх единого API-слоя.

- **Стек:** TypeScript 5.9 · React 19 · pnpm workspaces · Vite · TailwindCSS 3 · Radix UI · shadcn/ui · Lucide
- **Приложения:**
  - `apps/web` — React + Vite + MUI + Radix UI + TailwindCSS
  - `apps/desktop` — Tauri 2 (Rust shell + React UI)
  - `apps/mobile` — Expo / React Native + expo-router
- **Shared-пакеты:** `@platform/api-client` · `@platform/auth-core` · `@platform/auth-react` · `@platform/auth-storage` · `@platform/control-plane-client` · `@platform/smart-domain-client` · `@platform/types`
- **Развёртывание:** Docker + nginx для production; pnpm для разработки

### ⌨️ [`home-console-cli`](https://github.com/home-console/home-console-cli)

**CLI-утилита `hc`** для управления платформой через HTTP API CoreRuntime.

- **Стек:** Python 3.11+ · Typer · Rich · httpx · anyio · prompt_toolkit · tomlkit
- **Команды:**
  - `hc connect <host>` — подключение и сохранение конфигурации
  - `hc status` — состояние платформы
  - `hc plugin list|start|stop|info` — управление плагинами
  - `hc module list|status` — управление модулями
  - `hc install/remove <name>` — установка/удаление расширений
  - `hc deploy platform [--mode image]` — развёртывание платформы одной командой
  - `hc logs [--follow]` — потоковое чтение логов
  - `hc shell` — интерактивный shell
- **Конфигурация:** `~/.config/hc/config.toml`, JWT-токен через `HC_TOKEN` или флаг

### 🛠️ [`python-sdk`](https://github.com/home-console/python-sdk) — `home-console-sdk`

**SDK для разработки плагинов** на Python с dependency injection моделей и автоматическим управлением роутами.

- **Стек:** Python 3.11+ · FastAPI совместимость · публикуется как `home-console-sdk` на PyPI
- **Возможности:**
  - Базовый класс `InternalPluginBase` с lifecycle-хуками (`on_load`, `on_start`, `on_stop`, `on_unload`)
  - Dependency injection моделей: `self.models.get('Device')` вместо прямых импортов из ядра
  - Автоматический mount/unmount FastAPI-роутеров
  - Группировка endpoints через `APIRouter`
  - Маркер `infrastructure = True` для плагинов на `/api` без префикса
  - Конфигурация через `plugin.json` (namespace, capabilities, allowed_services, provides_services)
- **Документация:** [DEV_SETUP.md](https://github.com/home-console/python-sdk/blob/main/DEV_SETUP.md) · [OAUTH_INTEGRATION.md](https://github.com/home-console/python-sdk/blob/main/OAUTH_INTEGRATION.md) · [examples.py](https://github.com/home-console/python-sdk/blob/main/examples.py)

---

## 🚀 Быстрый старт

### Запуск платформы локально

```bash
# 1. Поднять ядро (core-runtime-service)
git clone https://github.com/home-console/core-runtime-service.git
cd core-runtime-service
docker build -t home-console/core .
docker run -p 8080:8080 home-console/core

# 2. Поднять marketplace (опционально)
git clone https://github.com/home-console/marketplace-api.git
cd marketplace-api
docker compose up -d

# 3. Установить CLI и подключиться
pip install -e home-console-cli/
hc connect localhost --port 8080
hc status
```

### Подключение клиента

```bash
git clone https://github.com/home-console/platform-home-console.git
cd platform-home-console
pnpm install
pnpm web        # http://localhost:5173
# или
pnpm desktop    # Tauri-приложение
pnpm mobile     # Expo dev-сервер
```

### Разработка плагина

```bash
pip install home-console-sdk
```

```python
from home_console_sdk import InternalPluginBase
from fastapi import APIRouter

class TemperatureSensor(InternalPluginBase):
    id = "temperature_sensor"
    name = "Temperature Sensor"
    version = "1.0.0"

    async def on_load(self):
        router = APIRouter(prefix="/api", tags=["temperature"])
        router.add_api_route("/read", self.read_temperature, methods=["GET"])
        self.router = router

    async def read_temperature(self):
        return {"celsius": 22.5}
```

С манифестом `plugin.json`:

```json
{
  "id": "temperature_sensor",
  "name": "Temperature Sensor",
  "version": "1.0.0",
  "namespace": "temperature",
  "allowed_services": ["logger.*", "storage.get", "storage.set"],
  "provides_events": ["temperature.changed"],
  "provides_services": ["temperature.read"]
}
```

---

## 🔒 Безопасность плагинов: 7 уровней изоляции

Каждый плагин, установленный через Marketplace, проходит через `PluginSandbox.create_isolation_context()`. Контекст представляет собой `PluginRuntimeFacade` — frozen-dataclass со следующими полями:

| Поле фасада | Прокси-класс | Что проверяет |
|---|---|---|
| `storage` | `NamespacedStorageProxy` | namespace плагина, формат ключей |
| `service_registry` | `ServiceRegistryProxy` | allowlist из манифеста + защита от service-squatting |
| `event_bus` | `EventBusProxy` | namespace публикуемых и подписываемых событий |
| `operations` | `OperationRegistryProxy` | provides_operations из манифеста |
| `http` | `HttpRegistryProxy` | префикс HTTP-маршрутов |
| `capabilities` | `CapabilityRegistryProxy` | provides_capabilities из манифеста |
| `agent_manager` | `AgentManagerProxy` | взаимодействие только со своими агентами |
| `vault` | `None` (принудительно) | секреты ядра плагину недоступны |

Сырые ссылки на `plugin_manager`, `module_manager`, `orchestration_service` в фасад **не передаются вообще**, что архитектурно исключает возможность плагина управлять другими плагинами или ядром.

---

## 📚 Документация и публикации

- **Магистерская диссертация:** Коновалов М. А. «Проектирование и реализация распределённой микросервисной платформы управления умным домом с интерфейсами для безопасного расширения функциональности компонентами». — НИЯУ МИФИ, кафедра №42 «Криптология и кибербезопасность», 2026.
- **Архитектурная политика ядра:** [core-runtime-service/docs/CORE_KERNEL_POLICY_RU.md](https://github.com/home-console/core-runtime-service/blob/main/docs/CORE_KERNEL_POLICY_RU.md)
- **Правила разработки:** [core-runtime-service/docs/CORE_DEVELOPMENT_RULES_RU.md](https://github.com/home-console/core-runtime-service/blob/main/docs/CORE_DEVELOPMENT_RULES_RU.md)
- **ADR (Architecture Decision Records):** в каждом репозитории под `docs/adr/`
- **OpenAPI 3.0:** `openapi.yaml` в `core-runtime-service` и `marketplace-api` (Swagger по `/docs`)

---

## 🤝 Контакты и участие

Это исследовательский проект, разработанный в рамках магистерской квалификационной работы. Открытие репозиториев для публичного участия и приём pull request планируется после защиты ВКР.

Вопросы по архитектуре, изоляции расширений, контрактам SDK — через issues соответствующего репозитория.

---

<div align="center">

Made with ☕ and concern for sandbox safety
in [НИЯУ МИФИ](https://mephi.ru/) · [Кафедра №42 «Криптология и кибербезопасность»](https://eng.mephi.ru/colleges/icis/structure/criptology)

</div>
