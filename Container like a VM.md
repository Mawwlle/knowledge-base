#ubuntu20_04 #cuda #docker #resources 
Так как виртуальные машины очень плохо делят между собой видеокарты (в теории они конечно могут что-то поделить, но это супер сложно и только на крутых видюхах, динамически оно не умеет). Будем использовать Nvidia-way, который пропагандирует использование контейнеров.

Разберем на примере задачи: Создать два виртуальных хоста, которые имеют доступ к друг другу (находятся в одной подсети), а также конфигурируются различными выч.ресурсами

Самое сложное сейчас - Dockerfile, так как CUDA в России сейчас недоступна. 
>[!info]- TLDR;
>```Dockerfile
> FROM nvidia/cuda:12.6.3-cudnn-devel-ubuntu20.04
> ARG DEBIAN_FRONTEND=noninteractive
> ENV TZ=UTC
> RUN rm -f /etc/apt/sources.list.d/cuda.list \
 >  && apt update \
>    && apt install -y --no-install-recommends \
>    ca-certificates \
>    gnupg \
>    && rm -rf /var/lib/apt/lists/* 
> RUN apt update \
>    && apt install -y --no-install-recommends \
>    openssh-server \
>    net-tools \
>    iputils-ping \
>    curl \
 >    wget \
>    git \
>    python3-pip \
>    sudo \
>    zsh \
>    && mkdir /var/run/sshd \
>    && rm -rf /var/lib/apt/lists/*
> RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/'
>    /etc/ssh/sshd_config \
>   && sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
>   RUN useradd -ms /bin/zsh developer \
>	&& echo "developer:developer" | chpasswd \
>	&& usermod -aG sudo developer
>RUN apt update \
>	&& apt install -y --no-install-recommends \
> 	   apt-transport-https \
>      ca-certificates \
>	curl \
> 	gnupg-agent \
> 	software-properties-common \
> 	&& curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - \
>	&& add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
>	&& apt update \
>	&& apt install -y --no-install-recommends docker-ce-cli \
>	&& rm -rf /var/lib/apt/lists/* \
>	&& groupadd -f docker \
>	&& usermod -aG docker developer
>	COPY entrypoint.sh /entrypoint.sh
> RUN chmod +x /entrypoint.sh
> EXPOSE 22
> ENTRYPOINT ["/entrypoint.sh"]

Алгоритм:
1. Скачать нужный образ CUDA с dockerhub (благо он доступен и существует много зеркал)
2. ОПЦИОНАЛЬНО. Выставить неинтерактивный режим в дебиане (чтобы не вылезали диалоговые окна про текущую тайм-зону). И выставить  переменную окружения `TZ` в нужное значение (или оставить по умолчанию `UTC`)
3. Далее, так как APT не может установить зависимости из репозитория nvidia, это будет нам ломать весь процесс загрузки ЛЮБОГО пакета. Но так как CUDA уже установлена, мы можем опустить этот момент. Просто удаляем репозиторий
4. Устанавливаем все нужные пакеты (в том числе и openssh-server, для доступа по ssh)
5. ОПЦИОНАЛЬНО. Конфигурируем ssh для доступа по паролю (многим это привычнее)
6. ОПЦИОНАЛЬНО. Добавляем пользователя
7. Устанавливаем докер. Немного поясню за этот шаг, он далеко необязательный, так как в целом мы итак внутри контейнера, и если вы используете контейнер для разработки, то это вообще лучше исключить. 
8. Создаём энтрипоинт, для динамической генерации паролей
9. Настраиваем оркестрацию

## Образ с CUDA
Я предпочитаю DEB-совместимые образы, поэтому везде фигурирует Ubuntu 20.04. Однако папира актуальна для любого DEB образа с ядром 5.15 (я не проверял но так должно быть).
```Dockerfile
FROM nvidia/cuda:12.6.3-cudnn-devel-ubuntu20.04
```

## Конфигурирование Debian
см. шаг 2 алгоритма
```Dockerfile
ARG DEBIAN_FRONTEND=noninteractive 
ENV TZ=UTC
```

## Удаление репозитория NVIDIA
```Dockerfile
RUN rm -f /etc/apt/sources.list.d/cuda.list \
    && apt update \
    && apt install -y --no-install-recommends \
        ca-certificates \
        gnupg \
    && rm -rf /var/lib/apt/lists/*
```

## Установка всех пакетов
Для примера установим что-то базовое
```Dockerfile
RUN apt update \
    && apt install -y --no-install-recommends \
        openssh-server \
        net-tools \
        iputils-ping \
        curl \
        wget \
        git \
        python3-pip \
        sudo \
        zsh \
    && mkdir /var/run/sshd \
    && rm -rf /var/lib/apt/lists/*
```

## Конфигурируем SSH для доступа по паролю

```Dockerfile
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config \
    && sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
```

## Создание пользователя
```Dockerfile
RUN useradd -ms /bin/zsh developer \
    && echo "developer:developer" | chpasswd \
    && usermod -aG sudo developer
```

## Установка Docker

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg-agent \
        software-properties-common \
    && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - \
    && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce-cli \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -f docker \
    && usermod -aG docker developer
```

## Копирование точки входа и декларация порта для SSH

```Dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 22

ENTRYPOINT ["/entrypoint.sh"]
```

## Настраиваем оркестрацию
Docker Compose крайне удобен для подобного рода задач - в одном файле мы можем легко собрать все нужные нам "зависимости" (обзовём так) для контейнера.

```yaml
services:
  master:
    build:
	    context: .
    environment:
      - ROOT_PASSWORD=everest
      - DEVELOPER_PASSWORD=brodpik
    restart: always
```

Для начала, установим [политику рестарта](https://docs.docker.com/engine/containers/start-containers-automatically/) (в нашем случае мы хотим чтобы контейнер был максимально стабилен, поэтому устанавливаем always)
В качестве переменных окружения, мы задаём пароли ([их можно поместить в .env файл ](https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/))
[Директива build](https://docs.docker.com/reference/compose-file/build/) указывает каким образом мы собираем контейнер (в какую папку смотрим при сборке)

#### Добавим бекапы, подсеть, прокинем сокет докера и порт SSH
Для примера, будем бекапить /opt. Подсеть будет типа [bridge](https://docs.docker.com/engine/network/drivers/bridge/). 
**!!Сокет докер прокидывать необязательно!! см. Алгоритм**
А вот SSH порт прокинуть уже обязательно (потому что контейнеров вероятнее всего будет много и коллизий не хочется)

```yaml
services:
  master:
    build:
	    context: .
    volumes:
      - master:/opt
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ROOT_PASSWORD=everest
      - DEVELOPER_PASSWORD=brodpik
    restart: always
    ports:
      - 4432:22
    networks:
      - primary_network

networks:
  primary_network:
    driver: bridge

volumes:
  master: {}
```

Дальше самое интересное - [конфигурирование ресурсов](https://docs.docker.com/engine/containers/resource_constraints/). Тема довольно обширная, обойду вскользь.
```yaml
services:
  master:
    build:
	    context: .
    volumes:
      - master:/opt
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ROOT_PASSWORD=everest
      - DEVELOPER_PASSWORD=brodpik
    restart: always
    deploy:
      resources:
        limits:
          cpus: '8'
          memory: 16000M
        reservations:
          cpus: '4'
          memory: 8000M
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    ports:
      - 4432:22
    networks:
      - primary_network

networks:
  primary_network:
    driver: bridge

volumes:
  master: {}
```

Вкратце,  у нас есть пределы (limits) и резервации (reservations). Резервация гарантирует, что у нас есть определённый ресурс в единицу времени. "В единицу времени" - довольно размытое понятие, однако так и есть, там везде квантование времени и всё такое, ну то есть можно просто закрыть пальчиком это выражение и жить в прекрасном мире (пока что-то не сломается и придётся вникать...). А предел - собственно говоря, куда мы можем уйти от нашей резервации - верхний потолок. 
В нашем случае у нас всегда доступна одна видеокарта (любая), 4 потока процессора и 8 гигабайт памяти. Максим может быть использовано 8 потоков и 16 гигабайт.

## Финальный docker-compose.yml для задачи
```Dockerfile
services:
  master:
    build:
	    context: .
    volumes:
      - master:/opt
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ROOT_PASSWORD=everest
      - DEVELOPER_PASSWORD=brodpik
    restart: always
    deploy:
      resources:
        limits:
          cpus: '8'
          memory: 64000M
        reservations:
          cpus: '4'
          memory: 32000M
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    ports:
      - 4432:22
    networks:
      - primary_network

  worker:
    build:
	    context: .
    volumes:
      - worker:/opt
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ROOT_PASSWORD=jomanlungma
      - DEVELOPER_PASSWORD=gasherbroom
    restart: always
    deploy:
      resources:
        limits:
          cpus: '8'
          memory: 16000M
        reservations:
          cpus: '4'
          memory: 8000M
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    ports:
      - 4433:22
    networks:
      - primary_network

networks:
  primary_network:
    driver: bridge

volumes:
  master: {}
  worker: {}
```

# Итог
Мы можем напрямую подключаться с гостевой машины к нашему хосту, достаточно знать лишь SSH порт и пароль. Контейнеры работают в собственной подсети, что обеспечивает безопасность и достаточную изолированность. Установлены все нужные пакеты, а также выставлены пределы ресурсов. **Стоит учитывать, что прокидывание Docker-сокета создаёт отдельные дыры в безопасности и стоит относиться к этому осторожно.** 

Пример работы в контейнере Master
![[Pasted image 20250110030608.png]]
Пример работы в контейнере Worker
![[Pasted image 20250110030703.png]]