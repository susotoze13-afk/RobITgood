# Analyst-Agent — Design Spec
Date: 2026-04-25

## Overview

Single agent file `analyst-agent.md` that operates in two modes depending on the session:
1. **Design mode** — creates a design specification Google Doc
2. **Technical mode** — creates a technical specification Google Doc

The agent fits into the pipeline between `hypothesis-verifier-agent` and `designer-agent`/`developer-agent`. Input is always the hypothesis verifier report. Both modes include research via Brave Search before writing the spec.

---

## Section 1: General Flow and Mode Detection

**Input:** Link to hypothesis-verifier-agent Google Doc (or pasted text directly).

**Step 1 — Load data:** Use `firecrawl_scrape` on the Google Doc link. Extract:
- Product name / theme
- Target audience problem
- Target audience description
- Chosen hypothesis (section 6 of verifier report)
- List of functions (section 7 of verifier report)

**Step 2 — Detect mode:** If the user's message already contains keywords ("дизайн-спецификацию", "дизайн-спек", "техническую", "ТЗ", "архитектуру") — detect the mode automatically without asking. Otherwise ask:
> "Какую спецификацию создаём в этом сеансе: дизайн-спецификацию или техническую?"

**Step 3 — Optional design doc input (technical mode only):** If technical mode is selected and a design spec has already been created, ask for its link so architecture and components are aligned.

**Step 4 — Research:** 2–3 targeted Brave Search queries:
- Design mode: UX/UI patterns for similar products, best design examples in this domain
- Technical mode: open-source tech stack choices for similar products, architecture patterns, existing implementations

Research results are used directly in the spec: design mode → fills sections 5 (style) and 8 (references); technical mode → justifies stack choice in section 3 and informs architecture in section 2.

**Step 5 — Write spec:** Create Google Doc via gdrive tool per the relevant structure below.

**Step 6 — Reply to user:** Send doc link + 2–3 sentence summary.

---

## Section 2: Design Specification Structure

**Google Doc title:** `[Product Name] — Дизайн-спецификация [DD.MM.YYYY]`

```
# Дизайн-спецификация: [Название продукта]
Дата: [ДД.ММ.ГГГГ] | Аудитория: [из отчёта] | Версия: 1.0

## 1. Контекст продукта
— Проблема: [из отчёта верификатора, 1-2 предложения]
— Аудитория: [из отчёта, 1-2 предложения]
— Решение: [выбранная гипотеза, 2-3 предложения]

## 2. Целевые персоны
### Персона 1: [Имя]
— Роль: [кто этот человек]
— Цели: [чего хочет достичь с помощью продукта]
— Боли: [что мешает сейчас]
— Tech-грамотность: [низкая / средняя / высокая]

### Персона 2: [Имя]
...

## 3. Основные экраны и пользовательские флоу
### Список экранов
— [Экран 1]: [одна строка — что это]
— [Экран 2]: ...

### Флоу 1: [Название ключевого сценария]
1. [Шаг 1]
2. [Шаг 2]
...

### Флоу 2: [Название]
...

## 4. Wireframe-описания ключевых экранов
### Экран: [Название]
— Верхняя зона: [что здесь]
— Основной контент: [что здесь, какие блоки]
— Нижняя зона / навигация: [что здесь]
— Действия пользователя: [что можно сделать на экране]

### Экран: [Название]
...

## 5. Визуальный стиль
### Цветовая палитра
— Primary: #[HEX] — [назначение]
— Secondary: #[HEX] — [назначение]
— Neutral: #[HEX] — [назначение]
— Success: #[HEX] | Error: #[HEX] | Warning: #[HEX]

### Типографика
— Шрифт: [название] (бесплатный / Google Fonts)
— H1: [размер, вес] | H2: [размер, вес] | Body: [размер, вес] | Caption: [размер, вес]

### Принципы дизайна
— [Принцип 1]: [1 предложение]
— [Принцип 2]: ...

## 6. Компонентная система
### Компоненты
— [Компонент 1] (кнопка / карточка / форма / ...): [описание]
— [Компонент 2]: ...

### Состояния
— Default, Hover, Active, Disabled, Error, Loading — для каждого интерактивного компонента

## 7. Адаптивность и доступность
### Breakpoints
— Mobile: до 768px
— Tablet: 768–1024px
— Desktop: от 1024px

### Accessibility
— WCAG уровень: [AA / AAA]
— Требования к контрасту: минимум 4.5:1 для текста
— Alt-тексты для всех изображений
— Keyboard navigation для всех интерактивных элементов

## 8. Референсы и рекомендации для дизайнера
### Референсы (из результатов поиска)
— [Сервис 1]: [ссылка] — [что именно взять как референс]
— [Сервис 2]: ...

### Ключевые рекомендации
— [Что принципиально важно сохранить при детальном дизайне]
— [Что нельзя менять]
```

