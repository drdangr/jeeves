# Стратегия чанкирования для документов базы знаний

## Введение

Разработка стратегии чанкирования направлена на обеспечение семантической целостности данных, удобства поиска и эффективного хранения информации. Важнейшими принципами являются:

- Сохранение логических блоков.
    
- Минимизация дублирования информации.
    
- Поддержание контекста ключевых терминов и тегов.
    
- Адаптивность параметров в зависимости от структуры и размера документа.
    

---

## Категоризация документов

|Категория|Размер (символы)|Стратегия|
|---|---|---|
|Микро|< 800|Не чанкировать|
|Малые|800–2000|2–3 чанка, мягкое перекрытие|
|Средние|2000–5000|Стандартное чанкирование|
|Большие|> 5000|Стандартное чанкирование + постобработка на эмбеддингах|

---

# Алгоритм выбора стратегии

```python
def smart_chunking(text, filename):
    doc_stats = analyze_document(text)

    if doc_stats['length'] < 800:
        return create_whole_document_chunk(text, filename)
    elif doc_stats['length'] < 2000:
        return create_small_document_chunks(text, filename, doc_stats)
    elif doc_stats['length'] <= 5000:
        return create_standard_document_chunks(text, filename, doc_stats)
    else:
        return create_big_document_chunks(text, filename, doc_stats)
```

---

# Методы чанкирования

## 1. Чанкирование микро документов

```python
def create_whole_document_chunk(text, filename):
    return [enrich_chunk(text, filename, text, 0)]
```

- Весь документ становится одним чанком.
    
- Обогащение метаданными.
    

---

## 2. Чанкирование малых документов (800–2000 символов)

```python
def create_small_document_chunks(text, filename, stats):
    chunks = simple_chunk_split(text)
    chunks = handle_tag_block(chunks, text)
    return [enrich_chunk(chunk, filename, text, i) for i, chunk in enumerate(chunks)]
```

- Разделение на 2–3 чанка.
    
- Мягкое перекрытие.
    
- Обработка блоков тегов.
    
- Обогащение метаданными.
    

---

## 3. Чанкирование средних документов (2000–5000 символов)

```python
def create_standard_document_chunks(text, filename, stats):
    chunks = advanced_chunk_split(text)
    chunks = handle_tag_block(chunks, text)
    return [enrich_chunk(chunk, filename, text, i) for i, chunk in enumerate(chunks)]
```

- Продвинутое разбиение по структуре и смысловым блокам.
    
- Учет абзацев, терминов, маркдауна.
    
- Обогащение чанков метаданными.
    

---

## 4. Чанкирование больших документов (>5000 символов)

```python
def create_big_document_chunks(text, filename, stats):
    chunks = advanced_chunk_split(text)
    chunks = semantic_chunking_postprocess(chunks)
    chunks = handle_tag_block(chunks, text)
    return [enrich_chunk(chunk, filename, text, i) for i, chunk in enumerate(chunks)]
```

- Стандартное продвинутое разбиение.
    
- Постобработка на эмбеддингах для объединения логически связанных чанков.
    
- Обогащение метаданными.
    

---

# Базовые подфункции

## Простое разбиение на чанки

```python
def simple_chunk_split(text):
    """Простое разбиение текста на чанки по абзацам и предложениям с мягким перекрытием."""
    import re

    # Этап 1: Делим текст по двойным переносам строк (абзацы)
    paragraphs = re.split(r'\n\s*\n', text.strip())

    # Этап 2: Склеиваем абзацы в чанки целевого размера
    chunks = []
    current_chunk = ""
    target_size = 800  # Целевой размер чанка
    overlap_size = 100  # Перекрытие между чанками

    for para in paragraphs:
        para = para.strip()
        if not para:
            continue
        if len(current_chunk) + len(para) + 2 <= target_size:
            current_chunk += "\n\n" + para if current_chunk else para
        else:
            # Завершаем текущий чанк
            if current_chunk:
                chunks.append(current_chunk.strip())
            # Стартуем новый
            current_chunk = para

    if current_chunk:
        chunks.append(current_chunk.strip())

    # Этап 3: Добавляем перекрытие между чанками
    overlapped_chunks = []
    for i, chunk in enumerate(chunks):
        if i > 0:
            # Добавляем перекрытие из конца предыдущего чанка
            overlap = chunks[i-1][-overlap_size:].strip()
            chunk = overlap + "\n\n" + chunk
        overlapped_chunks.append(chunk.strip())

    return overlapped_chunks

```

## Продвинутое разбиение на чанки

