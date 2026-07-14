# Домашнее задание «Оркестрация группой Docker-контейнеров на примере Docker Compose»

**Выполнил:** Александр Аксёнов  

## Задача 1. Установка Docker и Docker compose на ВМ и создание собственного образа Nginx.

На виртуальной машине были установлены Docker и Docker Compose.

Создан публичный репозиторий Docker Hub: https://hub.docker.com/repository/docker/folau/custom-nginx/general

Для создания образа использовался базовый образ:

```text
nginx:1.29.0
```

### Dockerfile

```dockerfile
FROM nginx:1.29.0

COPY index.html /usr/share/nginx/html/index.html
```
### Содержимое index.html

```html
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```

Образ был собран с тегом `1.0.0`:

```bash
docker build -t folau/custom-nginx:1.0.0 .
```

После сборки образ был загружен в Docker Hub:

```bash
docker push folau/custom-nginx:1.0.0
```

Полученный digest:

```text
sha256:f0a382bbe9e42b1499b9085503406eb5c400356dc5c40e17c7a341f05efe1d06
```

## Задача 2. Запуск контейнера custom-nginx

Контейнер был запущен в фоновом режиме с публикацией порта `8080` только на локальном интерфейсе хоста:

```bash
docker run -d \
  --name aksenov-aleksandr-custom-nginx-t2 \
  -p 127.0.0.1:8080:80 \
  folau/custom-nginx:1.0.0
```

После запуска контейнер был переименован без удаления и пересоздания:

```bash
docker rename aksenov-aleksandr-custom-nginx-t2 custom-nginx-t2
```

Для проверки состояния контейнера были выполнены команды:

```bash
date +"%d-%m-%Y %T.%N %Z" ; \
sleep 0.150 ; \
docker ps ; \
ss -tlpn | grep 127.0.0.1:8080 ; \
docker logs custom-nginx-t2 -n1 ; \
docker exec -it custom-nginx-t2 base64 /usr/share/nginx/html/index.html
```

Работа веб-сервера проверена командой:

```bash
curl http://127.0.0.1:8080
```

В ответ была получена созданная HTML-страница:

```html
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```

### Скриншоты

<img width="862" height="731" alt="nginx2" src="https://github.com/user-attachments/assets/9884e4b7-75da-423d-9bed-8169a4c539f0" />


<img width="748" height="439" alt="nginx_console" src="https://github.com/user-attachments/assets/359a7012-75f3-4f39-bc89-297309c83ac0" />

## Задача 3. Подключение к контейнеру и изменение порта Nginx

Для того, чтобы подключения к стандартному потоку ввода и вывода контейнера, нужно использовать:

```bash
docker attach custom-nginx-t2
```

После нажатия `Ctrl+C` контейнер остановился.

Контейнер остановился потому, что Nginx являлся главным процессом контейнера с PID 1. Комбинация `Ctrl+C` передала главному процессу сигнал прерывания. После завершения главного процесса завершил работу и сам контейнер.

Состояние было проверено командой:

```bash
docker ps -a
```

После запустили контейнер повторно:

```bash
docker start custom-nginx-t2
```

Чтобы войти внутрь контейнера используем:

```bash
docker exec -it custom-nginx-t2 bash
```

Устанавливаем внутри контейнера инструмент nano:

```bash
apt-get update
apt-get install -y nano
```

В файле:

```text
/etc/nginx/conf.d/default.conf
```

порт Nginx был изменён с:

```nginx
listen 80;
```

на:

```nginx
listen 81;
```

Конфигурация Nginx была перезагружена:

```bash
nginx -s reload
```

Проверка:

```bash
curl http://127.0.0.1:80
curl http://127.0.0.1:81
curl: (7) Failed to connect to 127.0.0.1 port 80 after 0 ms: Couldn't connect to server
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```
На порту `80` соединение не устанавливалось, а на порту `81` Nginx возвращал созданную страницу.

Docker продолжал перенаправлять порт 80 контейнера. Однако после изменения конфигурации Nginx начал слушать порт `81`.

После проверки запущенный контейнер был удалён без предварительной остановки:

```bash
docker rm -f custom-nginx-t2

### Дополнительное задание

Для того, чтобы исправить ситуацию с контейнерами и перенаправления портами, можно воспользоваться Socat.

После его установки, нужно внутри контейнера сделать следущее:

```bash
docker exec -d custom-nginx-t2 \
  socat TCP-LISTEN:80,fork,reuseaddr TCP:127.0.0.1:81
