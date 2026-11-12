# Лаба 4.5 — пошаговая инструкция (только команды)

Каждый шаг — команда в терминал. Выполняй по порядку **из папки `лаба-4.5-starrocks-agent/`**.
Всё, что ниже, поднимается и удаляется прямо здесь — других лаб делать не нужно.

> **Своя область** (подставляй вместо `oil-gas` во всех командах `--domain`; в именах таблиц дефис меняется на `_`):
> 1 = Нефтегаз → `oil-gas` (таблицы `*_oil_gas`); 2 = Металлургия → `metallurgy`;
> 3 = Химтех → `chemical`; 4 = ИВТ → `cs`.

---

## 0. Перед стартом

```bash
# Docker Desktop должен быть запущен. Проверь, что порты свободны (пустой вывод = ок):
lsof -nP -iTCP:9030 -iTCP:6333 -iTCP:11434 -sTCP:LISTEN
```

## 1. StarRocks (база данных)

```bash
docker run -d --name starrocks -p 9030:9030 -p 8030:8030 -p 8040:8040 starrocks/allin1-ubuntu:3.3.11
```

Подожди 30–60 с (пока FE/BE поднимутся), создай базу `dw`:

```bash
docker exec starrocks mysql -P 9030 -h 127.0.0.1 -u root -e "CREATE DATABASE IF NOT EXISTS dw; SHOW DATABASES;"
```

Должно вывести `dw`. (Интерактивно: `docker exec -it starrocks mysql -P 9030 -h 127.0.0.1 -u root`.)

## 2. python-контейнер + зависимости

```bash
docker run -d --name sr-py \
  --add-host host.docker.internal:host-gateway \
  -v "$(pwd):/app" \
  -v hf-cache:/root/.cache/huggingface \
  python:3.12-slim sleep infinity
```

Установи зависимости (сначала CPU-сборку torch, чтобы не качать CUDA ~2 ГБ):

```bash
docker exec sr-py sh -c "pip install -q torch --index-url https://download.pytorch.org/whl/cpu"
docker exec sr-py sh -c "pip install -q -r /app/requirements.txt"
```

Проверка (должно напечатать `ok`):

```bash
docker exec sr-py python -c "import pymysql, qdrant_client, sentence_transformers, mcp, requests; print('ok')"
```

## 3. Qdrant + Ollama (RAG-часть)

```bash
docker run -d --name qdrant -p 6333:6333 qdrant/qdrant:latest
docker run -d --name ollama -p 11434:11434 -v ollama-models:/root/.ollama ollama/ollama:latest
```

> Совет: модель лучше качать заранее, пока делаешь шаги 1–2 (в отдельном терминале). Если качать в этот момент — команда ниже заблокирует терминал на несколько минут (модель ~1 ГБ):

```bash
docker exec ollama ollama pull qwen2.5:1.5b
```

Проверка (в конце должно быть `success`; и далее):

```bash
docker exec ollama ollama list          # в списке должен быть qwen2.5:1.5b
curl -s http://localhost:6333/collections   # Qdrant отвечает JSON
```

> ⚠️ Не запускай pull через `docker exec -d …` (в фоне внутри контейнера) — такой процесс может тихо оборваться, и модель не появится. Используй обычный (foreground) pull.

## 4. STG — сырые данные (своя область)

```bash
docker exec sr-py python /app/loader.py --domain oil-gas
```

Сверь число строк (таблицы с суффиксом `_oil_gas`):

```bash
docker exec starrocks mysql -P 9030 -h 127.0.0.1 -u root -e \
  "SELECT count(*) FROM dw.stg_dim_oil_gas; SELECT count(*) FROM dw.stg_fact_oil_gas; SELECT count(*) FROM dw.stg_changes_oil_gas;"
```

## 5. ODS + SCD2 — история измерений

```bash
docker exec sr-py python /app/scd2.py --domain oil-gas
```

Проверь историю («как было на дату X» для W-01 — у изменённого объекта две версии):

```bash
docker exec starrocks mysql -P 9030 -h 127.0.0.1 -u root -e \
  "SELECT id, status, valid_from, valid_to, is_current FROM dw.ods_dim_oil_gas WHERE id='W-01' ORDER BY valid_from;"
```

## 6. DM — витрина-агрегат + исследование моделей

```bash
docker exec sr-py python /app/dm.py --domain oil-gas
docker exec sr-py python /app/research.py --domain oil-gas
```

`research.py` напечатает таблицу «модель → строки / время» (Duplicate/Aggregate/Primary) — запиши её в отчёт.

## 7. MCP-сервер — проверка инструмента `run_sql`

```bash
docker exec sr-py python -c "
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    async with stdio_client(StdioServerParameters(command='python', args=['/app/mcp_server.py'])) as (r, w):
        async with ClientSession(r, w) as s:
            await s.initialize()
            tools = await s.list_tools()
            print('инструменты:', [t.name for t in tools.tools])
            res = await s.call_tool('run_sql', arguments={'sql': 'SELECT count(*) FROM dw.stg_fact_oil_gas'})
            print('ответ run_sql:', res.content[0].text)

asyncio.run(main())
"
```

Должно напечатать `инструменты: ['run_sql']` и число. Read-only-защита (запрещённый запрос даст `ERROR`):

```bash
docker exec sr-py python -c "
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
async def main():
    async with stdio_client(StdioServerParameters(command='python', args=['/app/mcp_server.py'])) as (r, w):
        async with ClientSession(r, w) as s:
            await s.initialize()
            res = await s.call_tool('run_sql', arguments={'sql': 'DROP TABLE dw.stg_fact_oil_gas'})
            print(res.content[0].text)
asyncio.run(main())
"
```

## 8. Агент — эмбеддинги + 4 режима

Построй эмбеддинги документов своей области (первый запуск качает MiniLM ~470 МБ):

```bash
docker exec sr-py python /app/embed.py --domain oil-gas
```

Прогони один и тот же вопрос в 4 режимах (вопрос подставь под свою область):

```bash
docker exec sr-py python /app/agent.py --domain oil-gas --mode none "Сколько добыла скважина W-02 в январе 2026?"
docker exec sr-py python /app/agent.py --domain oil-gas --mode sql  "Сколько добыла скважина W-02 в январе 2026?"
docker exec sr-py python /app/agent.py --domain oil-gas --mode docs "Сколько добыла скважина W-02 в январе 2026?"
docker exec sr-py python /app/agent.py --domain oil-gas --mode both "Сколько добыла скважина W-02 в январе 2026?"
```

Сравни: `none` галлюцинирует, `sql` даёт цифру из БД, `docs` — методику из документов, `both` — и то и другое.

> Если `sql`-режим отдаёт странный SQL — это предел модели `qwen2.5:1.5b` (норма). Подскажи таблицу в вопросе (например, `… W-02 … (таблица stg_fact_oil_gas)`) или возьми `qwen2.5:3b`.

## 9. Заполни отчёт

Заполни `отчёт-исследование.xlsx` (листы «Отчёт», «Агент — 6 вопросов», «Заключение») и ответь на 6 вопросов заключения из `guide.md` §10.

## 10. Удалить всё (после сдачи)

```bash
docker rm -f sr-py starrocks qdrant ollama
docker volume rm ollama-models hf-cache
docker rmi starrocks/allin1-ubuntu:3.3.11 qdrant/qdrant:latest ollama/ollama:latest python:3.12-slim
cd .. && rm -rf лаба-4.5-starrocks-agent
```

> Volume/образы лабы 4 (`rag-4_*`) не трогай — они от другой лабы.
