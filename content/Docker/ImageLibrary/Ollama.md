## Запуск Ollama в контейнере

> Для выполнения этого задания возможно понадобится ВПН

- > ## Для выполнения этого задания используйте WSL или любой другой Linux или Mac!

- > Никогда в разработке не используйте русские имена файлов и каталогов!
- > Никогда в разработке не используйте пробелы и спец.символы в именах файлов и каталогов!
- > Перед созданием проекта убедитесь, что порт, который собираетесь использовать не занят другим приложением!
- > Создание проекта лучше начать с "чистого листа", предварительно остановив и удалив все другие контейнеры и образы!

> ### Можно выполнить один из нескольких вариантов

### Подготовка

> **Создание проекта лучше начать с "чистого листа", предварительно остановив и удалив все другие контейнеры и образы!**

Перед выполнением этого задания рекомендуется завершить и удалить все имеющиеся учебные контейнеры:

Остановить все запущенные контейнеры:
```shell
docker stop $(docker ps -q)
```
Удалить все остановленные или не запущенные (Exited, Created) учебные контейнеры:
```shell
docker container prune
```
> **Prune** — удаляйте ненужные контейнеры, чтобы не засорять ваш Docker!

Или так удалить все учебные контейнеры (включая запущенные — принудительно):
```shell
docker rm -f $(docker ps -aq)
```
Очистить все ненужные (учебные) тома:
```shell
docker volume prune -a
```
Опционально: можно удалить образы и кеш сборок в Docker

Удаление всех образов
```shell
docker image prune -a
```
Очистить кэш всех сборок:
```shell
docker builder prune -a
```

- [Подробней про управление Docker в шпаргалке](/content/Docker/DockerReminder.md)

***

### Вариант 1. Ollama+phi4

Получить ollama:
```shell
docker run -d -v ollama_data:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```
Запустите эту нейросеть и задайте ей вопрос: "Что ты можешь?"
```shell
docker exec -it ollama ollama run phi4
```
выйти из командного режима нейросети:
```shell
/exit
```
и
```shell
/bye
```

