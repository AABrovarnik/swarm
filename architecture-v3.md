# Архитектура v3: Программный рой AI-агентов (Venya Core + Izya + Moysha++)

> **Версия:** 3.0 (2026-06-14).
> **Назначение:** сводный архитектурный документ, объединяющий требования исходного промпта v1, рекомендации red-team review v1 (2026-06-11) и v2 (2026-06-12), и замечания «совета директоров» (SRE + COO + Security, 2026-06-14).
> **Связанные документы:** `Промпт-v3.md`, `redteam-prompt-v2-with-openclaw.md`, `redteam-llm-swarm-review.md`, `redteam-review-2026-06-11.md`, `server-spec-and-software.md`, `Posts/swarm-review-2026-06-14.md`.

---

## 0. TL;DR

**Что строим:** отказоустойчивый программный рой из 3 LLM-агентов + 1 процедурный оркестратор, способный пережить:
- исчерпание лимита LLM-провайдера (исходный инцидент — 37 ч простоя);
- одновременный отказ 2 LLM-провайдеров из 4;
- отказ Telegram-канала;
- потерю VPS-1 (cold start с offsite backup ≤10 мин);
- 30% потерю агентов в неделю на 2–3 дня.

**Размер:** 3 агента, 1 Venya Core, 4 LLM-провайдера (1 локальный + 3 облачных), 3 канала уведомлений.

**Бюджет:** $158/мес прямых + $9000–17000/год реальных (включая скрытые операционные).

**Срок MVP:** 10–14 дней при условии, что P0-блокеры из §10 закрыты **до** deploy в прод.

**People:** минимум 2 человека (владелец + backup-человек). Без backup-человека MVP не запускается.

---

## 1. Архитектурные принципы

1. **Venya Core без LLM.** Оркестратор — Python-процесс, не принимает решений через LLM. Только 4 команды: `HOLD | QUEUE | STATUS | ESCALATE`.
2. **Один командир одновременно.** Single Venya Core, нет second-instar, нет split-brain, нет flock/etcd/Raft.
3. **Ed25519-подписи всех решений.** `decisions` подписываются приватным ключом Core, verify при чтении. Hash-chain (`prev_decision_hash`) для обнаружения рассинхронизации.
4. **Один пакет передачи, не три.** Структурированный JSON с `next_action_type` ∈ enum, не свободная форма.
5. **Outbox-pattern для in-flight запросов.** Core пишет в `outbox` **до** отправки агенту, retry с idempotency-key, переживает рестарт OpenClaw.
6. **4 LLM-провайдера, разные AS egress.** Ollama local + Anthropic + OpenAI + Google. Падение любых 3 — рой живёт. 2-account strategy для Anthropic.
7. **litestream → S3-compatible offsite.** RPO ≤ 30 сек, cold start за ≤10 мин, restore-drill еженедельно.
8. **Watchman = 3 проверки + external monitor.** systemd-timer 30 сек, не cron 60 сек. Healthchecks.io + Venya Core сам алертит если Watchman молчит.
9. **Дедуп уведомлений по `(class, id, channel)`, не общий hash.** Каждый канал получает отдельный retry, alert-fatigue предотвращён.
10. **OpenClaw в systemd-песочнице + nftables egress filter.** Никаких RCE через prompt injection, никакого shell-доступа через tmux/coding-agent.
11. **2-account strategy для LLM, 2 Telegram-аккаунта, 2 уровня эскалации.** Канальная избыточность на каждом слое.
12. **People SPOF — неприемлемо.** Backup-человек обязателен, Bitwarden TTL, 2-tier runbook, bus-factor ≥ 2.

---

## 2. Компоненты

### 2.1. Venya Core (Python, single instance)

- **Процесс:** 1 экземпляр на VPS-1, systemd `Type=notify`.
- **API:** REST `/healthz`, `/readyz`, `/status`, `/hold`, `/queue`, `/escalate`.
- **БД:** SQLite WAL + Ed25519-подписи всех `decisions`.
- **Подписи:** Ed25519 keypair в `/root/venya/keys/`, права 0600, owner `root`. Публичный ключ компилируется в код Core для verify.
- **Outbox:** таблица `outbox(task_id, payload, state, channel_attempts_json, last_attempt_at)`.
- **Circuit breaker:** таблица `providers(name, circuit_state, half_open_at)`. State machine CLOSED→OPEN→HALF_OPEN.
- **Песочница systemd:**
  ```
  [Service]
  User=venya-core
  DynamicUser=yes
  ProtectSystem=strict
  ProtectHome=yes
  NoNewPrivileges=yes
  RestrictNamespaces=yes
  SystemCallFilter=@system-service
  SystemCallErrorNumber=EPERM
  ```
