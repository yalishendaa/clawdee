# CLAWDEE Post-Install Agent Task: Memory + OpenViking

Ты запущен как Claude CLI на VPS, где уже прошёл базовый installer CLAWDEE.

Это не инструкция для человека. Это task brief для тебя как для maintenance-agent. Твоя задача - самому выполнить настройку, задавая оператору только те вопросы, без которых нельзя безопасно продолжать.

Цель:

- проверить результат базового installer-а
- починить недостающие элементы файловой памяти, не затирая существующее содержимое
- проверить memory cron и scripts
- установить локальный OpenViking или подключить внешний, если оператор выбрал OpenViking
- подключить `clawdee-telegram-gateway` к OpenViking
- выполнить end-to-end проверки и дать короткий отчёт

Жёсткие правила:

- Не делай пользователя `clawdee` системным администратором.
- Не добавляй passwordless sudo для `clawdee`.
- Не создавай и не восстанавливай `/etc/sudoers.d/clawdee-agents`.
- Не запускай базовый installer заново без явного разрешения оператора.
- Не переписывай конфиги целиком, если можно внести точечное изменение.
- Перед изменением существующего конфига создай backup с timestamp.
- Не печатай секреты полностью. В отчёте показывай только masked preview.
- Если тебе нужны root/systemd действия, а текущая сессия не root и без sudo, остановись и попроси оператора перезапустить эту Claude CLI session от root/operator shell.
- Telegram-агент должен оставаться ограниченным профилем `/home/clawdee`; системные действия выполняются только этой явной CLI-сессией.
- Если OpenViking API в установленной версии отличается от ожидаемого, зафиксируй точную ошибку и минимальный следующий fix. Не делай вид, что всё работает.

Перед изменениями:

1. Сделай аудит.
2. Спроси только недостающие решения и секреты.
3. Покажи короткий план на 4-6 пунктов.
4. Выполни план.
5. Проверь результат командами.
6. Дай короткий финальный отчёт.

Спроси у оператора только это:

- Включать OpenViking сейчас?
  - `yes-local`: установить локальный OpenViking на этот VPS
  - `yes-external`: подключить существующий OpenViking URL
  - `no`: проверить только файловую память и cron
- Если OpenViking включён: account namespace, default `clawdee`.
- Если `yes-external`: OpenViking URL и API key.
- Если `yes-local`: model provider API key для OpenViking extraction/embeddings.

Для локального OpenViking API key сгенерируй сам:

```bash
openssl rand -hex 32
```

Не путай provider API key для OpenViking с Claude subscription/OAuth.

---

## Phase 1: Audit Current Installer Result

Выполни эти проверки сам и кратко зафиксируй факты:

```bash
id
id clawdee
getent passwd clawdee
test -d /home/clawdee && ls -ld /home/clawdee
test ! -e /etc/sudoers.d/clawdee-agents && echo "OK: no clawdee sudoers"
sudo -u clawdee -i bash -lc 'which claude && claude --version'
sudo -u clawdee -i bash -lc 'ls -la ~/.claude ~/.claude-lab/clawdee/.claude'
systemctl is-enabled claude-gateway || true
systemctl is-active claude-gateway || true
systemctl status claude-gateway --no-pager || true
crontab -u clawdee -l || true
```

Ожидаемое состояние:

- `clawdee` exists.
- `/home/clawdee/.local/bin/claude` exists.
- `/home/clawdee/.claude/.credentials.json` exists after `claude login`.
- `/home/clawdee/.claude-lab/clawdee/.claude` exists.
- No `/etc/sudoers.d/clawdee-agents`.
- `claude-gateway` is enabled; it may be inactive until Telegram token and OAuth are ready.

Если OAuth отсутствует, остановись и попроси оператора выполнить login. Не пытайся обходить OAuth автоматически:

```bash
sudo -u clawdee -i bash -lc 'claude login'
```

---

## Phase 2: Verify File Memory

Installer уже должен был создать файловую память. Проверь её сам и восстанови только недостающие элементы. Не перезаписывай существующее содержимое.

Target workspace:

```text
/home/clawdee/.claude-lab/clawdee/.claude
```

Required structure:

```text
core/
  USER.md
  rules.md
  MEMORY.md
  LEARNINGS.md
  hot/
    recent.md
    handoff.md
  warm/
    decisions.md
skills/
logs/
```

Проверки:

```bash
WS=/home/clawdee/.claude-lab/clawdee/.claude
sudo -u clawdee test -f "$WS/CLAUDE.md"
sudo -u clawdee test -f "$WS/core/MEMORY.md"
sudo -u clawdee test -f "$WS/core/hot/recent.md"
sudo -u clawdee test -f "$WS/core/hot/handoff.md"
sudo -u clawdee test -f "$WS/core/warm/decisions.md"
```

Если файл отсутствует, создай только этот файл с минимальным заголовком:

```bash
sudo -u clawdee bash -lc 'mkdir -p ~/.claude-lab/clawdee/.claude/core/hot ~/.claude-lab/clawdee/.claude/core/warm ~/.claude-lab/clawdee/.claude/logs'
sudo -u clawdee bash -lc 'test -f ~/.claude-lab/clawdee/.claude/core/MEMORY.md || printf "# MEMORY.md\n\nLong-term notes.\n" > ~/.claude-lab/clawdee/.claude/core/MEMORY.md'
sudo -u clawdee bash -lc 'test -f ~/.claude-lab/clawdee/.claude/core/hot/recent.md || printf "# recent.md -- full journal (NOT in @include)\n" > ~/.claude-lab/clawdee/.claude/core/hot/recent.md'
sudo -u clawdee bash -lc 'test -f ~/.claude-lab/clawdee/.claude/core/hot/handoff.md || printf "# handoff.md -- last 10 entries (@include)\n" > ~/.claude-lab/clawdee/.claude/core/hot/handoff.md'
sudo -u clawdee bash -lc 'test -f ~/.claude-lab/clawdee/.claude/core/warm/decisions.md || printf "# decisions.md -- last 14 days of decisions (@include)\n" > ~/.claude-lab/clawdee/.claude/core/warm/decisions.md'
```

Проверь memory cron:

```bash
crontab -u clawdee -l | sed -n '/clawdee-install v.*memory rotation/,/clawdee-install memory rotation end/p'
```

Ожидаемые scripts:

```text
/home/clawdee/.claude-lab/clawdee/scripts/rotate-warm.sh
/home/clawdee/.claude-lab/clawdee/scripts/trim-hot.sh
/home/clawdee/.claude-lab/clawdee/scripts/compress-warm.sh
/home/clawdee/.claude-lab/clawdee/scripts/ov-session-sync.sh
/home/clawdee/.claude-lab/clawdee/scripts/memory-rotate.sh
```

Запусти безопасную operational-проверку:

```bash
sudo -u clawdee bash -lc '~/.claude-lab/clawdee/scripts/trim-hot.sh'
sudo -u clawdee bash -lc '~/.claude-lab/clawdee/scripts/compress-warm.sh'
sudo -u clawdee bash -lc '~/.claude-lab/clawdee/scripts/rotate-warm.sh'
sudo -u clawdee bash -lc '~/.claude-lab/clawdee/scripts/memory-rotate.sh'
sudo -u clawdee bash -lc 'tail -50 ~/.claude-lab/clawdee/logs/memory-cron.log'
```

Не считай фазу готовой, пока memory scripts не выполняются без fatal errors.

---

## Phase 3: Choose OpenViking Mode

Используй выбор оператора из начального блока. Если оператор выбрал `no`, пропусти OpenViking целиком и переходи к финальной проверке файловой памяти.

### Option A: Connect to an existing OpenViking server

Используй этот путь, если у оператора уже есть работающий OpenViking server.

Настрой только:

- `/home/clawdee/claude-gateway/config.json`
- `/home/clawdee/.claude-lab/clawdee/.env`
- `/home/clawdee/.claude-lab/clawdee/secrets/openviking.key`

### Option B: Install local OpenViking on this VPS

Используй этот путь для self-contained VPS setup.

Ориентируйся на текущие OpenViking docs:

- `pip install openviking --upgrade --force-reinstall`
- config file at `~/.openviking/ov.conf`
- `openviking-server --port 1933`
- `GET /health` to verify the server

Reference:

- https://github.com/volcengine/OpenViking/blob/main/docs/en/getting-started/02-quickstart.md
- https://github.com/volcengine/OpenViking/blob/main/docs/en/getting-started/03-quickstart-server.md
- https://github.com/volcengine/OpenViking/blob/main/docs/en/guides/01-configuration.md

Установи через dedicated system user и isolated venv:

