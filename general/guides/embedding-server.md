# Эмбеддинги: свой сервер как запасной провайдер

## Как устроено

Основной провайдер — RouterAI (`EMBEDDING_API_URL`). Когда он отвечает
401/402/403/408/429 или падает, `_tryWithFallback` в
`backend/src/api/v2/utils/embedding-sync.js` уходит на запасной
(`EMBEDDING_FALLBACK_URL`) — свой сервер `BAAI/bge-m3`.
Исходник: `backend/scripts/embedding-server.py`, юнит и зависимости рядом.

## Почему сервер именно такой

Векторы в базе совместимы только с эталонной реализацией
`sentence-transformers` в fp32. Сверка 13.08.2026 на 12 прод-фрагментах дала
косинус 1.00000. Замена на TEI, Infinity, ollama или квантованные веса — это
переиндексация 155 тысяч записей (около 70 часов CPU).

## Регламент любой замены

    node backend/scripts/check-embedding-parity.mjs --url <URL> --key <KEY> --limit 20

Худший косинус ≥ 0.9999 — можно. Ниже — нельзя, что бы ни обещал поставщик.
Прогонять и после обновления torch или sentence-transformers: версии закреплены
в `backend/scripts/embedding-server.requirements.txt` именно теми, на которых
сверка сошлась.

## Установка

    mkdir -p /opt/embedding-server && cd /opt/embedding-server
    # положить embedding-server.py и embedding-server.requirements.txt
    python3 -m venv venv
    venv/bin/pip install --extra-index-url https://download.pytorch.org/whl/cpu -r embedding-server.requirements.txt
    printf 'EMBED_SERVER_KEY=<секрет>\n' > .env && chmod 600 .env
    cp embedding-server.service /etc/systemd/system/
    systemctl daemon-reload && systemctl enable --now embedding-server

Первый запуск качает 2,27 ГБ весов — до пяти минут. Готовность: `curl -s localhost:3098/health`.
Ручка `/health` открыта; `/v1/embeddings` требует `Authorization: Bearer <ключ>`.
Наружу (`--host 0.0.0.0`) сервер без `EMBED_SERVER_KEY` не стартует вовсе.

## Диагностика

| Симптом | Куда смотреть |
|---|---|
| В логе `primary failed, trying fallback` | норма при мёртвом основном провайдере |
| В логе `fallback also failed` | сервер лежит: `systemctl status embedding-server` |
| 401 на каждый запрос | `EMBED_SERVER_KEY` на сервере ≠ `EMBEDDING_FALLBACK_KEY` у бэкенда |
| Один сервер достучался, другой нет | правила `ufw` для адреса конкретного сервера |
| Запрос висит и не отвечает | не выставлен `OMP_NUM_THREADS` до импорта torch (sentence-transformers#3839) |
| Убит по памяти | пришла пачка длинных текстов; уменьшить `EMBEDDING_BATCH_SIZE` у бэкенда |
| Векторы не пишутся, ошибок нет | очередь `_v2_embedding_sync`: статус `error`, `retry_count` |

## Расход

Модель 834 МБ, короткий фрагмент 2,0 ГБ, пачка из 8 длинных 2,9 ГБ.
0,6 фрагмента в секунду на 4 потоках, одиночный запрос 200–300 мс.
Спрос платформы — около 400 эмбеддингов в сутки.