- **Зависимости:** Python 3.12, `sqlite3`, `cryptography` (Ed25519), `requests`, `pydantic`.
- **Лимиты:** ≤2000 строк кода.

### 2.2. Агенты (3 шт., Python subprocess)

| Агент | Primary LLM | Backup 1 | Backup 2 | Local fallback |
|---|---|---|---|---|
| **Izya-Speedy** | Claude Haiku 4.5 | GPT-5-mini | — | ollama/llama3.1:8b |
| **Izya-Deep** | Claude Sonnet 4.6 | GPT-5 | Gemini 2.5 Pro | ollama/glm-5.1:cloud |
| **Moysha++** | Gemini 2.5 Pro | Claude Sonnet 4.6 (другой аккаунт) | — | — (критик) |

**Приоритет лидерства:** `Deep > Speedy > Moysha++`. Если Deep умер — Speedy, если Speedy умер — Moysha++.

**Решение «какого провайдера использовать»** — функция Venya Core (таблица `provider_priority`), а не агента. Агент видит только `provider_id` в ответе Core.

**Изменения v3:**
- Moysha++ может работать на другом **аккаунте** того же провайдера (2-account strategy), не обязательно на другом провайдере.
- DeepSeek **отключён** (GDPR Chap. V).

### 2.3. OpenClaw Gateway (systemd sandbox)

- **Конфиг:** `~/.openclaw/openclaw.json`, версия 2026.6.1 зафиксирована в `/etc/venya/openclaw.pin`.
- **Автообновление:** **отключено**.
- **Плагины в проде:** только `ollama` + `model-usage`. `coding-agent`, `tmux`, `himalaya` — **отключены на runtime** (security).
- **Tools:** `web.search` с domain-whitelist, `db.query` (только SELECT).
- **Песочница systemd:** `User=openclaw`, `ProtectSystem=strict`, `ProtectHome=yes`, `NoNewPrivileges=yes`, `ReadWritePaths=/var/lib/venya/spool`.
- **Egress filter nftables:** только IP-диапазоны Anthropic, OpenAI, Google, Telegram Bot API, Ollama local.
- **Session-logs:** отключены.
- **Analytics:** отключены (`metrics_opt_out=true`).

### 2.4. Watchman (systemd-timer + Python скрипт)

- **systemd-timer:** `OnUnitActiveSec=30s`, `Persistent=true`.
- **Проверяет** (только 3):
  1. `curl http://127.0.0.1:8000/readyz` (Venya Core)
  2. `curl http://127.0.0.1:18789/health` (OpenClaw Gateway)
  3. `sqlite3 context.db "INSERT INTO _watchman_heartbeat VALUES (...)"`
- **Шлёт:** apprise (multi-channel с per-channel backoff).
- **External Healthchecks.io** — пинг каждые 5 мин.
- **Venya Core мониторит Watchman** — если `/var/run/watchman.heartbeat` старше 120 сек, Core сам шлёт SEV-1.

### 2.5. БД (SQLite WAL + litestream)

- **Файл:** `/var/lib/venya/context.db`, режим WAL, `synchronous=FULL`.
- **Структура:**
  - `decisions(decision_id PK, agent, decision_json, prev_hash, signature, ts)`
  - `leadership_log(decision_id, agent, from, to, ts)`
  - `incidents(id, severity, first_seen, last_seen, class, channel_status_json, hash, count)`
  - `outbox(task_id, payload, state, channel_attempts_json, last_attempt_at, ts)`
  - `providers(name, status, last_ok, last_fail, daily_tokens, circuit_state, half_open_at)`
  - `_watchman_heartbeat(ts)`
- **Подписи:** Ed25519 Core для каждой `decisions`.
- **Snapshot:** `VACUUM INTO` ежедневно в `/var/backups/venya/`.
- **Offsite replica:** `litestream` → Backblaze B2 EU bucket, retention 30 дней.
- **Verify:** `verify_backup.sh` в cron (ежечасно) — pull, `PRAGMA integrity_check`, PASS/FAIL в Telegram.