```bash
useradd --system --create-home --home-dir /opt/openviking --shell /usr/sbin/nologin openviking 2>/dev/null || true
install -d -m 0750 -o openviking -g openviking /opt/openviking
python3 -m venv /opt/openviking/venv
/opt/openviking/venv/bin/pip install --upgrade pip
/opt/openviking/venv/bin/pip install openviking --upgrade --force-reinstall
```

Если Python слишком старый, сообщи оператору и спроси разрешение на установку Python 3.12. Не меняй `/usr/bin/python3` глобально.

Подготовь config paths:

```bash
install -d -m 0700 -o openviking -g openviking /opt/openviking/.openviking /opt/openviking/data
```

Сгенерируй root API key для OpenViking и запроси model provider key скрытым prompt:

```bash
OPENVIKING_API_KEY="$(openssl rand -hex 32)"
read -r -s -p "Model provider API key for OpenViking: " MODEL_PROVIDER_API_KEY
printf '\n'
```

Создай `/opt/openviking/.openviking/ov.conf` с provider-ключом оператора. В JSON ниже замени `REPLACE_WITH_OPENVIKING_API_KEY` на `$OPENVIKING_API_KEY`, а `REPLACE_WITH_OPENAI_API_KEY` на `$MODEL_PROVIDER_API_KEY`. После записи config сразу выполни `unset MODEL_PROVIDER_API_KEY`.

OpenAI example:

```json
{
  "server": {
    "host": "127.0.0.1",
    "port": 1933,
    "auth_mode": "api_key",
    "root_api_key": "REPLACE_WITH_OPENVIKING_API_KEY",
    "cors_origins": ["*"]
  },
  "storage": {
    "workspace": "/opt/openviking/data",
    "vectordb": {
      "name": "context",
      "backend": "local"
    },
    "agfs": {
      "backend": "local"
    }
  },
  "embedding": {
    "dense": {
      "api_base": "https://api.openai.com/v1",
      "api_key": "REPLACE_WITH_OPENAI_API_KEY",
      "provider": "openai",
      "dimension": 1536,
      "model": "text-embedding-3-small"
    }
  },
  "vlm": {
    "api_base": "https://api.openai.com/v1",
    "api_key": "REPLACE_WITH_OPENAI_API_KEY",
    "provider": "openai",
    "model": "gpt-4o-mini"
  }
}
```

Важно:

- Если установленная версия OpenViking ожидает другой формат config, проверь `openviking-server init`, `openviking-server doctor`, `openviking-server --help` и адаптируй config.
- Не печатай реальные ключи в logs или final report.

Перед созданием unit сделай backup, если unit уже существует:

```bash
if test -f /etc/systemd/system/openviking.service; then
  cp -a /etc/systemd/system/openviking.service "/etc/systemd/system/openviking.service.bak.$(date +%Y%m%d-%H%M%S)"
fi
```

Создай systemd unit:

```ini
[Unit]
Description=OpenViking semantic memory server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=openviking
Group=openviking
WorkingDirectory=/opt/openviking
Environment=HOME=/opt/openviking
Environment=OPENVIKING_CONFIG_FILE=/opt/openviking/.openviking/ov.conf
ExecStart=/opt/openviking/venv/bin/openviking-server --config /opt/openviking/.openviking/ov.conf
Restart=on-failure
RestartSec=5
UMask=0077
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/openviking
CapabilityBoundingSet=
RestrictRealtime=true
LockPersonality=true
SystemCallArchitectures=native

[Install]
WantedBy=multi-user.target
```

Сохрани как:

```text
/etc/systemd/system/openviking.service
```

Включи и проверь:

```bash
systemctl daemon-reload
systemctl enable --now openviking
systemctl status openviking --no-pager
curl -fsS http://127.0.0.1:1933/health
```

Если `/health` не работает, изучи logs:

```bash
journalctl -u openviking -n 100 --no-pager
```

Не продолжай к gateway, пока OpenViking health не OK.

Для следующей фазы установи:

```bash
OPENVIKING_URL=http://127.0.0.1:1933
OPENVIKING_ACCOUNT="${OPENVIKING_ACCOUNT:-clawdee}"
```

---

## Phase 4: Wire CLAWDEE Gateway to OpenViking

Выполняй этот раздел только если оператор выбрал `yes-local` или `yes-external`.

Установи runtime variables на основе выбранного режима:

```bash
GW_CONFIG=/home/clawdee/claude-gateway/config.json
OPENVIKING_KEY_FILE=/home/clawdee/.claude-lab/clawdee/secrets/openviking.key
# For external mode, set this from the operator's answer.
# For local mode, it should already be http://127.0.0.1:1933.
: "${OPENVIKING_URL:?Set OPENVIKING_URL before wiring gateway}"
OPENVIKING_ACCOUNT="${OPENVIKING_ACCOUNT:-clawdee}"
```

Gateway config должен быть здесь:

```text
/home/clawdee/claude-gateway/config.json
```

Проверь, что config валидный JSON:

```bash
python3 -m json.tool "$GW_CONFIG" >/dev/null
```

Создай backup:

```bash
cp -a "$GW_CONFIG" "$GW_CONFIG.bak.$(date +%Y%m%d-%H%M%S)"
```

Нужные поля под `agents.clawdee`:

```json
{
  "openviking_url": "http://127.0.0.1:1933",
  "openviking_key_file": "/home/clawdee/.claude-lab/clawdee/secrets/openviking.key",
  "openviking_account": "clawdee"
}
```

Внеси поля через JSON-aware edit. Не используй fragile `sed`.

Если есть `jq`:

```bash
tmp="$(mktemp)"
jq --arg url "$OPENVIKING_URL" \
   --arg keyfile "$OPENVIKING_KEY_FILE" \
   --arg account "$OPENVIKING_ACCOUNT" \
   '.agents.clawdee.openviking_url = $url
    | .agents.clawdee.openviking_key_file = $keyfile
    | .agents.clawdee.openviking_account = $account' \
   "$GW_CONFIG" > "$tmp"
python3 -m json.tool "$tmp" >/dev/null
install -m 0640 -o clawdee -g clawdee "$tmp" "$GW_CONFIG"
rm -f "$tmp"
```

Если `jq` нет:

```bash
python3 - "$GW_CONFIG" "$OPENVIKING_URL" "$OPENVIKING_KEY_FILE" "$OPENVIKING_ACCOUNT" <<'PY'
import json
import sys
from pathlib import Path

path = Path(sys.argv[1])
url, keyfile, account = sys.argv[2:5]
data = json.loads(path.read_text())
agents = data.setdefault("agents", {})
agent = agents.setdefault("clawdee", {})
agent["openviking_url"] = url
agent["openviking_key_file"] = keyfile
agent["openviking_account"] = account
path.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
PY
chown clawdee:clawdee "$GW_CONFIG"
chmod 0640 "$GW_CONFIG"
```

Создай secret file:

```bash
install -d -m 0700 -o clawdee -g clawdee /home/clawdee/.claude-lab/clawdee/secrets
install -m 0600 -o clawdee -g clawdee /dev/null /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
```

Запиши OpenViking API key в:

```text
/home/clawdee/.claude-lab/clawdee/secrets/openviking.key
```

Если это external OpenViking, попроси key через скрытый prompt:

```bash
read -r -s -p "OpenViking API key: " OV_KEY_INPUT
printf '\n'
printf '%s\n' "$OV_KEY_INPUT" > /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
unset OV_KEY_INPUT
chown clawdee:clawdee /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
chmod 0600 /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
```

Если это local OpenViking, запиши сгенерированный тобой `OPENVIKING_API_KEY`:

```bash
printf '%s\n' "$OPENVIKING_API_KEY" > /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
chown clawdee:clawdee /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
chmod 0600 /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
```

Настрой cron sync env:

```text
/home/clawdee/.claude-lab/clawdee/.env
```

Сохрани или обнови эти значения, не уничтожая другие env-переменные:

```dotenv
OPENVIKING_URL=http://127.0.0.1:1933
OPENVIKING_ACCOUNT=clawdee
OPENVIKING_KEY=REPLACE_WITH_OPENVIKING_API_KEY
```

Для безопасного merge используй Python:

```bash
ENV_FILE=/home/clawdee/.claude-lab/clawdee/.env
cp -a "$ENV_FILE" "$ENV_FILE.bak.$(date +%Y%m%d-%H%M%S)" 2>/dev/null || true
OV_KEY="$(cat /home/clawdee/.claude-lab/clawdee/secrets/openviking.key)"
python3 - "$ENV_FILE" "$OPENVIKING_URL" "$OPENVIKING_ACCOUNT" "$OV_KEY" <<'PY'
import sys
from pathlib import Path

path = Path(sys.argv[1])
url, account, key = sys.argv[2:5]
updates = {
    "OPENVIKING_URL": url,
    "OPENVIKING_ACCOUNT": account,
    "OPENVIKING_KEY": key,
}
lines = []
seen = set()
if path.exists():
    for line in path.read_text().splitlines():
        if "=" in line and not line.lstrip().startswith("#"):
            name = line.split("=", 1)[0]
            if name in updates:
                lines.append(f"{name}={updates[name]}")
                seen.add(name)
                continue
        lines.append(line)
for name, value in updates.items():
    if name not in seen:
        lines.append(f"{name}={value}")
path.write_text("\n".join(lines).rstrip() + "\n")
PY
chown clawdee:clawdee "$ENV_FILE" /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
chmod 0600 "$ENV_FILE" /home/clawdee/.claude-lab/clawdee/secrets/openviking.key
unset OV_KEY
```

