# Полное руководство: Ollama + Open WebUI через Docker Compose с импортом модели из Hugging Face

Это руководство создаст полностью автономную локальную нейросеть, работающую без доступа к интернету после первичной настройки.

---

## 📋 Что мы создадим

*   **Контейнер `ollama`** — движок нейросети, хранит модели
*   **Контейнер `open-webui`** — веб-интерфейс для общения с моделью
*   **Доступ** через `http://localhost:81`
*   **Модель** импортируется вручную из файла `.gguf`, скачанного с Hugging Face

---

## Шаг 1. Подготовка рабочей папки

Создайте отдельную папку для проекта и перейдите в неё:

```bash
mkdir -p ~/ollama-stack && cd ~/ollama-stack
```

---

## Шаг 2. Создание файла `docker-compose.yml`

Создайте файл `docker-compose.yml` в этой папке:

```bash
nano docker-compose.yml
```

Вставьте в него следующее содержимое:

```yaml
version: '3.8'

services:
  # 1. Движок Ollama
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: always
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    dns:
      - 8.8.8.8
      - 1.1.1.1
    environment:
      - OLLAMA_HOST=0.0.0.0

  # 2. Веб-интерфейс Open WebUI
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: always
    ports:
      - "81:8080"
    volumes:
      - open-webui_data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    depends_on:
      - ollama

volumes:
  ollama_data:
  open-webui_data:
```

Сохраните файл (`Ctrl+O`, `Enter`, `Ctrl+X` в nano).

### Что здесь важно:
*   **`dns: 8.8.8.8`** — на случай, если в будущем захотите скачать модель через `ollama pull`
*   **`ports: "81:8080"`** — порт 81 снаружи, 8080 внутри (т.к. порт 80 у вас занят)
*   **`http://ollama:11434`** — внутренняя сеть Docker, контейнеры видят друг друга по именам сервисов
*   **Именованные volumes** — данные сохраняются между перезапусками

---

## Шаг 3. Запуск стека контейнеров

В папке `~/ollama-stack` выполните:

```bash
docker compose up -d
```

Дождитесь, пока оба контейнера скачаются и запустятся. Проверьте статус:

```bash
docker compose ps
```

Оба контейнера должны иметь статус `Up` (или `Up (healthy)` для open-webui).

---

## Шаг 4. Скачивание модели с Hugging Face

1. Откройте браузер и перейдите на страницу с квантованными моделями. Например:
   * **Llama 3.2 (3B):** https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/tree/main
   * **Qwen 2.5 (7B):** https://huggingface.co/bartowski/Qwen2.5-7B-Instruct-GGUF/tree/main

2. Скачайте файл с расширением `.gguf`. Рекомендуется вариант **`Q4_K_M`** — оптимальный баланс качества и скорости. Например:
   ```
   Llama-3.2-3B-Instruct-Q4_K_M.gguf
   ```

3. Файл сохранится в папку `~/Downloads` (или `~/Загрузки`).

---

## Шаг 5. Копирование модели в контейнер Ollama

Скопируйте скачанный файл в папку `/tmp` внутри контейнера `ollama`:

```bash
# Если файл в ~/Downloads
docker cp ~/Downloads/Llama-3.2-3B-Instruct-Q4_K_M.gguf ollama:/tmp/model.gguf

# ИЛИ если файл в ~/Загрузки
docker cp ~/Загрузки/Llama-3.2-3B-Instruct-Q4_K_M.gguf ollama:/tmp/model.gguf
```

⚠️ Замените имя файла на то, которое вы скачали.

---

## Шаг 6. Создание модели в Ollama

Теперь скажем Ollama собрать модель из этого файла.

### 6.1. Создайте файл Modelfile внутри контейнера:

```bash
docker exec -it ollama sh -c 'echo "FROM /tmp/model.gguf" > /tmp/Modelfile'
```

### 6.2. Соберите модель с именем `my-llama3.2`:

```bash
docker exec -it ollama ollama create my-llama3.2 -f /tmp/Modelfile
```

Вы увидите процесс сборки:
```
transferring model data
using existing layer sha256:...
creating new layer sha256:...
creating new metadata layer
writing manifest
success
```

### 6.3. Проверьте, что модель появилась:

```bash
docker exec -it ollama ollama list
```

Должен быть вывод вроде:
```
NAME                  ID              SIZE      MODIFIED
my-llama3.2:latest    56b0dceba6e9    2.0 GB    1 minute ago
```

---

## Шаг 7. Подключение модели в Open WebUI

1. Откройте браузер и перейдите по адресу: **http://localhost:81**
2. При первом входе создайте аккаунт администратора.
3. В выпадающем списке моделей сверху выберите **`my-llama3.2`**.
4. Напишите любое сообщение, например: *"Привет! Кто ты?"*

🎉 Готово! Вы общаетесь с локальной нейросетью.

---

## 🔧 Полезные команды на будущее

### Управление стеком (из папки `~/ollama-stack`):

```bash
# Остановить все контейнеры
docker compose stop

# Запустить снова
docker compose start

# Перезапустить
docker compose restart

# Посмотреть логи
docker compose logs -f

# Полное удаление (контейнеры + volumes, т.е. ВСЕ данные)
docker compose down -v
```

### Управление моделями:

```bash
# Список моделей
docker exec -it ollama ollama list

# Удалить модель
docker exec -it ollama ollama rm my-llama3.2

# Очистить временные файлы после импорта
docker exec -it ollama rm /tmp/model.gguf /tmp/Modelfile
```

### Добавление новой модели (повтор Шагов 4–6):

```bash
# 1. Скопировать новый файл
docker cp ~/Downloads/НоваяМодель.gguf ollama:/tmp/newmodel.gguf

# 2. Создать Modelfile
docker exec -it ollama sh -c 'echo "FROM /tmp/newmodel.gguf" > /tmp/Modelfile'

# 3. Собрать модель
docker exec -it ollama ollama create моя-новая-модель -f /tmp/Modelfile

# 4. Очистить временные файлы
docker exec -it ollama rm /tmp/newmodel.gguf /tmp/Modelfile
```

### Продвинутый Modelfile (с настройками):

Если хотите задать системный промпт или параметры генерации, создайте Modelfile так:

```bash
docker exec -it ollama sh -c 'cat > /tmp/Modelfile << EOF
FROM /tmp/model.gguf

PARAMETER temperature 0.7
PARAMETER num_ctx 4096

SYSTEM """Ты — полезный ассистент. Отвечай кратко и по делу."""
EOF'
```

---

## 📁 Где физически лежат данные

*   **Модели Ollama:** `/var/lib/docker/volumes/ollama-stack_ollama_data/_data/`
*   **Данные Open WebUI (чаты, юзеры):** `/var/lib/docker/volumes/ollama-stack_open-webui_data/_data/`

Эти тома сохраняются между перезапусками и пересозданиями контейнеров.

---

## ⚠️ Возможные проблемы и решения

| Проблема | Решение |
|---|---|
| `open-webui` не видит модель | `docker restart open-webui` + `Ctrl+F5` в браузере |
| Порт 81 занят | Замените `"81:8080"` на `"82:8080"` в `docker-compose.yml` |
| Контейнер `ollama` не стартует | `docker compose logs ollama` — посмотрите причину |
| Не хватает RAM для модели | Используйте модели поменьше: `Qwen2.5-1.5B` или `Llama-3.2-1B` |

---

Готово! Теперь у вас есть полностью документированная, воспроизводимая и автономная локальная нейросеть. 🚀