```python
def advanced_chunk_split(text):
    """Продвинутое разбиение текста на чанки с учётом структуры и маркдауна."""
    import re

    # Этап 1: Разделяем текст по двойным переносам строк (основные абзацы)
    paragraphs = re.split(r'\n\s*\n', text.strip())

    # Этап 2: Учитываем специальные структуры Markdown (списки, цитаты, заголовки)
    refined_paragraphs = []
    buffer = ""

    for para in paragraphs:
        if para.startswith(('-', '*', '+', '>')) or re.match(r'^#{1,6}\s', para):
            # Это элемент списка или заголовок — отдельный блок
            if buffer:
                refined_paragraphs.append(buffer.strip())
                buffer = ""
            refined_paragraphs.append(para.strip())
        else:
            # Иначе накапливаем абзацы в буфер
            if buffer:
                buffer += "\n\n" + para
            else:
                buffer = para

    if buffer:
        refined_paragraphs.append(buffer.strip())

    # Этап 3: Склеиваем параграфы в чанки определённого размера
    chunks = []
    current_chunk = ""
    target_size = 800  # Прицеливаемся на размер чанка ~800 символов

    for para in refined_paragraphs:
        if len(current_chunk) + len(para) + 2 <= target_size:
            current_chunk += "\n\n" + para if current_chunk else para
        else:
            if current_chunk:
                chunks.append(current_chunk.strip())
            current_chunk = para

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks

```

## Обработка блоков тегов

```python
def handle_tag_block(chunks, text):
    """Обеспечивает целостную обработку блока тегов в конце документа."""
    import re

    # Пытаемся найти блок тегов в конце текста
    tag_block_match = re.search(r'(?:#\w+\s*)+$', text.strip())
    if not tag_block_match:
        return chunks

    tag_block = tag_block_match.group(0)

    # Ищем, содержится ли блок тегов отдельно в каком-то чанке
    for idx, chunk in enumerate(chunks):
        if chunk.strip() == tag_block.strip():
            # Присоединяем блок тегов к предыдущему чанку
            if idx > 0:
                chunks[idx-1] += '\n' + chunks[idx]
                del chunks[idx]
            break

    return chunks

```

## Семантическое объединение чанков

```python
def semantic_chunking_postprocess(chunks):
    embeddings = embedding_model.encode([chunk['text'] for chunk in chunks])
    merged_chunks = []
    buffer = chunks[0]['text']
    for i in range(1, len(chunks)):
        similarity = cosine_similarity(embeddings[i-1], embeddings[i])
        if similarity > THRESHOLD:
            buffer += ' ' + chunks[i]['text']
        else:
            merged_chunks.append(buffer)
            buffer = chunks[i]['text']
    merged_chunks.append(buffer)
    return [{'text': chunk} for chunk in merged_chunks]
```

## Обогащение метаданными

```python
def enrich_chunk(chunk, filename, full_text, chunk_index):
    """Расширенное обогащение чанка метаданными"""
    import re

    # Базовые метаданные
    metadata = {
        "text": chunk,
        "source": filename,
        "chunk_index": chunk_index,
        "position": {
            "start": full_text.find(chunk),
            "end": full_text.find(chunk) + len(chunk)
        }
    }

    # Извлечение вики-ссылок
    wiki_links = re.findall(r'\[\[(.*?)\]\]', chunk)

    # Извлечение неразмеченных терминов (ключевые слова)
    unmarked_terms = extract_key_terms(chunk, wiki_links)

    # Извлечение тегов из чанка
    chunk_tags = re.findall(r'#(\w+)', chunk)

    # Если тегов нет в чанке, но они есть в документе — наследуем
    if not chunk_tags and "#" in full_text:
        doc_tags = re.findall(r'#(\w+)', full_text)
        chunk_tags = doc_tags

    # Обогащаем метаданные
    metadata["links"] = wiki_links
    metadata["terms"] = unmarked_terms
    metadata["tags"] = chunk_tags

    return metadata

```

---

## Извлечение неразмеченных ключевых терминов

```
def extract_key_terms(text, known_links):
    """Извлекает ключевые термины, не оформленные как вики-ссылки"""
    import re

    # Извлекаем потенциальные термины: слова с заглавной буквы
    potential_terms = re.findall(r'\b([А-Я][а-я]+)\b', text)

    # Получаем список допустимых терминов из базы знаний
    key_terms_db = get_key_terms_database()

    # Фильтруем: оставляем только те термины, которые есть в базе и не входят в вики-ссылки
    extracted_terms = [
        term for term in potential_terms
        if term in key_terms_db and not any(term in link for link in known_links)
    ]

    return extracted_terms


```

---

## Перспективы развития

- Динамическое управление длиной чанков по плотности информации.
    
- Кластеризация чанков на тематические группы.
    
- Расширение обработки событийной разметки (TimeLine).
    
- Параллельная обработка для ускорения работы на больших базах.
    

---