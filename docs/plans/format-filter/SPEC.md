# SPEC: format filter + fallback + magic-byte detection

**Version:** 0.5.0 (minor — additive `prefer_format` param, behavior change in search filtering)

## Цель

Жёсткий whitelist `epub / fb2 / pdf / txt` на этапе **поиска** в твоей библиотеке — никаких azw3 / mobi / bin / djvu / cbz / cbr / zip даже не попадает в кандидаты, ни одного слота daily-quota на них не тратится. Плюс: корректный приоритет, fallback на следующий поисковый хит, и magic-byte проверка скачанного.

## Acceptance criteria

1. **Whitelist в `searchBooks`:** результаты с `format ∉ {epub, fb2, pdf, txt}` отбрасываются ДО возврата вызывающему. Регекс расширен — включает `txt`. Хиты с `format = "?"` (когда парсер не определил) — тоже отбрасываются (безопасный default).
2. **Логирование исключённых:** перед `return hits` в `searchBooks` пишем в stderr одну строку `[books] filtered out N hits (formats: <list>)` для прозрачности.
3. **Приоритет сортировки:** `epub > pdf > fb2 > txt` (поменять с текущего `epub > fb2 > pdf > txt` в `book-skill.ts:150`).
4. **`prefer_format` параметр:** опциональный, в схеме `BookSkillInputSchema` — `z.enum(["epub","pdf","fb2","txt"]).optional()`. Если задан — этот формат поднимается в топ сортировки (остальные сохраняют относительный порядок).
5. **Search-hit fallback:** в `resolveAndDownload` для пути «поиск по строке» — если для `hits[0]` все 4 partner-сервера упали ИЛИ скачанный файл провалил magic-byte (RTF/HTML/неизвестный), пробуем `hits[1]`, затем `hits[2]`. Дальше — surface error. Cached-by-md5 путь не затрагиваем.
6. **Magic-byte детектор** (новый `src/lib/magic-bytes.ts`): читает первые ≤8 байт скачанного файла, возвращает `"epub" | "pdf" | "fb2" | "rtf" | "html" | "unknown"`. Сигнатуры:
   - `PK\x03\x04` → `epub` (zip)
   - `%PDF-` → `pdf`
   - `<?xml` или `<FictionBook` (после возможного BOM) → `fb2`
   - `{\rtf` → `rtf`
   - `<!DOCTYPE html` / `<html` → `html` (часто — captcha-страница, замаскированная под `.epub`)
7. **Интеграция magic-byte в `resolveAndDownload`:** после `downloadFile` — проверяем актуальный формат. Если совпадает с ext — оставляем. Если detected ∈ {`rtf`, `html`, `unknown`} — удаляем файл и кидаем ошибку (fallback подхватит). Если detected ∈ whitelist, но ≠ ext — переименовываем файл (`${md5}.<detected>`) и продолжаем.
8. **Smoke test:** `src/smoke.ts` дополнен — три новых проверки:
   - `searchBooks` для книги, у которой есть azw3 + epub варианты, возвращает только epub-результат(ы).
   - `prefer_format: "pdf"` поднимает pdf-хит над epub в сортировке.
   - Magic-byte: на синтетических буферах (`PK\x03\x04…`, `%PDF-…`, `{\rtf1…`) возвращает корректные ярлыки.

## Files to touch

| File | Change | Approx. lines |
|---|---|---|
| `src/lib/annas-client.ts` | whitelist filter, txt в regex, prefer_format param, stderr log | +20 / -3 |
| `src/lib/magic-bytes.ts` | **новый файл** — `detectMagic(path): Promise<MagicKind>` | +60 |
| `src/lib/downloader.ts` | без изменений (детекция — на стороне book-skill после download) | 0 |
| `src/tools/book-skill.ts` | схема `prefer_format`, новый порядок сортировки, search-hit fallback, magic-byte интеграция | +40 / -10 |
| `src/smoke.ts` | три новых проверки | +30 |
| `package.json` | bump `0.4.1 → 0.5.0` | +1 / -1 |

Всего: ~150 строк изменений, 1 новый файл.

## Open questions

Нет. Допущения зафиксированы (fallback = next search hit; .bin/azw3/mobi не достигают этапа download; lockstep по quota — не тратим лимит на отфильтрованные).

## Known tradeoffs

- **Если у книги есть ТОЛЬКО azw3 версия** — `searchBooks` вернёт `[]`, и MCP скажет «no results from your library». Пользователь не отличит «нет книги» от «есть, но в неподдерживаемом формате». Принимаем — иначе придётся возвращать «отфильтрованные» хиты отдельным списком, что усложнит API и противоречит твоей формулировке «даже не считывать».
- **HTML-детект** ловит captcha-страницы, но не ловит «битый epub» (валидный zip с мусором внутри) — это уже работа `extractBookText`, не наша.
- **prefer_format не гарантирует** этот формат — только поднимает в топ. Если в whitelisted-хитах его нет, отдаём всё что есть в стандартном порядке.

## Out of scope

- Изменения partner-сервер ротации (0..3 как и было).
- Изменения logic в `extractBookText` сверх того, что magic-byte уже отбросит RTF/HTML до вызова экстрактора.
- Изменения в других MCP-эндпоинтах (`skill_audit` и пр.).
