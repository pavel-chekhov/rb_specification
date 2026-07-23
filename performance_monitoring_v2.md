# Техническое задание: мониторинг загрузки страницы через `dataLayer`

## 1. Цель

Нужно измерять загрузку HTML-документа на стороне браузера и публиковать результаты в `dataLayer`, не полагаясь только на событие `window.load`.

Мониторинг должен фиксировать:

1. Начало работы измерителя.
2. Готовность DOM как внутреннее состояние и метрику, без отдельного события в `dataLayer`.
3. Полное завершение загрузки, если оно произошло.
4. Последнее доступное состояние, если пользователь скрыл или покинул страницу до завершения загрузки либо загрузка длится более 30 секунд.
5. Восстановление документа из back/forward cache (bfcache) как отдельное событие, не являющееся новой загрузкой.

Все события одной загрузки связываются через единый `page_load_id`.

Выбрана сбалансированная модель публикации:

```text
Обычная загрузка:       started → complete
Незавершённая загрузка: started → snapshot
Скрытие и возврат:      started → snapshot → complete
```

Ожидание только одного итогового события не используется: `load` может не наступить, а первый переход документа в `hidden` не доказывает, что пользователь окончательно ушёл со страницы.

## 2. Область действия и ограничения

- Измеряются только полные загрузки HTML-документов.
- Переходы внутри SPA через History API, `popstate` или `hashchange` не считаются новой загрузкой документа и находятся вне этого ТЗ.
- Источником сетевых метрик служит `PerformanceNavigationTiming`. Для старых браузеров используется fallback на `performance.timing`.
- LCP, CLS и INP этим решением не измеряются. Это Navigation Timing, а не набор Core Web Vitals.
- `dataLayer.push()` синхронно помещает данные в очередь страницы, но сам по себе не доставляет их во внешнюю систему. Триггер и тег GTM должны прочитать событие и при необходимости передать его внешнему получателю, например Event Collector, PostHog или другому хранилищу. Конкретный получатель, consent и сетевой транспорт настраиваются отдельно и не являются зависимостью этого ТЗ.
- Если страница закрыта до запуска GTM-контейнера, клиентский код не сможет зарегистрировать событие.

## 3. Запуск в Google Tag Manager

Измеритель реализуется одним тегом **Custom HTML** и запускается по триггеру **Initialization — All Pages**. Вспомогательные операции вынесены в две переменные типа **Custom JavaScript**:

- `{{page_load_id}}` один раз генерирует ID текущего HTML-документа, сохраняет его в `window.PAGE_LOAD_ID` и возвращает сохранённое значение при повторных обращениях;
- `{{pwa_context}}` определяет текущий режим запуска и возвращает объект с полями `isPwa` и `displayMode`.

Обе переменные вызываются из основного Custom HTML-тега. Отдельные триггеры для них не создаются.

