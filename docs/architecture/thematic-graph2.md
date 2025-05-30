# Стратегия построения Тематического Графа и Словаря Тем для Jeeves

## Введение

Мы строим смысловую карту мира "Странника", в которой каждая тема, идея и фрагмент текста становятся узлами единой живой структуры. Это не просто база данных — это основа мышления и коммуникации внутри ассистента Дживса. Тематический Граф становится ядром, на которое опираются:

- обработка и хранение знаний;
    
- работа субличностей (Seeker, Learner, Analyst);
    
- навигация и генерация текстов;
    
- визуальный и когнитивный интерфейс.
    

### Почему DAG (направленный ациклический граф)?

- Он позволяет иметь **множественные родительские и дочерние связи** (в отличие от деревьев)
    
- Он даёт возможность строить **иерархии смыслов** без циклов и конфликтов
    
- Он масштабируется без потери структуры и может визуализироваться как **плоско**, так и **в гиперболическом пространстве**
    
- Он поддерживает **динамическую глубину**, что позволяет пользователю ориентироваться как по ширине, так и по смысловой глубине
    

### Пример DAG-графа тем

```
Мир Странника
├── Технологии
│   └── Подключение
│       └── Кресло
├── Воплощение
│   ├── Эссенция
│   └── Аватара
├── Альянсы
```

Здесь видно:

- У "Мира Странника" — три основные ветви
    
- "Кресло" вложено в "Подключение", а то — в "Технологии"
    
- "Эссенция" и "Аватара" — дочерние темы "Воплощения"
    

Эта структура:

- помогает анализировать глубину понятий
    
- позволяет визуализировать вложенность и соседство
    
- становится отправной точкой для поиска, вывода гипотез, генерации текста и подсказок Learner
    

Мы проектируем живую, развивающуюся структуру знаний, которая позволит Jeeves эффективно оперировать понятиями, документами, смысловыми связями и знаниями пользователя. В основе этой системы лежит **Тематический Граф** — сеть, построенная на принципах:

- **DAG (направленный ациклический граф)** — чтобы поддерживать вложенность, множественное наследование и смысловую иерархию
    
- **Мультипроходной архитектуры** — пошагового сбора, привязки, анализа и уточнения
    
- **Гиперболической визуализации (Poincaré-диск)** — для компактного и масштабируемого отображения структуры
    
- **Смены центра перспективы (объективный/субъективный режим)** — как основа интерфейса и мышления
    
- **Влияния на субличности Jeeves** — граф становится ядром, с которым работают модули Seeker, Learner, Analyst
    

---

## Шаг 1. Выявление тем и построение узлов

Все обнаруженные и созданные темы формируют так называемый **Словарь Тем** — централизованную структуру, в которой для каждой темы сохраняется её идентификатор, тип, источник, метаданные, связи и связанные чанки. Этот словарь:

- сохраняется в отдельном файле (или базе) и связан с Тематическим Графом;
    
- используется всеми модулями Дживса как основа для поиска, генерации и анализа;
    
- может быть обновлён автоматически или вручную при правке темы;
    
- является основой навигации и визуализации, предоставляя унифицированный доступ к понятийному содержанию знаний.
    

### A. Явные (ручные) темы

- Извлекаются из .md-документов, заголовков, ссылок, тегов
    
- Каждая такая тема становится узлом графа с типом `manual`
    

### B. Неявные темы

- Выявляются на основе эмбедингов чанков и анализа их смыслового сходства
    
- Такие темы создаются автоматически, но сохраняются как полноценные темы (тип: `discovered`), а не как временные кластеры
    
- В метаданных темы указывается её происхождение (явная или обнаруженная)
    
- Их можно уточнять, переименовывать или объединять вручную
    

> Процесс кластеризации тем в группы или выявления надтем будет описан отдельно — это не синоним "обнаружения темы", а надстройка над уже созданной структурой

#### Пример структуры узла темы:

```
{
  "id": "theme_42",
  "label": "Подключение",
  "type": "manual",
  "source": "Подключение.md",
  "embedding": [...],
  "chunks": ["chunk_15", "chunk_16"],
  "parents": ["Технологии"],
  "children": ["Кресло", "Аватара"],
  "related": ["Эссенция"]
}
```

---

## Шаг 2. Чанкинг и структура данных

- Документы нарезаются на чанки (см. `infest-chunking.md`) по логическим и смысловым границам
    
- Каждый чанк получает эмбединг (векторное представление смысла)
    
- Привязка чанков к темам осуществляется по ссылкам, тегам и близости эмбедингов
    

#### Пример структуры чанка:

```
{
  "id": "chunk_15",
  "text": "Кресло — это интерфейс между эссенцией и аватарой...",
  "embedding": [...],
  "source_doc": "Подключение.md",
  "linked_themes": ["Кресло", "Эссенция"]
}
```

---

## Шаг 3. Построение связей

Тематический Граф — это не просто набор тем и чанков, а прежде всего сеть их взаимосвязей. Связи между узлами могут быть различными по характеру, направлению и значению. Мы различаем несколько типов связей и используем разные подходы для их выявления и обработки.

