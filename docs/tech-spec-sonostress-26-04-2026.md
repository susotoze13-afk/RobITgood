# Техническая спецификация: СоноСтресс — PWA-трекер стресса и сна
Дата: 26.04.2026 | Версия: 1.0 | Стек: Vanilla JS / PWA / localStorage

---

## 1. Обзор архитектуры

### Паттерн: Single-Page Application (SPA) без бэкенда

Приложение реализуется как автономная PWA — один HTML-файл (точка входа `index.html`) плюс набор JS/CSS/assets. Весь стек работает исключительно на стороне клиента. Сервер нужен только для первоначальной раздачи статики (или работает без сети после кэширования).

```
[Браузер / PWA-оболочка]
        │
        ├── index.html  ← точка входа, app-shell
        ├── sw.js       ← Service Worker (кэш + push)
        ├── manifest.json ← PWA-манифест
        │
        ├── js/
        │   ├── app.js        ← роутер экранов, init
        │   ├── storage.js    ← обёртка localStorage
        │   ├── diary.js      ← модуль дневника
        │   ├── techniques.js ← модуль техник
        │   ├── cbti.js       ← CBT-I расчёты
        │   ├── thoughts.js   ← дневник мыслей КПТ
        │   ├── insights.js   ← алгоритм инсайтов
        │   ├── charts.js     ← SVG-графики
        │   ├── export.js     ← CSV / JSON экспорт
        │   └── push.js       ← Web Push / уведомления
        │
        ├── css/
        │   ├── base.css      ← переменные, reset, типографика
        │   ├── components.css← карточки, кнопки, формы
        │   ├── screens.css   ← стили экранов
        │   └── animations.css← CSS-анимации дыхания
        │
        └── assets/
            ├── icons/        ← PWA иконки (72, 96, 128, 144, 152, 192, 384, 512px)
            └── sounds/       ← опциональные WAV/MP3 для дыхательных техник
```

### Принципы архитектуры

- **Offline-first**: все ресурсы кэшируются при установке Service Worker; приложение полностью работает без сети
- **Нулевые зависимости**: ни одного npm-пакета, ни одного CDN (кроме Google Fonts, загружаемых один раз и кэшируемых SW)
- **Модульность через ES-модули**: каждый JS-файл — ES Module (`type="module"`)
- **Данные только локально**: `localStorage` — единственное хранилище, сервер с данными отсутствует
- **Производительность**: целевой показатель Lighthouse PWA Score ≥ 90, First Contentful Paint < 1.5 с на 3G

---

## 2. Файловая структура проекта

```
sonostress/
├── index.html                  # SPA-оболочка: разметка всех экранов, подключение модулей
├── manifest.json               # PWA App Manifest
├── sw.js                       # Service Worker
├── .htaccess                   # Заголовки кэширования для Apache (если нужен)
│
├── js/
│   ├── app.js                  # Инициализация, роутер, глобальные события
│   ├── storage.js              # Единственная точка доступа к localStorage
│   ├── diary.js                # Дневник сна и настроения (ввод, список, редактирование)
│   ├── techniques.js           # Дыхательные техники + ПМР + Заземление (таймер, анимация)
│   ├── cbti.js                 # CBT-I: расчёт эффективности сна, рекомендации окна
│   ├── thoughts.js             # Дневник мыслей КПТ (4 шага)
│   ├── insights.js             # Автоматические инсайты и корреляции
│   ├── charts.js               # SVG-графики (линейный, без библиотек)
│   ├── export.js               # Экспорт CSV и JSON через Blob API
│   └── push.js                 # Service Worker Push API, настройка напоминаний
│
├── css/
│   ├── base.css                # CSS Custom Properties, reset, типографика Inter
│   ├── components.css          # Карточки, кнопки, формы, бейджи, модалки
│   ├── screens.css             # Стили каждого из 7 экранов
│   └── animations.css          # Анимации дыхательного круга, transitions
│
├── assets/
│   ├── icons/
│   │   ├── icon-72.png
│   │   ├── icon-96.png
│   │   ├── icon-128.png
│   │   ├── icon-144.png
│   │   ├── icon-152.png
│   │   ├── icon-192.png         # Используется как иконка на рабочем столе
│   │   ├── icon-384.png
│   │   └── icon-512.png         # Используется в splash screen
│   └── sounds/
│       ├── bell.wav             # Сигнал смены фазы дыхания (опционально)
│       └── complete.wav         # Сигнал завершения упражнения
│
├── CLAUDE.md                   # Инструкции для Claude Code
└── README.md                   # Документация проекта
```

### Экраны (соответствуют разделам index.html)

| ID экрана | Навигация | Описание |
|---|---|---|
| `screen-dashboard` | Главная | Дашборд: показатели дня, быстрый вход в дневник, стрик |
| `screen-diary` | Дневник | Форма ввода сна/настроения, список записей |
| `screen-techniques` | Техники | Список техник релаксации, запуск таймера |
| `screen-cbti` | — | CBT-I модуль (достижим из дашборда) |
| `screen-thoughts` | — | Дневник мыслей КПТ (достижим из меню) |
| `screen-progress` | Прогресс | SVG-графики, инсайты, стрелки изменений |
| `screen-settings` | — | Уведомления, тема, экспорт данных |

---

## 3. Описание каждого модуля

### 3.1 `storage.js` — Абстракция хранилища

**Назначение**: единственная точка работы с localStorage. Все остальные модули используют только функции этого файла — никогда `localStorage` напрямую.

**API модуля:**
```js
// Получить массив записей по ключу
export function getRecords(key)         // → Array

// Сохранить массив записей
export function saveRecords(key, arr)   // → void

// Добавить одну запись
export function addRecord(key, obj)     // → void

// Обновить запись по индексу
export function updateRecord(key, index, obj) // → void

// Удалить запись по индексу
export function deleteRecord(key, index) // → void

// Получить настройки
export function getSettings()           // → Object

// Сохранить настройки (merge)
export function saveSettings(patch)     // → void

// Полный экспорт всего хранилища
export function exportAll()             // → Object {sleepDiary, thoughtDiary, settings}

// Очистить всё (с подтверждением)
export function clearAll()              // → void
```