Использовать триггер **Window Loaded** нельзя: он сработает только после полной загрузки и не позволит увидеть незавершённые загрузки. Согласно [документации GTM](https://support.google.com/tagmanager/answer/7679319), Initialization запускается раньше остальных обычных page-view-триггеров, кроме Consent Initialization.

Повторное выполнение тега на одном документе не должно повторно устанавливать слушатели или создавать новый `page_load_id`.

## 4. События жизненного цикла страницы

Не все полезные события принадлежат `window`: часть отправляется объектом `document`.

| Событие | Target | Использование |
| --- | --- | --- |
| `readystatechange` | `document` | Помогает восстановить пропущенную готовность DOM при позднем запуске измерителя. Обновляет внутреннее состояние, но не создаёт отдельный push. |
| `DOMContentLoaded` | `document` | Основной сигнал готовности DOM. Обновляет внутреннее состояние, но не создаёт отдельный push. |
| `load` | `window` | Сразу фиксирует достижение `load`, затем создаёт стадию `complete` через `setTimeout(..., 0)`, чтобы браузер успел заполнить `loadEventEnd`. |
| `visibilitychange` | `document` | При переходе в `hidden` создаёт `snapshot`, если `complete` ещё не наступил. |
| `pagehide` | `window` | Fallback для `snapshot`, если он ещё не был создан; `persisted` сохраняется в payload. |
| `pageshow` | `window` | При `persisted === true` создаёт отдельное событие `page_bfcache_restore`. |
| `freeze` / `resume` | `document` | Дополнительные Page Lifecycle-сигналы Chromium; не являются обязательными для базовой реализации. |
| `beforeunload` / `unload` | `window` | Не используются: ненадёжны и могут ухудшать работу bfcache. |
| `error` / `unhandledrejection` | `window` | Могут использоваться отдельной диагностикой, но не являются стадиями загрузки. |
| `online` / `offline` | `window` | Могут давать сетевой контекст, но не являются стадиями загрузки. |
| `focus` / `blur` | `window` | Не используются для расчёта загрузочных метрик. |
| `popstate` / `hashchange` | `window` | Относятся к SPA-навигации и находятся вне этого ТЗ. |

Переход документа в `hidden` часто является последним доступным для наблюдения состоянием, особенно на мобильных устройствах. Поэтому `visibilitychange` используется раньше `pagehide`, `beforeunload` и `unload`. Потеря фокуса через `blur` не считается таким сигналом: окно может потерять фокус, хотя страница остаётся видимой. Подробности: [Page Lifecycle API](https://developer.chrome.com/docs/web-platform/page-lifecycle-api), [Beacon API](https://www.w3.org/TR/beacon/), [bfcache](https://web.dev/articles/bfcache).

### 4.1. Состояние в памяти

Измеритель хранит состояние текущего HTML-документа в обычном JavaScript-объекте `monitor`, доступном через `window[GUARD_NAME]`.

Зафиксировать DOM Ready в памяти означает выполнить:

```javascript
monitor.domReadyReached = true;
```

Это действие не вызывает `dataLayer.push()`, не отправляет сетевой запрос и не записывает данные в cookie или `localStorage`. Флаг только запоминает достигнутую точку жизненного цикла. При последующем `complete` или `snapshot` его значение попадает в `milestones.dom_ready_reached`, а доступный timestamp — в `metrics.dom_ready_ms`.

Состояние существует, пока существует текущий документ. Оно исчезает при полной перезагрузке или уничтожении вкладки, но сохраняется при bfcache restore, потому что браузер восстанавливает тот же документ вместе с DOM и JavaScript-состоянием.

### 4.2. Восстановление из bfcache

Back/forward cache может сохранить документ целиком, когда пользователь уходит с него, а затем вернуть тот же экземпляр при переходе Назад или Вперёд. Это не новая загрузка HTML: сетевой запрос обычно отсутствует, новый `page_load_id` не создаётся, а Navigation Timing продолжает описывать первоначальную загрузку.

Восстановление определяется событием `pageshow` с `event.persisted === true`. Оно публикуется как отдельное событие `page_bfcache_restore` без `ttfb_ms`, `dom_ready_ms` и `full_load_ms`, чтобы повторно не учитывать старые значения как новую загрузку.

### 4.3. Граница ответственности доставки

Цепочка обработки выглядит так:

```text
Измеритель → dataLayer.push() → триггер и тег GTM → внешний получатель
```

Измеритель отвечает за создание объекта и его помещение в `dataLayer`. Тег GTM отвечает за чтение события и сетевую отправку в выбранную систему. Сам `dataLayer.push()` не гарантирует доставку данных за пределы страницы.

Для snapshot при переходе в `hidden` тегу-получателю нужен механизм отправки, рассчитанный на завершение жизненного цикла страницы, например `navigator.sendBeacon()` или `fetch()` с `{ keepalive: true }`. Выбор и реализация транспорта находятся вне референсного кода этого ТЗ.

## 5. Стадии измерения

Во всех случаях в `dataLayer` передаётся одно имя GTM-события:

```text
page_load_performance
```

Текущее состояние определяется полем `performance_monitoring.measurement_stage`.

| `measurement_stage` | Условие отправки | Повторение |
| --- | --- | --- |
| `started` | Сразу после установки измерителя | Один раз на документ |
| `snapshot` | До достижения `load` при первом `visibilitychange:hidden`, fallback `pagehide` или по таймауту 30 секунд | Не более одного раза на документ |
| `complete` | После завершения обработчиков `window.load` | Один раз на документ |

Snapshot не означает доказанный отказ от загрузки. Например, пользователь может переключиться на другую вкладку, затем вернуться, после чего загрузка завершится. В этом случае с тем же `page_load_id` последовательно публикуются `snapshot` и `complete`.

`DOMContentLoaded` не создаёт отдельную стадию. Он только обновляет состояние `dom_ready_reached`, которое передаётся в следующем `snapshot` или `complete`.

Поле `stage_source` принимает только следующие значения:

```text
initialization
ready_state_recovery
load
visibility_hidden
pagehide
timeout
```

## 6. Контракт `dataLayer`

Каждый push события `page_load_performance` должен содержать полную структуру. Недостигнутые или недоступные значения явно заполняются `null`. Это исключает наследование значений предыдущей стадии в GTM Data Layer Model. Отдельное событие `page_bfcache_restore` использует собственную структуру из раздела 6.3.

```javascript
window.dataLayer = window.dataLayer || [];

window.dataLayer.push({
  event: 'page_load_performance',
  performance_monitoring: {
    schema_version: 3,
    page_load_id: '550e8400-e29b-41d4-a716-446655440000',
    measurement_stage: 'started',
    stage_source: 'initialization',
    observed_at_ms: 123,
    timing_api: 'navigation_timing_2',
    navigation_type: 'navigate',

    milestones: {
      dom_ready_reached: false,
      load_reached: false
    },

    metrics: {
      redirect_time_ms: null,
      ttfb_ms: null,
      dom_ready_ms: null,
      full_load_ms: null
    },

    context: {
      page: '/path',
      host: 'example.test',
      page_url: 'https://example.test/path',
      entry_url: null,
      utm_source: null,
      utm_medium: null,
      is_pwa: false,
      display_mode: 'browser',
      document_ready_state: 'loading',
      visibility_state: 'visible',
      pagehide_persisted: null,
      was_discarded: false
    }
  }
});
```

### 6.1. Общие поля

| Поле | Тип | Описание |
| --- | --- | --- |
| `schema_version` | `number` | Версия контракта `page_load_performance`, для этой реализации всегда `3`. |
| `page_load_id` | `string` | Уникальный ID HTML-документа. Не меняется при bfcache restore. |
| `measurement_stage` | `string` | Стадия измерения из раздела 5. |
| `stage_source` | `string` | Lifecycle-сигнал, вызвавший push. |
| `observed_at_ms` | `number` | Количество миллисекунд от начала навигации до создания push. |
| `timing_api` | `string` | `navigation_timing_2`, `performance_timing_legacy` или `unavailable`. |
| `navigation_type` | `string \| null` | Обычно `navigate`, `reload` или `back_forward`; в legacy API и при отсутствии данных может быть `null`. |

### 6.2. Контекст

| Поле | Тип | Описание |
| --- | --- | --- |
| `page` | `string` | `window.location.pathname`. |
| `host` | `string` | `window.location.host`: домен и порт, если используется нестандартный порт. |
| `page_url` | `string` | Полный URL текущего документа. |
| `entry_url` | `string \| null` | `document.referrer` или `null`. |
| `utm_source` | `string \| null` | Значение query-параметра `utm_source`. |
| `utm_medium` | `string \| null` | Значение query-параметра `utm_medium`. |
| `is_pwa` | `boolean \| null` | `true` для app-like режима, `false` для подтверждённого режима `browser`, `null`, если режим определить нельзя. |
| `display_mode` | `string` | `ios_standalone`, `window-controls-overlay`, `standalone`, `minimal-ui`, `fullscreen`, `browser` или `unknown`. |
| `document_ready_state` | `string` | Текущее значение `document.readyState`. |
| `visibility_state` | `string \| null` | Текущее значение `document.visibilityState`. |
| `pagehide_persisted` | `boolean \| null` | Значение `event.persisted` для source `pagehide`; иначе `null`. |
| `was_discarded` | `boolean` | Был ли предыдущий экземпляр вкладки выгружен браузером из памяти. |

`is_pwa` описывает текущий режим запуска страницы, а не доказывает, что PWA установлена на устройстве. Универсального браузерного API для проверки факта установки нет. Стандартный media feature `display-mode` сообщает применённый режим текущего окна, который может отличаться от значения, объявленного в manifest. Подробности: [Web Application Manifest](https://www.w3.org/TR/appmanifest/).

Для совместимости со старыми версиями Safari на iOS дополнительно проверяется нестандартное булево свойство `navigator.standalone`, документированное Apple. Значение `true` преобразуется в `display_mode: "ios_standalone"`. Подробности: [Configuring Web Applications](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/ConfiguringWebApplications/ConfiguringWebApplications.html).

Режим `window-controls-overlay` учитывается для поддерживающих его desktop PWA. Подробности: [Window Controls Overlay](https://learn.microsoft.com/en-gb/microsoft-edge/progressive-web-apps/how-to/window-controls-overlay). Режим `fullscreen` считается app-like, но может быть активирован и для страницы без установленной PWA. Поэтому для точной интерпретации вместе с `is_pwa` всегда передаётся исходный `display_mode`.

Порядок определения: сначала `navigator.standalone === true`, затем media queries для известных display modes, после них `navigator.standalone === false` как подтверждение обычного browser-режима. Если ни один сигнал недоступен или не дал результата, передаются `display_mode: "unknown"` и `is_pwa: null`. Основной тег обращается к `{{pwa_context}}` один раз при создании `monitor`, поэтому результат остаётся одинаковым для всех стадий текущего `page_load_id`.

### 6.3. Контракт восстановления из bfcache

Восстановление документа публикуется отдельно от загрузочных метрик:

```javascript
window.dataLayer.push({
  event: 'page_bfcache_restore',
  page_restore_monitoring: {
    schema_version: 1,
    page_load_id: '550e8400-e29b-41d4-a716-446655440000',
    restore_index: 1,
    observed_at_ms: 12345,
    page: '/path',
    page_url: 'https://example.test/path'
  }
});
```

| Поле | Тип | Описание |
| --- | --- | --- |
| `schema_version` | `number` | Версия контракта `page_bfcache_restore`, всегда `1`. |
| `page_load_id` | `string` | ID первоначально загруженного HTML-документа. При восстановлении не меняется. |
| `restore_index` | `number` | Порядковый номер восстановления этого документа, начиная с `1`. |
| `observed_at_ms` | `number` | Количество миллисекунд от начала первоначальной навигации до события восстановления. |
| `page` | `string` | `window.location.pathname` восстановленного документа. |
| `page_url` | `string` | Полный URL восстановленного документа. |

Событие не содержит `measurement_stage`, `milestones` или `metrics` и анализируется отдельно от `page_load_performance`.

## 7. Расчёт метрик

### 7.1. Navigation Timing Level 2

Источник:

```javascript
performance.getEntriesByType('navigation')[0]
```

| Метрика | Формула |
| --- | --- |
| `redirect_time_ms` | `redirectEnd - redirectStart`, только когда `redirectCount > 0` |
| `ttfb_ms` | `responseStart` |
| `dom_ready_ms` | `domContentLoadedEventEnd` |
| `full_load_ms` | `loadEventEnd`, только для стадии `complete` |

В Navigation Timing Level 2 значения измеряются относительно начала навигации, поэтому вычитать `navigationStart` не требуется. Спецификация: [Navigation Timing Level 2](https://www.w3.org/TR/navigation-timing-2/).

### 7.2. Legacy fallback

Источник:

```javascript
performance.timing
```

| Метрика | Формула |
| --- | --- |
| `redirect_time_ms` | `redirectEnd - redirectStart`, если оба timestamp доступны |
| `ttfb_ms` | `responseStart - navigationStart` |
| `dom_ready_ms` | `domContentLoadedEventEnd - navigationStart` |
| `full_load_ms` | `loadEventEnd - navigationStart`, только для стадии `complete` |

Если исходный timestamp равен `0`, результат передаётся как `null`. Значение `redirect_time_ms` также равно `null` при cross-origin редиректе, поскольку браузер скрывает его timing.

`dom_ready_ms` означает окончание обработчиков `DOMContentLoaded`. Он не гарантирует визуальную готовность страницы.

## 8. Референсная реализация для GTM

Код намеренно не использует стрелочные функции, `const`, `let`, optional chaining и `URLSearchParams`, чтобы сохранить совместимость со старыми браузерами и WebView.

### 8.1. Custom JavaScript Variable `page_load_id`

Создать в GTM переменную типа **Custom JavaScript** с именем `page_load_id`. Переменная сохраняет первое сгенерированное значение в `window.PAGE_LOAD_ID`, поэтому повторное обращение на том же документе не создаёт новый ID.

```javascript
function () {
  var cryptoApi;
  var bytes;
  var hex = [];
  var i;

  if (window.PAGE_LOAD_ID) {
    return window.PAGE_LOAD_ID;
  }

  cryptoApi = window.crypto || window.msCrypto;

  if (cryptoApi && typeof cryptoApi.randomUUID === 'function') {
    window.PAGE_LOAD_ID = cryptoApi.randomUUID();
    return window.PAGE_LOAD_ID;
  }

  if (cryptoApi && typeof cryptoApi.getRandomValues === 'function') {
    bytes = new Uint8Array(16);
    cryptoApi.getRandomValues(bytes);
    bytes[6] = (bytes[6] & 15) | 64;
    bytes[8] = (bytes[8] & 63) | 128;

    for (i = 0; i < bytes.length; i += 1) {
      hex.push((bytes[i] + 256).toString(16).slice(1));
    }

    window.PAGE_LOAD_ID = (
      hex.slice(0, 4).join('') + '-' +
      hex.slice(4, 6).join('') + '-' +
      hex.slice(6, 8).join('') + '-' +
      hex.slice(8, 10).join('') + '-' +
      hex.slice(10, 16).join('')
    );

    return window.PAGE_LOAD_ID;
  }

  window.PAGE_LOAD_ID = (
    new Date().getTime().toString(36) + '-' +
    Math.random().toString(36).slice(2)
  );

  return window.PAGE_LOAD_ID;
}
```

### 8.2. Custom JavaScript Variable `pwa_context`

Создать в GTM переменную типа **Custom JavaScript** с именем `pwa_context`. Она возвращает контекст текущего режима запуска; основной тег сохраняет этот объект в `monitor` при инициализации.

```javascript
function () {
  var navigatorApi = window.navigator;
  var displayModes = [
    'window-controls-overlay',
    'standalone',
    'minimal-ui',
    'fullscreen',
    'browser'
  ];
  var i;

  if (navigatorApi && navigatorApi.standalone === true) {
    return {
      isPwa: true,
      displayMode: 'ios_standalone'
    };
  }

  if (typeof window.matchMedia === 'function') {
    for (i = 0; i < displayModes.length; i += 1) {
      try {
        if (
          window.matchMedia(
            '(display-mode: ' + displayModes[i] + ')'
          ).matches
        ) {
          return {
            isPwa: displayModes[i] !== 'browser',
            displayMode: displayModes[i]
          };
        }
      } catch (error) {
        // Продолжаем проверку других сигналов и fallback.
      }
    }
  }

  if (navigatorApi && navigatorApi.standalone === false) {
    return {
      isPwa: false,
      displayMode: 'browser'
    };
  }

  return {
    isPwa: null,
    displayMode: 'unknown'
  };
}
```

### 8.3. Custom HTML-тег измерителя

Основной тег использует обе Custom JavaScript Variables:

```html
<script>
(function (window, document) {
  'use strict';

  var GUARD_NAME = '__PAGE_LOAD_PERFORMANCE_MONITOR_V2__';
  var MAX_LOAD_WAIT_MS = 30000;

  if (window[GUARD_NAME]) {
    return;
  }

  var monitor = {
    schemaVersion: 3,
    pageLoadId: {{page_load_id}},
    pwaContext: {{pwa_context}},
    domReadyReached: false,
    loadReached: false,
    completeSent: false,
    snapshotSent: false,
    restoreCount: 0,
    timeoutId: null
  };

  window[GUARD_NAME] = monitor;
  window.dataLayer = window.dataLayer || [];

  function roundMilliseconds(value) {
    if (typeof value !== 'number' || !isFinite(value) || value < 0) {
      return null;
    }

    return Math.round(value);
  }

  function positiveTimestamp(value) {
    return typeof value === 'number' && isFinite(value) && value > 0;
  }

  function readTiming() {
    var performanceApi = window.performance;
    var entries;

    if (
      performanceApi &&
      typeof performanceApi.getEntriesByType === 'function'
    ) {
      entries = performanceApi.getEntriesByType('navigation');

      if (entries && entries.length > 0) {
        return {
          api: 'navigation_timing_2',
          value: entries[0]
        };
      }
    }

    if (performanceApi && performanceApi.timing) {
      return {
        api: 'performance_timing_legacy',
        value: performanceApi.timing
      };
    }

    return {
      api: 'unavailable',
      value: null
    };
  }

  function observedAtMilliseconds() {
    var timing;
    var now;

    if (
      window.performance &&
      typeof window.performance.now === 'function'
    ) {
      return Math.max(0, Math.round(window.performance.now()));
    }

    timing = readTiming();

    if (
      timing.api === 'performance_timing_legacy' &&
      positiveTimestamp(timing.value.navigationStart)
    ) {
      now = new Date().getTime() - timing.value.navigationStart;
      return Math.max(0, Math.round(now));
    }

    return 0;
  }

  function getQueryParameter(name) {
    var escapedName = name.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    var match = new RegExp('[?&]' + escapedName + '=([^&#]*)').exec(
      window.location.search
    );

    if (!match) {
      return null;
    }

    try {
      return decodeURIComponent(match[1].replace(/\+/g, ' '));
    } catch (error) {
      return match[1];
    }
  }

  function getMetrics(timing, stage) {
    var value = timing.value;
    var metrics = {
      redirect_time_ms: null,
      ttfb_ms: null,
      dom_ready_ms: null,
      full_load_ms: null
    };
    var navigationStart;

    if (!value) {
      return metrics;
    }

    if (timing.api === 'navigation_timing_2') {
      if (
        value.redirectCount > 0 &&
        typeof value.redirectStart === 'number' &&
        positiveTimestamp(value.redirectEnd) &&
        value.redirectEnd >= value.redirectStart
      ) {
        metrics.redirect_time_ms = roundMilliseconds(
          value.redirectEnd - value.redirectStart
        );
      }

      if (positiveTimestamp(value.responseStart)) {
        metrics.ttfb_ms = roundMilliseconds(value.responseStart);
      }

      if (positiveTimestamp(value.domContentLoadedEventEnd)) {
        metrics.dom_ready_ms = roundMilliseconds(
          value.domContentLoadedEventEnd
        );
      }

      if (stage === 'complete' && positiveTimestamp(value.loadEventEnd)) {
        metrics.full_load_ms = roundMilliseconds(value.loadEventEnd);
      }

      return metrics;
    }

    navigationStart = value.navigationStart;

    if (!positiveTimestamp(navigationStart)) {
      return metrics;
    }

    if (
      positiveTimestamp(value.redirectStart) &&
      positiveTimestamp(value.redirectEnd) &&
      value.redirectEnd >= value.redirectStart
    ) {
      metrics.redirect_time_ms = roundMilliseconds(
        value.redirectEnd - value.redirectStart
      );
    }

    if (positiveTimestamp(value.responseStart)) {
      metrics.ttfb_ms = roundMilliseconds(
        value.responseStart - navigationStart
      );
    }

    if (positiveTimestamp(value.domContentLoadedEventEnd)) {
      metrics.dom_ready_ms = roundMilliseconds(
        value.domContentLoadedEventEnd - navigationStart
      );
    }

    if (stage === 'complete' && positiveTimestamp(value.loadEventEnd)) {
      metrics.full_load_ms = roundMilliseconds(
        value.loadEventEnd - navigationStart
      );
    }

    return metrics;
  }

  function getNavigationType(timing) {
    if (
      timing.api === 'navigation_timing_2' &&
      timing.value &&
      typeof timing.value.type === 'string'
    ) {
      return timing.value.type;
    }

    return null;
  }

  function pushStage(stage, source, pagehidePersisted) {
    var timing = readTiming();

    window.dataLayer.push({
      event: 'page_load_performance',
      performance_monitoring: {
        schema_version: monitor.schemaVersion,
        page_load_id: monitor.pageLoadId,
        measurement_stage: stage,
        stage_source: source,
        observed_at_ms: observedAtMilliseconds(),
        timing_api: timing.api,
        navigation_type: getNavigationType(timing),

        milestones: {
          dom_ready_reached: monitor.domReadyReached,
          load_reached: monitor.loadReached
        },

        metrics: getMetrics(timing, stage),

        context: {
          page: window.location.pathname,
          host: window.location.host,
          page_url: window.location.href,
          entry_url: document.referrer || null,
          utm_source: getQueryParameter('utm_source'),
          utm_medium: getQueryParameter('utm_medium'),
          is_pwa: monitor.pwaContext.isPwa,
          display_mode: monitor.pwaContext.displayMode,
          document_ready_state: document.readyState,
          visibility_state: document.visibilityState || null,
          pagehide_persisted:
            typeof pagehidePersisted === 'boolean'
              ? pagehidePersisted
              : null,
          was_discarded: document.wasDiscarded === true
        }
      }
    });
  }

  function pushBfcacheRestore() {
    monitor.restoreCount += 1;

    window.dataLayer.push({
      event: 'page_bfcache_restore',
      page_restore_monitoring: {
        schema_version: 1,
        page_load_id: monitor.pageLoadId,
        restore_index: monitor.restoreCount,
        observed_at_ms: observedAtMilliseconds(),
        page: window.location.pathname,
        page_url: window.location.href
      }
    });
  }

  function markDomReady() {
    monitor.domReadyReached = true;
  }

  function markLoadReached() {
    markDomReady();
    monitor.loadReached = true;
  }

  function emitSnapshot(source, pagehidePersisted) {
    if (
      monitor.loadReached ||
      monitor.completeSent ||
      monitor.snapshotSent
    ) {
      return;
    }

    monitor.snapshotSent = true;
    pushStage('snapshot', source, pagehidePersisted);
  }

  function emitComplete(source) {
    if (monitor.completeSent) {
      return;
    }

    markLoadReached();
    monitor.completeSent = true;

    if (monitor.timeoutId !== null) {
      window.clearTimeout(monitor.timeoutId);
      monitor.timeoutId = null;
    }

    pushStage('complete', source, null);
  }

  function timingShowsDomReady() {
    var timing = readTiming();

    if (!timing.value) {
      return document.readyState !== 'loading';
    }

    return positiveTimestamp(timing.value.domContentLoadedEventEnd);
  }

  function timingShowsLoadComplete() {
    var timing = readTiming();

    if (!timing.value) {
      return document.readyState === 'complete';
    }

    return positiveTimestamp(timing.value.loadEventEnd);
  }

  document.addEventListener('DOMContentLoaded', function () {
    markDomReady();
  });

  document.addEventListener('readystatechange', function () {
    if (timingShowsDomReady()) {
      markDomReady();
    }
  });

  window.addEventListener('load', function () {
    markLoadReached();

    window.setTimeout(function () {
      emitComplete('load');
    }, 0);
  });

  document.addEventListener('visibilitychange', function () {
    if (document.visibilityState === 'hidden') {
      emitSnapshot('visibility_hidden', null);
    }
  });

  window.addEventListener('pagehide', function (event) {
    emitSnapshot('pagehide', event.persisted === true);
  });

  window.addEventListener('pageshow', function (event) {
    if (event.persisted === true) {
      pushBfcacheRestore();
    }
  });

  if (timingShowsDomReady()) {
    markDomReady();
  }

  if (document.readyState === 'complete' || timingShowsLoadComplete()) {
    markLoadReached();
  }

  pushStage('started', 'initialization', null);

  if (monitor.loadReached) {
    window.setTimeout(function () {
      emitComplete('ready_state_recovery');
    }, 0);
  }

  monitor.timeoutId = window.setTimeout(function () {
    emitSnapshot('timeout', null);
  }, Math.max(0, MAX_LOAD_WAIT_MS - observedAtMilliseconds()));
})(window, document);
</script>
```

## 9. Интерпретация событий

События одной загрузки нужно объединять по `page_load_id`. Количество загрузок считается по уникальным `page_load_id`, а не по количеству строк `page_load_performance`.

Итоговое состояние выбирается по следующему приоритету:

1. Если для `page_load_id` существует `complete`, он является итогом независимо от более раннего snapshot. Для загрузочных метрик используются значения из `complete`.
2. Если `complete` отсутствует, но существует `snapshot`, он является последним наблюдавшимся состоянием. Это не доказанный отказ: вкладка могла быть скрыта, а доставка более позднего события могла не состояться.
3. Если существует только `started`, исход считается неизвестным. Это может означать раннее уничтожение страницы, сбой измерителя или потерю доставки, но не должно автоматически классифицироваться как ошибка загрузки.

События `page_bfcache_restore` анализируются отдельно. Они связываются с первоначальной загрузкой через `page_load_id`, но не увеличивают количество загрузок и не участвуют в расчёте перцентилей `ttfb_ms`, `dom_ready_ms` или `full_load_ms`.

## 10. Перцентили и отчёты в GA4 BigQuery

### 10.1. Интерпретация перцентилей

Перцентиль — это порог времени, ниже которого находится заданная доля наблюдений. Он измеряется в тех же единицах, что и исходная метрика, то есть для этого ТЗ — в миллисекундах, а не в процентах.

| Перцентиль | Интерпретация |
| --- | --- |
| P50 | Медиана: 50% загрузок завершились не медленнее этого значения. |
| P75 | 75% загрузок завершились не медленнее этого значения, а 25% были медленнее. |
| P95 | 95% загрузок завершились не медленнее этого значения; оставшиеся 5% образуют наиболее медленный хвост. |

Например, `P75 full_load_ms = 3200` означает, что 75% полных загрузок завершились не более чем за 3,2 секунды. Перцентили предпочтительнее среднего значения для мониторинга производительности, потому что несколько аномально медленных загрузок могут заметно сместить среднее.

### 10.2. Регистрация параметров в GA4

GA4 Event-тег передаёт значения как event parameters. Чтобы использовать числовые значения в отчётах и Explorations GA4, зарегистрировать `redirect_time_ms`, `ttfb_ms`, `dom_ready_ms` и `full_load_ms` как **custom metrics** с единицей измерения **Milliseconds**. Категориальные параметры, используемые в фильтрах и разбивках, например `measurement_stage`, `stage_source`, `timing_api`, `navigation_type`, `is_pwa` и `display_mode`, регистрируются как **event-scoped custom dimensions**.

`page_load_id` нужно передавать в GA4, но не регистрировать как custom dimension: уникальные ID создают высокую кардинальность и могут привести к строке `(other)` в интерфейсе GA4. В BigQuery этот параметр остаётся доступен внутри `event_params` и используется для объединения стадий одной загрузки.

Подробности: [custom metrics](https://support.google.com/analytics/answer/14239619), [event-scoped custom dimensions](https://support.google.com/analytics/answer/14239696), [GA4 BigQuery Export schema](https://support.google.com/analytics/answer/7029846).

### 10.3. Набор отчётов

| График | Метрика | Разбивка | Условия |
| --- | --- | --- | --- |
| TTFB по регионам | Медиана `ttfb_ms` | `geo.city` | Все дедуплицированные загрузки с доступным TTFB. |
| Full load по устройствам | P50/P75/P95 `full_load_ms` | `device.category` + `device.operating_system` | Только итоговая стадия `complete`. |
| Редиректы | Медиана `redirect_time_ms` | `geo.country` | Только загрузки с доступным временем редиректа. |
| Тренд по времени | P75 `full_load_ms` | По дням | Только итоговая стадия `complete`. |
| PWA vs Web | P75 `full_load_ms` | `is_pwa` | Только `complete`; неизвестный режим исключается. |

Поля `geo.city`, `geo.country`, `device.category` и `device.operating_system` входят в стандартную схему экспорта GA4 и не извлекаются из `event_params`.

### 10.4. Подготовка данных

Примеры предполагают, что GA4 Event-тег преобразует листовые значения объекта `performance_monitoring` в плоские event parameters с именами `page_load_id`, `measurement_stage`, `redirect_time_ms`, `ttfb_ms`, `dom_ready_ms`, `full_load_ms` и `is_pwa`.

Заменить `` `project.dataset.events_*` `` на wildcard-таблицу своего GA4-датасета. Скрипт создаёт временную таблицу `page_loads`, доступную следующим запросам в рамках одной BigQuery-сессии. По умолчанию обрабатываются последние 30 дней, включая текущий день.

```sql
DECLARE date_from DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 29 DAY);
DECLARE date_to DATE DEFAULT CURRENT_DATE();

CREATE TEMP TABLE page_loads AS
WITH extracted AS (
  SELECT
    event_timestamp,
    PARSE_DATE('%Y%m%d', event_date) AS event_date,
    geo.city AS geo_city,
    geo.country AS geo_country,
    device.category AS device_type,
    device.operating_system AS os,

    (
      SELECT value.string_value
      FROM UNNEST(event_params)
      WHERE key = 'page_load_id'
      LIMIT 1
    ) AS page_load_id,

    (
      SELECT value.string_value
      FROM UNNEST(event_params)
      WHERE key = 'measurement_stage'
      LIMIT 1
    ) AS measurement_stage,

    (
      SELECT COALESCE(
        CAST(value.int_value AS FLOAT64),
        value.double_value,
        CAST(value.float_value AS FLOAT64)
      )
      FROM UNNEST(event_params)
      WHERE key = 'redirect_time_ms'
      LIMIT 1
    ) AS redirect_time_ms,

    (
      SELECT COALESCE(
        CAST(value.int_value AS FLOAT64),
        value.double_value,
        CAST(value.float_value AS FLOAT64)
      )
      FROM UNNEST(event_params)
      WHERE key = 'ttfb_ms'
      LIMIT 1
    ) AS ttfb_ms,

    (
      SELECT COALESCE(
        CAST(value.int_value AS FLOAT64),
        value.double_value,
        CAST(value.float_value AS FLOAT64)
      )
      FROM UNNEST(event_params)
      WHERE key = 'dom_ready_ms'
      LIMIT 1
    ) AS dom_ready_ms,

    (
      SELECT COALESCE(
        CAST(value.int_value AS FLOAT64),
        value.double_value,
        CAST(value.float_value AS FLOAT64)
      )
      FROM UNNEST(event_params)
      WHERE key = 'full_load_ms'
      LIMIT 1
    ) AS full_load_ms,

    (
      SELECT COALESCE(
        value.string_value,
        CAST(value.int_value AS STRING),
        CAST(value.double_value AS STRING),
        CAST(value.float_value AS STRING)
      )
      FROM UNNEST(event_params)
      WHERE key = 'is_pwa'
      LIMIT 1
    ) AS is_pwa_raw
  FROM `project.dataset.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', date_from)
      AND FORMAT_DATE('%Y%m%d', date_to)
    AND event_name = 'page_load_performance'
),
normalized AS (
  SELECT
    * EXCEPT (is_pwa_raw),
    CASE
      WHEN LOWER(is_pwa_raw) IN ('true', '1', '1.0') THEN TRUE
      WHEN LOWER(is_pwa_raw) IN ('false', '0', '0.0') THEN FALSE
      ELSE NULL
    END AS is_pwa
  FROM extracted
  WHERE page_load_id IS NOT NULL
),
ranked AS (
  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY page_load_id
      ORDER BY
        CASE measurement_stage
          WHEN 'complete' THEN 3
          WHEN 'snapshot' THEN 2
          WHEN 'started' THEN 1
          ELSE 0
        END DESC,
        event_timestamp DESC
    ) AS stage_rank
  FROM normalized
)
SELECT * EXCEPT (stage_rank)
FROM ranked
WHERE stage_rank = 1;
```

При последовательности `started → snapshot → complete` в `page_loads` останется только `complete`. Если `complete` отсутствует, будет выбрана стадия `snapshot`, а при её отсутствии — `started`. Поэтому TTFB и время редиректа не получают дополнительный вес из-за нескольких событий одного `page_load_id`.

### 10.5. Медиана TTFB по городам

```sql
SELECT
  COALESCE(NULLIF(geo_city, ''), '(unknown)') AS geo_city,
  COUNT(*) AS page_loads,
  APPROX_QUANTILES(ttfb_ms, 100 IGNORE NULLS)[OFFSET(50)]
    AS median_ttfb_ms
FROM page_loads
WHERE ttfb_ms IS NOT NULL
GROUP BY 1
ORDER BY median_ttfb_ms DESC;
```

### 10.6. P50/P75/P95 Full Load по устройствам

`full_load_ms` существует только у стадии `complete`, поэтому незавершённые загрузки не участвуют в этом отчёте.

```sql
SELECT
  COALESCE(NULLIF(device_type, ''), '(unknown)') AS device_type,
  COALESCE(NULLIF(os, ''), '(unknown)') AS os,
  COUNT(*) AS completed_loads,
  APPROX_QUANTILES(full_load_ms, 100 IGNORE NULLS)[OFFSET(50)]
    AS p50_full_load_ms,
  APPROX_QUANTILES(full_load_ms, 100 IGNORE NULLS)[OFFSET(75)]
    AS p75_full_load_ms,
  APPROX_QUANTILES(full_load_ms, 100 IGNORE NULLS)[OFFSET(95)]
    AS p95_full_load_ms
FROM page_loads
WHERE
  measurement_stage = 'complete'
  AND full_load_ms IS NOT NULL
GROUP BY 1, 2
ORDER BY p75_full_load_ms DESC;
```

### 10.7. Медиана времени редиректа по странам

В расчёт входят только загрузки с непустым `redirect_time_ms`. Загрузки без редиректа и загрузки, для которых браузер скрыл cross-origin redirect timing, не интерпретируются как нулевые значения.

```sql
SELECT
  COALESCE(NULLIF(geo_country, ''), '(unknown)') AS geo_country,
  COUNT(*) AS redirected_loads,
  APPROX_QUANTILES(redirect_time_ms, 100 IGNORE NULLS)[OFFSET(50)]
    AS median_redirect_time_ms
FROM page_loads
WHERE redirect_time_ms IS NOT NULL
GROUP BY 1
ORDER BY median_redirect_time_ms DESC;
```

### 10.8. Дневной тренд P75 Full Load

```sql
SELECT
  event_date,
  COUNT(*) AS completed_loads,
  APPROX_QUANTILES(full_load_ms, 100 IGNORE NULLS)[OFFSET(75)]
    AS p75_full_load_ms
FROM page_loads
WHERE
  measurement_stage = 'complete'
  AND full_load_ms IS NOT NULL
GROUP BY event_date
ORDER BY event_date;
```

### 10.9. P75 Full Load для PWA и Web

`is_pwa: null` означает, что режим запуска определить не удалось. Такие строки исключаются, чтобы они не попадали в группу Web.

```sql
SELECT
  IF(is_pwa, 'PWA', 'Web') AS launch_mode,
  COUNT(*) AS completed_loads,
  APPROX_QUANTILES(full_load_ms, 100 IGNORE NULLS)[OFFSET(75)]
    AS p75_full_load_ms
FROM page_loads
WHERE
  measurement_stage = 'complete'
  AND full_load_ms IS NOT NULL
  AND is_pwa IS NOT NULL
GROUP BY launch_mode
ORDER BY launch_mode;
```

### 10.10. Точный расчёт

`APPROX_QUANTILES` возвращает приближённые границы и рассчитан на большие объёмы данных. Для точного перцентиля можно использовать `PERCENTILE_CONT`, который выполняется как оконная функция и обычно требует больше ресурсов. Подробности: [approximate aggregate functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/approximate_aggregate_functions), [PERCENTILE_CONT](https://cloud.google.com/bigquery/docs/reference/standard-sql/navigation_functions#percentile_cont).

Пример точного P75 Full Load по устройствам:

```sql
SELECT DISTINCT
  COALESCE(NULLIF(device_type, ''), '(unknown)') AS device_type,
  COALESCE(NULLIF(os, ''), '(unknown)') AS os,
  PERCENTILE_CONT(full_load_ms, 0.75 IGNORE NULLS) OVER (
    PARTITION BY device_type, os
  ) AS exact_p75_full_load_ms
FROM page_loads
WHERE
  measurement_stage = 'complete'
  AND full_load_ms IS NOT NULL
ORDER BY exact_p75_full_load_ms DESC;
```

## 11. Сэмплирование

Код измерителя не выполняет сэмплирование: все предусмотренные события всегда публикуются в `dataLayer`.

Если тег GTM, отправляющий данные внешнему получателю, использует сэмплирование, решение принимается один раз для всего `page_load_id`, включая связанные события `page_bfcache_restore`. Нельзя независимо сэмплировать отдельные события, иначе последовательность `started → complete` или `started → snapshot → complete` станет неполной.

## 12. Критерии приёмки

1. Обычная загрузка создаёт только `started → complete` с одним `page_load_id`; отдельное событие `dom_ready` отсутствует.
2. У `complete` поля `milestones.dom_ready_reached` и `milestones.load_reached` равны `true`, а доступные `dom_ready_ms` и `full_load_ms` заполнены.
3. Уход до DOM Ready создаёт `started → snapshot` с `dom_ready_reached: false`.
4. Уход после DOM Ready, но до `load`, создаёт `started → snapshot` с `dom_ready_reached: true`.
5. Если страница остаётся видимой и не заканчивает загрузку, через 30 секунд создаётся snapshot со `stage_source: "timeout"`.
6. Последовательность hidden → visible → load создаёт snapshot, а затем complete с тем же `page_load_id`; итогом считается complete.
7. При позднем запуске GTM достигнутые milestone-флаги восстанавливаются до `started` через Navigation Timing и `document.readyState`, но синтетический `dom_ready` не публикуется.
8. После достижения `load` snapshot не создаётся, даже если отложенный через `setTimeout(..., 0)` push `complete` ещё не выполнен.
9. При `pageshow` с `persisted === true` публикуется отдельное событие `page_bfcache_restore` без загрузочных метрик. Новый `started` и новый `page_load_id` не создаются.
10. Каждое повторное восстановление того же документа увеличивает `restore_index` на единицу.
11. Потеря фокуса через `focus` или `blur` не создаёт snapshot.
12. `{{page_load_id}}` сохраняет созданный ID в `window.PAGE_LOAD_ID`; повторные обращения на том же документе возвращают то же значение.
13. Повторный запуск Custom HTML не создаёт новый `page_load_id`, новые listeners или повторные события.
14. Каждый push `page_load_performance` содержит полную схему версии `3`; недоступные значения равны `null`.
15. `{{pwa_context}}` возвращает объект с полями `isPwa` и `displayMode`, который основной тег сохраняет при создании `monitor`.
16. PWA-контекст не меняется между `started`, `snapshot` и `complete` одного `page_load_id`.
17. Режимы `ios_standalone`, `window-controls-overlay`, `standalone`, `minimal-ui` и `fullscreen` дают `is_pwa: true`; `browser` даёт `false`; `unknown` даёт `null`.
18. `utm_source` и `utm_medium` читаются из одноимённых query-параметров; отсутствующее значение передаётся как `null`.
19. При отсутствии Navigation Timing Level 2 используется legacy API.
20. При отсутствии обоих timing API события всё равно публикуются с `timing_api: "unavailable"` и `null` во всех метриках.
21. `full_load_ms` заполнен только у стадии `complete`.
22. `redirect_time_ms` не принимает ложное нулевое значение при недоступном cross-origin timing.
23. Подготовительный BigQuery-скрипт оставляет не более одной строки на `page_load_id` с приоритетом `complete → snapshot → started`.
24. Отчёты по `full_load_ms` используют только стадию `complete`, а TTFB и редиректы считаются по дедуплицированным загрузкам.
25. P50, P75 и P95 возвращаются в миллисекундах; `NULL` не заменяется нулём.
26. PWA vs Web исключает строки с `is_pwa: null`.
27. Все SQL-примеры используют стандартные поля GA4 BigQuery и плоские параметры из `event_params`.

## 13. Тестовые сценарии

| Сценарий | Ожидаемый результат |
| --- | --- |
| Быстрая обычная загрузка | Создаются только `started`, затем `complete`; snapshot и отдельный `dom_ready` отсутствуют. |
| Медленное изображение или iframe | DOM Ready фиксируется в памяти без push; более поздний `complete` содержит `dom_ready_reached: true` и `dom_ready_ms`. |
| Ресурс не завершает загрузку 30 секунд | Создаётся timeout snapshot; при более позднем load дополнительно создаётся complete. |
| Переход на другую страницу до DOM Ready | Создаётся snapshot с `visibility_hidden` или fallback `pagehide` и `dom_ready_reached: false`. |
| Скрытие после DOM Ready до load | Создаётся snapshot с `dom_ready_reached: true`; отдельного `dom_ready` нет. |
| Переключение вкладки во время загрузки | Создаётся snapshot; возврат и load разрешают последующий complete. |
| `blur` без перехода документа в hidden | Snapshot не создаётся. |
| GTM запущен после DOMContentLoaded | `started` сразу содержит `dom_ready_reached: true`; синтетический `dom_ready` не создаётся. |
| GTM запущен после load | В очереди появляются только `started`, затем восстановленный `complete`; оба milestone-флага истинны. |
| Hidden сразу после достижения load | Snapshot отсутствует; после заполнения `loadEventEnd` создаётся complete. |
| Назад/вперёд без bfcache | `pageshow` с `persisted: false` не создаёт restore-событие. |
| Назад/вперёд через bfcache | Создаётся `page_bfcache_restore` с прежним `page_load_id`, `restore_index: 1` и без `metrics`. |
| Повторное восстановление через bfcache | Создаётся следующее `page_bfcache_restore` с тем же ID и увеличенным `restore_index`. |
| Повторное обращение к `{{page_load_id}}` | Возвращается существующее значение `window.PAGE_LOAD_ID`; новый ID не генерируется. |
| `navigator.standalone === true` на iOS | `display_mode` равен `ios_standalone`, `is_pwa` равен `true`. |
| App-like display mode | Для `window-controls-overlay`, `standalone`, `minimal-ui` и `fullscreen` сохраняется совпадающий `display_mode`, `is_pwa` равен `true`. |
| Обычная вкладка браузера | `display_mode` равен `browser`, `is_pwa` равен `false`. |
| API определения display mode недоступны | `display_mode` равен `unknown`, `is_pwa` равен `null`. |
| Режим отображения изменился после инициализации | Все стадии текущего `page_load_id` сохраняют PWA-контекст, определённый при установке измерителя. |
| URL содержит `utm_medium=cpc` | Во всех стадиях `context.utm_medium` равен `cpc`. |
| URL содержит `utm_medium=paid+social%2Fvideo` | `context.utm_medium` равен `paid social/video`. |
| URL не содержит `utm_medium` | Во всех стадиях `context.utm_medium` равен `null`. |
| Cross-origin redirect | TTFB включает навигацию, `redirect_time_ms` равен `null`. |
| Повторное выполнение тега | `page_load_id` не меняется, новых push и слушателей не появляется. |
| Timing API недоступен | Контекст и стадии присутствуют, все метрики равны `null`. |
| BigQuery: `started → snapshot → complete` | В `page_loads` остаётся одна строка стадии `complete`. |
| BigQuery: `started → snapshot` без complete | В `page_loads` остаётся одна строка стадии `snapshot`. |
| BigQuery: `redirect_time_ms` отсутствует | Загрузка не участвует в медиане времени редиректа и не создаёт ложный ноль. |
| BigQuery: `is_pwa` неизвестен | Загрузка не участвует в сравнении PWA vs Web. |
