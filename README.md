# Как поставить своего AI-ассистента (OpenClaw) — гайд с нуля

> **🤖 Для Claude Code:** Этот файл — пошаговая инструкция по установке OpenClaw на VPS. Ты можешь выполнить её автоматически. Попроси пользователя предоставить: (1) SSH-доступ к серверу (IP + ключ или пароль), (2) Anthropic API key, (3) Telegram bot token и Telegram ID. Затем выполняй шаги 2-4 последовательно через SSH. Шаги 0 и 1 пользователь должен сделать сам (регистрации и аккаунты).

---

## Быстрый путь (если у тебя есть Claude Code)

Если у тебя уже установлен Claude Code и арендован VPS:

1. Открой Claude Code: `claude`
2. Скажи ему:

```
Прочитай этот гайд: https://github.com/kalichkin/openclaw-setup-guide-ru

Подключись к моему серверу по SSH (root@МОЙ_IP) и выполни шаги 2-4.
Вот мои данные:
- Anthropic API key: sk-ant-...
- Telegram bot token: 71234...:AAH...
- Мой Telegram ID: 123456789
```

3. Claude Code сделает всё остальное. Если что-то пойдет не так, он покажет ошибку и предложит решение.

**Если Claude Code нет — читай полную инструкцию ниже.**

---

Этот гайд проведет тебя от полного нуля до работающего AI-ассистента, который доступен 24/7 через Telegram с телефона.

**Что получишь в итоге:** AI-ассистент (Claude), который живет на твоем сервере и доступен через Telegram-бота. Можно писать ему с телефона из любой точки мира, он умеет работать с файлами, запускать код, выполнять задачи по расписанию и многое другое.

**Время:** ~45-60 минут вручную (с Claude Code — ~15 минут)
**Стоимость:** ~$5/мес за сервер + ~$10-30/мес за API модели
**Уровень:** Копировать команды в терминал — достаточно

---

## Как это работает (общая картина)

```
  Ты                               Твой сервер (VPS)
┌────────┐                        ┌──────────────────────┐
│📱      │── Telegram-сообщение ──>│ OpenClaw Gateway     │
│Телефон │                        │   │                  │
│        │<── ответ ──────────────│   ↓                  │
└────────┘                        │ Claude (Anthropic AI) │
                                  └──────────────────────┘
```

OpenClaw — это open-source мост между мессенджерами и AI. Работает на твоем сервере, данные никуда не утекают.

---

## Шаг 0. Что тебе понадобится заранее

Прежде чем начинать, подготовь:

### 0.1. Anthropic API ключ

Это ключ для доступа к Claude (AI-модели). Без него OpenClaw не сможет "думать".

