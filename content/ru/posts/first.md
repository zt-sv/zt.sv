+++
date = '2025-12-28T22:20:36+03:00'
draft = false
title = 'От идеи к публикации: как я построил этот блог'
keywords = [
    "hugo",
    "static site generator",
    "статический сайт",
    "блог на Hugo",
    "GitHub Pages",
    "GitHub Actions",
    "автоматический деплой",
    "continuous deployment",
    "CI/CD",
    "Git",
    "Cloudflare",
    "custom domain",
    "CNAME",
    "hugo-blog-awesome",
    "многоязычный сайт",
    "i18n",
    "multilingual blog",
    "markdown",
    "настройка темы",
    "кастомизация шаблонов",
    "CSS",
    "statically generated blog",
    "dev blog",
    "технический блог"
]
modified = '2025-12-31T15:41:36+03:00'
description = 'Как настроить статический сайт на Hugo, развернуть его на GitHub Pages и настроить автоматический деплой через GitHub Actions'
images = ['author.png']
+++

Привет! Я рад представить вам первый пост моего блога - и логично, что он будет посвящён самому блогу. Здесь я расскажу
о том, как я настроил статический сайт на Hugo, развернул его на GitHub Pages и настроил автоматический деплой через
GitHub Actions.

Этот пост одновременно служит руководством для тех, кто хочет сделать что-то подобное, и документацией для меня самого,
чтобы не потеряться в собственных конфигурациях через пару месяцев.

# Почему Hugo + GitHub Pages?

Перед тем как рассказать о технических деталях, пару слов о выборе инструментов.

**Hugo** - генератор статических сайтов, написанный на Go. Он быстрый, простой и не требует базы данных или сервера
приложений. Контент хранится в простых Markdown-файлах, что особенно удобно для блога.

**GitHub Pages** - бесплатный хостинг для статических сайтов, встроенный прямо в GitHub. Контент и конфигурация
находятся в одном репозитории - удобно, надежно и не требует отдельной подписки.

**GitHub Actions** - встроенная в GitHub автоматизация, которая собирает сайт и деплоит его на GitHub Pages с каждым
push'ем. Никаких ручных команд.

Комбинация этих трёх инструментов дает мне:

- Простоту контроля версий через Git;
- Отсутствие постоянных платежей (за исключением оплаты за собственный домен);
- Полную автоматизацию процесса публикации;
- Версионирование всего контента;
- Возможность писать посты как в обычном текстовом редакторе.

# Шаг 1: Подготовка к работе

## Что вам понадобится

1. **Установленный Git** - для работы с репозиториями;
2. **Установленный Hugo** - генератор сайта;
3. **Аккаунт GitHub** - для хостинга и автоматизации;
4. **Текстовый редактор или IDE** - я использую GoLand;
5. **Немного времени** - полный процесс займёт 30–60 минут.

## Установка Hugo

