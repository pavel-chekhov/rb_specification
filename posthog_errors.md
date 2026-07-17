# Техническое задание

## Мониторинг загрузки и работы PostHog на клиенте

> Техническая реализация остается на усмотрение специалистов. Решения и примеры в этом документе носят рекомендательный характер и не являются строго обязательными, если выбранный способ обеспечивает описанный результат.

### 1. Цель

Нужно отслеживать:

1. Успешный запуск Event Collector.
2. Начало попытки загрузки файла PostHog.
3. Успешную загрузку файла PostHog.
4. Ошибку загрузки файла PostHog.
5. Успешную инициализацию PostHog.
6. JavaScript-ошибки и необработанные ошибки Promise, связанные с PostHog.

Все диагностические события отправляются в Event Collector через функцию:

```javascript
sendToEventCollector(payload);
```

Event Collector не должен зависеть от PostHog. Загрузка и инициализация PostHog не должны зависеть от Event Collector или диагностического мониторинга.

### 2. `page_load_id`

При загрузке HTML-документа нужно один раз создать уникальный `page_load_id`. Он связывает события Event Collector и автоматически добавляется во все события PostHog.

Также необходимо записать её в переменную `dataLayer` (dataLayer.push({page_load_id: "<сгенерированное значение>"})).

```javascript
function createPageLoadId() {
  var cryptoApi = window.crypto || window.msCrypto;

  if (cryptoApi && typeof cryptoApi.randomUUID === 'function') {
    return cryptoApi.randomUUID();
  }

  if (cryptoApi && typeof cryptoApi.getRandomValues === 'function') {
    var bytes = new Uint8Array(16);
    cryptoApi.getRandomValues(bytes);

    return Array.from(bytes, function (byte) {
      return byte.toString(16).padStart(2, '0');
    }).join('');
  }

  return Date.now().toString(36) + '-' + Math.random().toString(36).slice(2);
}

window.PAGE_LOAD_ID = window.PAGE_LOAD_ID || createPageLoadId();

window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  page_load_id: window.PAGE_LOAD_ID
});
```

`page_load_id`:

- создается один раз при загрузке HTML-документа;
- не меняется до полной перезагрузки страницы;
- остается прежним при повторной инициализации на этой странице;
- передается в поле `page_load_id` событий Event Collector;
- регистрируется в PostHog как super property и добавляется во все последующие события PostHog.

### 3. События

#### 3.1. `event_collector_initialized`

Событие отправляется после успешного запуска Event Collector.

```javascript
{
  event_name: 'event_collector_initialized',
  page_load_id: string,
  page_url: string,
  referrer: string | null,
  user_agent: string,
  timestamp: number
}
```

При повторном запуске Event Collector (в норме такого не должно происходить) событие отправляется еще раз с тем же `page_load_id`.

#### 3.2. `posthog_script_load_requested`

Событие отправляется после того, как штатный loader успешно добавил элемент `<script>` PostHog в DOM и опубликовал lifecycle-сигнал о начале попытки загрузки.

```javascript
{
  event_name: 'posthog_script_load_requested',
  page_load_id: string,
  script_url: string,
  page_url: string,
  referrer: string | null,
  user_agent: string,
  timestamp: number
}
```

`posthog_script_load_requested` означает начало попытки загрузки, но не подтверждает успешную загрузку файла.

#### 3.3. `posthog_script_loaded`

Событие отправляется после успешной загрузки JavaScript-файла PostHog.

Условие отправки: обработчик `load`, установленный штатным loader на элементе `<script>` PostHog, опубликовал соответствующий lifecycle-сигнал.

```javascript
{
  event_name: 'posthog_script_loaded',
  page_load_id: string,
  script_url: string,
  load_time_ms: number | null,
  page_url: string,
  referrer: string | null,
  user_agent: string,
  timestamp: number
}
```

- `script_url` — URL загруженного файла.
- `load_time_ms` — время загрузки в миллисекундах, если его можно определить.