**Валидация при чтении:**
```js
function getRecords(key) {
  try {
    const raw = localStorage.getItem(key);
    if (!raw) return [];
    const parsed = JSON.parse(raw);
    return Array.isArray(parsed) ? parsed : [];
  } catch {
    console.warn(`storage: corrupt data for key "${key}", resetting`);
    localStorage.removeItem(key);
    return [];
  }
}
```

**Ключи localStorage:**

| Ключ | Тип | Описание |
|---|---|---|
| `sleepDiary` | JSON Array | Записи дневника сна |
| `thoughtDiary` | JSON Array | Записи дневника мыслей КПТ |
| `settings` | JSON Object | Настройки приложения |
| `streakData` | JSON Object | Данные стрика (lastDate, count) |

---

### 3.2 `diary.js` — Дневник сна и настроения

**Назначение**: ввод, редактирование, отображение записей дневника.

**Алгоритм расчёта длительности сна:**
```js
function calcDuration(bedtime, waketime) {
  // bedtime, waketime — строки "HH:MM"
  const [bH, bM] = bedtime.split(':').map(Number);
  const [wH, wM] = waketime.split(':').map(Number);

  let bedMinutes  = bH * 60 + bM;
  let wakeMinutes = wH * 60 + wM;

  // Если время пробуждения меньше — переход через полночь
  if (wakeMinutes <= bedMinutes) wakeMinutes += 24 * 60;

  const durationMinutes = wakeMinutes - bedMinutes;
  return {
    minutes: durationMinutes,
    hours: durationMinutes / 60,                // число (например: 7.5)
    formatted: formatDuration(durationMinutes)   // "7 ч 30 мин"
  };
}
```

**Структура записи:**
```js
{
  date:     "2026-04-26",       // ISO-дата (YYYY-MM-DD), ключ идентификации
  bedtime:  "23:30",            // время отхода ко сну (HH:MM)
  waketime: "07:00",            // время пробуждения (HH:MM)
  duration: 450,                // длительность сна в минутах (рассчитывается)
  quality:  7,                  // качество сна 1–10 (integer)
  stress:   4,                  // уровень стресса 1–10 (integer)
  note:     "Текст заметки"     // опционально, до 500 символов
}
```

**Ограничения и валидация:**
- `date`: обязательное поле, формат YYYY-MM-DD; нельзя сохранить две записи на одну дату (обновляется существующая)
- `bedtime` / `waketime`: обязательные, формат HH:MM, регулярное выражение `/^([01]\d|2[0-3]):[0-5]\d$/`
- `quality` / `stress`: целые числа от 1 до 10 включительно
- `note`: строка до 500 символов; символы `<>` экранируются для защиты от XSS

**Поведение "один день — одна запись":**
```js
function saveDiaryEntry(entry) {
  const records = getRecords('sleepDiary');
  const existingIndex = records.findIndex(r => r.date === entry.date);
  if (existingIndex >= 0) {
    updateRecord('sleepDiary', existingIndex, entry); // перезапись
  } else {
    addRecord('sleepDiary', entry);                   // новая запись
  }
}
```

---

### 3.3 `techniques.js` — Интерактивные техники релаксации

**Назначение**: управление таймером и анимацией для дыхательных техник, ПМР, заземления.

**Конфигурация техник (data-driven подход):**
```js
export const TECHNIQUES = {
  '478': {
    id: '478',
    title: 'Дыхание 4-7-8',
    totalSeconds: 300,  // 5 минут = 3 полных цикла × ~63 сек + запас
    phases: [
      { label: 'Вдох',    duration: 4000,  scale: 'scale(1.2)',  color: '#6c63ff' },
      { label: 'Задержка',duration: 7000,  scale: 'scale(1.2)',  color: '#a29bfe' },
      { label: 'Выдох',   duration: 8000,  scale: 'scale(0.85)', color: '#c3b4f7' },
    ]
  },
  'square': {
    id: 'square',
    title: 'Квадратное дыхание',
    totalSeconds: 240,
    phases: [
      { label: 'Вдох',    duration: 4000, scale: 'scale(1.15)', color: '#6c63ff' },
      { label: 'Задержка',duration: 4000, scale: 'scale(1.15)', color: '#a29bfe' },
      { label: 'Выдох',   duration: 4000, scale: 'scale(0.9)',  color: '#c3b4f7' },
      { label: 'Задержка',duration: 4000, scale: 'scale(0.9)',  color: '#8b8fa8' },
    ]
  },
  'pmr': {
    id: 'pmr',
    title: 'Прогрессивная мышечная релаксация',
    totalSeconds: 900,  // 15 минут
    phases: [
      { label: 'Напряжение',   duration: 6000,  scale: 'scale(1.25)', color: '#f87171' },
      { label: 'Расслабление', duration: 20000, scale: 'scale(0.8)',  color: '#4ade80' },
    ]
  },
  'grounding': {
    id: 'grounding',
    title: 'Заземление 5-4-3-2-1',
    totalSeconds: 300,
    steps: [  // особый режим: последовательные текстовые шаги
      { count: 5, sense: 'вижу',     icon: '👁️' },
      { count: 4, sense: 'ощущаю',   icon: '✋' },
      { count: 3, sense: 'слышу',    icon: '👂' },
      { count: 2, sense: 'чувствую', icon: '👃' },
      { count: 1, sense: 'ощущаю на вкус', icon: '👅' },
    ]
  }
};
```