```

Теперь обращения к порту `80` контейнера перенаправлялись на порт `81`, на котором работал Nginx.
После проверки командой curl, возвращал стандартную HTML страничку.


### Скриншоты
<img width="858" height="449" alt="attache" src="https://github.com/user-attachments/assets/f3cc2648-07dc-42ba-bbeb-beea2024d328" />
<img width="855" height="722" alt="nginx_contr_2" src="https://github.com/user-attachments/assets/e9b46b9c-32e8-454a-800e-8075f4d254c8" />
<img width="851" height="717" alt="nginx_conter" src="https://github.com/user-attachments/assets/3cb5974e-a631-4618-b81e-6984097c3d38" />
<img width="749" height="445" alt="repair_nginx" src="https://github.com/user-attachments/assets/9d54ec09-6bbc-4009-bb8e-42be28627c3a" />


## Задача 4. Совместное использование каталога контейнерами

Был создан отдельный рабочий каталог:

```bash
mkdir -p ~/docker-task4
cd ~/docker-task4
```

Запущен контейнер CentOS с подключением текущего каталога хоста в `/data`:

```bash
docker run -d \
  --name centos-t4 \
  -v "$(pwd):/data" \
  centos:7 \
  tail -f /dev/null
```

Запущен контейнер Debian с подключением того же каталога:

```bash
docker run -d \
  --name debian-t4 \
  -v "$(pwd):/data" \
  debian:12 \
  tail -f /dev/null
```

Внутри контейнера создали файл:
```bash
docker exec centos-t4 \
  sh -c 'echo "Файл создан в контейнере CentOS" > /data/from-centos.txt'
```

На хостовой машине был создан второй файл:

```bash
echo "Файл создан на хостовой машине" > from-host.txt
```
После этого содержимое каталога было проверено из контейнера Debian:

```bash
docker exec debian-t4 ls -la /data
docker exec debian-t4 cat /data/from-centos.txt
docker exec debian-t4 cat /data/from-host.txt
```

### Результат

Оба контейнера и хостовая система использовали один каталог.

Файл, созданный внутри CentOS, стал доступен на самой хостовой машине и внутри контейнера Debian.
Файл, созданный на хостовой машине, также стал доступен обоим контейнерам.

Это произошло благодаря bind mount:

```text
$(pwd) хоста → /data контейнера
```

### Скриншоты

<img width="859" height="858" alt="Devian_centos" src="https://github.com/user-attachments/assets/07f71d1b-6365-435c-857d-7dd67ea40e7a" />

## Задача 5. Docker Compose, Registry и Portainer

Для выполнения задания был создан каталог:

```bash
mkdir -p /tmp/netology/docker/task5
cd /tmp/netology/docker/task5
```

### Первоначальный файл compose.yaml

```yaml
version: "3"

services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

### Файл docker-compose.yaml

```yaml
version: "3"

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
```

При выполнении команды:

```bash
docker compose up -d
```

Docker Compose обнаружил оба файла, но запустил конфигурацию из:

```text
compose.yaml
```

### Почему был выбран compose.yaml

Если же в каталоге, имеется два файла `compose.yaml` и `docker-compose.yaml` то Docker compose отдает предпочтение именно первому варианту.
`compose.yaml` является основным современным именем манифеста, а `docker-compose.yaml` поддерживается для обратной совместимости.
Поэтому при первом запуске был создан только контейнер Portainer.

### Подключение второго Compose-файла
Файл `compose.yaml` был изменён следующим образом:

```yaml
version: "3"

include:
  - docker-compose.yaml

services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

После выполнения:

```bash
docker compose up -d
```

были запущены оба сервиса:

```text
task5-portainer-1
task5-registry-1
```

---

### Загрузка custom-nginx в локальный Registry

Существующему образу был назначен новый тег:

```bash
docker tag \
  folau/custom-nginx:1.0.0 \
  127.0.0.1:5000/custom-nginx:latest