#### 3.4. `posthog_script_load_error`

Событие отправляется, если браузер не смог загрузить JavaScript-файл PostHog.

Условие отправки: обработчик `error`, установленный штатным loader на элементе `<script>` PostHog, опубликовал соответствующий lifecycle-сигнал.

```javascript
{
  event_name: 'posthog_script_load_error',
  page_load_id: string,
  script_url: string,
  page_url: string,
  referrer: string | null,
  user_agent: string,
  timestamp: number
}
```

Причиной может быть сетевой сбой, недоступность CDN, DNS-ошибка, CSP, блокировщик, неверный URL или HTTP-ошибка. Браузер обычно не сообщает точную причину.

#### 3.5. `posthog_initialized`

Событие отправляется после успешной инициализации PostHog. Для этого используется callback `loaded` в `posthog.init()`.

```javascript
posthog.init('<ph_project_token>', {
  api_host: '<ph_api_host>',
  loaded: function (posthog) {
    posthog.register({
      page_load_id: window.PAGE_LOAD_ID
    });

    trackPostHogInitialized();
  }
});
```

Payload:

```javascript
{
  event_name: 'posthog_initialized',
  page_load_id: string,
  page_url: string,
  referrer: string | null,
  user_agent: string,
  timestamp: number
}
```

В `page_load_id` передается `window.PAGE_LOAD_ID`.