**Класс таймера:**
```js
class BreathTimer {
  constructor(techniqueId, callbacks) {
    this.technique   = TECHNIQUES[techniqueId];
    this.remaining   = this.technique.totalSeconds;
    this.phaseIndex  = 0;
    this.cycleCount  = 0;
    this.running     = false;
    this.callbacks   = callbacks; // { onPhase, onTick, onComplete }
  }

  start()  { this.running = true;  this._tick(); this._runPhase(); }
  pause()  { this.running = false; clearTimeout(this._phaseTimer); clearInterval(this._tickTimer); }
  resume() { this.running = true;  this._tick(); this._runPhase(); }
  reset()  { this.pause(); this.remaining = this.technique.totalSeconds; this.phaseIndex = 0; this.cycleCount = 0; }

  _runPhase() {
    if (!this.running) return;
    const phases = this.technique.phases;
    const phase  = phases[this.phaseIndex % phases.length];
    this.callbacks.onPhase(phase, this.cycleCount);
    if (this.phaseIndex > 0 && this.phaseIndex % phases.length === 0) this.cycleCount++;
    this.phaseIndex++;
    this._phaseTimer = setTimeout(() => this._runPhase(), phase.duration);
  }

  _tick() {
    this._tickTimer = setInterval(() => {
      this.remaining--;
      this.callbacks.onTick(this.remaining);
      if (this.remaining <= 0) {
        this.pause();
        this.callbacks.onComplete();
      }
    }, 1000);
  }
}
```

**Web Audio API (опциональные звуки):**
```js
// Генерация тона без внешних файлов — fallback когда sounds/ отсутствуют
function playTone(frequency = 440, duration = 0.3) {
  try {
    const ctx  = new (window.AudioContext || window.webkitAudioContext)();
    const osc  = ctx.createOscillator();
    const gain = ctx.createGain();
    osc.connect(gain);
    gain.connect(ctx.destination);
    osc.frequency.value = frequency;
    gain.gain.setValueAtTime(0.3, ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + duration);
    osc.start();
    osc.stop(ctx.currentTime + duration);
  } catch { /* AudioContext недоступен — тихая деградация */ }
}
```

---

### 3.4 `cbti.js` — CBT-I модуль ограничения сна

Детальный алгоритм описан в разделе 4.

---

### 3.5 `thoughts.js` — Дневник мыслей КПТ

**Структура записи:**
```js
{
  id:          "uuid-v4",             // уникальный ID (crypto.randomUUID())
  date:        "2026-04-26",
  time:        "21:35",               // время создания
  situation:   "Текст ситуации",      // шаг 1, до 500 символов
  thought:     "Автоматическая мысль",// шаг 2, до 500 символов
  emotion:     ["тревога", "страх"],  // шаг 3, массив из списка + free-input
  intensity:   8,                     // интенсивность эмоции 1–10
  alternative: "Альтернативная мысль" // шаг 4, до 500 символов
}
```

**Список эмоций (предустановленный):**
```js
export const EMOTIONS = [
  'тревога', 'страх', 'злость', 'грусть', 'стыд',
  'вина', 'обида', 'раздражение', 'разочарование',
  'одиночество', 'беспомощность', 'растерянность'
];
```

**Генерация UUID без внешних библиотек:**
```js
function generateId() {
  return crypto.randomUUID
    ? crypto.randomUUID()
    : 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
        const r = Math.random() * 16 | 0;
        return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
      });
}
```

---

### 3.6 `charts.js` — SVG-графики

**Назначение**: рендер линейных графиков без внешних библиотек.

**Алгоритм построения линейного графика:**
```js
/**
 * Строит SVG-линейный график
 * @param {HTMLElement} container — куда монтировать SVG
 * @param {number[]}    data      — массив значений
 * @param {object}      opts      — { color, label, min, max, width, height }
 */
export function renderLineChart(container, data, opts = {}) {
  const W      = opts.width  || 320;
  const H      = opts.height || 120;
  const PAD    = { top: 10, right: 10, bottom: 24, left: 28 };
  const minVal = opts.min ?? Math.min(...data);
  const maxVal = opts.max ?? Math.max(...data);
  const range  = maxVal - minVal || 1;

  // Нормализация значений в пиксели
  const toX = (i) => PAD.left + (i / (data.length - 1)) * (W - PAD.left - PAD.right);
  const toY = (v) => PAD.top + (1 - (v - minVal) / range) * (H - PAD.top - PAD.bottom);

  // Построение polyline points
  const points = data.map((v, i) => `${toX(i)},${toY(v)}`).join(' ');

  // Область под графиком (полигон с заливкой)
  const areaPoints = [
    `${toX(0)},${H - PAD.bottom}`,
    ...data.map((v, i) => `${toX(i)},${toY(v)}`),
    `${toX(data.length - 1)},${H - PAD.bottom}`
  ].join(' ');

  const svg = `
    <svg viewBox="0 0 ${W} ${H}" width="100%" style="overflow:visible">
      <defs>
        <linearGradient id="grad-${opts.id}" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%"   stop-color="${opts.color}" stop-opacity="0.3"/>
          <stop offset="100%" stop-color="${opts.color}" stop-opacity="0"/>
        </linearGradient>
      </defs>
      <!-- Сетка -->
      ${[0, 0.25, 0.5, 0.75, 1].map(t => {
        const y = PAD.top + t * (H - PAD.top - PAD.bottom);
        const v = (maxVal - t * range).toFixed(1);
        return `<line x1="${PAD.left}" y1="${y}" x2="${W - PAD.right}" y2="${y}"
                  stroke="#2a3a5c" stroke-width="1"/>
                <text x="${PAD.left - 4}" y="${y + 4}" font-size="9"
                  fill="#5a6280" text-anchor="end">${v}</text>`;
      }).join('')}
      <!-- Заливка -->
      <polygon points="${areaPoints}" fill="url(#grad-${opts.id})"/>
      <!-- Линия -->
      <polyline points="${points}" fill="none"
        stroke="${opts.color}" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
      <!-- Точки данных -->
      ${data.map((v, i) => `
        <circle cx="${toX(i)}" cy="${toY(v)}" r="3" fill="${opts.color}"/>
      `).join('')}
    </svg>
  `;

  container.innerHTML = svg;
}
```

---

### 3.7 `export.js` — Экспорт данных

