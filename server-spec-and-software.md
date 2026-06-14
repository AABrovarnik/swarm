# ТЗ на сервер для отказоустойчивого роя 3–5 LLM-агентов с OpenClaw

> **Документ-компаньон к** `redteam-prompt-v2-with-openclaw.md`.
> **Назначение**: определить минимальные и рекомендуемые технические характеристики выделенного сервера, на котором будет развёрнут OpenClaw + Venya Core + 3 LLM-агента, и состав программного обеспечения.
> **Дата**: 2026-06-12.

---

## 0. Резюме: что мы строим

| Параметр | Значение |
|---|---|
| **Архитектура** | OpenClaw Gateway + Venya Core (Python) + 3 LLM-агента + Watchman |
| **Целевая нагрузка** | 3 агента × 50–100K токенов/день = 150–300K токенов/день |
| **LLM-режим** | Гибрид: локальные (Ollama) + 3 облачных провайдера |
| **Каналы** | Telegram (бот) + email (SMTP) |
| **Хранилище** | SQLite с WAL, ssh-pull реплика на 2-й VPS |
| **Целевой аптайм** | 99,5% (4 часа простоя/мес — допустимо) |
| **Бюджет на сервер** | $15–40/мес (VPS) + $50–120/мес (LLM) = $65–160/мес |
| **Профиль потерь** | 30% агентов в неделю на 2–3 дня — система должна выживать |

---

## 1. Требования к железу

### 1.1 Расчёт нагрузки

#### 1.1.1 CPU

| Компонент | Использование CPU | Пик |
|---|---|---|
| OpenClaw Gateway | 1–5% (Node.js, event-loop) | 30% при старте |
| Ollama runtime (3 модели: gemma4, llama3.1:8b, glm-5.1:cloud) | 0% в простое, **100–400%** при inference на GPU | burst |
| Venya Core + Watchman | < 1% | < 5% |
| Python-агенты (оркестрация) | 2–5% | 20% при failover |
| Telegram-бот, email-клиент | < 1% | < 1% |
| SQLite (WAL, fsync) | < 1% | 5% при коммитах |
| SSH-pull реплика (cron) | 0% в idle, **100% на 1–2 сек** каждые 5 мин | — |
| **Итого среднесуточное** | **5–15%** | — |
| **Пиковое (при inference)** | **500–600%** (с GPU-offload) | — |

#### 1.1.2 RAM

| Компонент | Потребление |
|---|---|
| OpenClaw Gateway (Node.js) | 200–400 МБ |
| Ollama runtime (3 модели) | **12–18 ГБ** (зависит от моделей) |
| Модель в RAM (gemma4, 8B) | ~5 ГБ (Q4) / ~10 ГБ (Q8) |
| Модель в RAM (llama3.1:8b) | ~5 ГБ (Q4) |
| Venya Core + Watchman | 50–100 МБ |
| Python-агенты (оркестрация) | 200–500 МБ |
| SQLite cache | 100–500 МБ |
| Telegram/email буферы | 50–100 МБ |
| **Итого** | **18–24 ГБ** для комфортной работы |

#### 1.1.3 Диск

| Компонент | Потребление |
|---|---|
| ОС + базовое ПО | 5–8 ГБ |
| OpenClaw 2026.6.1 + Node.js | 1 ГБ |
| Ollama binary | 200 МБ |
| 3 модели (gemma4, llama3.1:8b, glm-5.1:cloud) | **30–60 ГБ** |
| SQLite БД (1 год работы) | 2–10 ГБ (с ротацией) |
| Локальный git-репозиторий артефактов | 10–50 ГБ |
| Логи (с ротацией) | 5–20 ГБ |
| Swap / tmp | 8 ГБ |
| **Итого** | **60–150 ГБ** |

#### 1.1.4 Сеть

| Поток | Объём |
|---|---|
| LLM API (входящий, в основном текст) | 100–500 МБ/день |
| LLM API (исходящий, промпты) | 30–100 МБ/день |
| Telegram (webhook/polling) | 1–10 МБ/день |
| Email (SMTP) | 1–5 МБ/день |
| SSH-pull реплика (5 мин) | 50–200 МБ каждые 5 мин (с дельтой) |
| **Итого** | **~3–10 ГБ/день**, пиково 50 МБ каждые 5 мин |

#### 1.1.5 GPU (опционально, но рекомендуется)

