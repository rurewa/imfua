# Полное руководство: Ollama + Open WebUI через Docker Compose с импортом модели из Hugging Face

Это руководство создаст полностью автономную локальную нейросеть, работающую без доступа к интернету после первичной настройки.

- > ## Для выполнения этого задания используйте WSL или любой другой Linux или Mac!
- > Никогда в разработке не используйте русские имена файлов и каталогов!
- > Никогда в разработке не используйте пробелы и спец.символы в именах файлов и каталогов!
- > Перед созданием проекта убедитесь, что порт, который собираетесь использовать не занят другим приложением!
- > Создание проекта лучше начать с "чистого листа", предварительно остановив и удалив все другие контейнеры и образы!

---

## 📋 Что мы создадим

*   **Контейнер `ollama`** — движок нейросети, хранит модели.
*   **Контейнер `open-webui`** — веб-интерфейс для общения с моделью.
*   **Доступ** через `http://localhost:81`.
*   **Модель** импортируется вручную из файла `.gguf`, скачанного с Hugging Face.

---

## Шаг 1. Подготовка рабочей папки

> *Примечание: Для выполнения используйте WSL (если вы на Windows) или любой дистрибутив Linux.*

Создайте отдельную папку для проекта и перейдите в неё:

```bash
mkdir -p ~/ollama-stack && cd ~/ollama-stack
```

---

## Шаг 2. Создание файла `docker-compose.yml`

Создайте файл `docker-compose.yml` в этой папке:

```bash
nano compose.yaml
```
Вставьте в него следующее содержимое:
```yaml
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
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_THREADS=6  # Оптимизация для i7-8550U (оставляем 2 потока системе)
    dns:
      - 8.8.8.8
      - 1.1.1.1
    networks:
      - ai-network

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
    networks:
      - ai-network

volumes:
  ollama_data:
  open-webui_data:

networks:
  ai-network:
    driver: bridge
```
*(Сохраните файл: `Ctrl+O`, `Enter`, `Ctrl+X` в nano. Примечание: строка `version` удалена, так как в современных Docker Compose она устарела и вызывает предупреждения).*

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

1. Откройте браузер и перейдите по прямой ссылке для скачивания (выберите одну):
   * **Qwen 2.5 (1.5B) - Рекомендуется:** [Скачать Qwen2.5-1.5B-Instruct-Q4_K_M.gguf](https://huggingface.co/bartowski/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/Qwen2.5-1.5B-Instruct-Q4_K_M.gguf)
   * **Llama 3.2 (1B):** [Скачать Llama-3.2-1B-Instruct-Q4_K_M.gguf](https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf)
2. Файл с расширением `.gguf` сохранится в папку `~/Downloads` (или `~/Загрузки`).

---

## Шаг 5. Копирование модели в контейнер Ollama

Скопируйте скачанный файл в папку `/tmp` внутри контейнера `ollama`:
```bash
# Если файл в ~/Downloads (английская версия):
docker cp ~/Downloads/Qwen2.5-1.5B-Instruct-Q4_K_M.gguf ollama:/tmp/light-model.gguf

# ИЛИ если файл в ~/Загрузки (русская версия):
docker cp ~/Загрузки/Qwen2.5-1.5B-Instruct-Q4_K_M.gguf ollama:/tmp/light-model.gguf
```
⚠️ *Замените имя файла на то, которое вы фактически скачали.*

---

## Шаг 6. Создание модели в Ollama

Теперь скажем Ollama собрать модель из этого файла с оптимальными настройками.

### 6.1. Создайте файл Modelfile внутри контейнера:
```bash
docker exec -it ollama sh -c 'cat > /tmp/Modelfile-light << EOF
FROM /tmp/light-model.gguf
PARAMETER num_ctx 8192
PARAMETER num_thread 6
EOF'
```

### 6.2. Соберите модель с понятным именем:
```bash
docker exec -it ollama ollama create qwen-1.5b-fast -f /tmp/Modelfile-light
```
*(Дождитесь сообщения `success`)*

### 6.3. Проверьте, что модель появилась:
```bash
docker exec -it ollama ollama list
```
Должен быть вывод вроде:
```text
NAME                     ID              SIZE      MODIFIED
qwen-1.5b-fast:latest    27e5ef3c667e    986 MB    3 seconds ago
```

---

## Шаг 7. Подключение модели в Open WebUI

1. Откройте браузер и перейдите по адресу: **[http://localhost:81](http://localhost:81)**
2. При первом входе создайте аккаунт администратора.
3. Нажмите кнопку **`+` (New Chat)** в левом верхнем углу (это критически важно для скорости!).
4. В выпадающем списке моделей сверху выберите **`qwen-1.5b-fast`**.
5. Напишите любое сообщение, например: *"Привет! Кто ты?"*

🎉 Готово! Вы общаетесь с локальной нейросетью.

---

## 🔧 Полезные команды на будущее

### Управление стеком (из папки `~/ollama-stack`):
```bash
docker compose stop          # Остановить все контейнеры
docker compose start         # Запустить снова
docker compose restart       # Перезапустить
docker compose logs -f       # Посмотреть логи
docker stats                 # Получить статистику по запущенным контейнерам (CPU/RAM)
docker compose down -v       # Полное удаление (контейнеры + volumes, т.е. ВСЕ данные)
```

### Управление моделями:
```bash
docker exec -it ollama ollama list                    # Список моделей
docker exec -it ollama ollama rm qwen-1.5b-fast       # Удалить модель
docker exec -it ollama rm /tmp/light-model.gguf /tmp/Modelfile-light  # Очистить временные файлы
```

### Добавление новой модели (повтор Шагов 4–6):
```bash
# 1. Скопировать новый файл
docker cp ~/Downloads/НоваяМодель.gguf ollama:/tmp/newmodel.gguf

# 2. Создать Modelfile
docker exec -it ollama sh -c 'echo "FROM /tmp/newmodel.gguf" > /tmp/Modelfile-new'

# 3. Собрать модель (имя должно быть только на латинице, с цифрами и дефисами!)
docker exec -it ollama ollama create my-new-model -f /tmp/Modelfile-new

# 4. Очистить временные файлы
docker exec -it ollama rm /tmp/newmodel.gguf /tmp/Modelfile-new
```

### Продвинутый Modelfile (с настройками):
Если хотите задать системный промпт или параметры генерации:
```bash
docker exec -it ollama sh -c 'cat > /tmp/Modelfile-adv << EOF
FROM /tmp/newmodel.gguf
PARAMETER temperature 0.7
PARAMETER num_ctx 4096
SYSTEM """Ты — полезный ассистент. Отвечай кратко и по делу."""
EOF'
docker exec -it ollama ollama create my-smart-model -f /tmp/Modelfile-adv
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
| Не хватает RAM / всё тормозит | Используйте модели поменьше: `Qwen2.5-1.5B` или `Llama-3.2-1B`. Всегда начинайте новый чат (`+`). |
| Ошибка `invalid model name` | Убедитесь, что имя модели содержит только латинские буквы, цифры, точки и дефисы (без пробелов и кириллицы). |


**Результат оформить в виде отдельного `README.md` со скриншотами результатов ваших действий и прислать ссылку преподавателю.**

> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!