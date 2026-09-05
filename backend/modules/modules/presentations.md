# Presentations — PPTX-движок воркспейса

Каноническая JSON-модель слайдов → экспорт .pptx (pptxgenjs) и PDF-превью (общий
Chromium из documents/pdf.js). Импорт .pptx → модель (JSZip, MVP-фиделити:
текст/картинки/позиции, SmartArt и анимации — в warnings). Версионность —
снапшоты (паттерн documents). Доступ — единый судья `checkPresAccess`
(owner → ws-роль → is_public → sharing), общий для REST и ИИ (ADR-027).

## Эндпоинты (`/:db/presentations`, requireModule('presentations'))

| Метод | Путь | Роль | Описание |
|-------|------|------|----------|
| GET | `/` | any | Список (фильтр доступа в запросе) |
| POST | `/` | any | Создать (title, slides?, is_public?) |
| GET | `/:id` | viewer | Презентация со слайдами |
| PUT | `/:id` | editor | title/is_public/status |
| DELETE | `/:id` | admin | Soft delete |
| PUT | `/:id/slides` | editor | Полная замена слайдов (+снапшот, троттл 2 мин) |
| GET/POST | `/:id/versions` | viewer/editor | Список / создать снапшот |
| POST | `/:id/versions/:versionId/restore` | editor | Откат (текущее — в снапшот) |
| GET | `/:id/export/pptx\|pdf` | viewer | Экспорт |
| POST | `/import` | any | Импорт .pptx (multipart, поле file, ≤50 МБ) |
| POST | `/generate` | any | Сгенерировать слайды из outline-текста модели (`{ outline }`) → `{ slides }`; вызывается редактором |
| GET/POST/DELETE | `/sharing/:id(…)` | GET — viewer; POST/DELETE — admin | Гранты |

## Ключевые файлы

| Файл | Роль |
|------|------|
| `modules/presentations/schema.js` | 4 таблицы, полный DDL (ADR-013) |
| `modules/presentations/model.js` | Zod-модель слайда (элементы: title/text/bullets/image/table) |
| `modules/presentations/service.js` | CRUD, версии, шаринг, судья доступа |
| `modules/presentations/engine/to-pptx.js` | модель → .pptx |
| `modules/presentations/engine/to-html.js` | модель → HTML/PDF (общий Chromium) |
| `modules/presentations/engine/from-pptx.js` | .pptx → модель + warnings |

## События

`presentation.created`, `presentation.updated`, `presentation.deleted` (bus; consumers — по мере появления).

## ИИ-инструменты

`pres_list`, `pres_get`, `pres_create_from_outline`, `pres_save_slides`, `pres_delete`
(группа `presentations`, гейт модуля в module-gate.js, судья checkPresAccess на каждом).
