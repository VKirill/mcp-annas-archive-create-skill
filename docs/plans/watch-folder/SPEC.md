# SPEC: watch-folder fallback for manual slow-download

**Version:** 0.6.0 (minor — additive param + new fallback path; no breaking changes)

## Цель

Когда у пользователя **нет ключа** в `ANNAS_ACCOUNT_KEY` ИЛИ **исчерпан дневной лимит** fast-download — MCP не падает, а:

1. Резолвит md5 (через `searchBooks` или принимает напрямую).
2. Печатает в stderr ссылку на `https://annas-archive.gl/md5/<md5>` и путь, куда положить файл.
3. Поллит папку `$DATA_DIR/books/` каждые 2 секунды, до `manual_download_wait_ms` (дефолт 10 минут).
4. Когда файл появился и стабилизировался (размер не меняется между двумя соседними поллами) — продолжает обычный flow (magic-byte валидация → extract → Gemini).
5. Таймаут истёк — кидает понятную ошибку.

## Acceptance criteria

1. **Soft config loader** в `annas-client.ts`: `loadConfigFromEnvSoft(): { baseUrl, accountKey: string }` возвращает объект даже когда ключа нет (`accountKey: ""`); существующий `loadConfigFromEnv` остаётся для путей где ключ обязателен.
2. **Класс ошибки квоты:** `class QuotaExhaustedError extends Error` экспортируется из `annas-client.ts`. `getFastDownloadUrl` бросает его, когда ответ API содержит `error` совпадающее по regex `/quota|limit|exhaust|exceed/i` ИЛИ `downloads_left === 0`. Остальные ошибки fast_download остаются обычными `Error`.
3. **Новый модуль** `src/lib/watch-folder.ts` экспортирует:
   ```ts
   export interface WaitResult { path: string; format: string; size_bytes: number; }
   export async function waitForBook(
     booksDir: string,
     md5: string,
     timeoutMs: number,
     onTick?: (elapsedMs: number, remainingMs: number) => void,
   ): Promise<WaitResult | null>;
   ```
   - Поллит каждые **2000 мс**.
   - Файл считается готовым: имя начинается с `${md5}.`, расширение в `{epub, fb2, pdf, txt, bin}`, размер `>0`, и размер **не изменился** между двумя соседними поллами (защита от mid-write).
   - `onTick` зовётся раз в **30 сек** для прогресс-логов.
   - Если в `timeoutMs` файл не появился — возвращает `null` (не бросает).
4. **Схема `BookSkillInputSchema`** в `book-skill.ts` получает:
   ```ts
   manual_download_wait_ms: z.number().int().min(0).max(3_600_000).optional()
     .describe("Wait this many ms for a manually-downloaded file in data/books/ when fast-download is unavailable. 0 = no wait (fail fast). Default 600_000 (10 min) or env MANUAL_DOWNLOAD_WAIT_MS."),
   ```
5. **Интеграция в `resolveAndDownload`:**
   - **Search path:** даже без ключа — пробуем `searchBooks` (он не требует ключа, скрейпит HTML). Берём `hits[0].md5` (после whitelist+priority sort).
   - **Md5 path:** md5 уже известен.
   - Дальше:
     - Если ключ ЕСТЬ — пробуем fast-download как сейчас (top-3 hits × 4 partner). При `QuotaExhaustedError` на ВСЕХ попытках для текущего md5 → переход в watch mode для **этого** md5 (не fallback на hits[1]).
     - Если ключа НЕТ — сразу watch mode для `hits[0].md5` (или прямого md5).
   - Watch mode: печатаем block stderr, зовём `waitForBook(booksDir, md5, waitMs, onTick)`. На success: `detectMagic` → rename если нужно → возврат `BookFile`. На null: throw с инструкцией «таймаут истёк».
   - `manual_download_wait_ms = 0` отключает watch — fast-fail с понятным сообщением (legacy поведение).
6. **Stderr UX-блок** (формат фиксирован — пользователь должен сразу понять):
   ```
   ─────────────────────────────────────────────────────────────
   [mcp-books] Manual download mode (slow path / no key / quota)
   ─────────────────────────────────────────────────────────────
     1. Open in browser:
          https://annas-archive.gl/md5/<md5>
     2. Click "Slow download" — wait for the timer.
     3. Save the file as:
          /home/ubuntu/tools/mcp-books/data/books/<md5>.<ext>
        Allowed extensions: .epub .fb2 .pdf .txt
        (Wrong extension is OK — magic-byte detector will fix it.)
   
     Waiting up to 10:00. Polling every 2s.
   ─────────────────────────────────────────────────────────────
   [mcp-books] 0:30 elapsed, 9:30 remaining...
   [mcp-books] 1:00 elapsed, 9:00 remaining...
   [mcp-books] Found file: /home/.../data/books/<md5>.epub (2.4 MB). Validating...
   ```
7. **README + README.ru** — короткая секция «Manual download / no key» с примером.
8. **Smoke test** (`src/smoke.ts`) — новая офлайн-проверка: `waitForBook` с timeoutMs=200 и пустой папкой → возвращает `null` за ~200ms. Существующие 3 проверки остаются.
9. **Version bump** `0.5.0 → 0.6.0`.

## Files to touch

| File | Change | ~Lines |
|---|---|---|
| `src/lib/annas-client.ts` | `loadConfigFromEnvSoft` + `QuotaExhaustedError` + детект квоты в `getFastDownloadUrl` | +25 / -2 |
| `src/lib/watch-folder.ts` | **новый файл** — `waitForBook` + size-stability check | +70 |
| `src/tools/book-skill.ts` | schema param + интеграция watch-mode в `resolveAndDownload` + stderr UX-блок | +60 / -10 |
| `src/smoke.ts` | новая offline-проверка `waitForBook` timeout | +15 |
| `README.md`, `README.ru.md` | секция «Manual download» | +20 каждый |
| `package.json` | bump 0.5.0 → 0.6.0 | 1 line |

Всего ~190 строк изменений + 1 новый файл.

## Open questions

Нет — таймаут 10 мин (по умолчанию) и polling раз в 2 сек зафиксированы. Если пользователь захочет иначе — есть `manual_download_wait_ms` параметр на вызов и `MANUAL_DOWNLOAD_WAIT_MS` env-переменная.

## Known tradeoffs

- **`fs.watch` vs poll:** выбран polling. Причины: (а) cross-platform — `fs.watch` нестабилен на Linux+некоторых ФС; (б) надо знать когда файл «дописался», для этого всё равно нужен размер-stability check; (в) 2-сек интервал минимально нагружает диск.
- **Если пользователь сохранит файл под именем без префикса md5** (например `mom-test.epub`) — watcher его не найдёт. Сообщение в stderr делает требование явным, но это всё ещё user error. Решаем явной инструкцией в UX-блоке.
- **Search-path без ключа:** делает HTTP-запрос к твоей библиотеке, может попасть под Cloudflare если IP заблокирован. Этой проблемой мы не управляем — если scrape не работает, юзер всё равно не сможет получить md5. На существующих setups (с прокси в `proxy.ts`) работает.
- **`hits[1]/hits[2]` fallback не запускается** в watch mode — слишком много контекстных сообщений и риск что юзер скачает не ту книгу. Watch строго на `hits[0]`.

## Out of scope

- Автоматизация Slow download (Playwright/cloudflare bypass) — обсуждали, дорого и хрупко.
- UI/dashboard для очереди.
- Webhook/уведомления когда файл подхвачен.
- Параллельная обработка нескольких книг в одном MCP-вызове.
