Устанавливаем CVAT в директорию `/srv`
```shell
cd /srv
```
На машине из примера нет sudo доступа, поэтому устанаваливаем в домашнюю директорию
## Reverse Proxy 
Для продакшн развёртывания CVAT есть классная [документация](https://docs.cvat.ai/docs/administration/basics/installation/#deploy-secure-cvat-instance-with-https) . Однако это перестаёт работать в тот момент, когда мы сталкиваемся с работой из под прокси. Ну то есть у нас есть внешний веб-сервер который регламентирует доступ.
![[Pasted image 20250114171121.png]]
Мы получаем ошибку CSRF failed при попытки создания задачи. Поэтому пройдёмся по всему процессу развёртывания

#### Клонируем репозиторий
```shell
git clone https://github.com/cvat-ai/cvat
cd cvat
```
#### Создаём файл .env из которого будут задаваться настройки развертывания

Создаём файл
```shell
touch .env
```

Выставляем версию и домен. Выставлять версию очень полезно, так как по умолчанию используется dev.

Содержимое .env:
```shell
CVAT_VERSION=v2.16.2
CVAT_HOST=cvat-smile.onti.actcognitive.org
```

Экспортировать переменные можно через команду
```shell
export $(grep -v '^#' .env | xargs)
```

Далее, нужно создать `docker-compose.override.yml` для того, чтобы можно было изменить содержимое `production.py`.
`docker-compose.override.yml
```yaml
services:
  cvat_server:
    volumes:
      - ./production.py:/home/django/cvat/settings/production.py:ro
    environment:
      # делаем CVAT_HOST доступным внутри докер контейнера
      CVAT_HOST: ${CVAT_HOST:-localhost}
```

Копируем файл с настройками развёртывания

```shell
cp ./cvat/settings/production.py ./production.py
```

Добавляем строчку в конец файла
```shell
echo "CSRF_TRUSTED_ORIGINS = [origin.rstrip('/') for origin in os.getenv('CSRF_TRUSTED_ORIGINS', f'http://{CVAT_HOST},https://{CVAT_HOST}').split(',')]" >> ./production.py
```

Запускаем CVAT
```shell
export $(grep -v '^#' .env | xargs) # если не сделано ранее
docker compose up -d
```

## Бекапы 
Бекапы также изначально [сделаны очень хорошо](https://docs.cvat.ai/docs/administration/advanced/backup_guide/), однако чуть-чуть автоматизации не помешает. В случае, если мы хотим периодически делать бекапы, сделаем два скрипта - `backup.sh` и `restore.sh`

`backup.sh`:
```shell
#! /bin/bash
export $(grep -v '^#' .env | xargs)
BACKUP_FOLDER=/var/essdata/backups # в нашем случае папка для сохранения бекапов

echo "Stopping CVAT..."
docker compose stop

docker run --rm --name temp_backup --volumes-from cvat_db -v $BACKUP_FOLDER:/backup ubuntu tar -czvf /backup/cvat_db.tar.gz /var/lib/postgresql/data
docker run --rm --name temp_backup --volumes-from cvat_server -v $BACKUP_FOLDER:/backup ubuntu tar -czvf /backup/cvat_data.tar.gz /home/django/data
docker run --rm --name temp_backup --volumes-from cvat_clickhouse -v $BACKUP_FOLDER:/backup ubuntu tar -czvf /backup/cvat_events_db.tar.gz /var/lib/clickhouse

echo "Done, results stored in backup folder"

ls $BACKUP_FOLDER

echo "Starting CVAT..."
docker compose up -d
echo "CVAT started"
```
`restore.sh`:
```shell
#! /bin/bash

export $(grep -v '^#' .env | xargs)
BACKUP_FOLDER=/var/essdata/backups

echo "CVAT stopping..."
docker compose stop

cd $BACKUP_FOLDER
docker run --rm --name temp_backup --volumes-from cvat_db -v $(pwd):/backup ubuntu bash -c "cd /var/lib/postgresql/data && tar -xvf /backup/cvat_db.tar.gz --strip 4"
docker run --rm --name temp_backup --volumes-from cvat_server -v $(pwd):/backup ubuntu bash -c "cd /home/django/data && tar -xvf /backup/cvat_data.tar.gz --strip 3"
docker run --rm --name temp_backup --volumes-from cvat_clickhouse -v $(pwd):/backup ubuntu bash -c "cd /var/lib/clickhouse && tar -xvf /backup/cvat_events_db.tar.gz --strip 3"

echo "Successfully restored"
cd ~/cvat
docker compose up -d
echo "CVAT started"
```

Создаём cronjob, чтобы периодически выполнять бекапы. Допустим, мы хотим делать в 18.00 каждый день.

Редактируем crontab:
```shell
crontab -e
```

Добавляем строчку (для примера)
```
0 18 * * * /home/mawwlle/cvat/backup.sh # путь должен быть /srv/cvat/backup.sh
```

## Подключение к удалённому Nuclio
Для подключения удалённого нуклио (его [сначала нужно развернуть](https://docs.cvat.ai/docs/manual/advanced/serverless-tutorial/)), достаточно указать в .env `CVAT_NUCLIO_HOST`
Таким образом, файл `.env` выглядит вот так:
```shell
CVAT_NUCLIO_HOST=10.32.15.90
CVAT_VERSION=v2.16.2
CVAT_HOST=cvat-smile.onti.actcognitive.org
```

## WIP