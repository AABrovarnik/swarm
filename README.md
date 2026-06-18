# Swarm — отказоустойчивая система AI-агентов

Проект «swarm» — это программный рой из 3 LLM-агентов и одного процедурного оркестратора, спроектированный так, чтобы выживать при:
- исчерпании лимита одного LLM-провайдера (исходный инцидент — 37 часов простоя);
- региональных сбоях облаков;
- отказе Telegram-канала;
- одновременной потере 30% агентов в неделю на 2–3 дня;
- компрометации одной ноды или инсайдера.

> **Не путать** с `Posts/swarm-robotics/` — там 18-документная серия про рой-робототехнику (UAV, IoT, подводные рои, космос). Этот каталог — про **программный** рой AI-агентов.

---

## Состав

### Версия 4.1 Operational (целевая реализация, 2026-06-18)

| Документ | Назначение |
|---|---|
| [`architecture-v4.1-operational.md`](architecture-v4.1-operational.md) | Единая диспетчерская Вени, P1 SQLite как реестр агентных задач, роли Студента/Тесли/Изи, границы дашборда и Git. |
| [`implementation-plan-v4.1.md`](implementation-plan-v4.1.md) | Переход от живых P0/P1-shadow и отдельных ботов к сквозному контуру задачи. |

### Версия 4 Lite (исходный MVP-переход, 2026-06-18)

| Документ | Назначение |
|---|---|
| [`architecture-v4-lite.md`](architecture-v4-lite.md) | Практический MVP для текущего сервера 4 vCPU / 8 ГБ: GPT-5.5 → MiniMax M3 → локальный Qwen, процедурные задания и внешний контроль. |
| [`exchange/README.md`](exchange/README.md) | Git-протокол обмена задачами, отчётами и решениями между владельцем, Core и агентами. |
| [`schemas/task.schema.json`](schemas/task.schema.json) | Машиночитаемая схема задания. |
| [`schemas/report.schema.json`](schemas/report.schema.json) | Машиночитаемая схема отчёта агента. |

### Версия 3 (актуальная, 2026-06-14)

| Документ | Назначение |
|---|---|
| [`architecture-v3.md`](architecture-v3.md) | **Сводный архитектурный документ v3.** Принципы, компоненты, бюджет, сценарии отказа, runbook, observability, инварианты. |
| [`Промпт-v3.md`](Промпт-v3.md) | **Промпт v3** для red-team review архитектуры v3. Передаётся архитектору/инженеру. |

### Предыдущие версии (для истории)

| Документ | Назначение |
|---|---|
| [`Промпт.md`](Промпт.md) | Исходный промпт v1: 12 принципов отказоустойчивой архитектуры (2026-06-11) |
| [`redteam-review-2026-06-11.md`](redteam-review-2026-06-11.md) | Первая рецензия v1: 5 сценариев отказа, SPOF, MVP-90д (2026-06-11) |
| [`redteam-llm-swarm-review.md`](redteam-llm-swarm-review.md) | Реализационная рецензия: подбор LLM, бюджет, код-костяк (2026-06-12) |
| [`redteam-prompt-v2-with-openclaw.md`](redteam-prompt-v2-with-openclaw.md) | Промпт v2: один пакет передачи, 3 LLM + 1 HOLD-агент, OpenClaw 2026.6.1 (2026-06-12) |
| [`server-spec-and-software.md`](server-spec-and-software.md) | ТЗ на сервер: железо, ПО, бюджет, observability, runbook (2026-06-12) |

### Сводный отчёт

Полный отчёт «совета директоров» с 30 замечаниями и кросс-валидацией трёх перспектив (SRE, COO, Security) — в [`../Posts/swarm-review-2026-06-14.md`](../Posts/swarm-review-2026-06-14.md).

---

## Архитектура v3 в одном абзаце

Один процедурный оркестратор **Venya Core** (Python, single instance, systemd `Type=notify`) координирует 3 LLM-агентов через OpenClaw Gateway, работающий в systemd-песочнице. Решения подписываются Ed25519 и хранятся в SQLite WAL с offsite-репликой через `litestream` → Backblaze B2 EU. Передача командования — структурированный JSON-пакет с `next_action_type` ∈ enum. In-flight запросы переживают рестарт OpenClaw через outbox-pattern. **Watchman** — 3 проверки в systemd-timer 30 сек + external Healthchecks.io. Дедуп уведомлений по `(incident_class, incident_id, channel)`. 4 LLM-провайдера (Ollama local + Anthropic + OpenAI + Google) на разных AS egress, 2-account strategy для Anthropic. **3 уровня эскалации SEV-1**: Telegram (основной + запасной аккаунт) + email + Pushover. **Backup-человек обязателен** (Bitwarden ssh-key TTL, 2-tier runbook, bus-factor ≥ 2).

---

## Вердикт «совета директоров» (TL;DR)

| | |
|---|---|
| **v1** | Архитектурное эссе, не спецификация |
| **v2** | Жизнеспособна для MVP за 10–14 дней, но требует доработки |
| **v3** | Принята с обязательными доработками (10 P0-блокеров, см. `architecture-v3.md` §10) |
| **Бюджет v3 MVP** | $158/мес прямых + $14000/год реальных (включая скрытые операционные) |
| **Срок MVP** | 10–14 дней (1 владелец + 1 backup-человек) |
| **Главный риск** | 1 разработчик = SPOF для надёжности, безопасности и операций |
| **Ключевое отличие v3 от v2** | Ed25519-подписи БД, litestream, systemd sandbox OpenClaw, threat model, GDPR compliance, backup-человек обязателен |

---

## Контакты

- Владелец: Александр Броварник
- GitHub: https://github.com/AABrovarnik/swarm