**CSV-экспорт дневника сна:**
```js
export function exportSleepCSV() {
  const records = getRecords('sleepDiary');
  const headers = ['Дата','Время засыпания','Время пробуждения',
                   'Длительность (мин)','Качество сна','Уровень стресса','Заметка'];

  const rows = records.map(r => [
    r.date, r.bedtime, r.waketime, r.duration, r.quality, r.stress,
    `"${(r.note || '').replace(/"/g, '""')}"` // экранирование кавычек
  ]);

  const csv = [headers, ...rows]
    .map(row => row.join(','))
    .join('\r\n');

  // BOM для корректного открытия в Excel
  const blob = new Blob(['﻿' + csv], { type: 'text/csv;charset=utf-8;' });
  downloadBlob(blob, `sonostress-sleep-${today()}.csv`);
}
```

**JSON-экспорт (полный бэкап):**
```js
export function exportFullJSON() {
  const data = exportAll(); // из storage.js
  const json = JSON.stringify(data, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  downloadBlob(blob, `sonostress-backup-${today()}.json`);
}
```

**Скачивание через Blob API (без сервера):**
```js
function downloadBlob(blob, filename) {
  const url = URL.createObjectURL(blob);
  const a   = Object.assign(document.createElement('a'), {
    href: url, download: filename
  });
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);   // освобождаем память
}
```

---

### 3.8 `push.js` — Push-уведомления

**Регистрация разрешения и подписки:**
```js
export async function requestNotificationPermission() {
  if (!('Notification' in window)) return 'unsupported';
  const permission = await Notification.requestPermission();
  return permission; // 'granted' | 'denied' | 'default'
}
```

**Локальные напоминания через setTimeout (без VAPID-сервера):**
```js
// Расчёт миллисекунд до следующего срабатывания
export function scheduleLocalReminder(timeStr) {
  // timeStr = "21:00"
  const [h, m] = timeStr.split(':').map(Number);
  const now    = new Date();
  const target = new Date();
  target.setHours(h, m, 0, 0);
  if (target <= now) target.setDate(target.getDate() + 1); // завтра
  const delay = target - now;

  setTimeout(() => {
    if (Notification.permission === 'granted') {
      new Notification('СоноСтресс', {
        body: 'Время заполнить дневник сна',
        icon: '/assets/icons/icon-192.png',
        badge: '/assets/icons/icon-72.png',
        vibrate: [200, 100, 200]
      });
    }
    scheduleLocalReminder(timeStr); // рекурсивный перенос на следующий день
  }, delay);
}
```

**Важное ограничение**: `setTimeout`-напоминания работают только пока вкладка открыта. Для фоновых уведомлений (когда приложение закрыто) требуется Web Push с VAPID-сервером. На текущем MVP-этапе — только локальные уведомления. Добавление VAPID — Фаза 2+.

---

## 4. Алгоритм CBT-I расчёта окна сна (детально)

Источник протокола: Stanford Health Care Sleep Restriction Guidelines; мета-анализ Trauer et al., Sleep 2015.

### 4.1 Входные данные

- Минимум **7 записей** дневника сна за последние 14 дней (берутся самые свежие 7+).
- Из каждой записи: `bedtime`, `waketime`, `duration` (в минутах).

### 4.2 Расчёт средних показателей

```js
function calcAverages(records) {
  // Берём последние min(N, 14) записей
  const recent = records.slice(-14);

  const avgTIB = average(recent.map(r => {
    // TIB (Time In Bed) = waketime - bedtime в минутах
    return calcDuration(r.bedtime, r.waketime).minutes;
  }));

  const avgTST = average(recent.map(r => r.duration)); // Total Sleep Time

  return { avgTIB, avgTST, count: recent.length };
}
```

### 4.3 Расчёт эффективности сна (Sleep Efficiency, SE%)

```
SE% = (avgTST / avgTIB) × 100
```

```js
function calcSleepEfficiency(avgTST, avgTIB) {
  if (avgTIB === 0) return 0;
  return Math.round((avgTST / avgTIB) * 100);
}
```

**Интерпретация SE%:**

| SE% | Статус | Действие |
|---|---|---|
| ≥ 90% | Отличная | Увеличить окно сна на 15 мин |
| 85–89% | Хорошая | Окно не менять |
| 80–84% | Удовлетворительная | Окно не менять (ещё неделю) |
| < 80% | Сниженная | Уменьшить окно сна на 15 мин |

### 4.4 Расчёт начального окна сна

```js
function calcInitialWindow(avgTST) {
  const MIN_WINDOW = 4.5 * 60; // 4 ч 30 мин — клинический минимум (Stanford)
  const windowMinutes = Math.max(avgTST, MIN_WINDOW);

  // Рекомендуемое время подъёма — фиксируется пользователем (например, 07:00)
  // Время отхода ко сну = время подъёма - окно
  return windowMinutes;
}
```

### 4.5 Корректировка окна (еженедельная)

```js
/**
 * Возвращает рекомендованное изменение окна сна в минутах
 * @param {number} se — Sleep Efficiency в %
 * @returns {number} delta в минутах (-15 | 0 | +15)
 */
