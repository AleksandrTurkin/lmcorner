{{ if eq .Site.Language.Lang "ru" -}}
+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
tags = [""]
author = ["Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

## Развертывание в деталях

## Болтовня

#### Спасибо! Улыбаемся и пашем! 🚀
{{ else -}}
+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
tags = [""]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

### Recap

## Step-by-Step Implementation

## Blabber

#### Thanks! Keep calm and code on! 🚀
{{- end }}