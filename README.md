# ABCENTRUM HTML email — Грекова, 5–7

Готовый проект HTML-письма для ABCENTRUM. Объект: медицинское здание на ул. Грекова, 5–7. Формат: продажа целиком, Shell & Core, целевой ввод — IV кв. 2027.

## Состав проекта

- `index.html` — готовое HTML-письмо для локального просмотра с относительными путями к изображениям.
- `email-cdn-template.html` — версия для рассылочного сервиса: замените `{{ASSET_BASE_URL}}` на HTTPS-папку с изображениями и `{{unsubscribe_url}}` на ссылку отписки.
- `plain-text.txt` — текстовая версия письма.
- `subject-preheader.txt` — тема письма и прехедер.
- `assets/` — оптимизированные изображения для письма.
- `docs/qa-checklist.md` — чек-лист приемки и тестирования.
- `CODEX_SEND_EMAIL_PROMPT.md` — инструкция для Codex по отправке рассылки.
- `.env.example` — шаблон переменных окружения для Mail.ru SMTP.

## Обязательные ссылки в письме

- Скачать презентацию: https://drive.google.com/file/d/1GBcdZ1wVF01tD5hMibftqpv3XyRkqc3R/view?usp=sharing
- Перейти на сайт: https://abcport.ru/
- Telegram: https://t.me/abcentrum_dev

## Перед отправкой

1. Загрузите файлы из `assets/` на HTTPS-хостинг или в рассылочный сервис.
2. В `email-cdn-template.html` замените `{{ASSET_BASE_URL}}` на абсолютный URL папки с изображениями. Рабочий вариант: `https://raw.githubusercontent.com/Zharov-s/Grekova/main/assets`.
3. Подставьте реальную ссылку отписки вместо `{{unsubscribe_url}}`.
4. Отправьте тесты в Gmail, Apple Mail, Outlook, Mail.ru и Яндекс Почту.

## Техническая база верстки

Письмо сверстано в подходе профессиональной email-разработки: HTML5 doctype, `lang="ru"`, табличная структура, inline CSS, media queries для мобильной адаптации, alt-тексты изображений, фиксированный контейнер 640 px с адаптацией на мобильных экранах.

## Отправка через Codex

Для запуска рассылки используйте инструкцию `CODEX_SEND_EMAIL_PROMPT.md`. В проекте зафиксированы правила: отправка идет с `s.zharov@abcentrum.ru`, `To` всегда только `s.zharov@abcentrum.ru`, все получатели рассылки идут только в `Bcc`, `Cc` не используется.

Секреты не храните в репозитории. Создайте `.env` на основе `.env.example` или передайте переменные окружения Codex. Реальный `SMTP_PASS` не должен попадать в markdown, HTML, JS, JSON, логи или git.