function adjustWindow(se) {
  if (se >= 90) return +15;
  if (se >= 85) return 0;
  if (se >= 80) return 0;
  return -15;
}
```

### 4.6 Генерация текстовых рекомендаций

```js
function generateCBTIRecommendations(se, avgTST, avgTIB, windowMinutes) {
  const recs = [];
  const wakeTime = getSettings().wakeTime || '07:00';

  // Вычисляем рекомендуемое время укладывания
  const [wH, wM] = wakeTime.split(':').map(Number);
  const bedMinutes = (wH * 60 + wM) - windowMinutes;
  const bedHour = Math.floor(((bedMinutes % 1440) + 1440) % 1440 / 60);
  const bedMin  = Math.floor(((bedMinutes % 1440) + 1440) % 1440 % 60);
  const bedTimeStr = `${String(bedHour).padStart(2,'0')}:${String(bedMin).padStart(2,'0')}`;

  recs.push(`Ложитесь спать не раньше ${bedTimeStr} и вставайте в ${wakeTime} каждый день, включая выходные.`);

  if (se < 85) {
    recs.push(`Ваша эффективность сна ${se}% ниже целевых 85%. Оставайтесь в постели только в отведённое время.`);
  } else {
    recs.push(`Эффективность сна ${se}% — хороший показатель. Постепенно увеличиваем время в постели.`);
  }

  recs.push(`Избегайте телефона и яркого света за 1 час до ${bedTimeStr}.`);
  recs.push(`При невозможности заснуть в течение 20 минут — встаньте, займитесь спокойной активностью, вернитесь когда почувствуете сонливость.`);

  return recs;
}
```

### 4.7 Полный экспортируемый интерфейс `cbti.js`

```js
export function getCBTIAnalysis() {
  const records = getRecords('sleepDiary');
  if (records.length < 7) {
    return { ready: false, daysCollected: records.length, daysNeeded: 7 };
  }

  const { avgTIB, avgTST, count } = calcAverages(records);
  const se = calcSleepEfficiency(avgTST, avgTIB);
  const windowMinutes = calcInitialWindow(avgTST);
  const delta = adjustWindow(se);
  const recommendations = generateCBTIRecommendations(se, avgTST, avgTIB, windowMinutes + delta);

  return {
    ready:           true,
    daysCollected:   count,
    avgTST:          Math.round(avgTST),        // в минутах
    avgTIB:          Math.round(avgTIB),        // в минутах
    sleepEfficiency: se,                         // %
    windowMinutes:   windowMinutes + delta,      // рекомендованное окно
    deltaMinutes:    delta,                      // изменение (-15/0/+15)
    recommendations                              // массив строк
  };
}
```

---

## 5. Service Worker: стратегия кэширования

**Файл**: `sw.js` в корне проекта (для максимального scope).

### 5.1 Версионирование кэша

```js
const CACHE_VERSION = 'v1.0.0';
const CACHE_STATIC  = `sonostress-static-${CACHE_VERSION}`;
const CACHE_DYNAMIC = `sonostress-dynamic-${CACHE_VERSION}`;
```

При выпуске новой версии меняется только строка `CACHE_VERSION` — событие `activate` автоматически очищает старые кэши.

### 5.2 Список ресурсов для precache (Cache-First)

```js
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/manifest.json',
  '/css/base.css',
  '/css/components.css',
  '/css/screens.css',
  '/css/animations.css',
  '/js/app.js',
  '/js/storage.js',
  '/js/diary.js',
  '/js/techniques.js',
  '/js/cbti.js',
  '/js/thoughts.js',
  '/js/insights.js',
  '/js/charts.js',
  '/js/export.js',
  '/js/push.js',
  '/assets/icons/icon-192.png',
  '/assets/icons/icon-512.png',
  // Шрифт Inter (загружается один раз, кэшируется навсегда)
  'https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap',
];
```

### 5.3 События Service Worker

```js
// INSTALL: precache всех статических ресурсов
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_STATIC)
      .then(cache => cache.addAll(PRECACHE_URLS))
      .then(() => self.skipWaiting()) // активируем немедленно
  );
});

// ACTIVATE: удаляем старые кэши
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(k => k !== CACHE_STATIC && k !== CACHE_DYNAMIC)
          .map(k => caches.delete(k))
      )
    ).then(() => self.clients.claim())
  );
});

// FETCH: стратегия по типу ресурса
self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // Навигационные запросы — Cache-First (app shell)
  if (event.request.mode === 'navigate') {
    event.respondWith(
      caches.match('/index.html').then(r => r || fetch(event.request))
    );
    return;
  }

  // Статические ресурсы — Cache-First
  if (PRECACHE_URLS.includes(url.pathname) || url.origin !== location.origin) {
    event.respondWith(
      caches.match(event.request)
        .then(r => r || fetch(event.request).then(res => {
          const clone = res.clone();
          caches.open(CACHE_STATIC).then(c => c.put(event.request, clone));
          return res;
        }))
    );
    return;
  }

  // Всё остальное — Network-First с fallback на кэш
  event.respondWith(
    fetch(event.request)
      .then(res => {
        const clone = res.clone();
        caches.open(CACHE_DYNAMIC).then(c => c.put(event.request, clone));
        return res;
      })
      .catch(() => caches.match(event.request))
  );
});

// PUSH: обработка входящих push-уведомлений (если VAPID реализован)
self.addEventListener('push', event => {
  const data = event.data?.json() ?? { title: 'СоноСтресс', body: 'Время заполнить дневник' };
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body:    data.body,
      icon:    '/assets/icons/icon-192.png',
      badge:   '/assets/icons/icon-72.png',
      vibrate: [200, 100, 200],
      data:    { url: '/' }
    })
  );
});

// NOTIFICATIONCLICK: открыть приложение
self.addEventListener('notificationclick', event => {
  event.notification.close();
  event.waitUntil(clients.openWindow('/'));
});
```

### 5.4 Регистрация SW в `app.js`

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW registered, scope:', reg.scope))
      .catch(err => console.warn('SW registration failed:', err));
  });
}
```

---

## 6. Схема localStorage (типы данных, валидация)

### 6.1 Ключ `sleepDiary` — массив записей дневника сна

**Тип**: `Array<SleepRecord>`

```ts
interface SleepRecord {
  date:     string;   // "YYYY-MM-DD" — уникальный ключ
  bedtime:  string;   // "HH:MM"
  waketime: string;   // "HH:MM"
  duration: number;   // целое число минут ≥ 0
  quality:  number;   // integer 1..10
  stress:   number;   // integer 1..10
  note:     string;   // "" | строка до 500 символов
}
```

**Валидация при записи:**
```js
function validateSleepRecord(r) {
  const errors = [];
  if (!/^\d{4}-\d{2}-\d{2}$/.test(r.date))          errors.push('invalid date');
  if (!/^([01]\d|2[0-3]):[0-5]\d$/.test(r.bedtime))  errors.push('invalid bedtime');
  if (!/^([01]\d|2[0-3]):[0-5]\d$/.test(r.waketime)) errors.push('invalid waketime');
  if (!Number.isInteger(r.quality) || r.quality < 1 || r.quality > 10) errors.push('invalid quality');
  if (!Number.isInteger(r.stress)  || r.stress  < 1 || r.stress  > 10) errors.push('invalid stress');
  if (typeof r.note === 'string' && r.note.length > 500)                errors.push('note too long');
  return errors; // пустой массив = запись валидна
}
```

