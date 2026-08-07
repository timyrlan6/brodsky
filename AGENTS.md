# Проект «Бродский»

Исследование творчества Иосифа Бродского: частотный анализ, поиск по словам, отсылки к слову в стихе.

## Продукт (видение)
Веб-приложение для чтения с **семантическим слоем**:
- Пользователь читает стихотворение; слова, входящие в семантическое поле концепта (напр. «Угол»), **подсвечены**.
- **Интенсивность подсветки = величина символического значения** (на MVP — частота вхождений в корпус).
- Клик по слову → страница концепта: **описание символа** + список **всех вхождений по всем томам** (с контекстом и ссылкой на стих).
- Подробный контекст и история решений — см. `docs/CONTEXT.md`.

## Технологии (решено)
- **.NET / C#** — обработка данных
- **PostgreSQL** — рабочее хранилище (единственное хранилище обработанных данных)
- **ASP.NET Core + веб-интерфейс** — форма приложения
- **EF Core** — доступ к данным, миграции
- **LLM** — семантический слой: черновики описаний символов, (в будущем) классификация вхождений. НЕ используется для подсчётов и цитат.

## Принципы
- Канонические исходники не изменяем; БД — источник истины для обработанных данных.
- LLM работает на **выборках из БД**, каждая гипотеза верифицируется детерминированным слоем (SQL).
- LLM не считает частоты и не генерирует цитаты.

## Данные
- Исходники: папка `torrent/` — «Сочинения Иосифа Бродского» (2001–2003), 7 томов, форматы **fb2 / epub / mobi**.
- FB2 — исходный формат для импорта: структурированный XML, содержит разметку стиха.
- Файлы FB2 в кодировке **windows-1251** (не UTF-8!). Читать: `Encoding.GetEncoding(1251)`.
- Исходные файлы не изменяем — источник истины (canonical).

## Структура FB2 (важно для парсинга)
Иерархия: **том → раздел → год/цикл → стихотворение**.
- `<section>` — разделы (части, годы, циклы); вложенные `section` образуют дерево
- `<title><p>` — заголовок секции/стихотворения (внутри могут быть сноски `<a type="note">`)
- `<poem>` — стихотворение; `<stanza>` — строфа; `<v>` — строка стиха
- `<subtitle>` — подзаголовок
- Без названия стихи называются `* * *`
- В 1-м томе: 219 `<poem>`, 1032 `<stanza>`, 7421 `<v>`
- В томах есть проза (Нобелевская лекция): для неё строки = абзацы, `stanza_no = NULL`

## Схема БД `brodsky` (ревизия 2)
```
-- текстовый слой
volumes           (id, volume_no UNIQUE, title, year)
sections          (id, volume_id, parent_id NULL, title, sort_order)
works             (id, section_id, title NULL, subtitle, year, is_poem, source_ref, sort_order)
lines             (id, work_id, line_no, stanza_no NULL, text, UNIQUE(work_id, line_no))
words             (id, form UNIQUE, form_norm, stem)          -- form_norm: ё→е
word_occurrences  (id, word_id, work_id, line_id, char_start, char_end, position)
search_tsv        (work_id, tsv)  -- tsvector + GIN

-- концептный слой
concepts                  (id, slug UNIQUE, name, description, description_source, status)
concept_members           (id, concept_id, display_name, is_phrase, note)
concept_member_keys       (member_id, key_type, key_value, key_order)
                            -- key_type: 'stem' | 'form' | 'phrase'
concept_member_words      (member_id, word_id)   -- разрешённый список слов (авто + ручной)
concept_occurrences       (id, concept_id, member_id, work_id, line_id,
                            char_start, char_end, matched_text)  -- слова и фразы
concept_occurrence_notes  (concept_occurrence_id, is_symbolic, significance,
                            type, note, source, status)
```

Важно:
- Интенсивность подсветки = `COUNT(concept_occurrences) GROUP BY member_id` (нормировка 1–100 в UI). Граф зависимостей позже подменит источник метрики без изменения схемы.
- Заметки (`notes`) висят на **вхождении в концепт** (`concept_occurrences`), а не на слове — слово может быть членом нескольких концептов.
- `word_occurrences.char_start/char_end` — смещения в `lines.text`, нужны для рендера подсветки.
- Фразы («конец пера») поддержаны: `is_phrase`, span в `concept_occurrences`.

Индексы: `word_occurrences(word_id), (line_id), (work_id), (word_id, line_id)`; `concept_occurrences(concept_id, work_id), (member_id), (work_id)`; `lines(work_id, line_no)`; `works(section_id, sort_order)`; `words(stem)`; `concept_member_words(word_id)`; `search_tsv` GIN.

## Решения
- **Лемматизация MVP**: стеммер PostgreSQL + таблица исключений (случай «угол/угл»: Snowball не склеивает базовую форму с падежными — поиск «угла» не находит «угол»). Полноценный лемматизатор — позже.
- **Описание символа**: LLM-черновик + ручные правки (хранится в `concepts.description`).
- **UTF-8 выгрузки**: не нужны, БД — единственное хранилище.

## Открытые вопросы
1. Ё/ё: матчинг по `form_norm` (ё→е) или по обеим формам?
2. Фразы («конец пера»): в MVP или второй итерацией? (схема уже готова)
3. Где крутить LLM: локально (Ollama) или через API?
4. Готовый состав концепта «Угол» от исследователя или собираем с нуля (LLM-кандидаты)?

## План реализации
1. Импортёр FB2 → PostgreSQL (консоль, C#)
2. Индекс словоформ + стемминг + счётчики
3. Модель концептов + seed «Угол» (угол, локоть, колено, мыс, шпиль, «конец пера» + LLM-кандидаты)
4. Веб: чтение с подсветкой, страницы символов, список вхождений
5. (будущее) LLM-классификация вхождений, граф зависимостей, интерфейс валидации

## Нюансы среды (Windows, PowerShell 5.1)
- Оператор `&&` НЕ поддерживается — использовать `;` или `if ($?) { ... }`.
- Кириллица в консоли: ставить `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` перед выводом.
- `"$i: "` — ошибка `InvalidVariableReferenceWithDrive`; использовать `${i}:` или конкатенацию.
- Загрузка XML в cp1251: `[System.Text.Encoding]::GetEncoding(1251)`.
