+++
date = '2026-04-30T01:33:34+02:00'
draft = true
title = 'Локальная LLM. Глава 2'
tags = ["'privateLLM'"]
author = ["Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

* **Фреймворк**: .Net 9
* **Библиотека**: [LLamaSharp](https://github.com/SciSharp/LLamaSharp)

```
1. Создаем новый проект `Console Application` на .Net 9
2. NuGet: Install-Package LLamaSharp              <-- Устанавливаем пакет LLamaSharp
3. NuGet: Install-Package LLamaSharp.Backend.Cpu  <-- Устанавливаем бэкенд (вычисления на CPU)
```

## Развертывание в деталях

1. Создаем новый проект `Console Application`. Фрэймворк .Net 9.0.
2. Устанавливаем пакет LLamaSharp через NuGet:
```
    PM> Install-Package LLamaSharp (Tools > NuGet Package Manager > Package Manager Console)    
```
3. Устанавливаем `бэкенд` пакет. Так разработчики LLamaSharp называют нативную скомпилированную c++ библиотеку, нашу любимую [llama.cpp](https://lmcorner.net/ru/posts/local-llm/).
```
    PM> Install-Package LLamaSharp.Backend.Cpu
```
В моем случае, я использую `Cpu` бэкенд. Но в зависимости от вашего сетапа, вы можете выбрать следующие варианты:
  - `LLamaSharp.Backend.Cpu` - для процессоров 
  - `LLamaSharp.Backend.Cuda11` - для старых видеокарт NVIDIA.
  - `LLamaSharp.Backend.Cuda12` - для современных видеокарт NVIDIA (RTX 30-й, 40-й серий и выше).
  - `LLamaSharp.Backend.OpenCL` - для видеокарт AMD и Intel. Универсальный стандарт, который позволяет запустить вычисления на GPU от других производителей (не NVIDIA) под Windows и Linux.



## Болтовня

#### Спасибо! Улыбаемся и пашем! 🚀
