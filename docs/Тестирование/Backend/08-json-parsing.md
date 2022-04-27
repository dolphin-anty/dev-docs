---
id: testing-backend-json-parsing
title: Парсинг JSON ответов
---

# Тестирование списков

Для получения ответа от API в JSON в родительском классе `TestCase` реализован метод `toJson`, **использование которого обязательно** в случае, если нужно получить ответ от API и продолжить взаимодействовать с ним.

Например: 

```php
$request = $this->get('/proxy', $this->adminUser->headers);
$responseJson = $this->parseJsonResponse($request);
```

## paginationAssertions

Делается assertions наличия всех необходимых метаданных пагинации.

## listAssertions

Проводит проверку:

- Но наличие поля `data` в ответе
- На то, что поле `data` - это массив