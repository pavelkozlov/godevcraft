---
author: "Pavel Kozlov"
title: "Hugo + CI/CD + Telegram = ❤️"
date: "2024-06-22"
description: "Рассказываю как сделал автодеплой своего мини блога"
tags: ["guide", "hugo"]
ShowToc: true
TocOpen: true
---

После того как я сделал свой мини блог на Hugo, я решил что было бы круто сделать автодеплой. Ведь зачем мне каждый раз заходить на сервер и делать `git pull`?

<!--more-->

# Что мы имеем изначально

Hugo прекрасный генератор статических сайтов. Написан на go, работает быстро, имеет хорошую документацию и большое комьюнити, контент удобно писать в markdown. Но есть одно но: много ручной работы. Например, чтобы опубликовать новый пост, нужно сделать следующее:

- Сбилдить проект
- Закомитить изменения
- Запушить их в удаленный репозиторий (я использую github)
- Зайти на сервер
- Сделать `git pull`

# Что мы хотим

Минимум ручной работы. Я хочу чтобы после того как я написал пост и закоммитил его, он сам опубликовался на сайте.

# Что нам для этого нужно

- Первым делом я оплатил себе сервер на DigitalOcean, я взял один из недорогих планов, за 6 долларов в месяц. 1 Гб оперативной памяти, 25 Гб диска, 1 процессор. Думаю этого должно хватить
- Купил себе домен [godevcraft.com](https://godevcraft.com)
- Накатил на сервер nginx, настроил его так чтобы он отдавал статику из моего проекта и указал пути до сертификатов

```nginx
server {
        listen 443 ssl;
        keepalive_timeout 30;
        ssl_certificate /local/cert_chain.crt;
        ssl_certificate_key /local/godevcraft.key;
        ssl_session_timeout 5m;
        ssl_session_cache shared:SSL:10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        server_name godevcraft.com;

       root /root/godevcraft/;
       index index.html; 

       location / {
               try_files $uri $uri/ =404;
       }
}
```

- Поднял на этом же сервере self hosted github runner. Это нужно что бы он мог делать pull, да и минутки в CI не бесконечные

- Настроил github actions. Вот так выглядит мой workflow:

```yaml
name: Build and Deploy Hugo Site

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - name: Update submodules
        run: git submodule update --init --recursive

      - name: Install Hugo modules
        run: hugo mod get -u

      - name: Build the website
        run: hugo --gc --minify

      - name: Deploy to external repo
        if: success()
        env:
          TARGET_REPO: ${{ secrets.TARGET_REPO }}  # Add this secret in your repository settings
          TARGET_TOKEN: ${{ secrets.TARGET_TOKEN }}  # Add this secret in your repository settings
          TARGET_BRANCH: main  # Or any other branch you want to push to
        run: |
          cd public
          git init
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "Deploy Hugo site"
          git push --force "https://x-access-token:${{ env.TARGET_TOKEN }}@github.com/${{ env.TARGET_REPO }}.git" master:${{ env.TARGET_BRANCH }}

      - name: Mark directory as safe
        run: git config --global --add safe.directory /local/godevcraft
      - name: Git pull
        run: cd /local/godevcraft/ && git fetch --all && git reset --hard origin/main

```

Таким образом при пуше в мастера, github actions посылает задачу на моего воркера, который забирает проект с гитахаба, обновляет все темы и модули, билдит проект (с минификацией и сборкой мусора), и пушит результат в отдельный репозиторий. После этого он делает `git pull` и все обновляется. Отдельный репозиторий я выбрал осознанно, что бы не хранить рядом сгенерированный фронтенд и исходники в markdown. Кажется от этого репозиторий может стать довольно тяжелым.

# Минутка безопасности

- Все секреты я решил хранить в Github Secrets. Это удобно и безопасно.
- Оба репозитория приватные
- На сервере отключен вход по паролю, только по ssh ключу
- Общение сервера с гитхабом тоже происходит по ssh ключу
- Для воркера я создал отдельного пользователя, который не имеет прав на выполнение команд от имени root. Воркер использует токен для доступа к репозиторию, который я всегда могу отозвать

# Подключаем телеграм

Что бы быть модным, стильным и молодежным, я подключил telegram app. Это делается довольно легко, через [@BotFather](https://t.me/BotFather). Теперь мой блог доступен как через браузер, так и как приложение в телеграме. Под мобилку hugo отлично адаптируется, даже тему можно поменять на светлую или темную.

![alt text]({{- (resources.Get "telegram-app.jpg").RelPermalink -}})
![alt text]({{- (resources.Get "web.png").RelPermalink -}})

![alt text](/static/telegram-app.jpg)
![alt text](/static/web.png)