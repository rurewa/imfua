## Alt Linux в Docker

- > ## Для выполнения этого задания используйте WSL или любой другой Linux!

Выполните все этапы работы с проектом по примеру с [Nginx](/content/Docker/ImageLibrary/Nginx.md)

- > Никогда в разработке не используйте русские имена файлов и каталогов!
- > Никогда в разработке не используйте пробелы и спец.символы в именах файлов и каталогов!
- > Перед созданием проекта убедитесь, что порт, который собираетесь использовать не занят другим приложением!
- > Создание проекта лучше начать с "чистого листа", предварительно остановив и удалив все другие контейнеры и образы!

### 1. Подготовка

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
- [Подробней про управление Docker в шпаргалке](/content/Docker/DockerReminder.md)

***

Загрузить готовый образ Alt
```shell
docker pull alt:sisyphus
```
Запустить и использовать
```shell
docker run -ti --rm --name alt alt:sisyphus /bin/bash
```
Установить приложение Fastfetch в контейнере
```shell
apt-get update && apt-get install fastfetch
```
Запустить Fastfetch
```shell
fastfetch
```
Выйти из контейнера с Alt
```shell
exit
```

**Результат оформить в виде отдельного `README.md` со скриншотами результатов ваших действий и прислать ссылку преподавателю.**

### Полезные ссылки

- [alt Docker Official Image](https://hub.docker.com/_/alt/)
- [Dockerfile](https://github.com/alt-cloud/docker-brew-alt/blob/p10/x86_64/Dockerfile)
- [Docker Alt Linux Image](https://github.com/sibsau/docker-alt/blob/master/README.md)

> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!
