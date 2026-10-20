# cAdvisor хоста: источник метрик lab1-gvisor

## Текущий этап

Grafana открыта пользователем через port-forward и SSH-туннель. Далее запустить cAdvisor на Debian-хосте, проверить наличие метрик cgroup lab1-gvisor и только после этого подключить Prometheus. cAdvisor из kubelet узла kind не заменяет этот источник: lab1-gvisor работает во внешнем Docker хоста.

Используется официальный ghcr.io/google/cadvisor:0.60.6. cAdvisor присоединяется к Docker-сети kind для будущего доступа из Kubernetes, а порт хоста 8084 опубликован только на 127.0.0.1. Адрес в сети kind получаем из inspect и не угадываем. Будущий scrape будет настроен по фактическому адресу, он может измениться при пересоздании контейнера.

## Запуск пользователем на удалённом Debian

~~~bash
cd ~/lab1
sudo docker run -d --name lab1-cadvisor \
  --runtime=runc --privileged --cgroupns=host \
  --network kind \
  --publish 127.0.0.1:8084:8080 \
  --volume /:/rootfs:ro \
  --volume /var/run:/var/run:ro \
  --volume /sys:/sys:ro \
  --volume /var/lib/docker:/var/lib/docker:ro \
  ghcr.io/google/cadvisor:0.60.6 \
  --housekeeping_interval=5s \
  --allow_dynamic_housekeeping=false \
  --store_container_labels=false \
  --enable_metrics=cpu,memory
~~~

--privileged, host cgroup namespace и монтирования нужны агенту для наблюдения cgroups и метаданных Docker хоста. Это полномочия агента мониторинга; ограничения сервиса lab1-gvisor не меняются. Read-only mount Docker socket не превращает Docker API в API только для чтения. Использовать здесь как доверенный учебный агент. Дисковые/сетевые/прочие группы метрик не включаются: текущая цель — память, CPU и throttling. ОOM event exporter на этом шаге не включён.

При ошибке не создавать повторно одноимённый контейнер; прислать полный текст. После запуска дать агенту около 20 секунд на обнаружение контейнеров, затем:

~~~bash
sudo docker ps -a --filter 'name=^/lab1-cadvisor$'
sudo docker logs --tail 50 lab1-cadvisor
sudo docker inspect --format 'Status={{.State.Status}} IP={{(index .NetworkSettings.Networks "kind").IPAddress}}' lab1-cadvisor
~~~

## Проверка конкретного контейнера

~~~bash
cd ~/lab1
mkdir -p results
curl --noproxy '*' -fsS --max-time 15 \
  http://127.0.0.1:8084/metrics -o results/cadvisor-metrics.prom
~~~

Если curl завершился успешно:

~~~bash
CID=$(sudo docker inspect --format '{{.Id}}' lab1-gvisor)
grep -E '^container_(memory_usage_bytes|spec_memory_limit_bytes|cpu_usage_seconds_total|cpu_cfs_(periods|throttled_periods|throttled_seconds)_total)' \
  results/cadvisor-metrics.prom | grep -F "$CID"
~~~

Ожидаются серии с полным ID lab1-gvisor в id и, при распознавании Docker, name=lab1-gvisor: память, лимит 134217728 байт, накопленное CPU-время и CFS throttling. Значение памяти может отличаться от исходного замера: это новый момент времени. Если строки отсутствуют или видна только часть метрик, прислать вывод и логи: не считать scrape настроенным только по успешному HTTP-запросу.

~~~bash
sudo docker image inspect ghcr.io/google/cadvisor:0.60.6 --format '{{json .RepoDigests}}'
df -h /var/lib/docker
~~~

Сохранить вывод запуска, адрес в сети kind, выбранные серии, digest образа и свободный диск. Метрики относятся к cgroup sandbox с накладными расходами gVisor, а не только к heap HTTP-приложения. Сырые счётчики CPU/CFS впоследствии преобразуются в скорости/доли по интервалу.

## Следующий этап

По подтверждённому IP cAdvisor настроить scrape Prometheus через Helm values, проверить up и наличие серий lab1-gvisor в Prometheus/Grafana. На текущем шаге Helm values не меняются. Установка источника и проверка сбора объединяются в один учебный этап; вопросы по роли накопительных счётчиков и источника — после успешного scrape, без проверки Kubernetes/Helm.

Источники: [официальный запуск cAdvisor](https://github.com/google/cadvisor/tree/v0.60.6#quick-start-running-cadvisor-in-a-docker-container), [runtime options](https://github.com/google/cadvisor/blob/v0.60.6/docs/runtime_options.md), [release](https://github.com/google/cadvisor/releases/tag/v0.60.6).

Статус: команды подготовлены, cAdvisor и метрики lab1-gvisor ещё не проверены пользователем.
