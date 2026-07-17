# Техническое задание: мониторинг загрузки страницы через `dataLayer`

## 1. Цель

Нужно измерять загрузку HTML-документа на стороне браузера и публиковать результаты в `dataLayer`, не полагаясь только на событие `window.load`.

Мониторинг должен фиксировать:

1. Начало работы измерителя.
2. Готовность DOM.
3. Полное завершение загрузки, если оно произошло.
4. Последнее доступное состояние, если пользователь скрыл или покинул страницу до завершения загрузки либо загрузка длится более 30 секунд.
5. Восстановление документа из back/forward cache (bfcache).

Все стадии одной загрузки связываются через единый `page_load_id`.

## 2. Область действия и ограничения

- Измеряются только полные загрузки HTML-документов.
- Переходы внутри SPA через History API, `popstate` или `hashchange` не считаются новой загрузкой документа и находятся вне этого ТЗ.
- Источником сетевых метрик служит `PerformanceNavigationTiming`. Для старых браузеров используется fallback на `performance.timing`.
- LCP, CLS и INP этим решением не измеряются. Это Navigation Timing, а не набор Core Web Vitals.
- `dataLayer.push()` синхронно помещает данные в очередь страницы, но сам по себе не доставляет их во внешнюю систему. Доставка, consent и обработка downstream-тегами настраиваются отдельно.
- Если страница закрыта до запуска GTM-контейнера, клиентский код не сможет зарегистрировать событие.

## 3. Запуск в Google Tag Manager

Измеритель реализуется одним тегом **Custom HTML** и запускается по триггеру **Initialization — All Pages**.