Для локального inference с приемлемой скоростью:
- **NVIDIA с 8+ ГБ VRAM** (RTX 3060/4060, A4000, L4).
- **Или** Apple Silicon M1/M2/M3/M4 (через Ollama + Metal).
- **Или** CPU-only (в 5–20× медленнее, но работает).

### 1.2 Минимальная конфигурация (для 3 агентов, режим «экономия»)

| Параметр | Минимум | Комментарий |
|---|---|---|
| **CPU** | 4 vCPU, x86_64 (или ARM64) | Ollama неплохо работает на современных ARM |
| **RAM** | **16 ГБ** | Тесно, но 1 локальная модель + облако потянет |
| **Диск** | 80 ГБ NVMe SSD | NVMe критичен для скорости загрузки моделей |
| **Сеть** | 100 Мбит/с, безлимитный трафик | — |
| **GPU** | Опционально (CPU-only) | Скорость inference 5–20 tok/s на 7–8B модели |
| **ОС** | Ubuntu 24.04 LTS / Debian 12 | — |
| **Цена** | **$15–25/мес** (Hetzner, DigitalOcean, Vultr) | — |

### 1.3 Рекомендуемая конфигурация (для 3 агентов + локальные модели)

| Параметр | Рекомендация | Комментарий |
|---|---|---|
| **CPU** | 8 vCPU, x86_64 (Intel Xeon или AMD EPYC) | — |
| **RAM** | **32 ГБ** | Достаточно для 2–3 локальных моделей одновременно |
| **Диск** | 200 ГБ NVMe SSD | С запасом под рост моделей и логов |
| **Сеть** | 1 Гбит/с, безлимит | — |
| **GPU** | **NVIDIA L4 (24 ГБ VRAM)** или **RTX 4090 (24 ГБ)** | Inference 30–80 tok/s для 7–13B моделей |
| **ОС** | Ubuntu 24.04 LTS Server | — |
| **Цена** | **$80–200/мес** (в зависимости от провайдера) | RunPod, Vast.ai, Lambda Labs, Hetzner |
| **Провайдеры** | RunPod, Vast.ai, Lambda Labs (GPU); Hetzner, OVH (CPU) | — |

### 1.4 Альтернативные варианты

#### 1.4.1 Apple Silicon (M2/M3/M4 Max)

| Параметр | Значение |
|---|---|
| **Плюсы** | Нет шумных вентиляторов, отличная энергоэффективность, unified memory |
| **Минусы** | Нет облачного варианта, только bare-metal/colo |
| **Конфиг** | Mac Studio M3 Max (64 ГБ unified memory) |
| **Цена** | $4000+ (разово) + colocation $30/мес |
| **Скорость inference** | 40–60 tok/s для 7–8B моделей (Metal) |

#### 1.4.2 Домашний сервер

| Параметр | Значение |
|---|---|
| **Плюсы** | Полный контроль, нет месячной платы, можно сэкономить на GPU |
| **Минусы** | Зависит от домашнего интернета (uptime 95–99%), нет реплики в облако |
| **Конфиг** | MiniPC (Intel NUC, Beelink) + внешний GPU через Thunderbolt |
| **Цена** | $1000–2000 (разово) + электричество ~$10/мес |

**Не рекомендуется** для mission-critical сценариев без резервного канала связи.

### 1.5 Active-warm standby (опционально, +1 VPS)

Для устойчивости к потере VPS-1:

| Параметр | Значение |
|---|---|
| **Конфиг** | Минимальный: 2 vCPU, 4 ГБ RAM, 40 ГБ SSD |
| **Роль** | Cold standby: реплицирует БД, поднимает Venya Core при отсутствии heartbeat от VPS-1 |
| **Цена** | **$5–10/мес** (Hetzner, DigitalOcean) |
| **Failover** | Автоматический через cron + ssh-pull каждые 5 мин |
| **Время переключения** | ≤ 5 минут |

---

## 2. Состав программного обеспечения

### 2.1 Системный уровень (ОС)

| Компонент | Версия | Назначение |
|---|---|---|
| **Ubuntu Server LTS** | 24.04 (Noble Numbat) | ОС, до 2029 года поддержка |
| ИЛИ **Debian** | 12 (Bookworm) | Более консервативный выбор |
| **systemd** | ≥ 255 | Управление сервисами |
| **OpenSSH Server** | ≥ 9.6 | Доступ, ssh-pull реплика |
| **ufw** | 0.36+ | Файрвол (минимум правил) |
| **fail2ban** | 1.1+ | Защита SSH от брутфорса |
| **chrony / systemd-timesyncd** | — | Точное время (для логов и cron) |
| **cron / systemd-timer** | — | Планировщик |

