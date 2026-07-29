+++
date = '2026-07-30T00:25:35+02:00'
draft = true
title = 'Copilot CLI vs Gemini CLI: Cравнение команд'
tags = ["aiTools"]
author = ["Александр Т."]
+++

### Всем привет! 🖖

Здесь сопоставлены фиксированные встроенные интерактивные команды Copilot CLI и Gemini CLI, задокументированные на 30 июля 2026 года. В таблицу вошли slash-команды и документированные формы `@` и `!`. Параметры запуска, переменные среды, сочетания клавиш и пользовательские команды не включены.

`X` означает, что у другого CLI нет прямого документированного аналога. Для близких, но не идентичных функций различие указано в описании.

Источники: [справочник GitHub Copilot CLI](https://docs.github.com/ru/copilot/reference/copilot-cli-reference/cli-command-reference) и [команды Gemini CLI](https://geminicli.com/docs/reference/commands/). Официальная страница Gemini CLI пока доступна только на английском; описания Gemini в этой статье переведены с нее.

| Команда Copilot CLI | Команда Gemini CLI | Краткое описание |
| --- | --- | --- |
| `/model [--repo\|--local\|--session]`<br>` [MODEL] /models` | `/model`<br>`[manage\|set <model> [--persist]]` | Выбор и настройка ИИ-модели. Copilot также меняет настройки рассуждений и контекстного окна. |
| `/plan [PROMPT]` | `/plan [copy]` | Использование режима планирования без записи перед реализацией; Gemini умеет копировать утвержденный план. |
| `/compact [FOCUS-INSTRUCTIONS]` | `/compress` | Сжатие контекста диалога, чтобы освободить место в контекстном окне. |
| `/context` | `X` | Показывает использование токенов и визуализацию контекстного окна Copilot. |
| `/ask QUESTION` | `X` | Задает Copilot вопрос в стороне, не добавляя его в историю диалога. |
| `/refine TEXT` | `X` | Переписывает черновой запрос в более ясную формулировку для проверки. |
| `/resume [SESSION-ID]`<br>`/continue [SESSION-ID]` | `/resume`<br>`/chat` | Просмотр и открытие сохраненной сессии; у Gemini `/chat` является псевдонимом и управляет контрольными точками. |
| `/session`, `/sessions` | `/resume`<br>`[list\|save\|resume\|delete\|share]` | Просмотр, переименование, очистка и удаление сессий и контрольных точек. |
| `/clear [PROMPT]`<br>`/new [PROMPT]`, `/reset [PROMPT]` | `X` | Начинает новый диалог Copilot; это не команда очистки экрана Gemini. |
| `X` | `/clear` | Очищает видимую историю и прокрутку терминала Gemini, не обязательно сбрасывая сохраненную сессию. |
| `/share`, `/export` | `/resume share [FILENAME]` | Экспорт или публикация текущего диалога. Copilot поддерживает ссылки, Markdown, HTML и gist, Gemini - Markdown или JSON. |
| `/copy` | `/copy` | Копирует последний ответ модели в буфер обмена. |
| `/undo` | `/restore [TOOL_CALL_ID]` | Отмена изменений: Copilot откатывает последний ход, Gemini восстанавливает контрольную точку перед вызовом инструмента. |
| `/rewind` | `/rewind` | Переход назад по истории диалога с возможностью отменить связанные изменения файлов. |
| `/review [PROMPT]` | `X` | Запускает агент Copilot для проверки изменений локального кода. |
| `/security-review [PROMPT]` | `X` | Запускает специализированную проверку безопасности Copilot для активных локальных изменений. |
| `/diff` | `X` | Открывает интерфейс Copilot для проверки изменений рабочего дерева или ветки. |
| `/sandbox [enable\|disable]` | `X` | Настраивает изоляцию Copilot на уровне ОС для shell, MCP, LSP и встроенных инструментов. |
| `/permissions [show\|reset]`<br>`/reset-allowed-tools` | `/permissions`<br>`trust [DIRECTORY]` | Просмотр или сброс подтверждений Copilot в сессии; Gemini управляет доверием к папкам. |
| `/allow-all [on\|off\|show]`, `/yolo [on\|off\|show]` | `X` | Включает или показывает режим Copilot с разрешениями для всех действий. |
| `/mcp`<br>`[list\|show\|add\|edit\|delete]`<br>`[disable\|enable\|auth\|reload\|search]` | `/mcp`<br>`[auth\|desc\|disable\|enable]`<br>`[list\|reload\|schema]` | Управление MCP-серверами. Gemini дополнительно показывает описания и схемы, Copilot умеет интерактивно добавлять и редактировать серверы. |
| `/add-dir PATH`, `/list-dirs` | `/directory [add\|show]`, `/dir` | Добавление и просмотр дополнительных каталогов рабочего пространства. |
| `/cwd`, `/cd [PATH]` | `X` | Показывает или меняет рабочий каталог сессии Copilot. Команда Gemini добавляет корни рабочего пространства, а не меняет текущий каталог. |
| `/env` | `/tools [desc\|nodesc]` | Просмотр загруженных ресурсов среды Copilot или доступных инструментов Gemini; это родственные, но не одинаковые диагностики. |
| `/lsp`<br>`[show\|test\|reload\|logs\|help]`<br>`[SERVER-NAME]` | `X` | Управление настройкой и журналами языковых серверов Copilot. |
| `/ide` | `/ide`<br>`[enable\|disable\|install\|status]` | Подключение к IDE или управление интеграцией с IDE. Gemini предоставляет явные подкоманды жизненного цикла. |
| `/init` | `/init` | Анализ проекта и создание инструкций репозитория: инструкций Copilot или `GEMINI.md`. |
| `/instructions` | `/memory`<br>`[list\|refresh\|show]` | Просмотр загруженных файлов инструкций Copilot или иерархической памяти Gemini из `GEMINI.md`. |
| `/agent` | `/agents`<br>`[list\|reload\|enable\|disable\|config]` | Выбор агента Copilot либо обнаружение, настройка и включение подагентов Gemini. |
| `/subagents`, `/agents` | `/agents config <agent-name>` | Настройка моделей и лимитов по умолчанию или для конкретных агентов. У Copilot команда сфокусирована на моделях подагентов. |
| `/fleet [PROMPT]` | `X` | Параллельное выполнение частей задачи Copilot подагентами. |
| `/tasks` | `/shells`, `/bashes` | Просмотр и управление задачами Copilot (подагенты и shell) или фоновыми shell-процессами Gemini. |
| `/delegate [PROMPT]` | `X` | Делегирует изменения репозитория Copilot coding agent и создает pull request с помощью ИИ. |
| `/rubber-duck [PROMPT]` | `X` | Запрашивает у агента Copilot на комплементарной модели второе мнение о планах, коде или тестах. |
| `/research TOPIC` | `X` | Запускает глубокое исследование Copilot с GitHub Search и веб-источниками. |
| `/pr [view\|create\|fix\|auto\|automerge]` | `X` | Управляет pull request текущей ветки с помощью Copilot. |
| `X` | `/setup-github` | Настраивает GitHub Actions для триажа задач и проверки pull request с Gemini. |
| `/plugins`, `/plugin`<br>`/extensions`, `/extension` | `/extensions`<br>`[config\|disable\|enable\|explore\|install]`<br>`[link\|list\|restart\|uninstall\|update]` | Управление экосистемами расширений и плагинов. Copilot также управляет плагинами, marketplace, MCP-серверами и навыками из одной панели. |
| `/skills [list\|info\|add\|remove\|reload]` | `/skills [disable\|enable\|list\|reload]` | Обнаружение, включение и управление повторно используемыми навыками агента. |
| `X` | `/commands [list\|reload]` | Просмотр или перезагрузка определений пользовательских slash-команд Gemini. |
| `/settings`, `/config` | `/settings` | Просмотр и изменение настроек CLI. Copilot умеет обращаться к пользовательской, репозиторной и локальной областям прямо в команде. |
| `/theme`<br>`[default\|github\|dim]`<br>`[high-contrast\|colorblind]` | `/theme` | Просмотр или изменение цветовой темы терминала. |
| `/statusline`, `/footer` | `X` | Настройка строки состояния или подвала Copilot. |
| `/terminal-setup` | `/terminal-setup` | Настройка сочетаний клавиш терминала для многострочного ввода. |
| `/voice [on\|off\|models\|devices]` | `X` | Включение голосового режима или выбор голосовой модели и микрофона Copilot. |
| `X` | `/vim` | Включение или выключение режима ввода Gemini в стиле Vim. |
| `/keep-alive`, `/caffeinate` | `X` | Не дает компьютеру перейти в сон, пока сессия Copilot активна или занята. |
| `/experimental [on\|off\|show]` | `X` | Просмотр или переключение экспериментальных функций Copilot. |
| `/after [DELAY PROMPT]` | `X` | Планирует одно отложенное приглашение, навык или команду Copilot в экспериментальной сессии. |
| `/every [INTERVAL PROMPT]` | `X` | Планирует повторяющееся приглашение, навык или команду Copilot в экспериментальной сессии. |
| `/fork [NAME]`, `/branch [NAME]` | `X` | Создает отдельную экспериментальную сессию на основе текущей сессии Copilot. |
| `/worktree [branch\|task]` | `X` | Создает изолированное Git worktree и переключает Copilot на него. |
| `/move [branch\|task]` | `X` | Переносит незафиксированные изменения в новое Git worktree Copilot. |
| `/remote [on\|off]` | `X` | Включает или показывает удаленное управление сессией Copilot. |
| `/rename [NAME]` | `X` | Переименовывает текущую сессию Copilot. |
| `/restart` | `X` | Перезапускает Copilot CLI, сохраняя текущую сессию. |
| `/search [QUERY]`, `/find [QUERY]` | `X` | Ищет по таймлайну диалога Copilot; доступно в экспериментальном режиме. |
| `/changelog`, `/release-notes` | `X` | Показывает журнал изменений Copilot CLI, при необходимости с ИИ-сводкой. |
| `/downgrade VERSION` | `X` | Перезапускает Copilot на выбранной более старой версии CLI; доступно для командных аккаунтов. |
| `/update`, `/upgrade` | `/upgrade` | Обновляет Copilot CLI или открывает страницу повышения уровня Gemini Code Assist. |
| `/usage` | `/stats [session\|model\|tools]` | Показывает использование и статистику. Gemini разделяет данные по сессии, модели и инструментам. |
| `/limits [set\|unset]` | `X` | Показывает или задает лимиты AI Credits Copilot для каждого ответа. |
| `/version` | `/about` | Показывает версию и сведения о CLI; Copilot также проверяет обновления. |
| `/user [show\|list\|switch]` | `/auth` | Просмотр или переключение аккаунта GitHub в Copilot; диалог Gemini меняет способ входа. |
| `/login`, `/logout` | `/auth` | Вход и выход из Copilot либо открытие диалога метода аутентификации Gemini. |
| `/feedback`, `/bug` | `/bug [TITLE]` | Отправка обратной связи или сообщения об ошибке CLI. Gemini по умолчанию открывает issue в своем GitHub-репозитории. |
| `/app` | `X` | Запускает приложение GitHub Copilot или показывает URL его загрузки. |
| `/clikit [COMPONENT]`, `/tuikit [COMPONENT]` | `X` | Предварительный просмотр бизнес-компонентов или компонентов дизайн-системы Copilot CLI. |
| `X` | `/docs` | Открывает документацию Gemini CLI в браузере. |
| `X` | `/editor` | Выбирает поддерживаемый редактор для Gemini CLI. |
| `X` | `/hooks`<br>`[disable-all\|disable\|enable-all\|enable\|list]` | Управляет хуками жизненного цикла Gemini. |
| `X` | `/policies [list]` | Показывает активные политики Gemini по режимам. |
| `X` | `/privacy` | Показывает уведомление о конфиденциальности Gemini и настройку согласия на сбор данных. |
| `/help`, `?` | `/help`, `/?` | Показывает помощь по интерактивным командам. |
| `/exit`, `/quit` | `/exit`, `/quit [--delete]` | Завершает CLI. Gemini дополнительно может удалить историю текущей сессии и временные файлы. |
| `@ FILENAME` | `@<path_to_<wbr>file_or_directory>` | Добавляет содержимое файла или каталога в следующий запрос. Gemini поддерживает Git-aware раскрытие каталогов. |
| `X` | `@` | Передает одиночный символ at в Gemini без изменений. |
| `! COMMAND` | `!<shell_command>` | Выполняет одну shell-команду без отправки ее модели. |
| `!` | `!` | Включает shell-режим для нескольких команд, после чего можно вернуться в CLI. |

#### Спасибо! Улыбаемся и пашем! 🚀