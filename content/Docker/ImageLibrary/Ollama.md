## Запуск Ollama в контейнере

> Для выполнения этого задания возможно понадобится ВПН

> **Это задание лучше выполнять в WSL или в другом полноценном Linux!**

- > Никогда в разработке не используйте русские имена файлов и каталогов!
- > Никогда в разработке не используйте пробелы и спец.символы в именах файлов и каталогов!
- > Перед созданием проекта убедитесь, что порт 8084 не занят другим приложением!
- > Создание проекта лучше начать с "чистого листа", предварительно остановив и удалив все другие контейнеры и образы!

### Вариант 1. Ollama+phi4

Получить ollama:
```shell
docker run -d -v ollama_data:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```
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

---

### Вариант 2. Open WebUI и Ollama

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
[Открыть чат в браузере](http://localhost:8081)

Загрузить модель, например `llama3.2:1b`
```shell
docker exec -it open-webui ollama pull llama3.2:1b
```
показать установленные модели:
```shell
docker exec -it open-webui ollama list
```

### Вариант 3. Open WebUI и Ollama в двух контейнерах

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

- [Открываем в браузере http://localhost:81/](http://localhost:81/)

> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!


