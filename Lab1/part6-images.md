# Часть 6. Сборка и проверка образов

Текущий этап: задача 1 [плана](part6-plan.md). Все команды выполняет пользователь на удалённом Debian из каталога lab1. Ассистент не выполняет сборки и запуски.

## 1. Перенести файлы

В каталоге сервера должны быть main.go, go.mod, Dockerfile.single, Dockerfile и .dockerignore. Существующий go.mod пользователя уже совпадает с подготовленным. Бинарник хоста lab1-serv не копируется в образы: оба Dockerfile компилируют исходный код сами.

~~~bash
ls -la Dockerfile Dockerfile.single .dockerignore go.mod main.go
~~~

.dockerignore исключает отчёт, скриншоты и остальные артефакты из контекста. Не заменять точный COPY на COPY всего каталога.

## 2. Проверить сборщик и получить базовый образ

~~~bash
sudo docker buildx version
sudo docker pull golang:1.24.4
sudo docker image inspect golang:1.24.4 --format '{{json .RepoDigests}}'
~~~

Сохранить версию buildx либо сообщение о его отсутствии, а также digest полученного образа. Ошибка отсутствия buildx сама по себе не требует установки: Docker 26 может использовать legacy builder. Если pull завершился ошибкой, остановиться и прислать её. Не менять тег Go молча.

## 3. Собрать оба образа

~~~bash
sudo env BUILDKIT_PROGRESS=plain docker build -f Dockerfile.single -t lab1-api:single .
sudo env BUILDKIT_PROGRESS=plain docker build -f Dockerfile -t lab1-api:multi .
~~~

BUILDKIT_PROGRESS=plain задаёт читаемый лог для BuildKit; legacy builder этот параметр окружения не использует. Использованный builder определяется по версии и выводу. Если Docker требует отсутствующий buildx вместо доступного legacy builder, прислать ошибку перед изменением способа сборки.

В single конечный образ сохраняет Go SDK, исходники и кэш компилятора. В multi компиляция та же, но конечная стадия scratch получает только /lab1-serv. CGO_ENABLED=0 исключает зависимость бинарника от libc; в scratch нет ни libc, ни shell. Одинаковые команды сборки нужны для честного сравнения состава образов.

Сохранить логи обеих сборок. Первая multi-сборка уже может переиспользовать слои single-сборки; это не ошибка и не повод очищать кэш.

## 4. Запустить single и проверить HTTP

~~~bash
sudo docker run -d --rm --name lab1-images-single \
  --memory 128m --memory-swap 128m --cpus 0.5 --pids-limit 64 \
  --cap-drop ALL --security-opt no-new-privileges=true \
  --publish 8081:8080 lab1-api:single
~~~

Перед проверкой пользователь читает логи запуска:

~~~bash
sudo docker logs lab1-images-single
curl --noproxy '*' -i --max-time 5 http://127.0.0.1:8081/health
test "$(curl --noproxy '*' -sS --max-time 5 -w '%{http_code}' http://127.0.0.1:8081/health)" = ok200 \
  && echo 'single: HTTP 200 и точное тело ok — OK'
~~~

Последняя команда проверяет вместе точное тело и HTTP-статус; при несовпадении test возвращает ненулевой код и строка OK не печатается. Если сервис ещё не успел стартовать, сначала прочитать логи; сам по себе запуск в фоне не доказывает готовность. При ошибке имени или порта не удалять чужой контейнер и не завершать неизвестный процесс.

## 5. Запустить multi и повторить проверку

~~~bash
sudo docker run -d --rm --name lab1-images-multi \
  --memory 128m --memory-swap 128m --cpus 0.5 --pids-limit 64 \
  --cap-drop ALL --security-opt no-new-privileges=true \
  --publish 8082:8080 lab1-api:multi
sudo docker logs lab1-images-multi
curl --noproxy '*' -i --max-time 5 http://127.0.0.1:8082/health
test "$(curl --noproxy '*' -sS --max-time 5 -w '%{http_code}' http://127.0.0.1:8082/health)" = ok200 \
  && echo 'multi: HTTP 200 и точное тело ok — OK'
~~~

Для scratch нельзя использовать docker exec с sh, curl или ls: этих программ в образе нет. HTTP проверяется с хоста.

## 6. Проверить метаданные

~~~bash
sudo docker image inspect lab1-api:single lab1-api:multi \
  --format '{{json .RepoTags}} User={{.Config.User}} Entrypoint={{json .Config.Entrypoint}} Ports={{json .Config.ExposedPorts}}'
~~~

Для обоих тегов ожидается User=1000:1000, Entrypoint=["/lab1-serv"] и объявленный 8080/tcp. EXPOSE описывает порт, а фактическую публикацию задают 8081:8080 и 8082:8080.

Сохранить HTTP-проверки и метаданные в screenshots/part6-health.png. Прислать логи сборок, digest сборщика, результат проверок и метаданные. Контейнеры можно остановить после снимков:

~~~bash
sudo docker stop lab1-images-single lab1-images-multi
~~~

Из-за --rm тестовые контейнеры удалятся, образы останутся для следующего опыта. Порты 8081 и 8082 во время работы опубликованы на интерфейсах сервера.

## Дальше по плану

После подтверждения двух запусков сравним фактический размер, файловые слои и кэш повторной сборки. После проверки понимания этих механизмов перейдём к writable layer и named volume. Команды файлового эксперимента будут добавлены на его этапе.

Статус: файлы и проверки подготовлены; pull, build, run и проверки на сервере ещё не выполнены.

Источники: [multi-stage](https://docs.docker.com/build/building/multi-stage/), [контекст сборки и dockerignore](https://docs.docker.com/build/concepts/context/), [BUILDKIT_PROGRESS](https://docs.docker.com/build/building/variables/#buildkit_progress).