**Оценочный объём**: 1 запись ≈ 200 байт. 365 записей ≈ 73 КБ. Лимит localStorage 5 МБ достигается примерно через 25 000 записей (~68 лет) — не ограничивает.

---

### 6.2 Ключ `thoughtDiary` — массив записей КПТ

**Тип**: `Array<ThoughtRecord>`

```ts
interface ThoughtRecord {
  id:          string;   // UUID v4
  date:        string;   // "YYYY-MM-DD"
  time:        string;   // "HH:MM"
  situation:   string;   // до 500 символов
  thought:     string;   // до 500 символов
  emotion:     string[]; // массив строк из EMOTIONS + free-input
  intensity:   number;   // integer 1..10
  alternative: string;   // до 500 символов
}
```

---

### 6.3 Ключ `settings` — объект настроек

```ts
interface AppSettings {
  notifications:  boolean;  // разрешены ли уведомления
  reminderTime:   string;   // "HH:MM" — время ежедневного напоминания, default "21:00"
  theme:          'dark' | 'light'; // default 'dark'
  wakeTime:       string;   // "HH:MM" — целевое время подъёма для CBT-I, default "07:00"
  onboardingDone: boolean;  // флаг завершения онбординга
}
```

**Значения по умолчанию:**
```js
const DEFAULT_SETTINGS = {
  notifications: false,
  reminderTime:  '21:00',
  theme:         'dark',
  wakeTime:      '07:00',
  onboardingDone: false,
};
```

---

### 6.4 Ключ `streakData` — данные стрика

```ts
interface StreakData {
  lastDate:     string;  // "YYYY-MM-DD" — дата последней записи
  currentStreak: number; // текущая серия дней подряд
  longestStreak: number; // рекорд серии
}
```

**Алгоритм обновления стрика:**
```js
function updateStreak() {
  const today    = new Date().toISOString().slice(0, 10);
  const raw      = localStorage.getItem('streakData');
  const streak   = raw ? JSON.parse(raw) : { lastDate: null, currentStreak: 0, longestStreak: 0 };
  const yesterday = new Date(Date.now() - 86400000).toISOString().slice(0, 10);

  if (streak.lastDate === today) {
    return streak; // уже считали сегодня
  } else if (streak.lastDate === yesterday) {
    streak.currentStreak++;
  } else {
    streak.currentStreak = 1; // серия прервана
  }

  streak.lastDate     = today;
  streak.longestStreak = Math.max(streak.longestStreak, streak.currentStreak);
  localStorage.setItem('streakData', JSON.stringify(streak));
  return streak;
}
```

---

## 7. Алгоритмы генерации инсайтов

**Файл**: `insights.js`

Инсайты — автоматические текстовые выводы, которые формируются при наличии минимум 7 записей в дневнике.

### 7.1 Инсайт 1: корреляция стресс → качество сна

```js
function stressToQualityInsight(records) {
  if (records.length < 7) return null;
  const recent = records.slice(-14);

  // Разбиваем на дни с высоким стрессом (≥6) и низким (<6)
  const highStress = recent.filter(r => r.stress >= 6);
  const lowStress  = recent.filter(r => r.stress < 6);

  if (highStress.length < 2 || lowStress.length < 2) return null;

  const avgQualityHigh = average(highStress.map(r => r.quality));
  const avgQualityLow  = average(lowStress.map(r => r.quality));
  const diff = avgQualityLow - avgQualityHigh;

  if (diff >= 1.5) {
    return `В дни с высоким стрессом ваш сон на ${diff.toFixed(1)} балла хуже. Техники релаксации вечером могут помочь.`;
  }
  return null;
}
```

### 7.2 Инсайт 2: тренд качества сна (неделя к неделе)

```js
function sleepQualityTrendInsight(records) {
  if (records.length < 14) return null;

  const sorted = [...records].sort((a,b) => a.date.localeCompare(b.date));
  const prevWeek = sorted.slice(-14, -7);
  const thisWeek = sorted.slice(-7);

  const prevAvg = average(prevWeek.map(r => r.quality));
  const thisAvg = average(thisWeek.map(r => r.quality));
  const diff    = thisAvg - prevAvg;

  if (diff >= 0.5)  return `Качество сна улучшилось на ${diff.toFixed(1)} балла по сравнению с прошлой неделей. Отличная динамика!`;
  if (diff <= -0.5) return `Качество сна снизилось на ${Math.abs(diff).toFixed(1)} балла. Попробуйте CBT-I модуль.`;
  return `Качество сна стабильно на уровне ${thisAvg.toFixed(1)}/10.`;
}
```

### 7.3 Инсайт 3: оптимальное время пробуждения

```js
function wakeTimeInsight(records) {
  if (records.length < 7) return null;
  const recent = records.slice(-14);

  // Находим записи с качеством сна ≥ 7
  const goodSleep = recent.filter(r => r.quality >= 7);
  if (goodSleep.length < 3) return null;

  // Средняя длительность при хорошем сне
  const avgGoodDuration = average(goodSleep.map(r => r.duration));
  const hours = (avgGoodDuration / 60).toFixed(1);

  return `При хорошем самочувствии вы спите в среднем ${hours} ч. Это ваша личная норма сна.`;
}
```

### 7.4 Инсайт 4: регулярность режима (вариабельность времени подъёма)

