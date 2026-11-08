# Требования и особенности формата EDT (файловая разработка)

Знания, полученные при ручной правке файлов EDT вне среды. Обязательны к учёту
при любом изменении файлов проекта вне EDT.

## Структура каталогов проекта

Объекты метаданных размещаются **в корне `src/` по типам**, а не во вложенной
`src/Configuration/`:

```
src/
├── Configuration.mdo? — нет: конфигурация лежит в src/Configuration/Configuration.mdo
├── Configuration/
│   ├── Configuration.mdo
│   ├── CommandInterface.cmi
│   └── MainSectionCommandInterface.cmi
├── CommonModules/<Имя>/<Имя>.mdo + Module.bsl
├── CommonCommands/<Имя>/<Имя>.mdo + CommandModule.bsl
├── Constants/<Имя>/<Имя>.mdo
└── ...
```

Файлы верхнего уровня (Configuration.mdo, *.cmi) остаются в `src/Configuration/`,
объектные папки — в `src/<Класс>/`. Именно так их создаёт сама EDT данной версии.
Папки объектных типов идут **без префикса типа** в имени: `Catalogs/<Имя>/`,
`Enums/<Имя>/`, `InformationRegisters/<Имя>/`, `Reports/<Имя>/`; префиксы
(`Catalog.`, `EnumRef.` и т.п.) встречаются только внутри ссылок.

## Имена файлов модулей по типам объектов

| Тип объекта | Файл модуля |
|---|---|
| Общий модуль | `Module.bsl` |
| Общая команда | `CommandModule.bsl` |

Неверное имя файла модуля: модуль не отображается в редакторе EDT, анализаторы
применяются некорректно (модуль не распознаётся как модуль команды/формы).

## Технические требования к файлам

- **Окончания строк**: EDT на Windows пишет CRLF — и фактическое состояние
  файлов репозитория — **CRLF** (подтверждено побайтовой проверкой 2026-11-08;
  запись про LF, сделанная ранее в этот же день, была ошибочной). Новые
  и перезаписанные файлы пишем в CRLF, как существующие; массовую конвертацию
  не выполняем, чтобы не шуметь в истории git.
- Кодировка: UTF-8 (настройка проекта `encoding/<project>=UTF-8`), без BOM.
- Порядок XML-элементов — строго по схеме EDT (ecore-метамодель пакета
  `http://g5.1c.ru/v8/dt/metadata/mdclass`).

## Особенности метаданных

- **Константа**: обязателен блок `producedTypes` (managerType, valueManagerType,
  valueKeyType — по 2 UUID на каждый); строковый тип пишется как `String`
  (не `xs:string`); эталон также содержит `minValue`/`maxValue`
  (`<minValue xsi:type="core:UndefinedValue"/>`) и `dataLockControlMode`.
- **Общая команда**: блок `<commandParameterType/>` пустой — допустим; группа —
  предопределённая строка, например `NavigationPanelOrdinary`.
- **Общие модули**: наборы флагов должны соответствовать каноническим типам
  (стандарт ИТС 439/469, проверка `common-module-type`): «Клиентский модуль» =
  `Клиент (управляемое приложение)` + `Клиент (обычное приложение)`; серверный
  вызов с клиента = `Сервер` + `Вызов сервера` (без флагов обычного приложения
  и внешнего соединения).
- **Ссылки из Configuration.mdo**: `<commonModules>CommonModule.Имя</commonModules>`
  и т.п. — формат верный. Секции добавляются в фиксированном порядке схемы:
  ... `commonModules`, `commonCommands`, `constants`, `commonForms`, `catalogs`,
  `documents`, `enums`, `reports`, `dataProcessors`, `informationRegisters` ...
  Блок `containedObjects` (7 classId) — константный список классов корня,
  новыми объектами не пополняется.
- **Перечисление**: `producedTypes` — refType, listType, managerType (по 2 UUID);
  значения — блоки `enumValues` (uuid + name + synonym). Имена значений —
  валидные идентификаторы; латиница в нижнем регистре допустима (`low`, `max`).
- **Справочник**: `producedTypes` — 5 пар UUID (objectType, refType, selectionType,
  listType, managerType); блоки `standardAttributes` (9 шт.: PredefinedDataName,
  Predefined, Ref, DeletionMark, IsFolder, Owner, Parent, Description, Code) EDT
  пишет всегда; далее `codeLength`, `descriptionLength`, `levelCount` и реквизиты
  в `attributes` (после `quickChoice`/`choiceMode`). Строка без ограничения — пустой
  `<stringQualifiers/>`.
