# Graph: структура и стратегия создания графа связей

## Модель двухуровневого графа

### Стандартизация идентификаторов

- Узлы документов имеют ID в виде пути или имени файла без расширения.
    
- Узлы тегов имеют ID вида `tag:название_тега`.
    
- Все ID нормализуются (без `.md`, единый регистр).
    

### Типы узлов

- **Документы** — основные узлы, представляющие файлы `.md`
    
- **Теги** — специальные узлы, связывающие документы по тематике
    
- **Документы** — основные узлы, представляющие файлы `.md`
    
- **Теги** — специальные узлы, связывающие документы по тематике
    

### Типы связей

- **Прямые** (`document → document`) — созданные через `[[кросс-ссылки]]`
    
- **Тематические** (`document → tag`) — созданные через хэштеги `#тег`
    

---

## Алгоритм создания графа

### 1. Инициализация структуры

```python
graph = {
  "nodes": [],     # Узлы (документы и теги)
  "edges": [],     # Связи между узлами
  "metadata": {}   # Метаданные графа (дата создания, статистика)
}
```

### 2. Проход по документам

Перед обработкой каждый документ и его ссылки нормализуются:

- Ссылки `[[...]]` очищаются от `.md`, приводятся к единому формату ID.
    
- Теги выделяются отдельно и нормализуются.
    

```python
for document in documents:
    # Добавление узла документа
    doc_node = {
        "id": document.path,
        "type": "document",
        "title": document.title or document.filename,
        "meta": {
            "last_modified": document.last_modified,
            "size": document.size
        }
    }
    graph["nodes"].append(doc_node)

    # Обработка кросс-ссылок
    for link in extract_wiki_links(document.content):
        edge = {
            "source": document.path,
            "target": resolve_link_path(link),
            "type": "link",
            "weight": 1
        }
        add_or_update_edge(graph["edges"], edge)

    # Обработка тегов
    for tag in extract_tags(document.content):
        if not node_exists(graph["nodes"], f"tag:{tag}"):
            tag_node = {
                "id": f"tag:{tag}",
                "type": "tag",
                "title": tag,
                "count": 1
            }
            graph["nodes"].append(tag_node)
        else:
            increment_tag_count(graph["nodes"], f"tag:{tag}")

        edge = {
            "source": document.path,
            "target": f"tag:{tag}",
            "type": "has_tag"
        }
        add_or_update_edge(graph["edges"], edge)
```

### 3. Дополнительный анализ

#### Расчёт важности узлов

Дополнительно к расчёту PageRank для оценки связности, каждый узел получает поле `link_weight`, отражающее общее количество входящих ссылок (как через кросс-ссылки, так и через связи с тегами):

- Каждая прямая ссылка `[[...]]` увеличивает вес на 1
    
- Каждая связь с тегом `#тег` увеличивает вес на 1
    

Итоговый вес используется для визуализации и может учитываться при ранжировании результатов.

```python
# Расчёт важности узлов через PageRank
import networkx as nx

G = nx.DiGraph()
for node in graph["nodes"]:
    G.add_node(node["id"])

for edge in graph["edges"]:
    if edge["type"] == "link":
        G.add_edge(edge["source"], edge["target"], weight=edge["weight"])

pageranks = nx.pagerank(G)

for node in graph["nodes"]:
    if node["id"] in pageranks:
        node["pagerank"] = pageranks[node["id"]]
```

---

## Хранение и обновление

### Формат хранения

```python
import json

with open("knowledge_graph.json", "w") as f:
    json.dump(graph, f, indent=2)
```

### Процесс обновления

```python
def update_graph(knowledge_base_path, changed_documents):
    graph = load_graph_from_file(f"{knowledge_base_path}/knowledge_graph.json")

    for doc in changed_documents:
        remove_document_from_graph(graph, doc.path)

    for doc in changed_documents:
        add_document_to_graph(graph, doc)

    recalculate_metrics(graph)

    save_graph(graph, f"{knowledge_base_path}/knowledge_graph.json")
```

---

## Интеграция с модулями

### Seeker

- Улучшение ранжирования результатов с использованием PageRank
    
- Фильтрация по тегам через связи графа
    

### Interface

- Прямая визуализация графа через D3.js
    
- Переключение режимов отображения (только документы, документы + теги, фокус на узле)
    

### Analyst

- Выявление кластеров и центральных концепций
    
- Поиск слабосвязанных зон базы знаний
    

---

## Пример использования графа в Seeker

```python
def search_with_graph(query, knowledge_base_path, top_k=10):
    basic_results = vector_search(query, top_k=top_k*2)
    graph = load_graph_from_file(f"{knowledge_base_path}/knowledge_graph.json")

    for result in basic_results:
        node = find_node_by_id(graph, result.document_id)
        if node and "pagerank" in node:
            result.score *= (1 + node["pagerank"] * 0.5)

    return sorted(basic_results, key=lambda x: x.score, reverse=True)[:top_k]
```

---

## Интеграция с обновлением базы знаний

- Обновление графа синхронизировано с процессом инжеста
    
- Граф загружается из файла при старте приложения или по запросу
    
- Перерасчёт метрик выполняется один раз после обновления
    

---

## Пример структуры графа связей

```json
{
  "nodes": [
    {
      "id": "Эссенция.md",
      "type": "document",
      "title": "Эссенция",
      "meta": {
        "last_modified": "2023-12-10T12:34:56Z"
      },
      "pagerank": 0.0532
    },
    {
      "id": "tag:эссенция",
      "type": "tag",
      "title": "эссенция",
      "count": 12
    }
  ],
  "edges": [
    {
      "source": "Эссенция.md",
      "target": "Апостолы.md",
      "type": "link",
      "weight": 2
    },
    {
      "source": "Эссенция.md",
      "target": "tag:эссенция",
      "type": "has_tag"
    }
  ],
  "metadata": {
    "created": "2023-12-15T08:00:00Z",
    "last_updated": "2023-12-17T14:30:00Z",
    "document_count": 150,
    "tag_count": 45,
    "link_count": 312
  }
}
```

---

## Перспективы развития

- Поддержка динамического добавления новых слоёв связей (например, событийная корреляция)
    
- Интеграция поиска ближайших соседей по графу для рекомендаций
    
- Расширение графовой аналитики для выявления тематических пустот