**Rules:**
- Strictly 8 sections
- Minimum 2 personas
- Minimum 3 key screens with wireframe descriptions
- Minimum 2 user flows
- All text in Russian

---

## Section 3: Technical Specification Structure

**Google Doc title:** `[Product Name] — Техническая спецификация [DD.MM.YYYY]`

```
# Техническая спецификация: [Название продукта]
Дата: [ДД.ММ.ГГГГ] | Версия: 1.0

## 1. Контекст и цели
— Проблема: [1-2 предложения]
— Решение: [выбранная гипотеза, 2-3 предложения]
— Ключевые нефункциональные требования: [производительность, масштабируемость, надёжность]

## 2. Архитектура системы
— Паттерн: [monolith / microservices / BFF] — [обоснование выбора]
— Компоненты системы и их взаимодействие:
  [Client] → [API Gateway / Backend] → [Database]
                                      ↘ [External Services]
— Описание каждого компонента: [что делает, с чем взаимодействует]

## 3. Технологический стек
### Обоснование (на основе исследования)
— Фронтенд: [технология] — [почему]
— Бэкенд: [технология] — [почему]
— База данных: [технология] — [почему]
— Инфраструктура: [технология] — [почему]
— Дополнительные сервисы: [если нужны]

## 4. База данных
### Таблица: [название]
| Поле | Тип | Ограничения | Описание |
|------|-----|-------------|----------|
| id   | UUID | PK, NOT NULL | ... |
| ...  | ... | ... | ... |

### Связи
— [Таблица A] → [Таблица B]: [тип связи, через какое поле]

### Индексы
— [Таблица].[поле]: [причина индексирования]

## 5. API
### Аутентификация
— Метод: [JWT / OAuth2 / session]
— Описание: [как работает]

### Эндпоинты
#### [Группа ресурсов]
| Метод | Путь | Описание | Тело запроса | Ответ |
|-------|------|----------|--------------|-------|
| GET   | /api/v1/... | ... | — | { ... } |
| POST  | /api/v1/... | ... | { ... } | { ... } |

## 6. Бэкенд
### Структура модулей
[корень]/
├── [модуль 1]/   # [назначение]
├── [модуль 2]/   # [назначение]
└── ...

### Ключевые сервисы
— [Сервис 1]: [ответственность]
— [Сервис 2]: [ответственность]

## 7. Фронтенд
### Структура приложения
[src]/
├── pages/      # [описание]
├── components/ # [описание]
├── store/      # [описание]
└── ...

### Страницы
— [Страница 1]: [соответствующий экран из дизайн-спецификации]
— [Страница 2]: ...

### State Management
— Инструмент: [выбранный подход]
— Что хранится в стейте: [список]

## 8. Инфраструктура и деплой
### Локальная разработка
— docker-compose.yml: [сервисы, порты]
— Переменные окружения: [список с описанием]

### CI/CD
— Пайплайн: [инструмент и шаги]
— Окружения: dev → staging → prod

### Хостинг
— [Рекомендованный вариант деплоя — предпочтительно бесплатный/дешёвый open-source friendly]

## 9. Безопасность
— Аутентификация/авторизация: [подход, роли]
— Защита данных: [шифрование, хранение паролей]
— OWASP Top-10 меры: [SQL injection, XSS, CSRF, rate limiting, ...]
— Переменные окружения: [что никогда не хранится в коде]

## 10. Тестирование
— Unit-тесты: [что покрываем, инструмент]
— Integration-тесты: [что покрываем, инструмент]
— E2E-тесты: [ключевые флоу, инструмент]
— Минимальное покрытие: [%]

## 11. Мониторинг и логирование
— Логи: [что логируем, уровни логирования, инструмент]
— Метрики: [какие метрики отслеживаем]
— Алертинг: [когда и как уведомляем]

## 12. Структура проекта
[project-root]/
├── frontend/
├── backend/
├── docker-compose.yml
├── .env.example
├── CLAUDE.md
└── README.md

### CLAUDE.md (содержимое для Claude Code)
— Команды запуска: [dev, test, build, migrate]
— Переменные окружения: [список обязательных]
— Структура проекта: [краткое описание]
— Частые задачи: [как добавить новый эндпоинт, как добавить миграцию, ...]

## 13. План разработки
### Фаза 1: [Название] (примерно [N] дней)
— [Задача 1]
— [Задача 2]

### Фаза 2: [Название]
...

### Зависимости
— [Задача B] зависит от [Задача A]
— ...
```

**Rules:**
- Strictly 13 sections
- Stack choice must be justified by research results
- CLAUDE.md section must include startup commands and env vars
- All text in Russian

---

## Trigger Keywords

Agent should be invoked when user says:
- "сделай технические спецификации", "подготовь ТЗ", "опиши архитектуру"
- "напиши спеки на дизайн и разработку"
- "дизайн-спецификация", "техническая спецификация"
- "аналитик"