1. Зайди на [console.anthropic.com](https://console.anthropic.com)
2. Зарегистрируйся (или войди)
3. Привяжи карту для оплаты (Settings → Billing)
4. Зайди в Settings → API Keys → Create Key
5. Скопируй ключ (начинается с `sk-ant-...`) — **сохрани его, он показывается один раз!**

Стоимость: платишь за использование. Для персонального ассистента обычно $10-30/мес.

### 0.2. Telegram бот

1. Открой Telegram, найди [@BotFather](https://t.me/BotFather)
2. Напиши ему `/newbot`
3. Придумай имя бота (например: "My AI Assistant")
4. Придумай username бота (например: `my_ai_assistant_bot`) — должен заканчиваться на `bot`
5. BotFather выдаст тебе **токен** — длинная строка вида `7123456789:AAH...`. Сохрани его
6. Чтобы бот мог читать сообщения в группах, напиши BotFather: `/setprivacy`, выбери своего бота, выбери `Disable`

### 0.3. Узнай свой Telegram ID

1. Открой Telegram, найди бота [@userinfobot](https://t.me/userinfobot)
2. Напиши ему что угодно
3. Он ответит твоим числовым ID (например: `123456789`). Сохрани его

Этот ID нужен для whitelist — чтобы только ты мог писать боту.

### 0.4. Терминал на компьютере

- **macOS:** Spotlight → "Terminal" (или iTerm2, если установлен)
- **Windows:** Поставь [Windows Terminal](https://apps.microsoft.com/detail/9N0DX20HK701) из Microsoft Store. Затем установи WSL2: открой PowerShell от админа и запусти `wsl --install`. После перезагрузки у тебя будет Linux-терминал
- **Linux:** Ты и так знаешь :)

---

## Шаг 1. Арендовать сервер на Hetzner

### 1.1. Создать аккаунт

1. Зайди на [hetzner.com/cloud](https://www.hetzner.com/cloud/)
2. Нажми "Sign Up" (или "Get Started")
3. Заполни регистрацию, подтверди email
4. Привяжи карту (или PayPal)
5. Возможно потребуется верификация (паспорт/ID) — Hetzner серьезно относится к KYC. Обычно проходит за несколько часов

### 1.2. Создать SSH-ключ (если нет)

SSH-ключ — это безопасный способ подключаться к серверу без пароля. Один раз настраиваешь — и забываешь.

**Проверь, есть ли у тебя ключ:**

```bash
ls ~/.ssh/id_ed25519.pub
```

Если файл существует — у тебя уже есть ключ, пропусти создание.

**Если ключа нет — создай:**

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

Жми Enter на все вопросы (пустой пароль — ок для начала).

**Скопируй публичный ключ:**

```bash
cat ~/.ssh/id_ed25519.pub
```

Скопируй всю строку, которая начинается с `ssh-ed25519 ...`

### 1.3. Создать сервер

1. В Hetzner Cloud Console нажми **"Create Server"**
2. **Location:** Выбери ближайший регион (Falkenstein или Helsinki для EU; Ashburn для US)
3. **Image:** Ubuntu 24.04
4. **Type:** Shared vCPU → **CX22** (2 vCPU, 4GB RAM) — ~€4.50/мес (~$5). Хватит с запасом
5. **SSH Key:** Нажми "Add SSH Key", вставь свой публичный ключ (из шага 1.2)
6. **Name:** Дай серверу имя (например `ai-assistant`)
7. Нажми **"Create & Buy Now"**

Через 30 секунд сервер будет готов. Запиши его **IP-адрес** (виден в панели).

### 1.4. Подключиться к серверу

```bash
ssh root@ТВОЙ_IP_АДРЕС
```

Если спросит "Are you sure you want to continue connecting?" — напиши `yes`.

**Ты внутри.** Должен появиться prompt типа `root@ai-assistant:~#`

---

## Шаг 2. Установить OpenClaw на сервер

Теперь ты подключен к серверу по SSH. Все команды ниже выполняются на сервере.

### 2.1. Обновить систему

```bash
apt update && apt upgrade -y
```

Подожди пару минут, пока всё обновится.

### 2.2. Установить OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Скрипт сам установит Node.js (если нет) и OpenClaw. Займет 2-3 минуты.

### 2.3. Запустить онбординг

```bash
openclaw onboard --install-daemon
```

Визард спросит несколько вещей:
- **API provider:** выбери Anthropic
- **API key:** вставь свой ключ из Шага 0.1
- **Install as service:** Yes (чтобы работал 24/7)
- Остальные настройки можно оставить по умолчанию

### 2.4. Проверить, что работает

```bash
openclaw gateway status
```

Должно показать, что gateway запущен и слушает.

---

## Шаг 3. Подключить Telegram

### 3.1. Настроить конфиг

Открой конфиг-файл:

```bash
nano ~/.openclaw/openclaw.json
```

Найди (или добавь) секцию `channels` и добавь Telegram:

```json
{
  "channels": {
    "telegram": {
      "token": "ВСТАВЬ_ТОКЕН_ОТ_BOTFATHER",
      "allowFrom": ["ВСТАВЬ_СВОЙ_TELEGRAM_ID"]
    }
  }
}
```

**Важно:**
- `token` — строка от BotFather из Шага 0.2
- `allowFrom` — массив строк (!) с твоим Telegram ID из Шага 0.3

Сохрани файл: `Ctrl+O`, Enter, `Ctrl+X`

### 3.2. Перезапустить Gateway

```bash
openclaw gateway restart
```

### 3.3. Проверить

Открой Telegram, найди своего бота и напиши ему что-нибудь. Если всё настроено правильно, он ответит через Claude!

---

## Шаг 4. Базовая безопасность

Несколько настроек, чтобы сервер был защищен:

### 4.1. API ключи в .env (не в конфиге)

```bash
nano ~/.openclaw/.env
```

Добавь:

```
ANTHROPIC_API_KEY=sk-ant-ваш-ключ-здесь
```

И убери ключ из `openclaw.json` (OpenClaw подхватит из `.env` автоматически).

```bash
chmod 600 ~/.openclaw/.env
```

### 4.2. Firewall

```bash
apt install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw enable
```

Это закроет все входящие порты кроме SSH. Gateway слушает на localhost по умолчанию — он и так недоступен снаружи.

### 4.3. Автоматические обновления безопасности

```bash
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

Выбери "Yes" — система будет ставить security-патчи автоматически.

---

## Готово!

Теперь у тебя есть:
- ✅ AI-ассистент, работающий 24/7
- ✅ Доступ через Telegram с телефона
- ✅ Данные на твоем сервере (не на чужих)
- ✅ Базовая безопасность настроена

### Что дальше (опционально)

- **Веб-интерфейс:** С ноутбука запусти `ssh -L 18789:127.0.0.1:18789 root@ТВОЙ_IP` и открой `http://127.0.0.1:18789/` в браузере
- **WhatsApp:** `openclaw channels login` на сервере и отсканируй QR
- **Discord:** Добавь Discord bot token в конфиг
- **Кастомизация:** Настрой workspace, агентов, скиллы — [docs.openclaw.ai](https://docs.openclaw.ai)
- **Аудит безопасности:** `openclaw security audit` покажет, что можно улучшить

---

## Про безопасность OpenClaw (для скептиков)

Валидный вопрос, если ставишь open-source проект с доступом к shell.

**Факты:**
- Код полностью открыт: [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) (MIT лицензия)
- Нет телеметрии, нет "phone home", нет облачной зависимости
- Данные живут только на твоем VPS — единственный внешний трафик это API-вызовы к модели (Anthropic/OpenAI), которые ты бы делал в любом случае
- Формальная Threat Model по MITRE ATLAS фреймворку
- Встроенный security audit: `openclaw security audit --deep`
- Активное community (Discord) + security-контакт (trust.openclaw.ai)
- Модель безопасности: один пользователь = один сервер. Не multi-tenant

**По сравнению с "конвенциональными" продуктами:**
- Claude.ai / ChatGPT: твои данные на их серверах, их правила, их retention policy
- OpenClaw: твои данные на твоем сервере, твои правила, ты контролируешь всё

**Выделенный VPS за $5/мес — это и есть уровень безопасности.** Даже если в OpenClaw найдется баг, он затронет только изолированный сервер, на котором нет ничего кроме ассистента.

---

## Ссылки

| Что | Куда |
|-----|------|
| Документация | [docs.openclaw.ai](https://docs.openclaw.ai) |
| GitHub | [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) |
| Discord | [discord.com/invite/clawd](https://discord.com/invite/clawd) |
| Security | [trust.openclaw.ai](https://trust.openclaw.ai) |
| Гайды по VPS | [docs.openclaw.ai/vps](https://docs.openclaw.ai/vps) |
| Claude Code | [code.claude.com](https://code.claude.com) |

---

## Troubleshooting

**"openclaw: command not found"**
```bash
export PATH="$(npm prefix -g)/bin:$PATH"
echo 'export PATH="$(npm prefix -g)/bin:$PATH"' >> ~/.bashrc
```

**Gateway не стартует**
```bash
openclaw doctor          # диагностика
journalctl -u openclaw-gateway -n 50   # логи
```

**Бот не отвечает в Telegram**
- Проверь, что токен правильный
- Проверь, что твой ID в `allowFrom` (как строка!)
- Проверь `openclaw gateway status`
- Посмотри логи: `journalctl -u openclaw-gateway -f`

**Hetzner не верифицирует аккаунт**
Попробуй DigitalOcean ($6/мес) или Oracle Cloud (бесплатно). Установка аналогична.
