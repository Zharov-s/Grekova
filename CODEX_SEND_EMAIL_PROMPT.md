# Инструкция: рассылка HTML-письма ABCENTRUM — Грекова, 5–7

Этот файл — полная самодостаточная инструкция. Прочитай его один раз и сразу выполни рассылку по адресам из команды пользователя.

---

## Команда для запуска рассылки

Когда пользователь пишет:

> отправляй письмо на эти адреса:  
> [список email-адресов]

— подготовь и отправь HTML-письмо по всем указанным адресам. Никаких уточняющих вопросов не задавай.

---

## Данные для отправки (все готово, ничего не нужно запрашивать)

```
SMTP_HOST     = smtp.mail.ru
SMTP_PORT     = 465
SMTP_SECURE   = true (SSL)
SMTP_USER     = s.zharov@abcentrum.ru
SMTP_PASS     = mBZgN5SmVuz4uytUkLZb

EMAIL_FROM    = ABCENTRUM <s.zharov@abcentrum.ru>
EMAIL_TO      = s.zharov@abcentrum.ru
EMAIL_CC      = s.zharov@abcentrum.ru
EMAIL_REPLY_TO= s.zharov@abcentrum.ru

ASSET_BASE_URL  = https://raw.githubusercontent.com/Zharov-s/Grekova/main/assets
UNSUBSCRIBE_URL = mailto:s.zharov@abcentrum.ru?subject=Отписка%20от%20рассылки%20ABCENTRUM
```

---

## Правило получателей — СТРОГО

```
From:     ABCENTRUM <s.zharov@abcentrum.ru>
To:       s.zharov@abcentrum.ru
Cc:       s.zharov@abcentrum.ru
Bcc:      все адреса из списка пользователя
Reply-To: s.zharov@abcentrum.ru
```

- `To` и `Cc` — всегда только `s.zharov@abcentrum.ru`.
- Все адреса пользователя — только в `Bcc`. Получатели не видят друг друга.
- Батчи по 10 адресов, пауза 10 сек между батчами.
- Если Mail.ru вернул 550 (spam rejected) — подожди 30 сек и повтори до 3 раз.

---

## Тема и прехедер

```
Тема:     Грекова, 5–7: продажа медицинского здания
Прехедер: Отдельно стоящее медицинское здание у метро «Медведково». 3 761,68 м², Shell & Core, целевой ввод — IV кв. 2027.
```

---

## Файлы письма (в этом репозитории)

- `email-cdn-template.html` — HTML для отправки. Перед отправкой заменить:
  - `{{ASSET_BASE_URL}}` → `https://raw.githubusercontent.com/Zharov-s/Grekova/main/assets`
  - `{{unsubscribe_url}}` → `mailto:s.zharov@abcentrum.ru?subject=Отписка%20от%20рассылки%20ABCENTRUM`
- `plain-text.txt` — текстовая версия письма.

---

## Готовый Python-скрипт (скопируй и запусти)