- **Регистр сведений** (независимый, непериодический): `producedTypes` — 7 пар
  UUID (selectionType, listType, managerType, recordSetType, recordKeyType,
  recordType, recordManagerType); `standardAttributes` — Active, LineNumber,
  Recorder, Period; периодический регистр пишет `informationRegisterPeriodicity`
  (независимый непериодический — опускает, как и признак подчинения регистратору);
  порядок блоков: `resources`, `dimensions`, `attributes` (реквизиты), `forms`;
  у ссылочных измерений — `denyIncompleteValues` и `fillValue`
  (`core:ReferenceValue` → `Catalog.<Имя>.EmptyRef`); типы ссылок:
  `CatalogRef.<Имя>`, `EnumRef.<Имя>`.
- **Отчёт СКД**: `Reports/<Имя>/<Имя>.mdo` +
  `Templates/ОсновнаяСхемаКомпоновкиДанных/Template.dcs`; в mdo —
  `mainDataCompositionSchema` и блок `templates` с `templateType` =
  DataCompositionSchema; собственная форма не нужна (платформа даёт автоформу
  отчёта, открывается из кода через `ОткрытьФорму("Отчет.<Имя>")`).
- **Template.dcs**: корень `DataCompositionSchema` (namespace
  `http://v8.1c.ru/8.1/data-composition-system/schema`); порядок: `dataSource`
  (Local) → `dataSet` (DataSetQuery: перечисленные `field` + текст `query`,
  `&` экранируется `&amp;`) → `totalField` (dataPath + expression, напр.
  `Сумма(Поле)`) → `settingsVariant`. Варианты настроек: `dcsset:name` +
  `dcsset:presentation` (v8:LocalStringType) + `dcsset:settings` (с объявлением
  xmlns style/sys/web/win); группировки — `StructureItemGroup` с `GroupItemField`
  (groupType=Items, periodAdditionType=None, нулевые даты periodAddition),
  выборка — `SelectedItemAuto`/`SelectedItemField`, порядок — `OrderItemField`
  (orderType=Asc). Агрегаты по булевым полям считаем через целые поля, вычисленные
  в запросе: `ВЫБОР КОГДА Т.Верно ТОГДА 1 ИНАЧЕ 0 КОНЕЦ`.

## BSL-код

- Директивы `&НаКлиенте`, `&НаСервере`, `&НаСервереБезКонтекста` — только в
  модулях форм и команд (проверка `form-module-pragma`, стандарт ИТС 439).
- Имена метаданных, реквизитов формы, параметров и переменных не должны
  совпадать с зарезервированными словами встроенного языка (проверка на примере
  слова `Попытка`: ошибка компиляции «Ожидается имя формального параметра») и
  со свойствами глобального контекста (`Задачи`, `Отчеты`, `Справочники` и т.п. —
  неявная локальная переменная разрешается в read-only свойство, ошибка
  «Поле объекта недоступно для записи» во время выполнения; в модулях форм
  реквизиты глобальный контекст перекрывают). Именуя поле регистра, реквизит,
  параметр или переменную, сверяем слово с ключевыми словами и глобальным
  контекстом.
- `Сообщить()` подсвечивается плагином BSL Language Server как устаревший
  (диагностика DeprecatedMessage, важность «незначительная»). Осознанно
  используем как «консоль» вывода — в конфигурации нет БСП-обёртки.
- Тонкий клиент: ряд функций глобального контекста недоступен (например,
  `Округл` — только сервер/толстый клиент) — в клиентском коде используем
  клиентские аналоги (`Окр`), иначе ошибка компиляции «Процедура или функция
  с указанным именем не определена».
- Длительный клиентский цикл формы разбивается на шаги через
  `ПодключитьОбработчикОжидания(..., 0.1, Истина)`: управление возвращается
  в цикл обработки событий, интерфейс обновляет прогресс, кнопка «Остановить»
  успевает отработать между шагами.

## Обработки с формами

- Обработка `DataProcessor`: обязателен `producedTypes` (objectType + managerType,
  по 2 UUID на каждый); форма указывается через `defaultForm` и вложенный блок
  `<forms uuid="...">` внутри .mdo обработки (отдельного .mdo у формы нет).
- Папка формы: `Forms/<ИмяФормы>/` содержит `Form.form` (описание элементов),
  `Module.bsl` (модуль формы) и при наличии `ConditionalAppearance.dcssca`
  (условное оформление — подхватывается по имени файла, ссылка в Form.form не нужна).
