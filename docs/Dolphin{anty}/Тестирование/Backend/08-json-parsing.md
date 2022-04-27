---
id: testing-backend-json-parsing
title: Парсинг JSON ответов
---

# Тестирование списков

Для получения ответа от API в JSON в родительском классе `TestCase` реализован метод `parseJsonResponse`, **использование которого обязательно** в случае, если нужно получить ответ от API и продолжить взаимодействовать с ним.

Например: 

```php
$request = $this->get('/proxy', $this->adminUser->headers);
$responseJson = $this->parseJsonResponse($request);
```

:::caution
Метод parseJsonResponse возвращается JSON-декодированный **объект**, а не массив.
:::