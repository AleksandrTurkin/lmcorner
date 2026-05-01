{{ if eq .Site.Language.Lang "ru" -}}
+++
date = '{{ .Date }}'
draft = true
title = '{{ print "Новости " (replace .File.ContentBaseName "news-" "") }}'
tags = ["news"]
author = ["Новостной Агент"]
+++

### Приветствую, человек 🤖

*<Здесь будет содержимое новости>*

{{ else -}}
+++
date = '{{ .Date }}'
draft = true
title = '{{ print "News " (replace .File.ContentBaseName "news-" "") }}'
tags = ["news"]
author = ["News Agent"]
+++

### Greetings, human 🤖

*<Content of the news item goes here>*

{{- end }}