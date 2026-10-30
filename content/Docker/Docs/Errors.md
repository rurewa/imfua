## Различные ошибки работы Docker

Ошибка:
```shell
Unable to find image 'mysql:latest' locally docker: Error response from daemon: failed to resolve reference "docker.io/library/mysql:latest": failed to authorize: failed to fetch oauth token: Post "https://auth.docker.io/token": net/http: TLS handshake timeout
```

Решение:

Скопируйте весь блок кода ниже и замените им всё содержимое окна на вашем скриншоте. Я добавил запятые в нужных местах и включил параметры registry-mirrors и mtu:
```json
Plain Text
{  "builder": {    "gc": {      "defaultKeepStorage": "20GB",      "enabled": true    }  },  "experimental": false,  "registry-mirrors": [    "https://docker.1panel.live",    "https://hub.rat.dev",    "https://docker.m.daocloud.io"  ],  "mtu": 1400}
```
После того как вы вставили этот текст:
Нажмите кнопку **Apply & Restart** (обычно она появляется в правом нижнем углу или вверху).
Дождитесь перезапуска **Docker Desktop** (индикатор в левом нижнем углу должен стать зелёным).