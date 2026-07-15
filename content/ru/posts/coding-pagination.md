+++
date = '2026-07-11T01:04:10+02:00'
title = 'Агентское программирование. Часть 2.2: Cursor Pagination'
tags = ["best-practices", "software-design", "dotnet", "jardi-tips"]
author = ["Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

* **Фреймворк:** .NET 10
* **Технологии:** ASP.NET Core Web API, PostgreSQL, Entity Framework Core
* **Улучшение:** `Cursor Pagination`

Пример курсорной пагинации через кортежи в `PostgreSQL`:

```postgresql
select id, random_value, created_at
from sample_data
where (created_at, id) < ('2026-07-10 02:17:36.269', '2f9475be-67a9-4afe-8c7e-a0bba409dbea')
order by created_at desc, id desc
limit 100;
```

Реализация курсорной пагинации через кортежи средствами `Entity Framework Core`:

```csharp
public async Task<IEnumerable<T>> GetPagerResultAsync(IQueryable<T> query, DateTime? dateTime, Guid? lastId, int limit, CancellationToken cancellationToken)
{
    query = query.OrderByDescending(x => x.CreatedAt)
                    .ThenByDescending(x => x.Id);
    
    if (dateTime == null || lastId == null)
        return await query.Take(limit).AsNoTracking().ToListAsync(cancellationToken);

    query = query.Where(x => EF.Functions.LessThan(
        ValueTuple.Create(x.CreatedAt, x.Id),
        ValueTuple.Create(dateTime, lastId)))
        .Take(limit);

    return await query.AsNoTracking().ToListAsync(cancellationToken);
}
```

[Расширение AI-скиллов](https://github.com/AleksandrTurkin/api-jardi-tips/commit/5482fee44914917e1414c2124420bb9272ae970b) для работы с курсорной пагинацией.

## Реализация в деталях

Пагинация - одна из важнейших частей оптимизации работы с БД. Она позволяет эффективно извлекать данные из больших таблиц и не загружать всё сразу.

Давайте начнём немного с другого конца: сначала посмотрим, как эффективно сформировать запрос на уровне базы данных, а затем реализуем этот подход средствами `Entity Framework Core` в нашем приложении.

### Постраничная пагинация

Я уверен, что многие из вас сталкивались со стандартной постраничной пагинацией и реализовывали её.

Что для этого нужно:

- Посчитать, сколько всего записей есть в таблице.
- Определиться с количеством записей на странице и вычислить, сколько страниц потребуется для отображения всех записей.
- Извлечь данные для конкретной страницы.

1. Посчитаем количество записей в таблице `sample_data`. В нашем случае в таблице `1 100 000` записей.

```postgresql
select count(*)
from sample_data
```

Данная операция довольно легковесная:

```text
1 row(s) fetched - 0.055s
```

2. Определяем количество записей на странице и вычисляем, сколько страниц потребуется для отображения всех данных. В нашем случае пусть на одной странице будет `100` записей, тогда количество страниц будет равно `1 100 000 / 100 = 11 000`.

3. Давайте выгрузим данные для конкретной страницы, например для страницы почти в самом конце.

```postgresql
select id, random_value, created_at
from sample_data
order by created_at desc, id desc
limit 100 offset (10900 - 1) * 100;
```

А вот тут уже интересно: эта операция заняла почти 2 секунды.

```text
100 row(s) fetched - 1.912s
```

Выгрузить 100 записей за 2 секунды - это, мягко говоря, медленно. Но если выгрузить данные, например, со второй страницы, то всё работает очень быстро.

```text
100 row(s) fetched - 0.09s
```

При запросе глубокой страницы PostgreSQL спотыкается о прожорливый узел `Sort`: базе данных приходится поднять из диска/памяти и отсортировать `1 090 000` строк, потратив на это почти 2 секунды ради того, чтобы в самом конце отдать финальную сотню.

```text
->  Sort  (cost=155788.90..158538.90 rows=1100000 width=28) (actual time=1681.957..1976.727 rows=1090000.00 loops=1)
```

В то же время для второй страницы срабатывает оптимизация раннего выхода: параллельный узел `Gather Merge` обрабатывает всего 200 строк и укладывается в незаметные 0.09 секунды.

```text
->  Gather Merge  (cost=33481.18..161594.19 rows=1099999 width=28) (actual time=192.518..203.305 rows=200.00 loops=1)
```

### Курсорная пагинация

Курсорная пагинация позволяет эффективно получать следующую порцию данных без необходимости каждый раз пропускать большое количество строк через `OFFSET`.

Вместо номера страницы мы используем значения последней записи предыдущей страницы как курсор для следующей страницы.

Для реализации курсорной пагинации нам нужны:

- идентификатор последнего элемента предыдущей страницы;
- критерий сортировки, по которому мы получаем данные.

Важно, чтобы сортировка была стабильной. Если мы сортируем по `created_at`, то при одинаковых значениях `created_at` порядок строк может быть неоднозначным. Поэтому в качестве дополнительного tie-breaker мы используем `id`.

Пример реализации курсорной пагинации:

```postgresql
select id, random_value, created_at
from sample_data
where created_at < '2026-06-10 09:00:17.135'
   or (created_at = '2026-06-10 09:00:17.135'
       and id < 'fc0852b4-7f1c-47e3-8eda-dd4166cf86be')
order by created_at desc, id desc
limit 100;
```

Результаты запроса для глубокой страницы:

```text
100 row(s) fetched - 0.113s
```

Результаты для второй страницы:

```text
100 row(s) fetched - 0.154s
```

Как можно увидеть, у курсорной пагинации отсутствует деградация производительности при переходе на глубокие страницы, в отличие от стандартной пагинации с `OFFSET`.

PostgreSQL позволяет описать курсорную пагинацию в более лаконичном варианте через сравнение кортежей.

```postgresql
select id, random_value, created_at
from sample_data
where (created_at, id) < ('2026-07-10 02:17:36.269', '2f9475be-67a9-4afe-8c7e-a0bba409dbea')
order by created_at desc, id desc
limit 100;
```

Сравнение через кортежи - это не только синтаксический сахар, но и полезная оптимизация.

1. Один переход по индексу вместо ветвления

Конструкция с `OR` заставляет планировщик PostgreSQL оценивать несколько веток условий. В некоторых случаях это может приводить к менее оптимальным планам, например к `Bitmap Index Scan`. Сравнение кортежей база видит как единое условие и может выполнить быстрый `Index Scan` по B-Tree индексу.

2. Масштабируемость на 3+ колонки

Если пагинация идёт по трём и более полям, например `priority`, `created_at`, `id`, булева логика с `OR` быстро превращается в громоздкое условие. Кортеж `(a, b, c) < (x, y, z)` масштабируется намного проще и остаётся понятным для базы данных.

3. Более стабильные планы запросов

Запросы с большим количеством `OR` могут быть чувствительны к статистике распределения данных. Если распределение данных резко изменится, план запроса тоже может измениться не в лучшую сторону. Сравнение кортежей лучше ложится на структуру B-Tree индекса и обычно даёт более стабильный план.

Добавив индекс, курсорная пагинация работает максимально эффективно. Важно, чтобы порядок полей в кортеже соответствовал порядку полей в индексе. Тогда PostgreSQL сможет эффективно использовать B-Tree для быстрого перехода к нужной странице.

```postgresql
CREATE INDEX ix_sample_data_createdat_id ON sample_data ("created_at" DESC, "id" DESC);
```

Как результат, выполнение запроса даже на больших объёмах данных происходит очень быстро:

```text
100 row(s) fetched - 0.002s
```

### Реализация курсорной пагинации через кортежи

Реализация курсора:

```csharp
public record PagedCursor(DateTime DateTime, Guid LastId)
{
    public static string Encode(DateTime date, Guid lastId)
    {
        var cursor = new PagedCursor(date, lastId);
        var json = JsonSerializer.Serialize(cursor);
        
        return WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(json));
    }

    public static PagedCursor? Decode(string? cursor)
    {
        if (string.IsNullOrWhiteSpace(cursor))
            return null;
        
        try
        {
            var json = Encoding.UTF8.GetString(WebEncoders.Base64UrlDecode(cursor));
            return JsonSerializer.Deserialize<PagedCursor>(json);
        }
        catch
        {
            return null;
        }
    }
}

// Курсор будет иметь текстовое представление
// "pageContext": eyJEYXRlVGltZSI6IjIwMjYtMDctMTRUMDA6NTI6MzMuNjg5MjI0WiIsIkxhc3RJZCI6IjAxOWY1ZTFjLTdmN2QtNzk1Zi04ZTdkLWVlODU4NmMzZmU2ZSJ9"
```

Реализация курсорной пагинации через кортежи для `PostgreSQL` средствами `Entity Framework Core`:

```csharp
public async Task<IEnumerable<T>> GetPagerResultAsync(IQueryable<T> query, DateTime? dateTime, Guid? lastId, int limit, CancellationToken cancellationToken)
{
    query = query.OrderByDescending(x => x.CreatedAt)
                    .ThenByDescending(x => x.Id);
    
    if (dateTime == null || lastId == null)
        return await query.Take(limit).AsNoTracking().ToListAsync(cancellationToken);

    query = query.Where(x => EF.Functions.LessThan(
        ValueTuple.Create(x.CreatedAt, x.Id),
        ValueTuple.Create(dateTime, lastId)))
        .Take(limit);

    return await query.AsNoTracking().ToListAsync(cancellationToken);
}
```

Связка курсора и пагинации:

```csharp
protected async Task<PagedResult<TDto>> BaseHandle<TDto>(TQuery request, Func<TEntity, TDto> map, CancellationToken cancellationToken)
{
    var repository = unitOfWork.Repository<TEntity>();

    var query = repository.GetAll();
    var pageContext = ParsePageContext(request.PageContext);

    var limit = request.Limit ?? DefaultLimit;
    var items = (await repository.GetPagerResultAsync(query, pageContext.DateTime, pageContext.LastId, limit + 1, cancellationToken)).ToList();

    var hasMore = items.Count > limit;
    if (hasMore)
        items.RemoveAt(items.Count - 1);

    var cursor = hasMore
        ? PagedCursor.Encode(items[^1].CreatedAt, items[^1].Id)
        : null;
    
    return new PagedResult<TDto>(cursor, items.Select(map).ToList());
}

private (DateTime? DateTime, Guid? LastId) ParsePageContext(string? pageContext)
{
    if (string.IsNullOrWhiteSpace(pageContext))
        return (null, null);

    var context = PagedCursor.Decode(pageContext);
    
    if (context == null)
        return (null, null);

    return (context.DateTime, context.LastId);
}

/// items[^1] - последний элемент в списке, используемый для формирования курсора следующей страницы.
/// limit + 1 - мы запрашиваем на один элемент больше, чем нужно, чтобы определить, есть ли следующая страница.
/// Если следующего элемента нет, значит мы достигли конца списка и не формируем следующий курсор.
```

### AI-скиллы

Как обычно, корректируем наши AI-скиллы. В этот раз - чтобы формирование новых CRUD-операций учитывало пагинацию по курсору.

- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - скилл для создания обработчиков команд и запросов
- [api-endpoint-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md) - скилл для создания API-эндпоинтов
- [db-entity-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/db-entity-creator/SKILL.md) - скилл для создания сущностей базы и шагов миграции

Изменения в AI-скиллах можно посмотреть в [этом коммите](https://github.com/AleksandrTurkin/api-jardi-tips/commit/5482fee44914917e1414c2124420bb9272ae970b).

### Полезные источники

* [Understanding Cursor Pagination and Why It's So Fast (Deep Dive)](https://milanjovanovic.tech/blog/understanding-cursor-pagination-and-why-its-so-fast-deep-dive)
* [Cursor Pagination is the FASTEST - But you can't use it if...](https://www.youtube.com/watch?v=wepqVpRjp64)
* [THIS ONE Trick Made My Database Query 400x FASTER! (Cursor Pagination EXPOSED)](https://www.youtube.com/watch?v=U3LcKY19z_4)

## Болтовня

Я понимаю, что пример, где в таблице больше миллиона записей и надо отображать их по сто записей на одиннадцати тысячах страницах, похож на задачу по физике из жизни: "Давайте представим, что у нас есть невесомые качели в вакууме и на них катаются двое детей" 😁.

Это больше для наглядности: как происходит деградация производительности при работе с большими объёмами данных и почему стандартная постраничная пагинация может быть неэффективной в таких сценариях.

В пагинации по курсору есть свои ограничения. Например, мы не можем перейти на произвольную страницу. Мы можем только реализовать подгрузку данных как "бесконечная лента": подгружаем новые записи по мере прокрутки.

Для нашего будущего приложения такой сценарий подходит идеально.

#### Спасибо! Улыбаемся и пашем! 🚀