Использовать триггер **Window Loaded** нельзя: он сработает только после полной загрузки и не позволит увидеть незавершённые загрузки. Согласно [документации GTM](https://support.google.com/tagmanager/answer/7679319), Initialization запускается раньше остальных обычных page-view-триггеров, кроме Consent Initialization.

Повторное выполнение тега на одном документе не должно повторно устанавливать слушатели или создавать новый `page_load_id`.

## 4. События жизненного цикла страницы

Не все полезные события принадлежат `window`: часть отправляется объектом `document`.

| Событие | Target | Использование |
| --- | --- | --- |
| `readystatechange` | `document` | Помогает восстановить пропущенную готовность DOM при позднем запуске измерителя. |
| `DOMContentLoaded` | `document` | Основной источник стадии `dom_ready`. |
| `load` | `window` | Источник стадии `complete`; обработка откладывается через `setTimeout(..., 0)`, чтобы браузер успел заполнить `loadEventEnd`. |
| `visibilitychange` | `document` | При переходе в `hidden` создаёт `snapshot`, если `complete` ещё не наступил. |
| `pagehide` | `window` | Fallback для `snapshot`, если он ещё не был создан; `persisted` сохраняется в payload. |
| `pageshow` | `window` | При `persisted === true` создаёт стадию `bfcache_restore`. |
| `freeze` / `resume` | `document` | Дополнительные Page Lifecycle-сигналы Chromium; не являются обязательными для базовой реализации. |
| `beforeunload` / `unload` | `window` | Не используются: ненадёжны и могут ухудшать работу bfcache. |
| `error` / `unhandledrejection` | `window` | Могут использоваться отдельной диагностикой, но не являются стадиями загрузки. |
| `online` / `offline` | `window` | Могут давать сетевой контекст, но не являются стадиями загрузки. |
| `focus` / `blur` | `window` | Не используются для расчёта загрузочных метрик. |
| `popstate` / `hashchange` | `window` | Относятся к SPA-навигации и находятся вне этого ТЗ. |

Переход документа в `hidden` часто является последним состоянием, которое браузер гарантированно позволяет наблюдать, особенно на мобильных устройствах. Поэтому `visibilitychange` используется раньше `pagehide`, `beforeunload` и `unload`. Подробности: [Page Lifecycle API](https://developer.chrome.com/docs/web-platform/page-lifecycle-api), [Beacon API](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon), [bfcache](https://web.dev/articles/bfcache).

## 5. Стадии измерения

Во всех случаях в `dataLayer` передаётся одно имя GTM-события:

```text
page_load_performance
```

Текущее состояние определяется полем `performance_monitoring.measurement_stage`.

| `measurement_stage` | Условие отправки | Повторение |
| --- | --- | --- |
| `started` | Сразу после установки измерителя | Один раз на документ |
| `dom_ready` | После `DOMContentLoaded` либо при восстановлении уже достигнутого состояния | Один раз на документ |
| `snapshot` | До `complete` при первом `visibilitychange:hidden`, fallback `pagehide` или по таймауту 30 секунд | Не более одного раза на документ |
| `complete` | После завершения обработчиков `window.load` | Один раз на документ |
| `bfcache_restore` | При каждом `pageshow` с `persisted === true` | Один раз на каждое восстановление |

Snapshot не означает доказанный отказ от загрузки. Например, пользователь может переключиться на другую вкладку, затем вернуться, после чего загрузка завершится. В этом случае с тем же `page_load_id` последовательно публикуются `snapshot` и `complete`.

Поле `stage_source` принимает только следующие значения:

```text
initialization
dom_content_loaded
ready_state_recovery
load
visibility_hidden
pagehide
timeout
pageshow
```

## 6. Контракт `dataLayer`

Каждый push должен содержать полную структуру. Недостигнутые или недоступные значения явно заполняются `null`. Это исключает наследование значений предыдущей стадии в GTM Data Layer Model.

```javascript
window.dataLayer = window.dataLayer || [];

window.dataLayer.push({
  event: 'page_load_performance',
  performance_monitoring: {
    schema_version: 1,
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
      page_url: 'https://example.test/path',
      entry_url: null,
      utm_source: null,
      is_pwa: false,
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
| `schema_version` | `number` | Версия контракта, для этой реализации всегда `1`. |
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
| `page_url` | `string` | Полный URL текущего документа. |
| `entry_url` | `string \| null` | `document.referrer` или `null`. |
| `utm_source` | `string \| null` | Значение query-параметра `utm_source`. |
| `is_pwa` | `boolean` | Признак режима `display-mode: standalone`. |
| `document_ready_state` | `string` | Текущее значение `document.readyState`. |
| `visibility_state` | `string \| null` | Текущее значение `document.visibilityState`. |
| `pagehide_persisted` | `boolean \| null` | Значение `event.persisted` для source `pagehide`; иначе `null`. |
| `was_discarded` | `boolean` | Был ли предыдущий экземпляр вкладки выгружен браузером из памяти. |

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
    schemaVersion: 1,
    domReadySent: false,
    completeSent: false,
    snapshotSent: false,
    timeoutId: null
  };

  window[GUARD_NAME] = monitor;
  window.dataLayer = window.dataLayer || [];

  function createPageLoadId() {
    var cryptoApi = window.crypto || window.msCrypto;
    var bytes;
    var hex = [];
    var i;

    if (cryptoApi && typeof cryptoApi.randomUUID === 'function') {
      return cryptoApi.randomUUID();
    }

    if (cryptoApi && typeof cryptoApi.getRandomValues === 'function') {
      bytes = new Uint8Array(16);
      cryptoApi.getRandomValues(bytes);
      bytes[6] = (bytes[6] & 15) | 64;
      bytes[8] = (bytes[8] & 63) | 128;

      for (i = 0; i < bytes.length; i += 1) {
        hex.push((bytes[i] + 256).toString(16).slice(1));
      }

      return (
        hex.slice(0, 4).join('') + '-' +
        hex.slice(4, 6).join('') + '-' +
        hex.slice(6, 8).join('') + '-' +
        hex.slice(8, 10).join('') + '-' +
        hex.slice(10, 16).join('')
      );
    }

    return (
      new Date().getTime().toString(36) + '-' +
      Math.random().toString(36).slice(2)
    );
  }

  window.PAGE_LOAD_ID = window.PAGE_LOAD_ID || createPageLoadId();
  monitor.pageLoadId = window.PAGE_LOAD_ID;

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

  function isStandalone() {
    return Boolean(
      window.matchMedia &&
      window.matchMedia('(display-mode: standalone)').matches
    );
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
          dom_ready_reached: monitor.domReadySent,
          load_reached: monitor.completeSent
        },

        metrics: getMetrics(timing, stage),

        context: {
          page: window.location.pathname,
          page_url: window.location.href,
          entry_url: document.referrer || null,
          utm_source: getQueryParameter('utm_source'),
          is_pwa: isStandalone(),
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

  function emitDomReady(source) {
    if (monitor.domReadySent) {
      return;
    }

    monitor.domReadySent = true;
    pushStage('dom_ready', source, null);
  }

  function emitSnapshot(source, pagehidePersisted) {
    if (monitor.completeSent || monitor.snapshotSent) {
      return;
    }

    monitor.snapshotSent = true;
    pushStage('snapshot', source, pagehidePersisted);
  }

  function emitComplete(source) {
    if (monitor.completeSent) {
      return;
    }

    if (!monitor.domReadySent) {
      emitDomReady('ready_state_recovery');
    }

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
    emitDomReady('dom_content_loaded');
  });

  document.addEventListener('readystatechange', function () {
    if (timingShowsDomReady()) {
      emitDomReady('ready_state_recovery');
    }
  });

  window.addEventListener('load', function () {
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
      pushStage('bfcache_restore', 'pageshow', null);
    }
  });

  pushStage('started', 'initialization', null);

  if (timingShowsDomReady()) {
    emitDomReady('ready_state_recovery');
  }

  if (document.readyState === 'complete') {
    window.setTimeout(function () {
      if (timingShowsLoadComplete()) {
        emitComplete('ready_state_recovery');
      }
    }, 0);
  }

  monitor.timeoutId = window.setTimeout(function () {
    emitSnapshot('timeout', null);
  }, Math.max(0, MAX_LOAD_WAIT_MS - observedAtMilliseconds()));
})(window, document);
</script>
```

## 9. Сэмплирование

Producer-код не выполняет сэмплирование: все стадии всегда публикуются в `dataLayer`.

Если downstream-тег использует сэмплирование, решение принимается один раз для всего `page_load_id`. Нельзя независимо сэмплировать отдельные стадии, иначе последовательность `started → dom_ready → complete` станет неполной.

## 10. Критерии приёмки

1. Обычная загрузка создаёт последовательность `started → dom_ready → complete` с одним `page_load_id`.
2. Уход до DOM Ready создаёт `started → snapshot`.
3. Уход после DOM Ready, но до `load`, создаёт `started → dom_ready → snapshot`.
4. Если страница остаётся видимой и не заканчивает загрузку, через 30 секунд создаётся snapshot со `stage_source: "timeout"`.
5. Последовательность hidden → visible → load создаёт snapshot, а затем complete с тем же `page_load_id`.
6. При позднем запуске GTM уже достигнутые стадии восстанавливаются через Navigation Timing и `document.readyState`.
7. При восстановлении из bfcache публикуется `bfcache_restore`, но не создаётся новый `started` и не меняется `page_load_id`.
8. Повторный запуск Custom HTML не создаёт новые listeners и повторные стадии.
9. Каждый push содержит всю схему из раздела 6; недоступные значения равны `null`.
10. При отсутствии Navigation Timing Level 2 используется legacy API.
11. При отсутствии обоих timing API события всё равно публикуются с `timing_api: "unavailable"` и `null` во всех метриках.
12. `full_load_ms` заполнен только у стадии `complete`.
13. `redirect_time_ms` не принимает ложное нулевое значение при недоступном cross-origin timing.

## 11. Тестовые сценарии

| Сценарий | Ожидаемый результат |
| --- | --- |
| Быстрая обычная загрузка | `started`, `dom_ready`, `complete`; snapshot отсутствует. |
| Медленное изображение или iframe | DOM Ready фиксируется раньше complete. |
| Ресурс не завершает загрузку 30 секунд | Создаётся timeout snapshot; при более позднем load дополнительно создаётся complete. |
| Переход на другую страницу до DOM Ready | Создаётся snapshot с `visibility_hidden` или fallback `pagehide`. |
| Переключение вкладки во время загрузки | Создаётся snapshot; возврат и load разрешают последующий complete. |
| GTM запущен после DOMContentLoaded | `dom_ready` восстанавливается с source `ready_state_recovery`. |
| GTM запущен после load | В очереди появляются `started`, восстановленный `dom_ready` и восстановленный `complete`. |
| Назад/вперёд через bfcache | Создаётся `bfcache_restore` с прежним `page_load_id`. |
| Cross-origin redirect | TTFB включает навигацию, `redirect_time_ms` равен `null`. |
| Повторное выполнение тега | Новых push и слушателей не появляется. |
| Timing API недоступен | Контекст и стадии присутствуют, все метрики равны `null`. |