### 2.2 Runtime и пакеты

| Компонент | Версия | Назначение |
|---|---|---|
| **Python** | 3.12+ | Venya Core, агенты, утилиты |
| **pip + venv** | — | Изоляция зависимостей |
| **Node.js** | 20 LTS | OpenClaw Gateway требует |
| **npm** | 10+ | Установка OpenClaw плагинов |
| **Git** | 2.40+ | Деплой конфигов |
| **curl, jq** | — | Скрипты Watchman |
| **rsync** | 3.2+ | ssh-pull реплика |
| **sqlite3 CLI** | 3.45+ | Отладка БД |
| **tmux** | 3.3+ | OpenClaw skill `tmux` |
| **himalaya** | 1.1+ | OpenClaw skill `email` |
| **msmtp** | 1.8+ | SMTP-клиент для SEV-1 |
| **msmtp-mta** | — | Sendmail-обёртка для msmtp |

### 2.3 OpenClaw (2026.6.1)

| Компонент | Назначение |
|---|---|
| **OpenClaw Gateway** | Слушает `127.0.0.1:18789`, координирует агентов |
| **OpenClaw CLI** | Управление: `openclaw start\|stop\|status\|logs` |
| **Конфиг** | `~/.openclaw/openclaw.json` (есть в вашей системе) |
| **Плагин `coding-agent`** | Coding-задачи (нужно **включить**) |
| **Плагин `ollama`** | Уже включён |
| **Плагин `telegram`** | **Включить** для Telegram-канала |
| **Плагин `himalaya`** | Email через himalaya CLI |
| **Плагин `mcporter`** | MCP-интеграция (если нужна) |
| **Плагин `model-usage`** | Сбор метрик использования LLM |
| **Плагин `session-logs`** | Логи сессий |
| **Плагин `tmux`** | Управление процессами агентов |

### 2.4 Ollama + модели

| Компонент | Версия | Назначение |
|---|---|---|
| **Ollama** | 0.5+ (latest) | Локальный LLM runtime |
| **gemma4** | latest | Уже есть в `openclaw.json`, основная быстрая модель |
| **glm-5.1:cloud** | latest | Уже есть, primary default |
| **llama3.1:8b-instruct-q4_K_M** | latest | Backup для Izya-Speedy |
| **qwen2.5-coder:7b** (опционально) | latest | Альтернатива для code-задач |

**Размер моделей на диске**: 4–8 ГБ каждая (Q4_K_M квантизация), ~12–24 ГБ для 3 моделей.

### 2.5 Venya Core + агенты (Python)

| Компонент | Назначение | Зависимости |
|---|---|---|
| **venya_core.py** | Оркестратор (HOLD/QUEUE/STATUS/ESCALATE) | sqlite3, fcntl, requests |
| **agent_izya_speedy.py** | Быстрый агент (gemma4 + claude-haiku fallback) | openclaw-cli, requests |
| **agent_izya_deep.py** | Сильный агент (claude-sonnet-4.6 + gpt-5 fallback) | openclaw-cli, requests |
| **agent_moysha.py** | Критик (gemini-2.5-pro + deepseek fallback) | openclaw-cli, requests |
| **watchman.sh** | Cron-скрипт внешнего арбитра | curl, pgrep, mail, sqlite3 |
| **replication.sh** | ssh-pull реплика | rsync, ssh |
| **chaos_test.sh** | Еженедельные chaos-тесты | iptables, tc, systemctl |

**Python-зависимости** (requirements.txt):
```
openclaw-cli>=1.0
anthropic>=0.40
openai>=1.50
google-generativeai>=0.8
requests>=2.32
python-telegram-bot>=21.0
pyyaml>=6.0
structlog>=24.1
tenacity>=9.0
```

### 2.6 Observability (минимум)

| Компонент | Назначение | Установка |
|---|---|---|
| **Prometheus** | Сбор метрик | `apt install prometheus` или Docker |
| **node_exporter** | Системные метрики (CPU, RAM, диск) | Бинарник |
| **Grafana** | Визуализация | `apt install grafana` или Docker |
| **Loki + Promtail** | Логи (опционально) | Docker |
| **Healthchecks.io** (опционально) | Внешний cron-мониторинг | SaaS, free tier |