### Типы связей:

#### 1. Горизонтальные связи

- Отражают **смысловую близость** тем
    
- Строятся на основе **косинусной близости эмбедингов** тем
    
- Часто связывают темы одного уровня или смежных категорий
    
- Могут использоваться для подсказок, синонимических переходов, ассоциативного поиска
    

##### Пример:

```
{
  "from": "Эссенция",
  "to": "Аватара",
  "type": "related",
  "weight": 0.83
}
```

#### 2. Вертикальные связи (вложенность)

- Отражают отношение **A ⊂ B** — одна тема логически входит в другую
    
- Строятся по совокупности критериев:
    
    - Доля пересекающихся чанков (`overlap = |A ∩ B| / |A|`)
        
    - Косинусная близость эмбедингов тем
        
    - Размер и плотность тем (например, тема B шире и охватывает больше чанков)
        
    - Контекстные метки и ссылки
        
- Используются для построения иерархии DAG и расчёта глубины
    
- Хранится вес `inclusion_score`, отражающий силу вложенности
    

##### Пример вертикальной связи:

```
{
  "from": "Кресло",
  "to": "Подключение",
  "type": "hierarchy",
  "subtype": "inclusion",
  "weight": 0.91
}
```

#### 3. Иерархия DAG

- Все вертикальные связи объединяются в **направленный ациклический граф**
    
- Используется `topological sort`, чтобы определить уровни вложенности
    
- Выявляются корни, листья, глубина, перекрытия и возможные циклы
    
- Learner предлагает оптимизации: объединение тем, уточнение вложенности, преобразование слабых связей в горизонтальные
    

#### 4. Пространственные связи

- Построены в гиперболическом пространстве (Poincaré-диск)
    
- Используются для визуализации: расстояние отражает уровень обобщения и логическую глубину
    
- Поддерживают переключение перспектив: от корневой темы к локальному фокусу
    

```
{
  "from": "Подключение",
  "to": "Технологии",
  "type": "hierarchy",
  "weight": 0.91
}
```

---

## Шаг 4. Динамическое смещение центра графа (режимы перспективы)

Тематический Граф поддерживает два режима навигации:

### Объективный режим

- Центр гиперболической визуализации зафиксирован на корневой теме (например, "Мир Странника")
    
- Показываются все узлы и связи по мере удаления от центра
    
- Полезен для анализа общей структуры, визуального картографирования знаний
    

### Субъективный режим

- Центр графа переносится на выбранную пользователем тему
    
- Граф пересчитывается: ближайшие — вложенные и родственные темы, дальние — контекст
    
- Полезен для исследования, генерации знаний, обнаружения лакун и гипотез Learner
    

> Переход между режимами меняет как визуальную перспективу, так и фокус аналитических запросов

---

## Шаг 5. Визуализация

Используется гиперболическая модель (Poincaré-диск):

- Центр = смысловой фокус
    
- Каждый слой = уровень вложенности
    
- Узлы ближе к центру — более обобщающие темы
    
- Узлы у границ — конкретные темы и документы
    

### Визуальные особенности:

- **Цвет/размер** — по глубине или весу темы
    
- **Хлебные крошки** — показывают путь от центра до текущего узла
    
- **Динамическая навигация** — смена центра без пересборки графа
    

---

## Интерфейсные элементы

Чтобы взаимодействие с Тематическим Графом было интуитивным и продуктивным, реализуется ряд интерфейсных элементов, отражающих структуру, глубину и контекст текущего положения пользователя.

### Хлебные крошки (breadcrumb)

- Показывают путь от текущей темы к корню графа (например: `Мир Странника > Технологии > Подключение > Кресло`)
    
- Позволяют быстро подняться на любой уровень
    
- Используются и в объективном, и в субъективном режимах
    

### Подсветка активного узла

- Текущая тема (центр субъективного режима) всегда визуально выделена
    
- Связанные темы (родители, потомки, соседи) подчеркиваются мягким свечением или цветом
    

### Панель контекста темы

- Показывает:
    
    - Название темы
        
    - Тип (ручная, обнаруженная, объединённая)
        
    - Краткое описание
        
    - Ссылки на связанные документы и чанки
        
    - Родительские и дочерние связи
        
- Используется как навигационная панель и интерфейс уточнения структуры
    

### Фильтрация и управление видимостью

- Возможность скрыть или показать:
    
    - Автоматически сгенерированные темы
        
    - Темы выше/ниже определённого уровня
        
    - Темы без связанных чанков
        

### Интерактивная панель смены центра

- Кнопка "Сделать центром"
    
- Перемещает выбранную тему в центр визуализации (субъективный режим)
    
- Обновляет панель связей и слои вокруг
    

---

## Заключение

Эта архитектура делает Тематический Граф не только основой навигации и генерации знаний, но и ключевым источником **поведенческих стратегий Дживса**, структурируя взаимодействие между Seeker, Analyst и Learner.

Хочешь — добавим следующий раздел про работу каждого из модулей с этой структурой?