[Если у вас нет ВПН, скачайте модель вручную](https://huggingface.co/bartowski/microsoft_Phi-4-mini-instruct-GGUF/blob/main/microsoft_Phi-4-mini-instruct-Q4_K_M.gguf)

Скопируйте скачанный файл в папку `/tmp` внутри контейнера `ollama`:
```shell
# Если файл в ~/Downloads (английская версия):
docker cp ~/Downloads/microsoft_Phi-4-mini-instruct-Q4_K_M.gguf ollama:/tmp/light-model.gguf
```
```shell
# ИЛИ если файл в ~/Загрузки (русская версия):
docker cp ~/Загрузки/microsoft_Phi-4-mini-instruct-Q4_K_M.gguf ollama:/tmp/light-model.gguf
```
⚠️ *Замените имя файла на то, которое вы фактически скачали.*

Войдите в контейнер:
```shell
docker exec -it ollama /bin/bash
```
Создайте файл Modelfile:
```shell
cat > /tmp/Modelfile << 'EOF'
FROM /tmp/light-model.gguf

SYSTEM """You are Phi-4-mini, a helpful AI assistant."""

PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER num_ctx 16384

TEMPLATE """<|system|>
{{ .System }}<|end|>
<|user|>
{{ .Prompt }}<|end|>
<|assistant|>
{{ .Response }}<|end|>"""
EOF
```
Создайте модель в Ollama:
```shell
ollama create phi4-mini-custom -f /tmp/Modelfile
```
Проверка доступных моделей:
```shell
docker exec -it ollama ollama list
```
Запустите модель
```shell
docker exec -it ollama ollama run phi4-mini-custom
```
Появится чат диалога с нейросетью, в котором вы зададите вопрос:
```text
What can you do?
```
или по-русски:
```
Ты умеешь по-русски?
```
выйти из диалога:
```shell
/exit
```
и
```shell
/bye
```

---

### Вариант 2. Ollama + Open WebUI

```shell
docker run -d -p 3000:8081 -v ollama:/root/.ollama -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:ollama
```
[Открыть чат в браузере](http://localhost:3000)
или
[Открыть чат в браузере](http://0.0.0.0:3000)

или через другой порт:
```shell
docker run -d -p 8081:8080 \
  -v ollama:/root/.ollama \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:ollama
```
Загрузить модель, например `llama3.2:1b`
```shell
docker exec -it open-webui ollama pull llama3.2:1b
```
показать установленные модели:
```shell
docker exec -it open-webui ollama list
```
[Открыть чат в браузере](http://localhost:8081)

Укажите e-mail и придумайте пароль для локального личного кабинета Open WebUI

Задайте ей вопрос: "Что ты можешь?"

---

### Вариант 3. Ollama + Open WebUI в двух контейнерах

Остановим и удалим контейнер с 2 вариантом Open WebUI
```shell
docker stop open-webui
docker rm open-webui
```
Запускаем "мозги" (Ollama)
```shell
docker run -d \
  --name ollama \
  --restart always \
  -v ollama:/root/.ollama \
  ollama/ollama:latest
```
Устанавливаем веб-интерфейс Open WebUI
```shell
docker run -d \
  -p 81:8080 \
  --name open-webui \
  --restart always \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://ollama:11434 \
  --link ollama \
  ghcr.io/open-webui/open-webui:main
```
Скачать модель
```shell
docker exec -it ollama ollama pull llama3.2
```
Посмотреть список скачанных моделей
```shell
docker exec -it ollama ollama list
```
Удалить модель (если заняла много места)
```shell
docker exec -it ollama ollama rm llama3.2
```
Если модель не установиться и если у вас нет ВПН, то можно выполнить импорт модели через **Hugging Face**

https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/tree/main?spm=a2ty_o01.29997173.0.0.7f59c921DR5mIT


- Перейдите на сайт Hugging Face. Для Llama 3.2 отлично подходят квантованные версии от пользователя bartowski.
- Ссылка для Llama 3.2 (3B): (https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/tree/main?spm=a2ty_o01.29997173.0.0.7f59c921DR5mIT)
- Ссылка для Qwen 2.5 (7B): (https://huggingface.co/bartowski/Qwen2.5-7B-Instruct-GGUF/tree/main?spm=a2ty_o01.29997173.0.0.7f59c921DR5mIT)
- Найдите файл с окончанием `Q4_K_M.gguf` (это оптимальный баланс скорости и качества). Например: `Llama-3.2-3B-Instruct-Q4_K_M.gguf`.
- Скачайте его на свой компьютер (в папку Загрузки / Downloads).
- Скопируйте файл внутрь контейнера Docker

Откройте терминал Linux и выполните команду (замените имя файла на то, которое вы скачали):

# Копируем файл с хоста в папку /tmp внутри контейнера ollama
```shell
docker cp ~/Загрузки/Llama-3.2-3B-Instruct-Q4_K_M.gguf ollama:/tmp/model.gguf
```
Теперь скажем Ollama собрать модель из этого файла. Выполните прямо в терминале:
```shell
docker exec -it ollama sh -c 'echo "FROM /tmp/model.gguf" > /tmp/Modelfile'
```
и
```shell
docker exec -it ollama ollama create my-llama3.2 -f /tmp/Modelfile
```
Вы увидете примерно это:
```
gathering model components
copying file sha256:6c1a2b41161032677be168d354123594c0e6e67d2b9227c84f296ad037c728ff 100%
parsing GGUF
using existing layer sha256:6c1a2b41161032677be168d354123594c0e6e67d2b9227c84f296ad037c728ff
writing manifest
success
```

- [Открываем в браузере http://localhost:81/](http://localhost:81/)

Укажите e-mail и придумайте пароль для локального личного кабинета Open WebUI

Задайте ей вопрос: "Что ты можешь?"

Если локальная модель не доступна, то приступайте с шагу ниже.

### Опционально: полная очистка Open WebUI (безопасная)

1. Останавливаем и удаляем контейнер
```shell
docker stop open-webui
docker rm open-webui
```

2. УДАЛЯЕМ том с данными (именно это сбросит кэш и настройки)
```shell
docker volume rm open-webui
```

3. Запускаем контейнер заново с чистого листа
```shell
docker run -d \
  -p 81:8080 \
  --name open-webui \
  --restart always \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://ollama:11434 \
  --link ollama \
  ghcr.io/open-webui/open-webui:main
```
Подождите 20–30 секунд, чтобы база данных внутри контейнера успела полностью проинициализироваться.

[Снова перейдите по адресу: http://localhost:81](http://localhost:81)

Задайте ей вопрос: "Что ты можешь?"

**Результат оформить в виде отдельного `README.md` со скриншотами результатов ваших действий и прислать ссылку преподавателю.**

> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!


