# LLM EdgeBox своими руками: приватный ИИ без облака

[English version](README.md)

Что получится: на вашем компьютере или сервере работает **локальная языковая модель** с привычным **чат-интерфейсом в браузере**, **поиском по вашим документам** и **API, совместимым с OpenAI**. Запросы и документы никуда не уходят.

Компоненты (всё бесплатное и с открытым кодом):
- **[Ollama](https://ollama.com)** — запускает модели;
- **[Open WebUI](https://openwebui.com)** — веб-чат, учётные записи, база документов.

> Хотите готовый сервер, настроенный под ваши документы и задачи, с поддержкой? Это [llm.cakescats.com](https://llm.cakescats.com/ru/). Покупая его, вы поддерживаете наш некоммерческий проект.

---

## 1. Какое оборудование нужно

Главное — сколько памяти доступно модели: видеопамять (GPU) или общая память (Apple silicon).

| Память | Какие модели | Для чего |
|---|---|---|
| 8–16 ГБ | 7–14 млрд параметров | Личный ассистент, черновики, переводы |
| 24–32 ГБ | до ~32 млрд | Работа с документами, команда из нескольких человек |
| 64–128 ГБ (например, Mac Studio или NVIDIA DGX Spark) | MoE-модели до ~120 млрд | Сильный ассистент для команды до 20 человек |
| 16 ГБ GPU + 30 ГБ RAM + быстрый NVMe | MoE ~125 млрд, медленно (~10 токенов/с) | Наш форк [QwFNfer Secure Multilang](https://github.com/cakescats/QwFNfer-Secure-Multilang) |

Операционная система: Linux (Ubuntu 24.04+), macOS на Apple silicon или Windows. Ниже — команды для Ubuntu; для macOS и Windows Ollama ставится обычным установщиком с [ollama.com/download](https://ollama.com/download), а Open WebUI — через Docker Desktop.

## 2. Ollama и модель

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama --version
```

Скачайте модель. Список — на [ollama.com/library](https://ollama.com/library). Выбирайте по размеру памяти из таблицы выше, например:

```bash
ollama pull qwen3        # семейство Qwen: хорошо понимает русский
ollama run qwen3 "Привет! Кратко: что ты умеешь?"
```

## 3. Open WebUI (чат в браузере)

Установите Docker:

```bash
curl -fsSL https://get.docker.com | sh
```

Запустите Open WebUI:

```bash
sudo docker run -d --network=host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

Откройте **http://IP_КОМПЬЮТЕРА:8080**. Первый созданный пользователь становится администратором.

Настройки администратора (**Admin Panel → Settings**):
- **General → Default User Role**: *pending* — новые пользователи ждут вашего одобрения;
- **General → Enable New Sign Ups**: выключите, когда все сотрудники зарегистрировались;
- **Models**: выберите модель по умолчанию.

## 4. Полностью офлайн

После того как модели скачаны, интернет серверу не нужен.

1. Перезапустите Open WebUI в офлайн-режиме:

```bash
sudo docker rm -f open-webui
sudo docker run -d --network=host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  -e OFFLINE_MODE=true -e HF_HUB_OFFLINE=1 \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

2. Закройте серверу выход в интернет на роутере или брандмауэре, оставив доступ только из локальной сети. Пример для `ufw` на самом сервере, если ваша сеть `192.168.1.0/24`:

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from 192.168.1.0/24
sudo ufw allow out to 192.168.1.0/24
sudo ufw enable
```

Чтобы обновить модели позже, временно откройте исходящий трафик (`sudo ufw default allow outgoing`), выполните `ollama pull …` и закройте снова.

## 5. Работа с вашими документами (RAG)

1. **Workspace → Knowledge → +** — создайте базу, например «Регламенты».
2. Загрузите файлы: PDF, DOCX, TXT, Markdown.
3. В чате напишите `#` и выберите базу — модель будет отвечать по вашим документам и показывать, откуда взяла ответ.

Ваш стиль и терминология: **Workspace → Models → +** — создайте свою модель на основе базовой, задайте системный промпт («Отвечай в деловом стиле, используй термины компании…») и привяжите базу знаний.

## 6. API для программ и ИИ-агентов

Ollama отдаёт API, совместимый с OpenAI: `http://IP_СЕРВЕРА:11434/v1`.

По умолчанию Ollama слушает только `127.0.0.1`. Чтобы открыть API в локальной сети:

```bash
sudo systemctl edit ollama
# добавьте:
# [Service]
# Environment="OLLAMA_HOST=0.0.0.0"
sudo systemctl restart ollama
```

Проверка с другого компьютера:

```bash
curl http://IP_СЕРВЕРА:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"Привет"}]}'
```

> У API Ollama нет авторизации. Открывайте его только в доверенной сети. Для доступа снаружи используйте Open WebUI: у него есть учётные записи и API-ключи (**Settings → Account → API keys**).

## 7. HTTPS в локальной сети (по желанию)

Чтобы браузер не предупреждал о незащищённом соединении, поставьте перед Open WebUI [Caddy](https://caddyserver.com) с внутренним сертификатом:

```bash
sudo apt install -y caddy
echo 'https://ai.local {
  tls internal
  reverse_proxy 127.0.0.1:8080
}' | sudo tee /etc/caddy/Caddyfile
sudo systemctl restart caddy
```

Добавьте `ai.local` в DNS роутера (или в файлы hosts) и один раз установите корневой сертификат Caddy на компьютеры сотрудников.

## Если что-то не работает

| Проблема | Решение |
|---|---|
| Open WebUI не видит модели | `curl http://127.0.0.1:11434/api/tags` должен вернуть список моделей; проверьте `OLLAMA_BASE_URL` |
| Модель отвечает очень медленно | Модель слишком большая для вашей памяти — возьмите меньшую или квантованную версию |
| Не хватает видеопамяти | `ollama ps` покажет, сколько модели работает на GPU; закройте другие программы |

Вопросы — в Telegram [@cakescats](https://t.me/cakescats) или на [mail@cakescats.com](mailto:mail@cakescats.com).