**Минимально** без Prometheus: log-файлы + cron-скрипт `metrics-collector.sh`, пишущий в SQLite.

### 2.7 Безопасность

| Компонент | Назначение |
|---|---|
| **UFW** (Uncomplicated Firewall) | Закрыть все порты кроме SSH (22) |
| **fail2ban** | SSH brute-force protection |
| **SSH keys** (только, без паролей) | Аутентификация |
| **AppArmor** (Ubuntu default) | Песочница для сервисов |
| **systemd-resolved** | DNS-over-TLS (опционально) |
| **OpenClaw token rotation** | Ротация токена раз в 90 дней |
| **LLM API keys** | В env-переменных или systemd EnvironmentFile, **не в git** |
| **Encrypted backups** | БД шифруется перед репликацией (опционально) |

### 2.8 Telegram-бот

| Компонент | Назначение |
|---|---|
| **@BotFather** | Создание бота (один раз) |
| **python-telegram-bot 21+** | Polling/webhook |
| **OpenClaw plugin `telegram`** | Альтернативный клиент |
| **Отдельный аккаунт** для Venya Command Bot | Изоляция от личного Telegram |

---

## 3. Структура файлов на сервере

```
/etc/venya/                          # конфиги
├── venya.yaml                       # главный конфиг роя
├── agents/                          # конфиги агентов
│   ├── izya-speedy.yaml
│   ├── izya-deep.yaml
│   └── moysha.yaml
├── providers.yaml                   # LLM-провайдеры
├── channels/                        # каналы уведомлений
│   ├── telegram.yaml
│   └── email.yaml
└── systemd/                         # systemd unit-файлы
    ├── venya-core.service
    ├── watchman.service
    └── replication.service

/var/lib/venya/                      # данные
├── context.db                       # SQLite (WAL)
├── context.db-wal                   # WAL-журнал
├── decisions/                       # экспортированные решения (ротация)
├── incidents/                       # экспортированные инциденты
└── outbox/                          # очередь уведомлений

/var/log/venya/                      # логи
├── core.log                         # Venya Core
├── watchman.log                     # Watchman
├── agents/                          # агенты
│   ├── izya-speedy.log
│   ├── izya-deep.log
│   └── moysha.log
├── openclaw.log                     # OpenClaw Gateway
└── replication.log                  # ssh-pull

/var/backups/venya/                  # бэкапы (ежедневно)
└── context-YYYY-MM-DD.db.gz

/root/venya/                         # исходники (git)
├── core/                            # venya_core.py
├── agents/                          # agent_*.py
├── watchman/                        # watchman.sh, replication.sh
├── chaos/                           # chaos_test.sh
├── systemd/                         # unit-файлы
├── tests/                           # pytest
└── requirements.txt
```

---

## 4. План развёртывания

### 4.1 MVP (10–14 дней)

| Этап | Дни | Что делается |
|---|---|---|
| **E1. Сервер** | 1 | Заказать VPS, настроить SSH, базовая безопасность (ufw, fail2ban) |
| **E2. Системный стек** | 1 | Установить Python 3.12, Node.js 20, Ollama, msmtp, jq, tmux, himalaya |
| **E3. OpenClaw** | 0.5 | Установить OpenClaw 2026.6.1, скопировать конфиг, проверить `openclaw doctor` |
| **E4. Модели Ollama** | 1 | Скачать gemma4, llama3.1:8b, glm-5.1:cloud (если ещё не) |
| **E5. Venya Core** | 2 | Написать `venya_core.py`, юнит-тесты, systemd unit |
| **E6. Агенты** | 3 | 3 агента (Speedy/Deep/Moysha), маршрутизация, fallback-логика |
| **E7. Watchman** | 1 | bash-скрипт, cron, Telegram-уведомления |
| **E8. Тесты failover** | 1 | Искусственно убить Anthropic API, проверить переключение |
| **E9. Репликация** | 1 | ssh-pull на VPS-2 (cron + systemd) |
| **E10. Email-канал** | 0.5 | msmtp + SEV-1 |
| **E11. Chaos-тесты** | 1 | 6 сценариев |
| **E12. Документация** | 0.5 | Runbook, восстановление после 2–3 дневного простоя |

**Итого**: 12.5 дней.