```

Образ был загружен в локальный Registry:

```bash
docker push 127.0.0.1:5000/custom-nginx:latest
```

Полученный digest:

```text
sha256:f0a382bbe9e42b1499b9085503406eb5c400356dc5c40e17c7a341f05efe1d06
```

Наличие образа в Registry проверено запросом:

```bash
curl http://127.0.0.1:5000/v2/custom-nginx/tags/list
```

Получен ответ:

```json
{
  "name": "custom-nginx",
  "tags": [
    "latest"
  ]
}
```

---

### Развёртывание Nginx через Portainer

В Portainer было выбрано локальное окружение.

На странице `Stacks` через `Web editor` был создан стек с именем:

```text
dep
```

Использованный Compose-манифест:

```yaml
version: "3"

services:
  nginx:
    image: 127.0.0.1:5000/custom-nginx
    ports:
      - "9090:80"
```

После развёртывания был создан контейнер:

```text
dep-nginx-1
```

Порт контейнера был опубликован следующим образом:

```text
9090:80
```

### Контейнер в Portainer

<img width="2048" height="459" alt="Portainer_dep_nginx" src="https://github.com/user-attachments/assets/a87d1d7b-a3eb-4c8c-9e79-d351b1426579" />

### Inspect контейнера

В разделе `Inspect` было выбрано представление `Tree`, раскрыт параметр `Config`.

На скриншоте видны параметры от `AppArmorProfile` до `Driver`.

<img width="502" height="834" alt="Portainer" src="https://github.com/user-attachments/assets/e89fd124-7471-4ec5-9ef6-baf53c2beec0" />

### Удаление одного из манифестов

Перед удалением файл `compose.yaml` был сохранён:

```bash
cp compose.yaml /tmp/netology/docker/compose.yaml
```

После этого исходный файл был удалён:

```bash
rm compose.yaml
```

В каталоге остался только:

```text
docker-compose.yaml
```

После выполнения:

```bash
docker compose up -d
```

появилось предупреждение:

```text
Found orphan containers (task5-portainer-1) for this project.
```

### Объяснение предупреждения

Ранее Compose-проект `task5` содержал два сервиса:

```text
portainer
registry
```

После удаления `compose.yaml` в текущей конфигурации остался только сервис:

```text
registry
```

Однако контейнер `task5-portainer-1`, созданный ранее, продолжал существовать и был связан с тем же Compose проектом.

Docker Compose обнаружил контейнер, который больше не описан в актуальном манифесте, и определил его как orphan-контейнер, или точнее контейнер-сироту.

В предупреждении Docker Compose предложил использовать параметр:

```text
--remove-orphans
```

Предложенное действие было выполнено:

```bash
docker compose up -d --remove-orphans
```

В результате:

```text
task5-registry-1  Running
task5-portainer-1 Removed
```

Предупреждение:

```text
the attribute `version` is obsolete
```

означает, что современная версия Docker Compose больше не использует поле `version`. Поле игнорируется и может быть удалено из файла.

---

### Остановка Compose-проекта одной командой

Проект был остановлен и удалён одной командой:

```bash
docker compose down
```

Результат:

```text
Container task5-registry-1 Removed
Network task5_default Removed
```
### Скриншоты выполнения задачи 5


<img width="858" height="853" alt="Task5-1" src="https://github.com/user-attachments/assets/2a4529fe-353f-40e2-9907-da42f552b2c3" />
<img width="864" height="864" alt="Task5-2" src="https://github.com/user-attachments/assets/aa11f34f-639b-404e-983d-bb71cabadbe6" />
<img width="858" height="682" alt="Task5-3" src="https://github.com/user-attachments/assets/b0013988-ef9c-4a52-8037-af5960b39be1" />

## Итог

В ходе выполнения домашнего задания были выполнены следующие действия:

- Установил Docker и Docker compose;
- Был создан собственный Docker образ на основе Nginx;
- Образ с nginx был опубликован в Docker Hub;
- Выполнен запуск, переименование и диагностика контейнера;
- Изучил взаимодействие контейнера с сигналами;
- Измеил порт Nginx внутри контейнера и исправил ошибку с перенаправлением портов;
- Настроил совместное использование каталога через bind mount;
- Запустил сервисы через Docker Compose;
- Подключил дополнительный Compose-файл через `include`;
- Развернул локально в VM Docker Registry;
- Выполнил собственный образ в Registry;
- Настроил Portainer;
- Через Portainer развёрнул Docker Compose стек;
- Изучил все предупреждения WARN в консоле после запуска контейнеров;
- Разобрался как избавиться от предупреждений во время запуска;
- Compose-проект остановлен одной командой.
