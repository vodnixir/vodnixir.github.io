# Лабораторная работа 3. CI/CD для статического сайта в SourceCraft и GitHub Actions

## Цель работы

Целью лабораторной работы было изучение основ CI/CD на примере автоматического развертывания статического сайта, созданного с помощью MkDocs.

В рамках работы требовалось:

- настроить автоматическое развертывание сайта на платформе SourceCraft;
- настроить автоматическое развертывание этого же сайта через GitHub Actions;
- использовать один локальный репозиторий и два удалённых репозитория;
- изучить, какие настройки необходимо выполнить в интерфейсах GitHub и SourceCraft для корректной публикации сайта.

## Исходные данные

В качестве основы использовался ранее созданный статический сайт-портфолио лабораторных работ на MkDocs.

Локально проект содержал:

- исходные Markdown-страницы сайта;
- конфигурацию `mkdocs.yml`;
- ранее собранную версию сайта;
- Git-репозиторий, связанный с GitHub.

## Что было сделано

### 1. Подготовка репозитория

В работе использовался один локальный Git-репозиторий и два удалённых репозитория:

- `origin` — GitHub-репозиторий;
- `sourcecraft` — репозиторий на платформе SourceCraft.

Для подключения второго удалённого репозитория была использована команда:

```bash
git remote add sourcecraft https://git.sourcecraft.dev/<имя_аккаунта>/<имя_репозитория>.git
```

После этого стало возможным пушить изменения сразу в оба репозитория независимо:

```bash
git push origin main
git push sourcecraft main
```

### 2. Настройка CI/CD в SourceCraft

В корне репозитория был создан файл `.sourcecraft/ci.yaml` с описанием pipeline:

```yaml
on:
  push:
    - workflows: [build-site]
      filter:
        branches: ["main"]

workflows:
  build-site:
    tasks:
      - name: build-mkdocs
        cubes:
          - name: build
            image: docker.io/library/python:3.13-slim
            script:
              - pip install mkdocs mkdocs-material
              - cd source
              - mkdocs build -d ../site
```

Pipeline запускается автоматически при каждом пуше в ветку `main`. Он устанавливает зависимости и собирает сайт в папку `site`, которую SourceCraft затем публикует.

Дополнительно был создан файл `.sourcecraft/sites.yaml`, указывающий SourceCraft, из какой папки брать готовый сайт:

```yaml
site:
  root: "site"
  ref: "main"
```

### 3. Настройка CI/CD в GitHub Actions

Для GitHub Pages был создан файл `.github/workflows/pages.yml`:

```yaml
name: Deploy MkDocs to GitHub Pages

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - run: pip install mkdocs mkdocs-material
      - run: cd source && mkdocs build --site-dir ../_site
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./_site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v4
```

В настройках репозитория на GitHub в разделе **Settings → Pages** источником была выбрана опция **GitHub Actions** вместо ветки `gh-pages`.

### 4. Проверка работы

После пуша в `main` оба pipeline запустились автоматически:

- в SourceCraft: статус сборки виден в интерфейсе платформы;
- в GitHub: статус сборки отображается во вкладке **Actions** репозитория.

Сайт стал доступен по адресу `https://vodnixir.github.io/vodnixir.github.io/` после завершения деплоя на GitHub и по адресу SourceCraft — после завершения их pipeline.

## Выводы

В ходе работы был реализован полноценный CI/CD-процесс для статического сайта с двойным деплоем. Ключевые наблюдения:

- один локальный репозиторий может иметь несколько удалённых — это позволяет деплоить на несколько платформ без дублирования кода;
- GitHub Actions и SourceCraft CI имеют схожую структуру: триггер на событие → шаги сборки → публикация;
- главное различие — в GitHub деплой разделён на два джоба (build и deploy) из соображений безопасности, у SourceCraft процесс более компактный;
- автоматическая сборка исключает ошибки ручного деплоя и гарантирует, что опубликованная версия сайта всегда соответствует коду в репозитории.