Официальная [документация](https://gohugo.io/installation/) содержит различные способы установки для популярных ОС. Я
использую пакетный менеджер Homebrew для macOS, поэтому для меня это было просто:

```bash
brew install hugo
```

Проверим, что Hugo установился успешно:

```bash
hugo version
```

Команда выводит установленную версию Hugo. В моём
случае: `hugo v0.153.2+extended+withdeploy darwin/amd64 BuildDate=2025-12-22T16:53:01Z VendorInfo=Homebrew`.

# Шаг 2: Создание Hugo-проекта

Создадим новый проект:

```bash
hugo new site zt.sv
```

После выполнения команды Hugo автоматически создаст базовую структуру папок:

```bash
zt.sv
├── archetypes # Шаблоны для новых постов
│   └── default.md
├── assets     # Статические ассеты (изображения, стили)
├── content    # Контент (посты, страницы)
├── data       # YAML/JSON данные
├── hugo.toml  # Конфигурация Hugo
├── i18n       # Локализация
├── layouts    # Кастомные шаблоны HTML
├── static     # Статические файлы
└── themes     # Темы
```

# Шаг 3: Конфигурация

По умолчанию `hugo new site` создает единый файл конфигурации `hugo.toml` в корне проекта, что кажется неудобным для
больших проектов. К счастью, Hugo
поддерживает [конфигурационную директорию](https://gohugo.io/configuration/introduction/#configuration-directory),
позволяющую разбить конфигурацию на несколько файлов. Сначала я конвертирую `toml` в `yaml`, а позже в эту директорию
попадут и другие конфигурационные файлы:

```bash
zt.sv
├── config
│   └── _default
│       └── hugo.yaml
```

Содержимое `config/_default/hugo.yaml`:

```yaml
---
baseURL: https://zt.sv/
languageCode: en-us
defaultContentLanguage: en
title: Aleksandr Zaytsev | ztsv's blog
contentDir: content/en
defaultContentLanguageInSubdir: false
```

Я выбрал путь `content/en` для `contentDir`, поскольку планирую вести блог на нескольких языках и использовать отдельные
директории для каждого. Эту директорию нужно создать самостоятельно:

```bash
mkdir -p content/en
```

# Шаг 4: Установка темы

Для Hugo существует огромное количество [готовых тем](https://themes.gohugo.io/), но я остановился
на [hugo-blog-awesome](https://github.com/hugo-sid/hugo-blog-awesome) от Hugo Sid.

Будем устанавливать тему как модуль, поэтому сначала инициализируем проект как Hugo-модуль:

```bash
hugo mod init github.com/zt-sv/zt.sv
```

Добавим конфигурационный файл с темой в `config/_default/module.yaml`:

```yaml
---
imports:
  - path: github.com/hugo-sid/hugo-blog-awesome
```

Установим тему через установку зависимостей:

```bash
hugo mod vendor
```

После этого можно запустить встроенный в Hugo локальный сервер и убедиться, что всё работает, перейдя по
адресу `http://localhost:1313/`:

```bash
hugo server
```

![First run](posts/first/01.png)

# Шаг 5: Настройка темы

Тема `hugo-blog-awesome` поддерживает различные дополнительные настройки, такие как иконки социальных сетей и выбор
цветовой схемы. К сожалению, в документации описано далеко не всё, но можно найти пример сайта и
его [конфигурацию](https://github.com/hugo-sid/hugo-blog-awesome/blob/main/exampleSite/hugo.toml) в репозитории темы.

Добавим конфигурационный файл с параметрами в `config/_default/params.yaml`:

```yaml
---
dateFormat: "January 2, 2006"

socialIcons:
  - name: github
    url: https://github.com/zt-sv
  - name: RSS
    url: /index.xml

sitename: Aleksandr Zaytsev | ztsv
defaultColor: dark
mainSections:
  - posts

author:
  avatar: author.png
  name: Aleksandr Zaytsev
  intro: Aleksandr Zaytsev | ztsv
  description: K8s, split-keyboards, IaC and something else

webmanifest:
  name: zt.sv
  short_name: zt.sv
  theme_color: "#434648"
  background_color: "#ffffff"
  display: standalone
  start_url: /
```

Добавим `author.png` в директорию `assets`. Кроме того, при
помощи [realfavicongenerator.net](https://realfavicongenerator.net/) я сгенерировал набор фавиконок для сайта и поместил
их в директорию `assets/icons`:

```bash
assets/
├── author.png
└── icons
    ├── apple-touch-icon.png
    ├── favicon-96x96.png
    ├── favicon.ico
    ├── favicon.svg
    ├── web-app-manifest-192x192.png
    └── web-app-manifest-512x512.png
```

# Шаг 6: Поддержка нескольких языков

Я планировал вести данный блог одновременно на двух языках: английском и русском. Поэтому нужно было добавить на сайт
функцию выбора языка.

Тема `hugo-blog-awesome` поддерживает выбор языка из коробки, поэтому достаточно определить локали
в `config/_default/languages.yaml`, в котором можно также переопределять отдельные параметры для каждого языка:

```yaml
---
en:
  disabled: false
  languageCode: en-us
  languageDirection: ltr
  languageName: 🇺🇸
  weight: 1
ru-ru:
  disabled: false
  languageCode: ru-ru
  languageDirection: ltr
  languageName: 🇷🇺
  weight: 2
  params:
    dateFormat: "2 January 2006"
    author:
      name: Александр Зайцев
      intro: Александр Зайцев | ztsv
      description: K8s, сплит-клавиатуры, IaC и всякая всячина
  contentDir: content/ru
```

После этого на сайте появится селектор выбора языка:

![Language select](posts/first/02.png)

# Шаг 7: Кастомизация

Любую тему можно донастроить или переделать отдельные шаблоны. Например, в выборе языка отображаются `LanguageCode`, что
кажется неэлегантным, поэтому я переделал это на `languageName`. Для этого достаточно
переопределить [оригинальный шаблон header](https://github.com/hugo-sid/hugo-blog-awesome/blob/main/layouts/partials/header.html),
разместив новый шаблон в `layouts/partials/header.html` и внеся свои правки.

Для использования собственных стилей я добавил `layouts/partials/custom-head.html` со следующим содержимым:

```html

<link href="{{ (resources.Get "css/custom.css").RelPermalink }}" rel="stylesheet">
```

И добавил соответствующий файл `assets/css/custom.css`:

```css
html.dark .author .author-avatar {
    border-color: white;
}

.author .author-avatar {
    width: 200px;
    height: 200px;
    border-radius: 25%;
    border: 5px #0d122b solid;
}

.lang-list {
    background: transparent;
    border: none;
    color: #0d122b;
    line-height: 2.25;
    text-decoration: none;
    padding: .3rem .5rem;
    opacity: .7;
    letter-spacing: .015rem;
    font-size: 16px;
}

html.dark .lang-list {
    color: #eaeaea;
}
```

После внесённых изменений сайт стал выглядеть так:

![Styled](posts/first/03.png)

# Шаг 8: Меню и страница «Обо мне»

Для добавления основного меню на сайте необходимо создать файл `config/_default/menu.yaml`. Пока добавлю в меню только
пункт «Обо мне»:

```yaml
---
main:
  - identifier: "menu.about"
    url: /about/
    pageRef: about
```

Поле `identifier` позволяет указать различные варианты текста для разных языков. Поэтому добавим файлы `i18n/en.yaml`
и `i18n/ru-ru.yaml`, в которых каждому идентификатору присвоим перевод:

```yaml
- id: "menu.about"
  translation: "About"
```

И создадим саму страницу:

```bash
hugo new pages/about.md
```

Эта команда создаст файл `pages/about.md` из шаблона `archetypes/default.md` в директории контента по умолчанию (
параметр `contentDir` в файле `hugo.yaml`).

# Шаг 9: Создание репозитория

Инициализируем локальный Git-репозиторий:

```bash
git init
```

И настраиваем файл `.gitignore`:

```gitignore
_vendor/
public/
resources/
.hugo_build.lock
```

После этого создаём репозиторий на [GitHub](https://github.com). При создании репозитория я пропускаю автоматическое
добавление файлов `README.md`, `.gitignore` и файла лицензии.

![Create repository on GitHub](posts/first/04.png)

Делаем первый коммит:

```bash
git add .
git branch -M main
git commit -m "Initial commit"
```

Подключаем созданный удалённый репозиторий на GitHub и пушим изменения:

```bash
git remote add origin git@github.com:zt-sv/zt.sv.git
git push -u origin main
```

# Шаг 10: Собственный домен

Поскольку я планирую использовать собственный домен для данного сайта, необходимо добавить простой текстовый
файл `CNAME`:

```text
zt.sv
```

И закоммитить его:

```bash
git add CNAME
git commit -m "Create CNAME"
git push
```

Я использую Cloudflare в качестве DNS-провайдера, поэтому внёс
IP-адреса [со страницы документации GitHub](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain)
в личный кабинет Cloudflare. Я добавил `A` и `AAAA` записи для основного домена и `CNAME` запись поддомена `www`.

![Cloudflare DNS management](posts/first/06.png)

После этого я внёс свой домен в настройках репозитория на GitHub в разделе `Pages` и выбрал ветку `gh-pages` в качестве
источника.

![GitHub custom domain setup](posts/first/07.png)

# Шаг 11: GitHub Actions и автоматическая публикация

Для автоматической сборки и публикации проекта я использую GitHub Actions, поэтому создам
файл `.github/workflows/deploy.yml`:

```yaml
name: Deploy Hugo site

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.153.2'
          extended: true

      - name: Install hugo modules
        run: hugo mod vendor

      - name: Build
        run: hugo --minify

      - name: Add CNAME file
        run: cp CNAME public/CNAME

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

Этот workflow:

1. Запускается каждый раз при push'е в ветку `main`;
2. Скачивает исходный код репозитория;
3. Устанавливает Hugo;
4. Устанавливает зависимости;
5. Собирает сайт (команда `hugo --minify`);
6. Добавляет к собранному сайту файл CNAME;
7. Деплоит результат на GitHub Pages (в специальную ветку `gh-pages`).

Перед тем как закоммитить изменения, необходимо перейти в настройки созданного репозитория и в разделе `Actions/general`
установить разрешение на чтение и запись для workflow'ов.

![Allow workflows to read and write](posts/first/05.png)

Отправляем изменения в GitHub:

```bash
git add .github/workflows/deploy.yml
git commit -m "Build and deploy hugo site"
git push
```

# Что дальше?

Теперь у вас есть:

- ✅ Работающий статический блог на Hugo;
- ✅ Бесплатный хостинг на GitHub Pages;
- ✅ Автоматический деплой при каждом push'е;
- ✅ Версионирование всего контента.

На этом фундаменте можно:

- Добавлять новые посты (просто новые `.md` файлы);
- Кастомизировать стили и макет;
- Добавлять новые функции (поиск, комментарии, аналитика).

**P.S.** Исходный код этого блога [доступен на GitHub](https://github.com/zt-sv/zt.sv). Если вы захотите посмотреть
точную конфигурацию или скопировать что-то оттуда, добро пожаловать!