### 4.2 Бюджет по этапам

| Этап | Стоимость (мес) |
|---|---|
| VPS (рекомендуемый, без GPU) | $30–50 |
| GPU-аренда (если нужны локальные модели) | $50–150 |
| VPS-2 (standby, минимальный) | $5–10 |
| LLM API (Anthropic, OpenAI, Google) | $50–120 |
| Домен + DNS (опционально) | $1–2 |
| **Итого** | **$136–332/мес** |
| **Минимум (только 1 VPS, минимум LLM)** | **$55–80/мес** |
| **Только локальные модели (без API)** | **$35–60/мес** |

---

## 5. Выбор облачного провайдера (что заказать)

### 5.1 Сравнение

| Провайдер | CPU 8/32 ГБ | GPU L4 24 ГБ | Standby 2/4 | Плюсы | Минусы |
|---|---|---|---|---|---|
| **Hetzner** | €30/мес | — (нет GPU) | €5/мес | Дёшево, стабильно, Европа | Нет GPU, нет Азии |
| **OVH** | €40/мес | €150/мес (если есть) | €7/мес | Европа, широкая линейка | Сложная панель |
| **DigitalOcean** | $48/мес | — (нет GPU) | $6/мес | Простой, хорошая документация | Дороже Hetzner |
| **Vultr** | $48/мес | — (нет GPU) | $6/мес | Много локаций | Средний uptime |
| **Linode (Akamai)** | $48/мес | — (нет GPU) | $6/мес | Стабильный, США/Европа/Азия | — |
| **RunPod** | — | $200–400/мес (L4) | — | GPU почасово | Нет persistent storage на GPU |
| **Vast.ai** | — | $100–250/мес (RTX 4090) | — | Дёшево GPU | Ненадёжные хосты |
| **Lambda Labs** | — | $300/мес (A10) | — | Стабильный, ML-focused | Очередь на инстансы |
| **AWS Lightsail** | $40/мес | — | $5/мес | Привычно | Дорого для характеристик |
| **Google Cloud (e2)** | $50/мес | — | $7/мес | — | Сложно, дорого |

### 5.2 Рекомендация

**Бюджетный вариант**: **Hetzner** (Германия/Финляндия) — CCX 23 (8 vCPU, 32 ГБ RAM, 240 ГБ NVMe) за €30/мес + CX 21 (2 vCPU, 4 ГБ RAM, 40 ГБ) за €5/мес как standby. **Итого €35/мес ≈ $38/мес** за серверы.

**С GPU**: добавить **Vast.ai** или **RunPod** с RTX 4090 / L4 на 50–80% времени (остальное — облачный API или standby).

**Локация**: Европа (Hetzner DE/FI) для GDPR-совместимости и низкого пинга до Anthropic/OpenAI API.

---

## 6. Observability — что мониторить

### 6.1 Метрики (обязательные)

| Метрика | Источник | Алерты |
|---|---|---|
| CPU usage | node_exporter / `top` | > 90% в течение 10 мин |
| RAM usage | node_exporter / `free` | > 85% |
| Disk usage | node_exporter / `df` | > 80% |
| OpenClaw Gateway health | `curl 127.0.0.1:18789/health` | Не отвечает 3 мин → SEV-2 |
| Venya Core process | `pgrep venya_core.py` | Нет процесса 30 сек → SEV-1, restart |
| Watchman last run | `stat /tmp/venya_alert_hash` | > 120 сек → SEV-1 |
| LLM daily tokens (per provider) | OpenClaw `model-usage` | > 80% лимита → WARN |
| LLM provider availability | HEAD-запрос | 3 ошибки подряд → SEV-2, switch |
| SQLite WAL size | `ls -la /var/lib/venya/context.db-wal` | > 100 МБ → SEV-3, checkpoint |
| Replication lag | `stat /var/backups/venya/` | > 30 мин → SEV-2 |
| Last decision in DB | `sqlite3 ... SELECT MAX(ts)` | > 10 мин (рабочее время) → SEV-2 |
| Telegram bot reachable | `getMe` | 3 ошибки → SEV-3, switch to email |

### 6.2 Логи

- **JSON-формат** (structlog) — для парсинга.
- **Ротация**: ежедневно, 14 дней онлайн + 90 дней в бэкапе.
- **Централизация** (опционально): Loki + Promtail, или просто файлы + `journalctl`.

### 6.3 Визуализация (минимальная)