```js
function scheduleConsistencyInsight(records) {
  if (records.length < 7) return null;
  const recent = records.slice(-7);

  const wakeTimes = recent.map(r => {
    const [h, m] = r.waketime.split(':').map(Number);
    return h * 60 + m;
  });

  const mean = average(wakeTimes);
  const variance = average(wakeTimes.map(t => Math.pow(t - mean, 2)));
  const stdDev = Math.sqrt(variance); // стандартное отклонение в минутах

  if (stdDev > 60) {
    return `Время подъёма за последнюю неделю варьируется более чем на ${Math.round(stdDev)} минут. Нерегулярный режим ухудшает качество сна.`;
  }
  if (stdDev <= 30) {
    return `Режим сна стабилен — время подъёма отклоняется не более чем на ${Math.round(stdDev)} минут. Это положительно влияет на сон.`;
  }
  return null;
}
```

### 7.5 Мастер-функция генерации всех инсайтов

```js
export function generateInsights() {
  const records = getRecords('sleepDiary');
  const insights = [
    stressToQualityInsight(records),
    sleepQualityTrendInsight(records),
    wakeTimeInsight(records),
    scheduleConsistencyInsight(records),
  ].filter(Boolean); // убираем null

  return insights; // массив строк, может быть пустым
}
```

### 7.6 Стрелки-индикаторы изменений

```js
/**
 * @returns {{ direction: 'up'|'down'|'same', diff: number, label: string }}
 */
export function getWeeklyDelta(field) {
  // field = 'quality' | 'stress' | 'duration'
  const records = getRecords('sleepDiary');
  if (records.length < 7) return null;

  const sorted   = [...records].sort((a,b) => a.date.localeCompare(b.date));
  const prevWeek = sorted.slice(-14, -7);
  const thisWeek = sorted.slice(-7);

  if (prevWeek.length < 3) return null; // нет данных для сравнения

  const prev = average(prevWeek.map(r => r[field]));
  const curr = average(thisWeek.map(r => r[field]));
  const diff = curr - prev;

  return {
    direction: diff > 0.1 ? 'up' : diff < -0.1 ? 'down' : 'same',
    diff:      Math.abs(diff).toFixed(1),
    label:     diff >= 0 ? `↑ +${Math.abs(diff).toFixed(1)}` : `↓ −${Math.abs(diff).toFixed(1)}`
  };
}
```

---

## 8. Требования к производительности и совместимости браузеров

### 8.1 Целевые метрики производительности

| Метрика | Целевое значение | Инструмент измерения |
|---|---|---|
| Lighthouse PWA Score | ≥ 90 | Chrome DevTools Lighthouse |
| Lighthouse Performance | ≥ 85 | Chrome DevTools Lighthouse |
| First Contentful Paint (FCP) | < 1.5 с (3G Fast) | Lighthouse / WebPageTest |
| Time to Interactive (TTI) | < 3.0 с (3G Fast) | Lighthouse |
| Total Blocking Time (TBT) | < 200 мс | Lighthouse |
| Cumulative Layout Shift (CLS) | < 0.1 | Lighthouse |
| JS Bundle Size | < 100 КБ (без иконок/шрифтов) | build output |
| localStorage операция | < 5 мс | performance.now() |
| SVG-рендер 30 точек | < 16 мс (60 fps) | requestAnimationFrame |

### 8.2 Техники оптимизации

**Загрузка:**
- Критический CSS инлайнится в `<head>` (< 14 КБ) — ни одного render-blocking ресурса
- JS-модули подключаются с `type="module"` (автоматически defer)
- Шрифт Inter загружается с `font-display: swap` — текст виден немедленно

**Рендеринг:**
- CSS-анимации дыхательного круга — только `transform` и `opacity` (не вызывают layout reflow)
- SVG-графики строятся один раз при открытии экрана, перестраиваются только при изменении данных
- DOM-манипуляции батчатся — обновления вносятся за один проход

**Память:**
- `URL.revokeObjectURL()` вызывается сразу после скачивания файла
- Аудио-контексты закрываются после завершения звука (`ctx.close()`)
- Обработчики событий навешиваются на родительские контейнеры (event delegation), а не на каждый элемент

### 8.3 Совместимость браузеров

**Целевые браузеры** (покрытие аудитории Россия/СНГ):

| Браузер | Версия | Особенности |
|---|---|---|
| Chrome для Android | 90+ | Полная поддержка PWA, Web Push, Web Audio |
| Safari iOS | 16.4+ | PWA с add-to-homescreen, уведомления с iOS 16.4 |
| Samsung Internet | 14+ | Полная поддержка PWA |
| Chrome Desktop | 90+ | Полная поддержка |
| Firefox Desktop | 88+ | Нет Web Push на десктопе без сервера |

**Полифиллы и деградация:**

| API | Поддержка | Стратегия при отсутствии |
|---|---|---|
| `crypto.randomUUID()` | Chrome 92+, Safari 15.4+ | Fallback на `Math.random()`-имплементацию |
| `Notification API` | Широкая | Graceful degradation: скрыть кнопку напоминаний |
| `Web Audio API` | Широкая | Try/catch: без звука, только визуальная анимация |
| `CSS backdrop-filter` | Широкая | Непрозрачный фон как fallback |
| Service Worker | Chrome 45+, Safari 11.1+ | Без SW: приложение работает, но без офлайн |
| `localStorage` | Все браузеры | Try/catch: показать предупреждение если недоступен |

**Обнаружение возможностей (feature detection):**
```js
const CAPABILITIES = {
  serviceWorker: 'serviceWorker' in navigator,
  notifications: 'Notification' in window,
  webAudio:      'AudioContext' in window || 'webkitAudioContext' in window,
  randomUUID:    typeof crypto !== 'undefined' && typeof crypto.randomUUID === 'function',
};
```

### 8.4 Требования к устройствам

- Минимальное разрешение экрана: 320×568px (iPhone SE 1-го поколения)
- Основное целевое разрешение: 390×844px (iPhone 14) и Android-эквиваленты
- Ориентация: портретная (альбомная не ограничена, но не оптимизируется на MVP)
- RAM: работа при 1 ГБ RAM (бюджетные Android-устройства)

---

## 9. Чеклист готовности к деплою

### 9.1 Функциональность