Перезапусти gateway:

```bash
systemctl restart claude-gateway
systemctl status claude-gateway --no-pager
journalctl -u claude-gateway -n 80 --no-pager
```

---

## Phase 5: Test OpenViking Writes

Выполняй этот раздел только если OpenViking был включён.

### Test direct OpenViking health

```bash
curl -fsS "$OPENVIKING_URL/health"
```

### Test gateway memory push

Попроси оператора отправить Telegram DM агенту CLAWDEE:

```text
Запомни: тестовая настройка памяти CLAWDEE, кодовое слово pineapple-memory-test.
```

После сообщения сам проверь gateway logs:

```bash
journalctl -u claude-gateway -n 120 --no-pager | grep -Ei 'ov|openviking|extracted|memory|warning|error' || true
```

Ожидаемо:

- нет fatal gateway errors
- в идеале есть строка вроде `extracted N memories`

### Test cron OpenViking sync

Запусти сам:

```bash
sudo -u clawdee bash -lc '~/.claude-lab/clawdee/scripts/ov-session-sync.sh'
sudo -u clawdee bash -lc 'tail -50 ~/.claude-lab/clawdee/logs/memory-cron.log'
```

Если script падает, потому что `/api/v1/capture` не поддерживается установленной версией OpenViking, не скрывай это. Зафиксируй:

```text
Gateway OpenViking push works/does not work.
Cron ov-session-sync endpoint is compatible/not compatible with this OpenViking version.
```

Если нужна правка `ov-session-sync.sh`, предложи вынести её в отдельную code session, а не смешивать с настройкой VPS.

### Test search

Используй настроенный API key:

```bash
OV_KEY=$(cat /home/clawdee/.claude-lab/clawdee/secrets/openviking.key)
curl -fsS -X POST "$OPENVIKING_URL/api/v1/search/find" \
  -H "X-API-Key: ${OV_KEY}" \
  -H "X-OpenViking-Account: ${OPENVIKING_ACCOUNT}" \
  -H "X-OpenViking-User: clawdee" \
  -H "Content-Type: application/json" \
  -d '{"query": "pineapple-memory-test", "limit": 10}'
```

Если текущая версия OpenViking ожидает другой search endpoint, проверь docs/help, адаптируй команду и в финальном отчёте назови endpoint, который реально сработал.

---

## Phase 6: Final Verification Checklist

Не утверждай, что готово, пока все релевантные проверки не выполнены:

- `id clawdee` works.
- `/etc/sudoers.d/clawdee-agents` does not exist.
- `sudo -u clawdee -i bash -lc 'claude --version'` works.
- `/home/clawdee/.claude/.credentials.json` exists.
- `/home/clawdee/claude-gateway/config.json` is valid JSON.
- `systemctl status claude-gateway --no-pager` is healthy or clearly waiting for OAuth/token.
- File memory exists:
  - `core/hot/recent.md`
  - `core/hot/handoff.md`
  - `core/warm/decisions.md`
  - `core/MEMORY.md`
- Memory cron block exists in `crontab -u clawdee -l`.
- Memory scripts run manually without fatal errors.
- Если OpenViking был включён:
  - OpenViking service health endpoint works.
  - OpenViking key is stored with `0600`.
  - gateway config has `openviking_url`, `openviking_key_file`, `openviking_account`.
  - a Telegram message produces either an OpenViking extraction log or a clear actionable error.
  - search test works or the reason it cannot work is documented.

Финальный ответ оператору:

```text
Готово.

Проверено:
- ...

Настроено:
- файловая память: да/нет
- cron памяти: да/нет
- OpenViking: local/external/not requested
- gateway -> OpenViking: да/нет

Что осталось вручную:
- ...
```

Держи финал коротким, чтобы его можно было использовать в видео.
