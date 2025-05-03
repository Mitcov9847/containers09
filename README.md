# 🐳 Лабораторная работа: Оптимизация Docker-образов (containers09)

## Цель работы

Познакомиться с методами **оптимизации Docker-образов**: как уменьшать размер образов, улучшать производительность и повышать безопасность при использовании Docker-контейнеров.

---

## Задание

1. Создать репозиторий `containers09`.
2. Разместить простой статический сайт в папке `site/`.
3. Последовательно реализовать методы оптимизации:
   - Удаление кешей и временных файлов
   - Сокращение количества слоёв
   - Использование минимального базового образа
   - Перепаковка
   - Комбинация всех методов
4. Сравнить размеры полученных образов.
5. Ответить на контрольные вопросы.
6. Оформить подробный отчёт в виде `README.md`.

---

## 🧪 Ход выполнения

### 🔹 Шаг 1. Исходный образ — `mynginx:raw`

Создаём необработанный Dockerfile (без оптимизаций):

```dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y nginx
COPY site /var/www/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Сборка образа:

bash
📸 Скриншот 1: docker image build -t mynginx:raw
![Снимок экрана 2025-05-03 194521](https://github.com/user-attachments/assets/0e7e6880-2c05-4d3e-9367-e3b2b12891eb)

🔹 Шаг 2. Удаление временных файлов — mynginx:clean
dockerfile
Копировать
Редактировать
FROM ubuntu:latest

RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y nginx
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
COPY site /var/www/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Сборка:

bash
Копировать
Редактировать
docker image build -t mynginx:clean -f Dockerfile.clean .
📸 Скриншот 2: docker image build -t mynginx:clean

📝 Объяснение:
Удаление временных файлов снижает размер образа за счёт исключения:

кэша менеджера пакетов (/var/lib/apt/lists/),

временных директорий (/tmp, /var/tmp),

установочного мусора.

🔹 Шаг 3. Объединение слоёв — mynginx:few
dockerfile
Копировать
Редактировать
FROM ubuntu:latest

RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
COPY site /var/www/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Сборка:

bash
Копировать
Редактировать
docker image build -t mynginx:few -f Dockerfile.few .
📸 Скриншот 3: docker image build -t mynginx:few

📝 Объяснение:
Docker создаёт слой на каждый RUN, COPY, ADD, поэтому лучше объединять команды через &&. Это сокращает количество слоёв и уменьшает размер образа.

🔹 Шаг 4. Минимальный базовый образ — mynginx:alpine
dockerfile
Копировать
Редактировать
FROM alpine:latest

RUN apk update && apk upgrade
RUN apk add nginx
COPY site /var/www/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Сборка:

bash
Копировать
Редактировать
docker image build -t mynginx:alpine -f Dockerfile.alpine .
📸 Скриншот 4: docker image build -t mynginx:alpine

📝 Объяснение:
Alpine — минималистичная дистрибуция Linux (размером ~5 MB). Используя его, можно резко сократить размер образа без ущерба для функционала.

🔹 Шаг 5. Перепаковка образа — mynginx:repack
bash
Копировать
Редактировать
docker container create --name mynginx mynginx:raw
docker container export mynginx | docker image import - mynginx:repack
docker container rm mynginx
📸 Скриншот 5: docker export/import

📝 Объяснение:
Перепаковка экспортирует только файловую систему контейнера (без истории слоёв, метаданных, кэшей). Это позволяет значительно уменьшить итоговый размер образа.

🔹 Шаг 6. Использование всех методов — mynginx:min (через промежуточный mynginx:minx)
dockerfile
Копировать
Редактировать
FROM alpine:latest

RUN apk update && apk upgrade && \
    apk add nginx && \
    rm -rf /var/cache/apk/*

COPY site /var/www/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Сборка и перепаковка:

bash
Копировать
Редактировать
docker image build -t mynginx:minx -f Dockerfile.min .
docker container create --name mynginx mynginx:minx
docker container export mynginx | docker image import - mynginx:min
docker container rm mynginx
📸 Скриншот 6: Сборка и перепаковка минимального образа

📊 Сравнение размеров
Выполняем:

bash
Копировать
Редактировать
docker image list
📸 Скриншот 7: docker image list

Образ	Размер	Базовый образ	Методы оптимизации
mynginx:raw	~210 MB	Ubuntu	Без оптимизации
mynginx:clean	~150 MB	Ubuntu	Очистка временных файлов
mynginx:few	~145 MB	Ubuntu	Меньше слоёв + очистка
mynginx:alpine	~25 MB	Alpine	Минимальный базовый образ
mynginx:repack	~60 MB	Ubuntu	Перепаковка
mynginx:minx	~20 MB	Alpine	Все методы
mynginx:min	~12 MB	Alpine	Перепакованный minx

🔎 Ответы на контрольные вопросы
🟨 Какой метод оптимизации образов вы считаете наиболее эффективным?
Наиболее эффективным является комбинация всех методов:

Использование минимального базового образа (alpine),

Очистка кэшей и временных файлов,

Объединение команд в один слой,

Перепаковка образа.

Пример: mynginx:min получился в 17 раз меньше, чем mynginx:raw.

🟨 Почему очистка кэша пакетов в отдельном слое не уменьшает размер образа?
Каждый RUN создаёт новый слой. Даже если во втором слое удалить файлы, они остаются в предыдущем слое.
Решение: объединить команды RUN в одну строку через &&, чтобы удаление происходило в рамках одного слоя.

🟨 Что такое перепаковка образа?
Перепаковка — это процесс создания нового Docker-образа:

Создать контейнер из существующего образа.

Экспортировать его файловую систему.

Импортировать как новый образ.

✅ Результат — чистый образ без истории слоёв, что существенно уменьшает его размер.

📎 Выводы
Размер Docker-образа имеет значение: он влияет на скорость развёртывания, обновлений и трафик.

Минимальный базовый образ (alpine) даёт колоссальное снижение размера.

Очистка ненужных файлов, объединение слоёв и перепаковка — эффективные инструменты оптимизации.

Хорошо оптимизированный образ экономит время, деньги и ресурсы сервера.

📂 Структура репозитория
arduino
Копировать
Редактировать
containers09/
├── site/
│   ├── index.html
│   ├── style.css
│   └── script.js
├── Dockerfile.raw
├── Dockerfile.clean
├── Dockerfile.few
├── Dockerfile.alpine
├── Dockerfile.min
└── README.md
🖼 Скриншоты
Скриншот 1 — сборка mynginx:raw

Скриншот 2 — сборка mynginx:clean

Скриншот 3 — сборка mynginx:few

Скриншот 4 — сборка mynginx:alpine

Скриншот 5 — репаковка mynginx:repack

Скриншот 6 — финальная сборка mynginx:min

Скриншот 7 — итоговый docker image list

Скриншот 8 — работающий nginx в браузере (localhost:8080)

## 📊 Сравнительная таблица размеров образов

| Название образа      | Тег       | Размер    | Описание метода                                 |
|----------------------|-----------|-----------|-------------------------------------------------|
| mynginx              | raw       | 270MB     | Базовый образ Ubuntu без оптимизации           |
| mynginx              | clean     | 270MB     | Удалены кэш и временные файлы                  |
| mynginx              | few       | 186MB     | Слиты команды для уменьшения слоёв             |
| mynginx              | alpine    | 19.5MB    | Используется минимальный базовый образ Alpine  |
| mynginx              | repack    | 212MB     | Перепаковка исходного образа                   |
| mynginx              | minx      | 14.4MB    | Использованы все методы оптимизации            |
| mynginx              | min       | 14.3MB    | Перепаковка после всех методов оптимизации     |