Callback `loaded` описан в [документации PostHog](https://posthog.com/docs/libraries/js/config). Внутри него можно вызвать `posthog.capture()`, но в этой реализации он не используется. Событие отправляется через независимый Event Collector.

Перед вызовом `trackPostHogInitialized()` значение регистрируется через `posthog.register()` как super property. Оно автоматически добавляется к последующим `$pageview`, autocapture и ручным событиям PostHog. `register_once()` использовать нельзя: после полной перезагрузки страницы старое значение должно быть заменено новым.

Регистрация выполняется внутри `loaded` до первого автоматического `$pageview`. `page_load_id` является свойством события и не создает person property.

При повторном срабатывании `loaded` событие отправляется еще раз с тем же `page_load_id`.

#### 3.6. `posthog_js_error`

Событие отправляется при ошибке, связанной с PostHog.

Поддерживаются два источника:

- обычная JavaScript-ошибка — `window_error`;
- необработанная ошибка Promise — `unhandled_rejection`.

```javascript
{
  event_name: 'posthog_js_error',
  error_source: 'window_error' | 'unhandled_rejection',
  error_name: string | null,
  message: string,
  stack: string | null,
  filename: string | null,
  lineno: number | null,
  colno: number | null,
  page_url: string,
  user_agent: string,
  timestamp: number
}
```

Для обычной ошибки данные берутся из события `window.error`.

Для Promise-ошибки обработчик слушает `window.unhandledrejection`. Поля `error_name`, `message` и `stack` берутся из `event.reason`, если они доступны. Если `reason` не является объектом `Error`, его значение приводится к строке для поля `message`.

### 4. Как определить связь ошибки с PostHog

Ошибка считается связанной с PostHog, если выполняется хотя бы одно условие:

1. `filename` содержит строку `posthog`.
2. `stack` содержит строку `posthog`.
3. URL файла совпадает с известным URL или доменом PostHog.

Сравнение выполняется без учета регистра.

```javascript
String(value).toLowerCase().includes('posthog');
```

### 5. Общие поля

К событиям добавляются общие поля:

```javascript
{
  page_load_id: window.PAGE_LOAD_ID,
  page_url: window.location.href,
  referrer: document.referrer || null,
  user_agent: navigator.userAgent,
  timestamp: Date.now()
}
```

### 6. Отправка в Event Collector

Функция `sendToEventCollector()` должна:

1. Не зависеть от PostHog.
2. Не выбрасывать наружу необработанные ошибки.
3. Не создавать цикл ошибок, если сбой произошел внутри Event Collector.
4. Использовать короткий сетевой тайм-аут.
5. По возможности использовать `navigator.sendBeacon()` или `fetch()` с `keepalive: true`.
6. Не блокировать загрузку страницы.
7. Не передавать чувствительные данные.

Защитный вызов:

```javascript
try {
  sendToEventCollector(payload);
} catch (collectorError) {
  // Ошибка Event Collector не должна нарушать работу страницы.
}
```

### 7. Дедупликация

К событию `posthog_js_error` применяется дедупликация.

Ключ дедупликации:

```text
error_source + message + filename + lineno + colno
```

Не нужно дедублицировать:

- `event_collector_initialized`;
- `posthog_initialized`;
- `posthog_script_load_requested`;
- `posthog_script_loaded`;
- `posthog_script_load_error`.

Каждая попытка загрузки должна формировать собственную последовательность lifecycle-событий. Повторные события инициализации также важны: они позволяют увидеть повторный запуск, которого в норме быть не должно.

### 8. Анализ инициализации

События нужно сравнивать по `page_load_id`.

- `posthog_script_load_requested` → `posthog_script_loaded` → `posthog_initialized` — файл успешно загружен, PostHog инициализирован.
- `posthog_script_load_requested` → `posthog_script_load_error` — файл не загрузился; `posthog_script_loaded` и `posthog_initialized` для этой попытки отсутствуют.
- `posthog_script_load_requested` → `posthog_script_loaded` без `posthog_initialized` — файл загрузился, но PostHog не был успешно инициализирован.
- Один `event_collector_initialized` и один `posthog_initialized` — нормальная работа.
- Есть `event_collector_initialized`, но нет `posthog_initialized` — PostHog не инициализировался.
- Несколько `event_collector_initialized` с одним `page_load_id` — Event Collector запущен повторно.
- Несколько `posthog_initialized` с одним `page_load_id` — callback успешной инициализации PostHog сработал повторно.

Сравнения только общего количества событий недостаточно. Количество нужно считать отдельно для каждого `page_load_id`.

### 9. Порядок подключения

Мониторинг подключается до основного скрипта PostHog.

Порядок:

1. Создать `window.PAGE_LOAD_ID`.
2. Инициализировать Event Collector.
3. Отправить `event_collector_initialized`.
4. Зарегистрировать обработчик внутреннего события `posthog:lifecycle`.
5. Зарегистрировать обработчики `window.error` и `window.unhandledrejection` для ошибок выполнения JavaScript и Promise.
6. В штатном loader назначить обработчики `load` и `error` элементу `<script>` PostHog.
7. В штатном loader добавить `<script>` в DOM и опубликовать `posthog_script_load_requested` через `posthog:lifecycle`.
8. Из обработчика `load` или `error` опубликовать соответствующий lifecycle-сигнал.
9. Из callback `loaded` отправить `posthog_initialized`.

Если диагностический мониторинг не загрузился или завершился ошибкой, штатный loader должен продолжить загрузку PostHog.

### 10. Контракт штатного loader и мониторинга

Создание и добавление элемента `<script>` PostHog в DOM выполняет штатный bootstrap-loader. Все попытки загрузки PostHog должны проходить через этот loader. Диагностический мониторинг не должен создавать, добавлять или заменять элемент `<script>` PostHog.

Loader публикует внутреннее событие `posthog:lifecycle`. Оно не отправляется напрямую в Event Collector и используется только как интерфейс между loader и мониторингом.

Поле `detail` имеет структуру:

```javascript
{
  event_name: 'posthog_script_load_requested'
    | 'posthog_script_loaded'
    | 'posthog_script_load_error',
  script_url: string,
  load_time_ms?: number | null
}
```

Пример публикации lifecycle-сигналов внутри штатного loader:

```javascript
function publishPostHogLifecycle(eventName, scriptUrl, loadTimeMs) {
  try {
    var detail = {
      event_name: eventName,
      script_url: scriptUrl
    };

    if (typeof loadTimeMs === 'number') {
      detail.load_time_ms = loadTimeMs;
    }

    window.dispatchEvent(new CustomEvent('posthog:lifecycle', {
      detail: detail
    }));
  } catch (lifecycleError) {
    // Ошибка публикации диагностики не должна влиять на loader.
  }
}

var script = document.createElement('script');
var loadStartedAt = null;

script.src = POSTHOG_SCRIPT_URL;
script.async = true;

script.addEventListener('load', function () {
  var loadTimeMs = loadStartedAt === null
    ? null
    : window.performance.now() - loadStartedAt;

  publishPostHogLifecycle(
    'posthog_script_loaded',
    script.src,
    loadTimeMs
  );
});

script.addEventListener('error', function () {
  publishPostHogLifecycle(
    'posthog_script_load_error',
    script.src
  );
});

if (window.performance && typeof window.performance.now === 'function') {
  loadStartedAt = window.performance.now();
}

document.head.appendChild(script);

publishPostHogLifecycle(
  'posthog_script_load_requested',
  script.src
);
```

Обработчики `load` и `error` назначаются до добавления элемента в DOM. Сигнал `posthog_script_load_requested` публикуется только после успешного выполнения `appendChild()`.

Мониторинг подписывается на lifecycle-событие до запуска штатного loader:

```javascript
function handlePostHogLifecycleEvent(event) {
  try {
    var detail = event.detail || {};

    if (detail.event_name === 'posthog_script_load_requested') {
      trackPostHogScriptLoadRequested(detail.script_url);
      return;
    }

    if (detail.event_name === 'posthog_script_loaded') {
      trackPostHogScriptLoaded(
        detail.script_url,
        typeof detail.load_time_ms === 'number'
          ? detail.load_time_ms
          : null
      );
      return;
    }

    if (detail.event_name === 'posthog_script_load_error') {
      trackPostHogScriptLoadError(detail.script_url);
    }
  } catch (monitoringError) {
    // Ошибка мониторинга не должна влиять на загрузку PostHog.
  }
}

window.addEventListener(
  'posthog:lifecycle',
  handlePostHogLifecycleEvent
);
```

Loader не вызывает `sendToEventCollector()` и не импортирует код Event Collector. Ошибка загрузки фиксируется только обработчиком `error` элемента `<script>` и передается мониторингу через `posthog:lifecycle`. Глобальный capture-обработчик для ошибок загрузки ресурсов не используется. Обработчики `window.error` и `window.unhandledrejection` используются только для ошибок выполнения JavaScript и Promise.

### 11. Возможные проблемы с Event Collector

Endpoint Event Collector рекомендуется размещать на собственном домене или first-party поддомене без слов `posthog`, `analytics` и `tracking` в адресе. В противном случае он может быть заблокирован решениями типа AdBlock.

### 12. Безопасность данных

Длину данных об ошибке нужно ограничить:

```javascript
message.slice(0, 1000);
stack.slice(0, 5000);
```

### 13. Структура реализации

Реализация должна содержать функции:

```javascript
createPageLoadId()
publishPostHogLifecycle()
handlePostHogLifecycleEvent()
trackEventCollectorInitialized()
trackPostHogInitialized()
trackPostHogScriptLoadRequested()
trackPostHogScriptLoaded()
trackPostHogScriptLoadError()
trackPostHogJavaScriptError()
trackPostHogPromiseError()
sendPostHogDiagnosticEvent()
```

`sendPostHogDiagnosticEvent()` должна:

- добавлять `page_load_id` и общие поля;
- выполнять дедупликацию только для `posthog_js_error`;
- безопасно вызывать `sendToEventCollector()`;
- перехватывать ошибки отправки.

`publishPostHogLifecycle()` относится к штатному loader и не должна обращаться к Event Collector. `handlePostHogLifecycleEvent()` относится к мониторингу и обрабатывает только известные значения `event_name` из контракта `posthog:lifecycle`.

### 14. Критерии приемки

1. При нормальной работе приходят `event_collector_initialized`, `posthog_script_load_requested`, `posthog_script_loaded` и `posthog_initialized` в указанном порядке и с одинаковым `page_load_id`.
2. Первый `$pageview`, ручные и autocapture-события PostHog содержат тот же `page_load_id`.
3. После полной перезагрузки страницы создается новый `page_load_id`, который заменяет старое значение super property.
4. В браузере без `crypto.randomUUID()` `page_load_id` создается через fallback.
5. При ошибке инициализации приходят `posthog_script_load_requested` и `posthog_script_loaded`, но не приходит `posthog_initialized` с тем же `page_load_id`.
6. При неверном URL, недоступности CDN, ограничении CSP или блокировке файла приходят `posthog_script_load_requested` и один `posthog_script_load_error`; `posthog_script_loaded` и `posthog_initialized` для этой попытки отсутствуют.
7. Повторная попытка загрузки формирует новую последовательность lifecycle-событий и не подавляется дедупликацией.
8. Повторные события инициализации отправляются и сохраняют тот же `page_load_id`.
9. `page_load_id` не создается как person property.
10. Ошибка или отсутствие lifecycle-мониторинга не препятствует загрузке и инициализации PostHog.
11. Ошибка `window.error`, связанная с PostHog, отправляется как `posthog_js_error` с `error_source: 'window_error'`.
12. Ошибка `window.unhandledrejection`, связанная с PostHog, отправляется как `posthog_js_error` с `error_source: 'unhandled_rejection'`.
13. Повтор одинаковой ошибки в течение 10 секунд не создает дубликаты `posthog_js_error`.
14. `posthog_initialized` отправляется через Event Collector без вызова `posthog.capture()`.
15. Ошибка внутри `sendToEventCollector()` не нарушает работу страницы, не запускает цикл ошибок и не препятствует загрузке PostHog.
16. `dataLayer` содержит `page_load_id`, значение которого совпадает с `window.PAGE_LOAD_ID`.

### 15. Итоговый список событий

| Событие | Назначение |
| --- | --- |
| `event_collector_initialized` | Event Collector успешно запущен |
| `posthog_script_load_requested` | Штатный loader начал попытку загрузки файла PostHog |
| `posthog_script_loaded` | Файл PostHog успешно загружен |
| `posthog_script_load_error` | Файл PostHog не удалось загрузить |
| `posthog_initialized` | PostHog успешно инициализирован |
| `posthog_js_error` | Ошибка выполнения PostHog или необработанная ошибка Promise |

---

## Пример реализации для понимания

> Этот раздел не является частью обязательных требований.

Ниже показан один из возможных вариантов обработки обычных JavaScript-ошибок и необработанных ошибок Promise.

```javascript
window.addEventListener('error', function (event) {
  var stack = event.error && event.error.stack;
  var filename = event.filename || null;

  if (!isPostHogRelated(filename, stack)) {
    return;
  }

  sendPostHogDiagnosticEvent({
    event_name: 'posthog_js_error',
    error_source: 'window_error',
    message: event.message || '',
    error_name: event.error && event.error.name || null,
    stack: stack || null,
    filename: filename,
    lineno: event.lineno || null,
    colno: event.colno || null
  });
});

window.addEventListener('unhandledrejection', function (event) {
  var reason = event.reason;
  var stack = reason && reason.stack || null;

  if (!isPostHogRelated(null, stack)) {
    return;
  }

  sendPostHogDiagnosticEvent({
    event_name: 'posthog_js_error',
    error_source: 'unhandled_rejection',
    message: reason && reason.message || String(reason),
    error_name: reason && reason.name || null,
    stack: stack,
    filename: null,
    lineno: null,
    colno: null
  });
});
```

В этом примере:

- `isPostHogRelated()` проверяет связь ошибки с PostHog по правилам из раздела 4;
- `sendPostHogDiagnosticEvent()` добавляет `page_load_id` и общие поля, ограничивает длину данных, выполняет дедупликацию и безопасно вызывает Event Collector.