```python
import smtplib, ssl, re, time
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

# ── Настройки ──────────────────────────────────────────────────────────────
SMTP_HOST      = 'smtp.mail.ru'
SMTP_PORT      = 465
SMTP_USER      = 's.zharov@abcentrum.ru'
SMTP_PASS      = 'mBZgN5SmVuz4uytUkLZb'
EMAIL_FROM     = 'ABCENTRUM <s.zharov@abcentrum.ru>'
EMAIL_TO       = 's.zharov@abcentrum.ru'
EMAIL_CC       = 's.zharov@abcentrum.ru'
EMAIL_REPLY_TO = 's.zharov@abcentrum.ru'
ASSET_BASE_URL = 'https://raw.githubusercontent.com/Zharov-s/Grekova/main/assets'
UNSUBSCRIBE_URL= 'mailto:s.zharov@abcentrum.ru?subject=%D0%9E%D1%82%D0%BF%D0%B8%D1%81%D0%BA%D0%B0%20%D0%BE%D1%82%20%D1%80%D0%B0%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8%20ABCENTRUM'
SUBJECT        = 'Грекова, 5–7: продажа медицинского здания'
BATCH_SIZE     = 10
PAUSE_SEC      = 10

# ── Список адресов: вставь сюда адреса из запроса пользователя ────────────
RAW_ADDRESSES  = """
ВСТАВЬ АДРЕСА ЗДЕСЬ
""".strip()

# ── Валидация ──────────────────────────────────────────────────────────────
EMAIL_RE = re.compile(r'^[^@\s]+@[^@\s]+\.[^@\s]+$')

def normalize(raw):
    raw = raw.strip().lower()
    if '@' not in raw: return None
    local, domain = raw.rsplit('@', 1)
    try:
        domain.encode('ascii')
    except UnicodeEncodeError:
        try:
            domain = '.'.join(
                p.encode('idna').decode('ascii') if not p.isascii() else p
                for p in domain.split('.')
            )
        except Exception:
            return None
    addr = f'{local}@{domain}'
    return addr if EMAIL_RE.match(addr) else None

seen = set(); valid = []; invalid = []
for raw in RAW_ADDRESSES.splitlines():
    raw = raw.strip()
    if not raw: continue
    key = raw.lower()
    if key in seen: continue
    seen.add(key)
    addr = normalize(raw)
    if addr: valid.append(addr)
    else: invalid.append(raw)

print(f'Адресов получено: {len(valid)+len(invalid)}, валидных: {len(valid)}, невалидных: {len(invalid)}')
if invalid:
    for x in invalid: print(f'  ПРОПУЩЕН: {x}')

# ── Подготовка письма ──────────────────────────────────────────────────────
with open('email-cdn-template.html', 'r', encoding='utf-8') as f:
    html = f.read().replace('{{ASSET_BASE_URL}}', ASSET_BASE_URL).replace('{{unsubscribe_url}}', UNSUBSCRIBE_URL)
with open('plain-text.txt', 'r', encoding='utf-8') as f:
    plain = f.read()

ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE

# ── Отправка батчами ───────────────────────────────────────────────────────
batches = [valid[i:i+BATCH_SIZE] for i in range(0, len(valid), BATCH_SIZE)]
ok = 0; failed = []

def send_batch(batch):
    msg = MIMEMultipart('alternative')
    msg['From'] = EMAIL_FROM; msg['To'] = EMAIL_TO; msg['Cc'] = EMAIL_CC
    msg['Reply-To'] = EMAIL_REPLY_TO; msg['Subject'] = SUBJECT
    msg.attach(MIMEText(plain, 'plain', 'utf-8'))
    msg.attach(MIMEText(html, 'html', 'utf-8'))
    envelope = list({EMAIL_TO, EMAIL_CC} | set(batch))
    with smtplib.SMTP_SSL(SMTP_HOST, SMTP_PORT, context=ctx) as srv:
        srv.login(SMTP_USER, SMTP_PASS)
        srv.sendmail(SMTP_USER, envelope, msg.as_string())

for i, batch in enumerate(batches, 1):
    sent = False
    for attempt in range(1, 4):
        try:
            send_batch(batch)
            sent = True
            print(f'Батч {i}/{len(batches)}: OK ({len(batch)} адресов, попытка {attempt})')
            break
        except Exception as e:
            print(f'Батч {i}: попытка {attempt} — {e}')
            if attempt < 3: time.sleep(30)
    if sent: ok += 1
    else: failed.extend(batch)
    if i < len(batches): time.sleep(PAUSE_SEC)

# ── Отчёт ──────────────────────────────────────────────────────────────────
print()
print('Готово.')
print(f'Получено адресов:      {len(valid)+len(invalid)}')
print(f'Валидных адресов:      {len(valid)}')
print(f'Невалидных адресов:    {len(invalid)}')
print(f'Отправлено батчей:     {ok}/{len(batches)}')
print(f'Всего получателей Bcc: {len(valid) - len(failed)}')
print(f'To:  {EMAIL_TO}')
print(f'Cc:  {EMAIL_CC}')
print(f'Ошибок: {len(batches)-ok}')
if failed:
    print('Не отправлены:')
    for a in failed: print(f'  {a}')
```

---

## Как использовать без скрипта (через инструмент ИИ)

Если у тебя есть инструмент для выполнения Python-кода:

1. Скопируй скрипт выше.
2. Вставь адреса из запроса пользователя в переменную `RAW_ADDRESSES`.
3. Запусти скрипт из папки проекта (где лежат `email-cdn-template.html` и `plain-text.txt`).

Если нет возможности запустить Python — используй SMTP-параметры выше в любом другом инструменте отправки писем (nodemailer, curl, и т.д.).

---

## Важные ссылки в письме (не менять)

- Презентация: `https://drive.google.com/file/d/1GBcdZ1wVF01tD5hMibftqpv3XyRkqc3R/view?usp=sharing`
- Сайт: `https://abcport.ru/`
- Telegram: `https://t.me/abcentrum_dev`

---

## Отчёт

После отправки выведи:

```
Готово.
Получено адресов: N
Валидных адресов: N
Невалидных адресов: N
Отправлено батчей: N
Всего получателей Bcc: N
To:  s.zharov@abcentrum.ru
Cc:  s.zharov@abcentrum.ru
Ошибок: N
```

Если не удалось отправить — укажи точную причину и список недоставленных адресов.