### 2.6. Каналы уведомлений (3 уровня)

| Канал | Когда | Дедуп ключ | Конфиг |
|---|---|---|---|
| **Telegram (основной)** | Все SEV | `(class, id, "telegram_main")` | Отдельный аккаунт, `ALLOWED_CHAT_IDS` env |
| **Telegram (запасной)** | SEV-1 | `(class, id, "telegram_backup")` | Другой номер, физ. SIM |
| **Email** | SEV-1 | `(class, id, "email")` | Fastmail/ProtonMail/Workspace (2FA-Recovery) |
| **Pushover** | SEV-1 | `(class, id, "pushover")` | $5 разово, мобильное приложение |

### 2.7. Backup offsite (Backblaze B2 EU)

- **Bucket:** `venya-backup`, регион EU, $5/TB-мес.
- **litestream replica** с RPO ≤ 30 сек.
- **Retention:** 30 дней daily + 12 месяцев monthly.
- **Restore-drill:** еженедельно, на test-VPS.

### 2.8. OpenClaw-less escape hatch

- **Скрипт `/root/venya/scripts/manual_mode.sh`** — Python, прямые HTTP-вызовы к Anthropic/OpenAI/Google API.
- Используется при: OpenClaw CVE, обновлении с breaking change, отказе upstream.
- Качество UX ниже (нет `coding-agent`), но **SEV-1 задачи работают**.

### 2.9. Люди

- **Владелец:** full access, Tier 2 runbook.
- **Backup-человек:** read-only + `triage.sh` + `ack` в Telegram, Tier 1 runbook.
- **Bitwarden:** ssh-ключи с TTL 1 час, логирование получения.
- **Onboarding:** 2 ч теории + 1 ч live-fire drill + 1 ч shadowing'а.

---

## 3. Бюджет

### 3.1. Прямые расходы

| Категория | Сумма/мес | Сумма/год |
|---|---|---|
| VPS (Hetzner CCX 23, 8 vCPU/32 ГБ) | €30 | $390 |
| VPS-2 (cold standby, 2 vCPU/4 ГБ) | €5 | $65 |
| LLM API (Anthropic, OpenAI, Google) | $50–120 | $600–1440 |
| Backup offsite (B2 EU) | $5 | $60 |
| Healthchecks.io | $0 (free tier) | $0 |
| Pushover | — | $5 (разово) |
| Домен | — | $10 |
| **ИТОГО прямых** | **$90–160/мес** | **$1130–1970/год** |

### 3.2. Скрытые операционные расходы (новое в v3)