- **Grafana dashboard** с 4 панелями:
  1. CPU/RAM/Disk (последние 24 ч).
  2. Количество решений/час (по каждому агенту).
  3. LLM tokens/день (по провайдерам).
  4. Инциденты за неделю (гистограмма).

### 6.4 Внешний мониторинг (обязательно)

- **Healthchecks.io** (free tier) — пинг от Watchman каждые 5 мин. Если пинг не пришёл 15 мин — email/SMS админу.
- **Uptime Kuma** (self-hosted) — проверка `127.0.0.1:18789/health` с публичного адреса (через nginx + let's encrypt).

---

## 7. План восстановления после инцидентов

### 7.1 Типичные сценарии

| Сценарий | Действие | Время |
|---|---|---|
| **Izya-Deep умер (Anthropic down)** | Venya Core переключает роль на Izya-Speedy | ≤ 5 мин |
| **OpenClaw Gateway умер** | systemd `Restart=always`, recovery ≤ 30 сек | ≤ 1 мин |
| **VPS-1 целиком умер** | cron на VPS-2 детектит отсутствие heartbeat, ssh-pull реплики, поднимает Venya Core | ≤ 10 мин |
| **Диск заполнился** | logrotate, удаление старых бэкапов, VACUUM SQLite | ≤ 15 мин |
| **SQLite corruption** | восстановление с реплики (VPS-2 или бэкап) | ≤ 30 мин |
| **Telegram-бот забанен** | автоматический fallback на email | ≤ 5 мин |
| **Все облачные LLM упали** | переключение на локальные Ollama-модели (gemma4, llama3.1:8b, glm-5.1) | ≤ 5 мин |
| **Случайный kill всех процессов** | systemd auto-restart + ssh-pull восстановление БД | ≤ 5 мин |

### 7.2 Runbook (краткий)

```bash
# 1. Проверить состояние
systemctl status venya-core openclaw watchman
openclaw status
/var/lib/venya/context.db "SELECT * FROM providers ORDER BY last_ok DESC"

# 2. Перезапустить всё
sudo systemctl restart openclaw venya-core
sudo systemctl status venya-core

# 3. Ручное переключение командира
sudo /root/venya/scripts/force_leader.sh moysha

# 4. Восстановить БД с реплики
sudo systemctl stop venya-core
sudo rsync -avz vps2:/var/lib/venya/context.db* /var/lib/venya/
sudo systemctl start venya-core

# 5. Экстренный HOLD-режим
sudo /root/venya/scripts/enter_hold.sh
```

---

## 8. Источники и стандарты

- **Hetzner Cloud**: https://www.hetzner.com/cloud
- **Vast.ai**: https://vast.ai
- **Ollama**: https://ollama.com
- **OpenClaw docs**: https://docs.openclaw.io (предположительно — нужно уточнить путь в установленной версии)
- **Anthropic API docs**: https://docs.anthropic.com
- **OpenAI API docs**: https://platform.openai.com/docs
- **Google Gemini API**: https://ai.google.dev
- **systemd.service(5)**: `man systemd.service`
- **SQLite WAL**: https://www.sqlite.org/wal.html
- **NATS** (опционально для pub/sub): https://nats.io
- **Prometheus + Grafana**: https://prometheus.io, https://grafana.com

---

## 9. Резюме для принятия решения

| Сценарий | VPS | Standby | GPU | LLM/мес | Итого/мес |
|---|---|---|---|---|---|
| **Минимум** (1 VPS, облачные LLM) | $25 | — | — | $50 | **$75** |
| **Рекомендуемый** (1 VPS + standby + облачные LLM) | $50 | $10 | — | $80 | **$140** |
| **Локальный LLM** (1 VPS с GPU + standby) | $150 | $10 | $0 (включено) | $30 | **$190** |
| **Максимум** (2 VPS + GPU-аренда) | $80 | $10 | $200 | $80 | **$370** |

**Рекомендую начать с варианта «Рекомендуемый»** за $140/мес. Через месяц эксплуатации, когда накопится статистика, переходить к варианту «Локальный LLM» с собственной GPU-арендой, если качество локальных моделей окажется приемлемым для ваших задач.

**Все остальные пункты промпта** (Venya Core, Watchman, эпохи, дедупликация, chaos-тесты) — **сохранить** из v2 без изменений, но реализовать в упрощённом виде согласно разделам §2–§7.
