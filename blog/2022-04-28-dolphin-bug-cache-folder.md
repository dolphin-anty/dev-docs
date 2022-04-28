---
slug: dolphin-bug-cache-folder
title: Перегрузка серверов у новых пользователей
authors:
  name: Denis Zhitnyakov
tags: [dolphin, dolphin-server, bugs, dolphin-server-bugs]
---

# Перегрузка серверов у новых пользователей

## Проблема у юзеров

https://dolphin-ru-com.atlassian.net/jira/software/projects/DSS/boards/51?selectedIssue=DSS-295

Новые пользователи после установки дельфина ловили бесконечный загруз сервера на 100%. Так, что было почти невозможно ничего делать.

## Проблема под капотом

У новых пользователей в обязательном порядке почти сразу после установки Dolphin стартует загрузка баз гео, которая скачивается [отсюда](https://github.com/dolphinrucom/fb-lists/releases/tag/v1) и затем вгружается в БД сервера юзера.

Примечательно, что в сжатом виде файл гео базы около 6MB. В распакованном - около 96MB. А в конечном счете базу данных сервера кладется около 200 000 записей - стран, регионов, городов, таргетов поведения, интересов и демографических признаков. В общем всего того, что есть в самом фб при настройке таргетов.

То есть такой объем данных не может вгрузиться в БД за 1 минуту. И именно поэтому на время, пока данные вгружаются мы делаем следующий финт:

```php
// ...
$this->load->driver('cache', array('adapter' => 'file'));

if ($this->cache->get('fb_lists')) {
    return;
}

try {
    $this->cache->save('fb_lists', '1', 3600);
// ...
```

Таким образом мы записываемся в кэш о том, что мы стартовали вгрузку данных в БД. И следующая задача по загрузке гео базы, которая стартует через минуту, уже не запустится.

**Но что, если кэш не работает в данный момент как функция?** В таком случае крон будет запускать процесс обновления гео базы каждую минуту, так как к началу новой минуты у нас не будет достаточно элементов в нужных таблицах. Это проверяется здесь:

```php
if ($this->db->count_all_results('fb_countries') <= self::MIN_COUNTRIES_COUNT) {
    $empty_countries = true;
}
if ($this->db->count_all_results('fb_regions') <= self::MIN_REGIONS_COUNT) {
    $empty_regions = true;
}
if ($this->db->count_all_results('fb_cities') <= self::MIN_CITIES_COUNT) {
    $empty_cities = true;
}
if ($this->db->count_all_results('fb_interests') <= self::MIN_INTERESTS_COUNT) {
    $empty_interests = true;
}
if ($this->db->count_all_results('fb_behaviors') <= self::MIN_BEHAVIORS_COUNT) {
    $empty_behaviors = true;
}
if ($this->db->count_all_results('fb_demographics') <= self::MIN_DEMOGRAPHICS_COUNT) {
    $empty_demographics = true;
}

if (
    ($empty_demographics || $empty_countries || $empty_regions || $empty_cities || $empty_interests || $empty_behaviors)
    || ($h == 3 && $m == 13)
) {
  // ... 
}
```

Смотря на содержание таблицы fb_cities (самая крупная таблица, около 150 000 записей), я увидел картинку - данные наполняются, наполняются. Наступает новая минута, и количество записей становится = 0. И наполняется заново.

Затем отправился смотреть на наличие файлов кэша. Во фреймворке CodeIgniter кэш хранится в директории `/var/www/html/new/application/cache`. Ее просто не существовало...

## Решение

В pipeline сборки приложения добавил команды:

```bash
mkdir -p /var/www/html/application/cache &&\
  chmod -R 777 /var/www/html/application/cache

mkdir -p /var/www/html/new/application/cache &&\
  chmod -R 777 /var/www/html/new/application/cache
```