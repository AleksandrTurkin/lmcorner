+++
date = '2026-07-30T00:25:35+02:00'
title = 'Copilot CLI vs Gemini CLI: сравнение команд'
tags = ["aiTools"]
author = ["Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

**Copilot CLI сильнее выглядит в:**

- GitHub workflow;
- code review и security review;
- управлении diff;
- sandbox и permissions;
- параллельной работе через `/fleet`;
- agentic-сценариях вроде `/delegate`, `/rubber-duck`, `/research`;
- работе с worktree и экспериментальными сессиями;
- управлении релизами и версией CLI.

**Gemini CLI сильнее выглядит в:**

- custom slash-командах;
- hooks;
- policies и privacy;
- иерархической памяти через `GEMINI.md`;
- явном управлении IDE integration;
- управлении фоновыми shell-процессами;
- folder trust;
- явных командах управления расширениями.

## Сравнение команд Copilot CLI и Gemini CLI

Я сравниваю только фиксированные встроенные интерактивные команды:

- slash-команды;
- документированные формы `@`;
- документированные формы `!`.

В сравнение не входят:

- параметры запуска CLI;
- переменные среды;
- сочетания клавиш;
- пользовательские команды;
- недокументированные или экспериментальные возможности.

Если команда есть у одного CLI, но у другого есть только похожий сценарий без прямого документированного аналога, я оставляю `X` и поясняю различие в описании.

## Основной рабочий процесс

### Модель, планирование и контекст

- **Выбор модели**
    - Copilot CLI: `/model`, `/models`
    - Gemini CLI: `/model manage`, `/model set`

    _Copilot также настраивает усилия рассуждений и контекстное окно._
- **Планирование**
    - Copilot CLI: `/plan`
    - Gemini CLI: `/plan`, `/plan copy`

    _Обе CLI поддерживают планирование без записи перед реализацией._
- **Сжатие контекста**
    - Copilot CLI: `/compact`
    - Gemini CLI: `/compress`

    _Обе команды резюмируют диалог, освобождая место в контекстном окне._
- **Использование контекста**
    - Copilot CLI: `/context`
    - Gemini CLI: `X`

    _Copilot визуализирует использование токенов контекстного окна._
- **Работа с запросом**
    - Copilot CLI: `/ask`, `/refine`
    - Gemini CLI: `X`

    _Можно задать побочный вопрос или улучшить черновой запрос, не теряя текущий контекст._

### Сессии и результаты

- **Возобновление сессии**
    - Copilot CLI: `/resume`, `/continue`
    - Gemini CLI: `/resume`, `/chat`

    _У Gemini `/chat` является псевдонимом и управляет контрольными точками._
- **Управление сессией**
    - Copilot CLI: `/session`, `/sessions`
    - Gemini CLI: `/resume list`, `/resume save`, `/resume delete`, `/resume share`

    _Обе CLI позволяют просматривать, сохранять, возобновлять, публиковать и удалять состояние диалогов._
- **Новый диалог**
    - Copilot CLI: `/clear`, `/new`, `/reset`
    - Gemini CLI: `X`

    _Эти команды начинают новый диалог Copilot._
- **Очистка экрана**
    - Copilot CLI: `X`
    - Gemini CLI: `/clear`

    _Команда очищает видимую историю терминала Gemini, но не обязательно сбрасывает сохраненную сессию._
- **Экспорт и копирование**
    - Copilot CLI: `/share`, `/export`, `/copy`
    - Gemini CLI: `/resume share`, `/copy`

    _Copilot поддерживает ссылки, Markdown, HTML и gist; Gemini экспортирует Markdown или JSON._

## Восстановление, проверка и безопасность

- **Отмена изменений**
    - Copilot CLI: `/undo`, `/rewind`
    - Gemini CLI: `/restore`, `/rewind`

    _Copilot откатывает ход, а Gemini восстанавливает контрольную точку перед вызовом инструмента._
- **Проверка кода**
    - Copilot CLI: `/review`, `/security-review`, `/diff`
    - Gemini CLI: `X`

    _У Copilot есть специализированные рабочие процессы для проверки, безопасности и diff._
- **Песочница**
    - Copilot CLI: `/sandbox`
    - Gemini CLI: `X`

    _Настройка изоляции Copilot на уровне ОС для shell, MCP, LSP и встроенных инструментов._
- **Разрешения**
    - Copilot CLI: `/permissions`, `/reset-allowed-tools`, `/allow-all`, `/yolo`
    - Gemini CLI: `/permissions trust`

    _Copilot управляет подтверждениями сессии, Gemini - доверием к папкам._

## Рабочее пространство, MCP и IDE

- **MCP-серверы**
    - Copilot CLI: `/mcp list`, `/mcp show`, `/mcp add`, `/mcp edit`, `/mcp delete`, `/mcp disable`, `/mcp enable`, `/mcp auth`, `/mcp reload`, `/mcp search`
    - Gemini CLI: `/mcp auth`, `/mcp desc`, `/mcp disable`, `/mcp enable`, `/mcp list`, `/mcp reload`, `/mcp schema`

    _Gemini показывает схемы; Copilot умеет интерактивно добавлять и редактировать серверы._
- **Каталоги рабочего пространства**
    - Copilot CLI: `/add-dir`, `/list-dirs`, `/cwd`, `/cd`
    - Gemini CLI: `/directory add`, `/directory show`, `/dir`

    _Gemini добавляет корни рабочего пространства, а Copilot также меняет каталог сессии._
- **Среда и инструменты**
    - Copilot CLI: `/env`, `/lsp`
    - Gemini CLI: `/tools`

    _Copilot показывает загруженные ресурсы и настраивает языковые серверы, Gemini перечисляет доступные инструменты._
- **Интеграция с IDE**
    - Copilot CLI: `/ide`
    - Gemini CLI: `/ide enable`, `/ide disable`, `/ide install`, `/ide status`

    _Gemini предоставляет явные команды жизненного цикла интеграции._
- **Инструкции проекта**
    - Copilot CLI: `/init`, `/instructions`
    - Gemini CLI: `/init`, `/memory list`, `/memory refresh`, `/memory show`

    _Обе CLI инициализируют инструкции репозитория; Gemini загружает иерархическую память из `GEMINI.md`._

## Агенты и автоматизация

- **Настройка агентов**
    - Copilot CLI: `/agent`, `/agents`, `/subagents`
    - Gemini CLI: `/agents list`, `/agents reload`, `/agents enable`, `/agents disable`, `/agents config`

    _Обе CLI предоставляют обнаружение и настройку агентов, но используют разные модели взаимодействия._
- **Параллельная и фоновая работа**
    - Copilot CLI: `/fleet`, `/tasks`
    - Gemini CLI: `/shells`, `/bashes`

    _Copilot запускает параллельных подагентов, Gemini показывает фоновые shell-процессы._
- **Делегирование и исследование**
    - Copilot CLI: `/delegate`, `/rubber-duck`, `/research`
    - Gemini CLI: `X`

    _Делегирование изменения, второе мнение и глубокое исследование темы._

- **GitHub workflow**
    - Copilot CLI: `/pr`
    - Gemini CLI: `/setup-github`

    _Copilot управляет текущим pull request, Gemini настраивает GitHub Actions для триажа и проверок._
- **Ветвление экспериментальной сессии**
    - Copilot CLI: `/fork`, `/branch`, `/worktree`, `/move`
    - Gemini CLI: `X`

    _Форк сессии или создание изолированного Git worktree._

## Расширения и настройка

- **Расширения и плагины**
    - Copilot CLI: `/plugins`, `/plugin`, `/extensions`, `/extension`
    - Gemini CLI: `/extensions config`, `/extensions disable`, `/extensions enable`, `/extensions explore`, `/extensions install`, `/extensions link`, `/extensions list`, `/extensions restart`, `/extensions uninstall`, `/extensions update`

    _Обе CLI управляют расширениями; Copilot также объединяет плагины, marketplace, MCP-серверы и навыки в одной панели._
- **Навыки**
    - Copilot CLI: `/skills list`, `/skills info`, `/skills add`, `/skills remove`, `/skills reload`
    - Gemini CLI: `/skills enable`, `/skills disable`, `/skills list`, `/skills reload`

    _Обе CLI обнаруживают, включают и управляют повторно используемыми навыками агента._
- **Пользовательские команды**
    - Copilot CLI: `X`
    - Gemini CLI: `/commands list`, `/commands reload`

    _Gemini показывает или перезагружает определения пользовательских slash-команд._
- **Настройки и оформление**
    - Copilot CLI: `/settings`, `/config`, `/theme`, `/statusline`, `/footer`
    - Gemini CLI: `/settings`, `/theme`

    _Copilot умеет обращаться к репозиторной или локальной области настроек и менять строку состояния._
- **Параметры ввода**
    - Copilot CLI: `/terminal-setup`, `/voice`
    - Gemini CLI: `/terminal-setup`, `/vim`

    _Обе CLI настраивают многострочный ввод; голосовой режим есть только у Copilot, режим Vim - только у Gemini._
- **Хуки и политики**
    - Copilot CLI: `X`
    - Gemini CLI: `/hooks`, `/policies`, `/privacy`

    _Gemini управляет хуками жизненного цикла, политиками и согласием на обработку данных._

## Операции и диагностика

- **Предотвращение сна и экспериментальные функции**
    - Copilot CLI: `/keep-alive`, `/caffeinate`, `/experimental`
    - Gemini CLI: `X`

    _Предотвращение сна компьютера и переключение экспериментальных функций Copilot._
- **Запланированная работа**
    - Copilot CLI: `/after`, `/every`
    - Gemini CLI: `X`

    _Планирование отложенных и повторяющихся запросов, навыков или поддерживаемых команд в экспериментальном режиме._
- **Удаленные сессии**
    - Copilot CLI: `/remote`, `/rename`, `/restart`, `/search`, `/find`
    - Gemini CLI: `X`

    _Управление, перезапуск, переименование или поиск по сессии Copilot._
- **Управление релизами**
    - Copilot CLI: `/changelog`, `/release-notes`, `/downgrade`, `/update`, `/upgrade`
    - Gemini CLI: `/upgrade`

    _Gemini открывает страницу повышения уровня, Copilot обновляет или меняет версию CLI._
- **Использование и лимиты**
    - Copilot CLI: `/usage`, `/limits`
    - Gemini CLI: `/stats session`, `/stats model`, `/stats tools`

    _Gemini разделяет статистику по сессии, модели и инструментам._
- **Идентификация и обратная связь**
    - Copilot CLI: `/version`, `/user`, `/login`, `/logout`, `/feedback`, `/bug`, `/app`
    - Gemini CLI: `/about`, `/auth`, `/bug`

    _Copilot дополнительно запускает приложение Copilot._
- **Интерфейс, справка и выход**
    - Copilot CLI: `/clikit`, `/tuikit`, `/help`, `?`, `/exit`, `/quit`
    - Gemini CLI: `/docs`, `/editor`, `/help`, `/?`, `/exit`, `/quit --delete`

    _Обе CLI предоставляют справку; остальные команды показывают компоненты, открывают документацию или завершают работу. Gemini может удалить историю сессии и временные файлы при выходе._

## Префиксы запроса

- **Добавление контекста**
    - Copilot CLI: `@ FILENAME`
    - Gemini CLI: `@<path>`, `@`

    _Обе CLI добавляют файл или каталог в следующий запрос; Gemini применяет Git-aware фильтрацию каталогов и принимает одиночный `@` как буквенный символ._
- **Выполнение shell-команд**
    - Copilot CLI: `! COMMAND`, `!`
    - Gemini CLI: `! COMMAND`, `!`

    _Можно выполнить одну команду или включить shell-режим для нескольких команд._

## Источники и ограничения

Сравнение сделано по официальной документации на 30 июля 2026 года:

- [справочник GitHub Copilot CLI](https://docs.github.com/ru/copilot/reference/copilot-cli-reference/cli-command-reference)
- [команды Gemini CLI](https://geminicli.com/docs/reference/commands/)

## Болтовня

В целом Copilot CLI и Gemini CLI имеют довольно схожий набор возможностей: работа с моделью, планирование, сессии, MCP, workspace, shell-команды, навыки и расширения.

Но различия всё равно хорошо видны. В Copilot CLI есть несколько интересных agentic-сценариев, которых я не нашёл в Gemini CLI как прямых документированных аналогов. Например, `/rubber-duck` - полезная концепция второго мнения от другой модели.

У Gemini CLI мне понравилась идея пользовательских команд через `/commands list` и `/commands reload`. Это удобный способ превращать повторяющиеся промпты в локальные команды. Плюс у Gemini интересно выглядят hooks, policies, privacy-настройки и иерархическая память через `GEMINI.md`.

В итоге я бы не сказал, что один CLI полностью заменяет другой. Copilot CLI больше похож на GitHub-native agentic coding environment, а Gemini CLI - на расширяемый локальный AI-инструмент с хорошей кастомизацией.

![Work Hard](work-hard_ru.png)

#### Спасибо! Улыбаемся и пашем! 🚀