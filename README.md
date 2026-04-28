# Mini API — Tasks (NestJS)

Небольшой REST API для работы со списком задач. Данные хранятся в памяти
(массив внутри сервиса), без базы данных.

## Стек

- [NestJS 10](https://docs.nestjs.com/)
- TypeScript
- `class-validator` + `class-transformer` для валидации входящих данных

## Структура проекта

```
src/
├── main.ts                      # точка входа, глобальный ValidationPipe
├── app.module.ts                # корневой модуль
└── tasks/
    ├── tasks.module.ts          # модуль задач
    ├── tasks.controller.ts      # HTTP-эндпоинты
    ├── tasks.service.ts         # бизнес-логика + in-memory хранилище
    ├── task.entity.ts           # интерфейс Task
    ├── task-status.enum.ts      # enum статусов: new | in_progress | done
    └── dto/
        └── create-task.dto.ts   # DTO для POST /tasks
```

## Установка и запуск

```bash
npm install
npm run start          # обычный запуск
npm run start:dev      # запуск в watch-режиме
```

Сервис поднимется на `http://localhost:3000` (порт можно переопределить
через переменную окружения `PORT`).

## Эндпоинты

### `POST /tasks` — создать задачу

Тело запроса:

```json
{
  "title": "Купить молоко",
  "description": "2 литра",
  "status": "new"
}
```

Поля:

- `title` — обязательное, непустая строка
- `description` — необязательное, строка
- `status` — обязательное, одно из: `new`, `in_progress`, `done`

Ответ `201 Created`:

```json
{
  "id": 1,
  "title": "Купить молоко",
  "description": "2 литра",
  "status": "new"
}
```

При ошибке валидации возвращается `400 Bad Request` с описанием полей.

### `GET /tasks` — получить список задач

Ответ `200 OK`:

```json
[
  {
    "id": 1,
    "title": "Купить молоко",
    "description": "2 литра",
    "status": "new"
  }
]
```

### `GET /tasks/:id` — получить задачу по id (бонус)

- `200 OK` — задача найдена
- `400 Bad Request` — `id` не является числом
- `404 Not Found` — задачи с таким `id` нет:

  ```json
  {
    "statusCode": 404,
    "message": "Task with id 42 not found",
    "error": "Not Found"
  }
  ```

## Примеры запросов (curl)

```bash
# создать задачу
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Купить молоко","description":"2 литра","status":"new"}'

# получить все задачи
curl http://localhost:3000/tasks

# получить задачу по id
curl http://localhost:3000/tasks/1
```

## Примеры запросов (PowerShell)

> Запускайте команды из второго окна PowerShell, пока в первом крутится
> `npm run start`. Чтобы кириллица в консоли отображалась корректно, один
> раз выполните:
>
> ```powershell
> [Console]::OutputEncoding = [Text.UTF8Encoding]::new()
> chcp 65001 | Out-Null
> ```

```powershell
# 1) Создать задачу (валидное тело)
$body = @{ title = 'Купить молоко'; description = '2 литра'; status = 'new' } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:3000/tasks `
  -Method Post `
  -ContentType 'application/json; charset=utf-8' `
  -Body ([Text.Encoding]::UTF8.GetBytes($body))

# 2) Создать задачу без description
$body = @{ title = 'Сделать ТЗ'; status = 'in_progress' } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:3000/tasks `
  -Method Post `
  -ContentType 'application/json; charset=utf-8' `
  -Body ([Text.Encoding]::UTF8.GetBytes($body))

# 3) Получить список всех задач
Invoke-RestMethod http://localhost:3000/tasks

# 4) Получить задачу по id
Invoke-RestMethod http://localhost:3000/tasks/1

# 5) Несуществующая задача — ожидаем 404
try {
  Invoke-RestMethod http://localhost:3000/tasks/999
} catch {
  $code = $_.Exception.Response.StatusCode.value__
  $text = (New-Object IO.StreamReader($_.Exception.Response.GetResponseStream())).ReadToEnd()
  Write-Host "HTTP $code"
  Write-Host $text
}

# 6) Невалидное тело (нет title и status) — ожидаем 400
try {
  Invoke-RestMethod -Uri http://localhost:3000/tasks `
    -Method Post -ContentType 'application/json' `
    -Body '{"description":"oops"}'
} catch {
  $code = $_.Exception.Response.StatusCode.value__
  $text = (New-Object IO.StreamReader($_.Exception.Response.GetResponseStream())).ReadToEnd()
  Write-Host "HTTP $code"
  Write-Host $text
}

# 7) Некорректный status — ожидаем 400
try {
  Invoke-RestMethod -Uri http://localhost:3000/tasks `
    -Method Post -ContentType 'application/json' `
    -Body '{"title":"x","status":"unknown"}'
} catch {
  $code = $_.Exception.Response.StatusCode.value__
  $text = (New-Object IO.StreamReader($_.Exception.Response.GetResponseStream())).ReadToEnd()
  Write-Host "HTTP $code"
  Write-Host $text
}

# 8) Лишнее поле в теле — ValidationPipe(forbidNonWhitelisted) вернёт 400
try {
  Invoke-RestMethod -Uri http://localhost:3000/tasks `
    -Method Post -ContentType 'application/json' `
    -Body '{"title":"x","status":"new","hacker":"yes"}'
} catch {
  $code = $_.Exception.Response.StatusCode.value__
  $text = (New-Object IO.StreamReader($_.Exception.Response.GetResponseStream())).ReadToEnd()
  Write-Host "HTTP $code"
  Write-Host $text
}
```

Альтернатива через `curl.exe` (настоящий curl, не алиас) — самый надёжный
способ передать JSON с кириллицей через файл:

```powershell
'{"title":"Купить молоко","description":"2 литра","status":"new"}' |
  Out-File -Encoding utf8 body.json

curl.exe -i -X POST http://localhost:3000/tasks `
  -H "Content-Type: application/json; charset=utf-8" `
  --data-binary "@body.json"

curl.exe -i http://localhost:3000/tasks
curl.exe -i http://localhost:3000/tasks/1
curl.exe -i http://localhost:3000/tasks/999
```