- [ ] Все 7 экранов открываются и закрываются без ошибок в консоли
- [ ] Дневник сна: создание записи, редактирование существующей, расчёт длительности корректен
- [ ] Дневник сна: валидация — нельзя сохранить запись с пустыми обязательными полями
- [ ] Техники релаксации: все 4 техники запускаются, таймер считает, анимация работает
- [ ] Техники: кнопка "Пауза" / "Продолжить" работает корректно
- [ ] CBT-I: при < 7 записей — прогресс-бар и сообщение "N из 7 дней данных"
- [ ] CBT-I: при ≥ 7 записях — расчёт эффективности и рекомендации отображаются
- [ ] Дневник мыслей: прохождение всех 4 шагов, сохранение, просмотр, удаление записи
- [ ] Прогресс: SVG-графики рендерятся за 7 и 30 дней при наличии данных
- [ ] Прогресс: инсайты отображаются при наличии 7+ записей
- [ ] Экспорт CSV: файл скачивается, открывается в Excel без проблем с кириллицей (BOM)
- [ ] Экспорт JSON: файл скачивается, содержит все данные localStorage
- [ ] Настройки: переключение темы сохраняется после перезагрузки страницы
- [ ] Напоминания: запрос разрешения появляется, время устанавливается

### 9.2 PWA

- [ ] `manifest.json` валиден — проверить через Chrome DevTools → Application → Manifest
- [ ] Service Worker зарегистрирован — Application → Service Workers → Status: activated
- [ ] Precache всех ресурсов выполнен — Application → Cache Storage → sonostress-static
- [ ] Приложение работает полностью офлайн (Network → Offline → перезагрузка страницы)
- [ ] "Добавить на главный экран" появляется на Android Chrome (A2HS prompt)
- [ ] Иконки корректно отображаются на iOS (Settings → Home screen) и Android
- [ ] Standalone-режим: строка браузера скрыта при запуске с рабочего стола
- [ ] `theme-color` применяется в шапке браузера (тёмно-синий)

### 9.3 Производительность

- [ ] Lighthouse PWA Score ≥ 90 (мобильный режим эмуляции)
- [ ] Lighthouse Performance ≥ 85
- [ ] FCP < 1.5 с при эмуляции Slow 3G
- [ ] CLS < 0.1 (отсутствуют скачки раскладки)
- [ ] Нет ресурсов, загружаемых без кэша, при повторном посещении (все отдаются из SW)

### 9.4 Совместимость

- [ ] Протестировано в Chrome for Android (актуальная версия)
- [ ] Протестировано в Safari iOS 16.4+ (реальное устройство или BrowserStack)
- [ ] Протестировано в Samsung Internet 14+
- [ ] Нет JavaScript-ошибок в консоли на всех браузерах
- [ ] При `localStorage` недоступен (режим "инкогнито" Safari) — показывается понятное сообщение
- [ ] При отключённых уведомлениях — приложение работает без ошибок

### 9.5 Безопасность

- [ ] Весь пользовательский ввод (заметки, текст мыслей) экранируется перед отображением в DOM
- [ ] Нет `innerHTML` с пользовательскими данными без экранирования
- [ ] Все внешние ресурсы (только шрифт Inter) указаны явно в precache-списке SW
- [ ] `Content-Security-Policy` настроен на сервере (запрещает inline-scripts кроме тех, что нужны)
- [ ] HTTPS — обязательно (required для Service Worker и Push API)

### 9.6 Хостинг (рекомендация)

**Вариант 1 — GitHub Pages (бесплатно, рекомендуется):**
- Деплой из ветки `gh-pages` или директории `/docs`
- HTTPS автоматически
- CDN по всему миру
- CI/CD: GitHub Actions — push в `main` → деплой в `gh-pages`

**Вариант 2 — Netlify (бесплатный tier):**
- Drag-and-drop деплой папки проекта
- Автоматические HTTPS + custom domain
- 100 ГБ трафика/месяц бесплатно

**Вариант 3 — Cloudflare Pages (бесплатно):**
- Связка с GitHub-репозиторием
- Неограниченный трафик
- Лучшее время отклика для СНГ

### 9.7 CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate HTML
        run: npx html-validate index.html
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v11
        with:
          urls: |
            https://your-username.github.io/sonostress/
          budgetPath: .lighthouserc.json
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: .
```

---

## CLAUDE.md (содержимое для Claude Code)

```markdown
# СоноСтресс — PWA-трекер стресса и сна

## Команды

— Запуск (dev): `npx serve . -p 3000` или `python -m http.server 3000`
  (HTTPS не нужен локально — SW работает на localhost)
— Сборка: сборка не нужна (vanilla JS, нет бандлера)
— Линтинг JS: `npx eslint js/*.js`
— Линтинг HTML: `npx html-validate index.html`
— Lighthouse: `npx lighthouse http://localhost:3000 --view`
— Обновить версию кэша SW: изменить строку CACHE_VERSION в sw.js

## Переменные окружения (обязательные)

Проект не использует переменные окружения — все настройки в localStorage.

## Структура проекта

index.html — точка входа, содержит HTML-разметку всех 7 экранов и подключает JS-модули.
sw.js — Service Worker, кэширует все ресурсы при установке.
js/ — все JS-модули, каждый отвечает за один функциональный блок.
css/ — стили разделены по назначению: base (переменные), components, screens, animations.
Данные пользователя хранятся только в localStorage браузера, сервер не используется.

## Частые задачи

— Добавить новую дыхательную технику: добавить объект в TECHNIQUES в js/techniques.js
— Добавить новый инсайт: написать функцию в js/insights.js, добавить вызов в generateInsights()
— Обновить версию кэша (новый релиз): изменить CACHE_VERSION в sw.js
— Добавить новое поле в дневник: 1) добавить поле в интерфейс SleepRecord в storage.js,
  2) обновить форму в index.html, 3) обновить функцию calcDuration/saveEntry в diary.js,
  4) обновить заголовки CSV в export.js
— Запустить один тест Lighthouse: `npx lighthouse http://localhost:3000 --only-categories=pwa`
```

---

*Техническая спецификация создана агентом-аналитиком AI-RobITGood. Версия 1.0 — 26.04.2026.*