| Категория | Чел-часов/год | $/год (при $50/ч) | Прямые $/год |
|---|---|---|---|
| Ротация API-ключей (12 процедур × 30 мин) | 6 | $300 | $0 |
| Ротация TLS-сертификатов (Let's Encrypt auto + verify) | 2 | $100 | $0 |
| Ротация OpenClaw token (4 × 30 мин) | 2 | $100 | $0 |
| Ротация ssh-pull ключа (2 × 1 ч) | 2 | $100 | $0 |
| Chaos-тесты (CI + game day 1×/мес × 2 ч) | 28 | $1400 | $50 (ephemeral VM) |
| Зависимости CVE-мониторинг (1 ч/нед) | 50 | $2500 | $0 |
| Cold start после 2–3 дневного простоя (3–5 раз/год × 6 ч) | 24 | $1200 | $200 (новый VPS) |
| Email-IP прогрев + DNS setup + повтор на новых VPS | 12 | $600 | $30 (PTR/SPF) |
| Backup-storage offsite retention рост | 4 | $200 | $60 |
| Обновление LLM (Sonnet 5/GPT 6/Gemini 3 — 4 раза/год × 2 дня) | 64 | $3200 | $200 (test tokens) |
| Юридические (DPA + compliance check, 1×/квартал) | 8 | $400 | $500 (юрист разово) |
| Onboarding backup-человека (1×/год, 3 дня) | 24 | $1200 | $0 |
| Healthchecks.io / Pushover / домен / 2-я SIM | 6 | $300 | $160 |
| **ИТОГО скрытых** | **230 ч/год** | **$11600/год** | **$1200/год** |

**Реальный годовой бюджет:** $1130–1970 (прямые) + $11600 (чел-часы) + $1200 (скрытые прямые) = **$14000–14800/год**.

При найме SRE/security-фрилансера: +$200–1500/мес = +$2400–18000/год.

**Итоговая вилка:** **$16400–32800/год** при полной нагрузке, **$14000/год** при самостоятельной поддержке.

---

## 4. Сценарии отказа и реакция

| Сценарий | Время реакции | Действие | SLO |
|---|---|---|---|
| **Izya-Deep умер (Anthropic down)** | ≤5 мин | Venya Core переключает роль на Izya-Speedy. Если тоже мёртв — Moysha++. | MTTR ≤5 мин |
| **OpenClaw Gateway умер** | ≤10 мин | systemd `Restart=always`, recovery ≤30 сек. In-flight запросы — outbox-pattern, retry после рестарта. | MTTR ≤10 мин |
| **VPS-1 целиком умер** | ≤10 мин | Cold start VPS-2 с litestream-реплики. Если VPS-2 не подхватил — ручной заказ нового VPS, `litestream restore`, запуск Core. | MTTR ≤30 мин |
| **2 LLM-провайдера упали** | ≤5 мин | Circuit breaker OPEN, переключение на оставшийся + локальный Ollama. | MTTR ≤5 мин |
| **Anthropic abuse review на 24–72 ч** | ≤10 мин | Core детектит 403_abuse, переключает на 2-й аккаунт Anthropic или Google. Appeals-шаблон отправлен. | MTTR ≤10 мин |
| **Telegram-бот забанен** | ≤5 мин | Автоматический fallback на email + Pushover. | MTTR ≤5 мин |
| **БД corrupted** | ≤30 мин | `litestream restore` с B2. | RPO ≤30 сек |
| **Все облачные LLM упали** | ≤5 мин | Все агенты — на локальных Ollama. Moysha++ тоже на локальной (degraded, но работает). | MTTR ≤5 мин |
| **OpenClaw CVE-RCE** | ≤1 ч | Escape hatch `manual_mode.sh`, OpenClaw отключён от сети. | MTTR ≤1 ч |
| **Владелец недоступен 7 дней (отпуск)** | — | Backup-человек Tier 1: triage + ack + эскалация на владельца. Полный доступ — после возврата. | MTTD ≤5 мин (через Watchman) |

---

## 5. Режимы работы

| Режим | Условие | Поведение |
|---|---|---|
| `NORMAL` | Все агенты живы, лидер — Izya-Deep | Полная функциональность |
| `DEGRADED_PRIMARY` | Упал primary LLM-провайдер | Backup-провайдер. Качество ↓, рой живёт |
| `DEGRADED_BACKUP` | Упали 2 LLM-провайдера | Локальный Ollama. Качество ↓↓, рой живёт |
| `DEGRADED_LOCAL` | Упали все облачные провайдеры | Все на Ollama. Moysha++ — на локальной (degraded). |
| `HOLD` | Все агенты мертвы или OpenClaw недоступен | Core: только QUEUE, STATUS, ESCALATE |
| `STANDBY` | Плановое обслуживание VPS-1 | Cold start VPS-2, IP-failover через DNS TTL ≤60 сек |
| `MAINTENANCE` | Владелец явно остановил рой | Ручной режим, всё через Core CLI |

---

## 6. Observability и метрики

### 6.1. Метрики (Prometheus + Grafana, минимум)

| Метрика | Источник | Алерт |
|---|---|---|
| `venya_core_up` | `/healthz` | SEV-1 при down > 30 сек |
| `venya_core_ready` | `/readyz` | SEV-1 при not-ready > 60 сек |
| `openclaw_gateway_up` | `/health` | SEV-2 при down > 3 мин |
| `watchman_last_run_seconds` | `/var/run/watchman.heartbeat` | SEV-1 при > 120 сек |
| `litestream_replica_lag_seconds` | `litestream snapshots -age` | SEV-2 при > 60 сек |
| `nodecrypto_offset_ms` | `chronyc tracking` | SEV-2 при > 250 мс |
| `daily_tokens_per_provider` | OpenClaw `model-usage` | WARN при > 80%, переключение при > 90% |
| `provider_circuit_state` | Core table | SEV-2 при OPEN > 5 мин |
| `outbox_pending_count` | Core table | SEV-3 при > 100, SEV-2 при > 1000 |
| `mttd_seconds` | Computed (incidents) | SEV-2 при > 10 мин |
| `mttr_per_mode` | Computed (leadership_log) | — (дашборд) |
| `pct_time_in_normal_mode` | Computed | SEV-2 при < 80% за неделю |
| `restore_drill_success_rate` | verify_backup.sh log | SEV-2 при < 80% за месяц |
| `openclaw_skills_enabled` | OpenClaw config | SEV-1 при `coding-agent` или `tmux` enabled |
| `nftables_egress_blocks_per_min` | nftables counter | SEV-3 при > 10 (возможная атака) |

### 6.2. Grafana dashboard (4 панели)

1. **Состояние роя** — uptime всех компонентов + current mode + last incident.
2. **MTTD/MTTR** — графики за неделю/месяц, разбивка по режимам.
3. **LLM cost** — stacked bar по провайдерам за 30 дней, лимиты.
4. **Инциденты** — гистограмма по SEV, breakdown по типу.

### 6.3. Внешний мониторинг

- **Healthchecks.io** — Watchman пингует каждые 5 мин. Если пинг не пришёл 15 мин — email/SMS админу.
- **Uptime Kuma** (self-hosted) — проверка `127.0.0.1:8000/healthz` с публичного адреса.
- **Status page** — Telegram-канал «Swarm Status», автоматически постит при смене режима.

---

## 7. Runbook (2-tier)

### 7.1. Tier 1: для backup-человека

**Имеет доступ:**
- `triage.sh` (read-only, выводит состояние всех компонентов)
- Inline `ack` в Telegram-канале
- Без sudo, без `force_leader.sh`, без deploy

**Процедура при SEV-1:**
1. Открыть `triage.sh` → увидеть `mode: HOLD` или `mode: DEGRADED_*`.
2. Прочитать последний инцидент в Telegram-канале.
3. Нажать `ack` (Telegram inline кнопка).
4. Если владелец не отвечает 15 мин → эскалация на 2-й контакт (см. `/etc/venya/escalation.md`).
5. НЕ делать restart, НЕ менять конфиг, НЕ запускать chaos-тесты.

### 7.2. Tier 2: для владельца

**Имеет доступ:**
- Полный sudo, deploy, `force_leader.sh`, `enter_hold.sh`, `restore.sh`, `chaos_test.sh`
- Bitwarden/Vault с TTL-ключами

**Процедура при SEV-1:**
1. `triage.sh` — диагностика.
2. `tail -100 /var/log/venya/core.log` — последние логи Core.
3. `sqlite3 context.db "SELECT * FROM incidents ORDER BY last_seen DESC LIMIT 10"` — инциденты.
4. `sqlite3 context.db "SELECT * FROM providers ORDER BY last_ok DESC"` — состояние провайдеров.
5. По типу инцидента — `restore.sh`, `force_leader.sh`, `enter_hold.sh`.
6. После восстановления — `verify_backup.sh`, post-mortem в `/var/log/venya/postmortems/`.

### 7.3. Appeal-шаблоны

- `/root/venya/runbooks/appeals-anthropic.md`
- `/root/venya/runbooks/appeals-openai.md`
- `/root/venya/runbooks/appeals-google.md`
- `/etc/venya/escalation.md` — контакт-карточка (хостер, Anthropic CSM, OpenAI support, Google support, владелец, backup-человек, 2-й контакт)

---

## 8. Порядок внедрения (10–14 дней)

| Этап | Дни | Что делается | Блокирует? |
|---|---|---|---|
| **E0. Threat model + IAM** | 1 | Документ угроз, IAM-роли, Bitwarden для backup-человека | Все остальные |
| **E1. Сервер + стек** | 1 | VPS (Hetzner CCX 23), ufw, fail2ban, chrony, Python 3.12, Node.js 20, OpenClaw (закреплён) | E2–E7 |
| **E2. Backup-стратегия** | 1 | litestream → B2 EU bucket, daily VACUUM, verify_backup.sh | E3 |
| **E3. Venya Core** | 2 | Ed25519-подписи, outbox, circuit breaker, systemd sandbox | E4–E7 |
| **E4. OpenClaw sandbox** | 0.5 | systemd unit, nftables egress, allowlist chat_id, отключение skills | E5 |
| **E5. Агенты + маршрутизация** | 2 | 3 агента, provider_priority, 2-account Anthropic | E6 |
| **E6. Watchman + каналы** | 1 | systemd-timer, 3 проверки, apprise, Telegram (2 акка) + email + Pushover, Healthchecks.io | E7 |
| **E7. Restore-drill + chaos** | 1 | Drill на test-VPS, chaos-test по чек-листу | E8 |
| **E8. Onboarding backup-человека** | 1 | 2 ч теории + 1 ч live-fire drill + 1 ч shadowing | E9 |
| **E9. Документация + runbook** | 0.5 | Decision tree, appeals-шаблоны, escalation contacts | — |
| **ИТОГО** | **11 дней** | | |

---

## 9. Обязательные инварианты и проверки (P0 + P1)

| # | Инвариант | Проверка | Приоритет |
|---|-----------|----------|-----------|
| I1 | **Один Venya Core одновременно** | systemd `Type=notify`, /readyz перед promotion | P0 |
| I2 | **Все `decisions` подписаны Ed25519** | Verify при чтении, не проходит → HOLD | P0 |
| I3 | **RPO ≤ 30 сек** | litestream replica lag, alert при > 60 сек | P0 |
| I4 | **Watchman ≤ 1 уведомление на (incident, channel)** | Дедуп по `(class, id, channel)`, per-channel backoff | P0 |
| I5 | **In-flight переживают рестарт OpenClaw** | outbox-pattern с retry+idempotency | P0 |
| I6 | **4 LLM-провайдера не на одной инфраструктуре** | Разные cloud account, разные AS egress | P0 |
| I7 | **Restore-drill проходит раз в неделю** | verify_backup.sh + cold-restore на test-VPS | P0 |
| I8 | **OpenClaw не имеет shell-доступа** | `coding-agent`, `tmux` отключены, egress-filter nftables | P0 |
| I9 | **Telegram-бот не может выполнить shell** | allowlist chat_id, санитайзер-парсер | P0 |
| I10 | **circuit breaker в Venya Core, не в агентах** | state machine CLOSED→OPEN→HALF_OPEN | P0 |
| I11 | **Backup-человек имеет read-only triage доступ** | Bitwarden ssh-key TTL 1ч, Tier 1 runbook | P0 |
| I12 | **Moysha++ на LLM, отличном от Deep/Speedy** | Другой провайдер или другой аккаунт | P0 |
| I13 | **NTP offset ≤ 250 ms** | chrony + alert | P1 |
| I14 | **Chaos-тесты в CI** | pytest на ephemeral VM, каждый PR | P1 |
| I15 | **session-logs в проде отключены** | OpenClaw config | P1 |
| I16 | **PII-фильтр на исходящие промпты** | presidio-analyzer или regex | P1 |
| I17 | **DeepSeek отключён** | GDPR Chap. V compliance | P0 |
| I18 | **2-account strategy для Anthropic** | 2 cloud account, 2 billing | P0 |
| I19 | **Appeals templates готовы** | 3 файла в /root/venya/runbooks/ | P1 |
| I20 | **GDPR retention policy** | 12 мес для incidents, 30 дней для outbox | P1 |

---

## 10. Что НЕ вошло в v3 (отклонено)

- 5-канальная схема уведомлений.
- 3-уровневая иерархия Izya-Speedy/Deep/Max.
- Автовыборы лидера (Raft/etcd).
- LLM-улучшатель формулировок.
- OpenClaw `coding-agent` и `tmux` в проде.
- DeepSeek как LLM-провайдер.
- Offline-llm-only режим как primary.
- 2 инстанса Venya Core для HA.
- flock как механизм distributed consensus.

---

## 11. Сводка изменений v2 → v3

| Что | v2 | v3 | Почему |
|---|---|---|---|
| **Лидерство** | flock + heartbeat + «внешний арбитр» | Single Venya Core + systemd Type=notify | flock не consensus, арбитра нет |
| **БД replica** | ssh-pull rsync 5 мин | litestream → B2 EU, 30 сек | rsync corrupted WAL |
| **Подписи БД** | Нет | Ed25519 + hash-chain | Insider threat, отравление context.db |
| **Outbox** | Не описан | Реализован | Потеря/дубль при рестарте OpenClaw |
| **Watchman** | 7 проверок, cron 60s | 3 проверки, systemd-timer 30s | Alert-fatigue |
| **Каналы** | 2 (TG + email) | 4 (TG×2 + email + Pushover) | SEV-1 single-channel SPOF |
| **Threat model** | Отсутствует | Документ + 10 контролей | Prompt injection, GDPR |
| **People SPOF** | 1 разработчик | 2 (владелец + backup) | Bus-factor = 1 |
| **DeepSeek** | Используется | Отключён | GDPR Chap. V |
| **Circuit breaker** | В агентах | В Core | Нужен обзор всех провайдеров |
| **NTP мониторинг** | Нет | chrony + alert | Ложный split-brain |
| **Appeals** | Не описаны | 3 шаблона | 24–72 ч abuse review |
| **PII-фильтр** | Нет | presidio-analyzer | Утечка контекста |
| **GDPR/ФЗ-152** | Не рассмотрен | Compliance plan | Штрафы |
| **Реальный бюджет** | $140/мес | $14000/год (включая скрытые) | Честная оценка |
| **Срок MVP** | 10–14 дней | 10–14 дней (E0 добавлен) | Threat model обязателен до кода |

---

## Приложение А. Карта «взломай X — получи Y»

| Компрометация | Что получает атакующий | Митигация в v3 |
|---|---|---|
| Venya Core RCE | Запись `decision_id=999999`, `prev_hash=...`, исполнение «решений» | Ed25519 verify на каждом read, systemd sandbox |
| `/var/lib/venya/context.db` | Полный дамп (outbox, decisions, providers) | Ed25519-подписи, offsite replica (litestream), аудит-лог |
| ssh-pull ключ VPS-2 | Lateral movement на VPS-1 | `command="rsync --server ..."` в authorized_keys |
| OpenClaw токен | Свой биллинг на счёт жертвы | systemd sandbox, EnvironmentFile права 0600, 2-account strategy |
| Telegram-бот | Ложные HOLD/ESCALATE | allowlist `chat_id`, 2 аккаунта, Pushover как 3-й канал |
| NTP-сервер | Ложный split-brain | chrony + alert при offset > 250 мс |
| LLM provider (BGP hijack) | Все 3 облачных провайдера падают | Ollama local fallback, hash-check при `ollama pull` |
| Владелец-инсайдер | RCE на VPS-1, модификация `decisions` | Ed25519 verify, tamper-evident log, 2-account strategy |
| Сосед по VPS | Утечка через shared infra | Hetzner/OVH dedicated, не shared hosting |

---

## Приложение B. Открытые вопросы

- **Что делать, если OpenClaw проект будет заброшен?** Нужен форк-план (см. §2.8 escape hatch + мониторинг upstream).
- **Экономика LLM-рой как продукта.** v3 описывает инфраструктуру, не монетизацию.
- **Disaster Recovery в другом регионе.** B2 EU — это страховка данных, не инфраструктуры.
- **On-call rotation через несколько людей.** При полном бюджете — managed-service или 2–3 фрилансера с TTL-доступом.
- **Юридические аспекты автоматического failover** (если преемник действует в юридически значимом контексте).
- **Open-source vs proprietary runtime.** Стоимость поддержки форка OpenClaw.

---

## Приложение C. Источники

- AWS Well-Architected Framework: Reliability Pillar
- Google SRE Book, Chapters 22–26
- Lamport L. «Time, Clocks, and the Ordering of Events» (1978)
- Ongaro D., Ousterhout J. «In Search of an Understandable Consensus Algorithm» (Raft, 2014)
- Anthropic Prompt Caching Documentation
- OpenAI Structured Outputs / Function Calling
- OpenAI Resilience Best Practices
- Google Gemini API Reliability
- SREcon, Chaos Engineering Conference proceedings
- NIST SP 800-53 (Security Controls)
- OWASP LLM Top 10
- GDPR Regulation 2016/679
- ФЗ-152 «О персональных данных»
- systemd.service(5), systemd.exec(5) — man pages
- SQLite WAL documentation
- Backblaze B2 + litestream documentation
