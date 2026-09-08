# Импорт словаря через ИИ

## Настройка

Интеграция записывает слова только в словарь владельца ключа. Откройте профиль нажатием на email, нажмите «Создать ключ» и сразу сохраните показанное значение. В БД хранится только SHA-256, поэтому посмотреть полный ключ позднее нельзя — его можно только перевыпустить.

В профиле доступна готовая инструкция для ИИ. Кнопка «Скопировать инструкцию» добавляет в неё адреса ручек, правила наполнения и только что созданный ключ. Ключ передаётся только в заголовке `X-API-Key`.

## Самоописание API

ИИ сначала должен запросить актуальную схему, языки, письменности и грамматический каталог:

```http
GET /api/v1/integrations/dictionary/schema?sourceLanguageCode=sr
X-API-Key: <key>
```

Для GPT Action импортируйте отдельную OpenAPI-схему по адресу
`/dictionary-action-openapi.json`. Она содержит только четыре словарные операции
и не раскрывает пользовательские JWT-ручки.

Полный OpenAPI-контракт доступен через `GET /docs-json`. Ответ `/schema` содержит допустимые значения из текущей БД, готовые шаблоны форм и пример тела импорта.

## Чтение и сравнение словаря

```http
GET /api/v1/integrations/dictionary/words?sourceLanguageCode=sr&targetLanguageCode=ru&limit=100
X-API-Key: <key>
```

Ответ содержит краткие карточки и `pageInfo.nextCursor`. Для следующей порции передайте `cursor`; для перехода по номерам страниц используйте `page` и `limit`.

Доступные фильтры:

- `q` — слово, вариант написания или перевод;
- `exact=true` — точное сравнение `q` со всеми написаниями и переводами;
- `sourceLanguageCode`, `targetLanguageCode` — языковая пара;
- `learningStatus` — `new`, `learning`, `review`, `mastered`, `suspended`;
- `deckId`, `partOfSpeech`, `scriptCode`;
- `hasGrammar=true|false`, `hasExamples=true|false`;
- `updatedAfter` — ISO 8601 дата для инкрементальной синхронизации;
- `cursor` или `page`, а также `limit` от 1 до 100.

Перед импортом ИИ должен проверить точное совпадение:

```http
GET /api/v1/integrations/dictionary/words?q=kuća&exact=true&sourceLanguageCode=sr&targetLanguageCode=ru
X-API-Key: <key>
```

Полная карточка со всеми написаниями, переводами, примерами, падежами и временами:

```http
GET /api/v1/integrations/dictionary/words/{wordId}
X-API-Key: <key>
```

## Импорт

```http
POST /api/v1/integrations/dictionary/import
Content-Type: application/json
X-API-Key: <key>
```

За один запрос принимается от 1 до 100 слов. Повторный импорт безопасен: новые слова создаются, существующие только дополняются отсутствующими написаниями, ударениями, переводами, примерами и формами.

## Полный формат JSON

```json
{
  "items": [
    {
      "sourceLanguageCode": "sr",
      "targetLanguageCode": "ru",
      "spellings": [
        {
          "scriptCode": "Latn",
          "value": "kuća",
          "stressedValue": "kȕća",
          "isPrimary": true
        },
        {
          "scriptCode": "Cyrl",
          "value": "кућа",
          "stressedValue": "ку̏ћа",
          "isPrimary": false
        }
      ],
      "translations": ["дом", "жилище"],
      "learningStatus": "new",
      "partOfSpeech": "noun",
      "grammar": {
        "gender": "feminine",
        "declension": "I склонение",
        "forms": [
          {
            "group": "case",
            "label": "родительный, ед. число",
            "value": "kuće",
            "attributes": [
              {
                "kind": "case",
                "value": "genitive",
                "label": "родительный"
              },
              {
                "kind": "number",
                "value": "singular",
                "label": "ед. число"
              }
            ]
          }
        ]
      },
      "note": "Частотное существительное.",
      "examples": [
        {
          "sourceText": "Ovo je moja kuća.",
          "translatedText": "Это мой дом."
        }
      ],
      "deckIds": []
    }
  ]
}
```

## Допустимые значения

- `partOfSpeech`: `noun`, `verb`, `adjective`, `adverb`, `pronoun`, `preposition`, `conjunction`, `interjection`, `numeral`, `particle`, `phrase`, `other`.
- `learningStatus`: `new`, `learning`, `review`, `mastered`, `suspended`. Поле используется только при импорте; если его нет, создаётся новое слово со статусом `new`.
- `gender`: `masculine`, `feminine`, `neuter`, `common`.
- `grammar.forms[].group`: `case`, `tense`, `other`.
- `grammar.forms[].attributes[].kind`: `case`, `number`, `gender`, `degree`, `definiteness`, `animacy`, `person`, `tense`, `mood`, `formType`, `extra`.
- Сербские `case`: `nominative`, `genitive`, `dative`, `accusative`, `vocative`, `locative`, `instrumental`.
- `number`: `singular`, `plural`.
- `degree`: `positive`, `comparative`, `superlative`.
- `definiteness`: `indefinite`, `definite`.
- `animacy`: `animate`, `inanimate`.
- `person`: `first`, `second`, `third`.
- Сербские `tense`: `present`, `future`, `aorist`, `imperfect`.
- `mood`: `imperative`.
- `formType`: `infinitive`, `activeParticiple`, `passiveParticiple`, `gerund`.

Значения грамматики могут расширяться для других языков, поэтому перед генерацией данных ИИ должен использовать актуальный ответ `/schema`, а не полагаться только на этот список.

## Серверная валидация

Одинаковые правила применяются к пользовательскому созданию, редактированию, JSON/JSONL-импорту и AI-import:

- языки слова и перевода должны существовать и отличаться;
- написания и значения форм должны соответствовать письменности исходного языка;
- переводы должны преимущественно соответствовать письменности языка перевода;
- HTML, невидимые управляющие символы, ссылки в лексических полях, избыток символов и очевидные клавиатурные или повторяющиеся последовательности отклоняются;
- пустые строки, дубли переводов, признаков и одинаковых грамматических форм отклоняются;
- грамматика требует `partOfSpeech`; род, склонение, спряжение, падежи и времена проверяются на совместимость с частью речи;
- для языка с грамматическим каталогом падежи, времена и `attributes` должны соответствовать значениям из `/schema`.

При ошибке API возвращает HTTP `422`. Поле `details.path` указывает ошибочное поле, а при пакетном импорте `details.itemIndex` содержит индекс слова внутри `items`.

## Правила для ИИ

1. Перед импортом вызвать `/schema` и использовать существующие коды языков и письменностей.
2. Перед добавлением искать каждое слово через `exact=true`, а при совпадении открывать полную карточку.
3. В `value` хранить написание без разметки, в `stressedValue` — то же слово с ударением.
4. Для сербского по возможности передавать обе письменности `Latn` и `Cyrl`, но только одну отмечать `isPrimary: true`.
5. Не смешивать кириллицу и латиницу внутри одного варианта написания.
6. В `forms[].value` передавать именно форму слова, а не её перевод или описание.
7. Не выдумывать формы: сомнительные поля лучше пропустить — все необязательные поля можно не отправлять.
8. Делить большие словари на пакеты не более 100 элементов и проверять ответ `{ received, created, extended, skipped }` после каждого пакета.
9. Если ключ скомпрометирован, перевыпустить или отключить его в профиле; старый ключ перестаёт работать сразу.