- `Form.form`: корневой `form:Form` (namespace `http://g5.1c.ru/v8/dt/form`), элементы
  с числовыми `id`, реквизиты в `attributes` (таблица значений — `types>ValueTable`
  с вложенными `columns`), команды формы в `formCommands` (обработчик — `handler/name`
  = имя процедуры модуля формы), обязательный пустой `commandInterface`.
  Эталон структуры: официальная демо-конфигурация 1С (dt-demo-configuration).
- События формы и элементов: блок `<handlers><event>OnCreateAtServer</event><name>ПриСозданииНаСервере</name></handlers>`
  (имена событий английские). На уровне формы — после `<autoCommandBar>`, перед
  `<saveWindowSettings>`; внутри `FormField` — после `dataPath`/`titleLocation`,
  перед `extendedTooltip`; у таблицы — после `dataPath`. Без этого блока платформа не вызовет процедуру.
- Выпадающий список у поля ввода: `<dropListButton>true</dropListButton>` в
  `InputFieldExtInfo`; значения для строкового реквизита в форме статически не
  хранятся — список выбора заполняется кодом (`Элементы.Поле.СписокВыбора`,
  обычно в ПриСозданииНаСервере).
- Флажок: `<type>CheckBoxField</type>` + `<extInfo xsi:type="form:CheckBoxFieldExtInfo"/>`
  (пустой блок допустим).
- Числовой реквизит формы: в `valueType` атрибута —
  `<numberQualifiers><precision>10</precision><scale>0</scale><nonNegative>true</nonNegative></numberQualifiers>`.
- `id` элементов формы уникальны только в рамках своего вида: `items`, `attributes`,
  `formCommands` — независимые пространства (в эталоне у атрибута и элемента
  совпадают id).
- Условное оформление (dcssca): default namespace
  `http://v8.1c.ru/8.1/data-composition-system/settings`; фильтр —
  `item xsi:type="FilterItemComparison"` с `left xsi:type="dcscor:Field"`,
  `comparisonType`, `right xsi:type="xs:string"`; оформление —
  `dcscor:item xsi:type="SettingsParameterValue"` с параметром (`ЦветТекста` и пр.)
  и значением. **Важно:** значение цвета строками `web:Имя`/`style:Имя` EDT
  не распознаёт (подставляет жёлтый 255,255,0) — ссылку на элемент стиля
  задавать атрибутами: `<dcscor:value xsi:type="v8ui:Color" ref="style:ИмяСтиля" kind="StyleItem"/>`.
  Сами цвета хранятся в элементах стиля (StyleItem) со значением
  `core:ColorValue` → `core:ColorRef` → `Web.Имя` (именно с точкой).

## Kimi API

- Endpoint: `api.moonshot.ai` (глобальный), протокол совместим с OpenAI
  (`POST /v1/chat/completions`).
- Модель `moonshot-v1-8k` устарела (deprecated). Используется `kimi-k3.1-pro-highspeed`;
  актуальный список: <https://platform.kimi.ai/docs/models>. Точка смены —
  функция `ИмяМодели()` общего модуля `KimiAPIСервер`.
- Проверка доступных моделей для конкретного ключа: `GET /v1/models`.
- Рассуждение (K3.1): всегда включено, отключить нельзя; управляется
  **верхнеуровневым** полем запроса `reasoning_effort` со значениями `low`,
  `high`, `max` (по умолчанию `max` — Дни 1–2 фактически работали с `max`,
  поле не передавали). В ответе отдельно приходит `reasoning_content`
  (цепочка рассуждения) рядом с `content`; токены рассуждения —
  `usage.completion_tokens_details.reasoning_tokens`. `temperature`/`top_p`
  фиксированы платформой — не отправлять.
- Управление форматом ответа (параметры запроса):
  - `max_tokens` (число) — ограничение длины ответа; малое значение обрезает вывод;
  - `stop` (строка или массив строк) — стоп-последовательность, генерация обрывается
    до её появления в тексте;
  - `response_format` — `{"type": "json_object"}` (JSON Mode: только валидный JSON,
    поля импровизируются; формат нужно описывать в промпте) или
    `{"type": "json_schema", "json_schema": {"name": ..., "strict": true, "schema": {...}}}`
    (Structured Output: структура строго по схеме; при `strict: true` схема должна
    соответствовать спецификации MFJS; семейство k3.1 держит надёжно, сложные
    конструкции `anyOf`/`$ref` не использовать). Руководство:
    <https://platform.kimi.ai/docs/guide/response_format>.
- В ответе полезны `choices[0].finish_reason` (`"length"` — обрезан по `max_tokens`)
  и `usage` (`prompt_tokens`, `completion_tokens`, `total_tokens`) — используем
  для статусной строки чата.
