# Soul Mind Cards — Лог разработки проекта

Лендинг для психолога, работающего с метафорическими ассоциативными картами (МАК).
Репозиторий: <https://github.com/ShirkoSergey/landingSMC>

---

## Исходные данные

- **Макет**: HTML-файл `Лендинг психолога МАК.dc.html` с полным дизайном, контентом и CSS-токенами
- **SEO-пакет**: директория с `meta-tags.html`, `SEO.md`, `README.md`, иконками, og-обложкой, robots.txt, manifest
- **Репозиторий**: пустой, GitHub Pages, удалённый origin на `ShirkoSergey/landingSMC`
- **Домен**: `soulmindcards.net` (изначально планировался `.com`, скорректирован в процессе)

---

## Стек технологий (утверждён до начала работы)

| Слой | Решение |
|------|---------|
| Фреймворк | Astro 7 (статическая генерация) |
| Контент | Markdown / MDX, content collections с Zod-схемой |
| Плагины | `@astrojs/mdx`, `@astrojs/sitemap`, `@astrojs/rss` |
| Rehype | `rehype-slug`, `rehype-autolink-headings` |
| Стили | Scoped CSS в `.astro`, CSS custom properties (oklch) |
| Шрифты | Google Fonts: Work Sans, Lora, IBM Plex Mono |
| JS | Ноль бандла — только `is:inline` скрипты (тема, бургер) |
| Деплой | GitHub Actions → GitHub Pages |
| Блог | Folder-per-post (`src/content/blog/<slug>/index.md`) |

---

## Ход разработки по фазам

### Фаза 0 — Инициализация проекта

**Коммит**: `bac2965 chore: init Astro 7 project with MDX, sitemap, rehype plugins`

**Действия**:
1. `git init` в `C:\Users\OS\work\landingSMC`, настройка remote origin
2. Создание Astro-проекта (`npm create astro@latest`) с минимальным шаблоном — установилась версия Astro 7.3.1
3. Установка зависимостей: `@astrojs/mdx`, `@astrojs/sitemap`, `@astrojs/rss`, `rehype-slug`, `rehype-autolink-headings`
4. Настройка `astro.config.mjs`: site, base, integrations, markdown processor

**Проблема — Sätteri**:
Astro 7 использует Sätteri как дефолтный Markdown-процессор вместо unified. Сборка упала с ошибкой:
> `markdown.rehypePlugins` run on the `unified` processor from `@astrojs/markdown-remark`, which is no longer installed by default now that Sätteri is the default Markdown processor.

**Решение**: установить `@astrojs/markdown-remark` и обернуть rehype-плагины в `unified()`:

```js
import { unified } from '@astrojs/markdown-remark';

export default defineConfig({
  markdown: {
    processor: unified({
      rehypePlugins: [
        rehypeSlug,
        [rehypeAutolinkHeadings, { behavior: 'wrap' }],
      ],
    }),
  },
});
```

**Первый коммит и push в main.**

---

### Фаза 1 — Каркас проекта

**Коммит**: `9d2412d feat: add project scaffold — layouts, components, pages, content collection`

**Действия**:
1. Чтение SEO-материалов из архива
2. Создание структуры директорий: `layouts/`, `components/`, `styles/`, `content/blog/`, `pages/blog/`, `assets/`
3. Копирование SEO-ассетов в `public/`: favicon.svg, apple-touch-icon.png, og-cover.jpg, robots.txt, site.webmanifest
4. Создание `BaseLayout.astro` с полным SEO `<head>` (title, description, OG, Twitter, Schema.org slot, тема-блокировка)
5. Создание `BlogPostLayout.astro` с `.prose :global()` типографикой для рендеренного Markdown
6. Создание 12 компонентов-заготовок с контентом из макета:
   - Header, Footer, Hero, Topics, Method, Steps, Platform, About, Pricing, FAQ, CTA, ThemeToggle
7. Создание страниц: `index.astro`, `blog/index.astro`, `blog/[...slug].astro`, `rss.xml.ts`
8. Создание `content.config.ts` с Zod-схемой
9. Создание примера поста блога (`primer-stati/index.md`, `draft: true`)

**Проблемы — API Astro 7 для content collections**:
- Файл конфигурации: `src/content.config.ts` (не `src/content/config.ts`)
- Импорт `z` из `astro/zod` (не из `astro:content`)
- Нужен `glob` loader из `astro/loaders`
- Используется `post.id` вместо `post.slug`
- `render()` импортируется из `astro:content`

**Итоговая конфигурация контента**:

```ts
import { defineCollection } from 'astro:content';
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.{md,mdx}', base: './src/content/blog' }),
  schema: ({ image }) =>
    z.object({
      title: z.string().max(70),
      description: z.string().min(80).max(160),
      pubDate: z.coerce.date(),
      updatedDate: z.coerce.date().optional(),
      cover: image(),
      coverAlt: z.string(),
      tags: z.array(z.string()).default([]),
      draft: z.boolean().default(false),
    }),
});

export const collections = { blog };
```

**Сборка прошла, коммит запушен.**

---

### Фаза 2 — Дизайн-система (CSS-токены)

**Коммит**: `479e236 feat: add design system — CSS tokens, light/dark theme, global reset`

**Действия**:
1. Создание `src/styles/global.css` со всеми CSS-токенами из макета
2. Светлая тема (`:root`) и тёмная тема (`[data-theme="dark"]`) — 17 custom properties каждая
3. Все цвета в `oklch()` для перцептуально равномерных переходов
4. CSS-ресет и базовая типографика (Work Sans 300)
5. Подключение в `BaseLayout` через `import '../styles/global.css'`

**Пример токенов**:

```css
:root {
  --bg: oklch(0.985 0.005 80);
  --accent: oklch(0.53 0.11 40);
  --accent-h: oklch(0.44 0.1 40);
  /* ... */
}

[data-theme="dark"] {
  --bg: oklch(0.19 0.01 60);
  --accent: oklch(0.72 0.1 45);
  /* ... */
}
```

---

### Фаза 3 — Переключатель темы

Реализован ещё в Фазе 1 в составе `ThemeToggle.astro`:
- Кнопка с солнцем/луной и анимированным «knob»
- `is:inline` скрипт: toggle `data-theme` на `<html>`, сохранение в localStorage
- Блокирующий скрипт в `<head>` BaseLayout для предотвращения FOUC (flash of unstyled content)

---

### Фаза 4 — Стилизация секций

**Коммит**: `ee392d4 feat: style all landing sections matching mockup design`

**Действия**: стилизация всех 9 секций лендинга точно по макету:

| Секция | Особенности стилей |
|--------|--------------------|
| Hero | Kicker, h1 clamp(32px–54px), два CTA-баттона, placeholder фото |
| Topics | 6-колоночная grid auto-fit, IBM Plex Mono нумерация (01–06) |
| Method | 2 колонки: текст + 4 факта с border-left accent |
| Steps | 4-step grid + CTA кнопка |
| Platform | surface-bg секция, feature dots, outline button |
| About | Photo placeholder + bio + 3 credentials |
| Pricing | 3 тарифные карточки |
| FAQ | 5 Q&A items |
| CTA | Финальный centered CTA блок |

Все размеры шрифтов, отступов и цветов соответствуют макету. Адаптивность через `clamp()` и CSS grid `auto-fit`.

---

### Фаза 5 — Блог-пайплайн

**Коммит**: `255c047 feat: verify blog pipeline, add blog link to navigation`

**Действия**:
1. Верификация пайплайна end-to-end (временно `draft: false`):
   - Страница поста рендерится корректно
   - `rehype-slug` добавляет id к заголовкам
   - `rehype-autolink-headings` оборачивает заголовки в ссылки
   - RSS-фид генерируется
   - Список постов фильтрует черновики
2. Добавлена ссылка «Блог» в Header и Footer навигацию
3. Возврат `draft: true` для примера поста

---

### Фаза 6 — SEO и производительность

**Коммит**: `bc14aa4 fix: SEO — base-aware icon/manifest paths, domain .com → .net`

**Действия**:

**6.1 — Аудит**: сравнение сгенерированного `dist/index.html` с эталоном `meta-tags.html` из SEO-пакета. Найдено 3 проблемы.

**6.2 — Пути с base prefix**:
Favicon, apple-touch-icon, manifest, og:image не учитывали `base: '/landingSMC'`.

До:
```html
<link rel="icon" href="/favicon.svg" />
```

После:
```html
<link rel="icon" href={`${base}/favicon.svg`} />
```

Важный нюанс: `import.meta.env.BASE_URL` в Astro 7 возвращает `/landingSMC` (без trailing slash), поэтому нужен явный `/` перед именем файла.

**6.3 — Домен .com → .net**: пользователь скорректировал домен в процессе. Заменено во всех файлах:
- `BaseLayout.astro` — fallback siteUrl
- `index.astro` — Schema.org (ProfessionalService, Person, FAQPage)
- `Platform.astro` — ссылка на платформу

**6.4 — robots.txt**: исправлен URL sitemap с `soulmindcards.com` на `shirkosergey.github.io/landingSMC/`

**6.5 — site.webmanifest**: обновлены `start_url` и пути иконок под base prefix

---

### Фаза 7 — Деплой

**Коммит**: `d9d7ac4 ci: add GitHub Actions workflow for Pages deploy`

**Действия**:
1. Создание `.github/workflows/deploy.yml`:
   - Триггер: push в `main` + `workflow_dispatch`
   - Node 22, `npm ci`, `npx astro build`
   - `actions/upload-pages-artifact` из `dist/`
   - `actions/deploy-pages` для деплоя
   - `concurrency` с `cancel-in-progress: false`
2. Permissions: `contents: read`, `pages: write`, `id-token: write`

**Ручной шаг**: в Settings → Pages репозитория выбрать Source: **GitHub Actions** (вместо "Deploy from a branch").

Результат: `https://shirkosergey.github.io/landingSMC/`

---

### Фаза 8 — Финальная шлифовка

**Коммит**: `0cda626 polish: base-aware links, a11y, mobile burger menu`

**8.1 — Аудит внутренних ссылок**:

Обнаружено, что `href="/blog/"` в Header, Footer и blog/index.astro не учитывают base prefix. На GitHub Pages эти ссылки вели бы на `shirkosergey.github.io/blog/` вместо `.../landingSMC/blog/`.

Исправлено во всех файлах через `${base}/blog/...`:
- `Header.astro` — ссылка «Блог»
- `Footer.astro` — ссылка «Блог»
- `blog/index.astro` — ссылки на посты
- `rss.xml.ts` — поле `link` в RSS items

**8.2 — Доступность (a11y)**:

- **Логотип как ссылка**: `<div class="logo">` → `<a href="${base}/" class="logo">` с сбросом стилей ссылки
- **Skip-to-content**: добавлен `<a href="#main" class="skip-link">Перейти к содержимому</a>` в BaseLayout, виден при фокусе (Tab)
- **id="main"**: добавлен на все `<main>` элементы (index, blog/index, BlogPostLayout)

Стиль skip-link в `global.css`:
```css
.skip-link {
  position: absolute;
  top: -100%;
  left: 16px;
  z-index: 999;
  /* ... */
}
.skip-link:focus {
  top: 0;
}
```

**8.3 — Мобильная навигация (бургер-меню)**:

На экранах `<680px`:
- Навигация скрывается, появляется кнопка-бургер
- Анимированная иконка: три полоски → крестик (CSS transitions)
- `aria-expanded` / `aria-controls` для скринридеров
- Меню закрывается при клике на ссылку
- Скрипт `is:inline` — ноль бандла

---

## Итоговая структура проекта

```
landingSMC/
├── .github/workflows/deploy.yml    # CI/CD
├── public/
│   ├── apple-touch-icon.png
│   ├── favicon.ico
│   ├── favicon.svg
│   ├── og-cover.jpg
│   ├── robots.txt
│   └── site.webmanifest
├── src/
│   ├── components/
│   │   ├── About.astro
│   │   ├── CTA.astro
│   │   ├── FAQ.astro
│   │   ├── Footer.astro
│   │   ├── Header.astro          # + бургер-меню
│   │   ├── Hero.astro
│   │   ├── Method.astro
│   │   ├── Platform.astro
│   │   ├── Pricing.astro
│   │   ├── Steps.astro
│   │   ├── ThemeToggle.astro
│   │   └── Topics.astro
│   ├── content/
│   │   └── blog/
│   │       └── primer-stati/
│   │           ├── cover.jpg
│   │           └── index.md      # draft: true
│   ├── layouts/
│   │   ├── BaseLayout.astro      # SEO head, тема, skip-link
│   │   └── BlogPostLayout.astro  # .prose :global() типографика
│   ├── pages/
│   │   ├── blog/
│   │   │   ├── [...slug].astro
│   │   │   └── index.astro
│   │   ├── index.astro           # лендинг + Schema.org JSON-LD
│   │   └── rss.xml.ts
│   ├── styles/
│   │   └── global.css            # oklch токены, light/dark, reset
│   └── content.config.ts         # Zod-схема blog collection
├── astro.config.mjs
├── package.json
└── tsconfig.json
```

---

## История коммитов

```
bac2965 chore: init Astro 7 project with MDX, sitemap, rehype plugins
9d2412d feat: add project scaffold — layouts, components, pages, content collection
479e236 feat: add design system — CSS tokens, light/dark theme, global reset
ee392d4 feat: style all landing sections matching mockup design
255c047 feat: verify blog pipeline, add blog link to navigation
bc14aa4 fix: SEO — base-aware icon/manifest paths, domain .com → .net
d9d7ac4 ci: add GitHub Actions workflow for Pages deploy
0cda626 polish: base-aware links, a11y, mobile burger menu
```

---

## Ключевые технические решения и подводные камни

### 1. Astro 7 — Sätteri вместо unified
Astro 7 заменил дефолтный Markdown-процессор на Sätteri. Для rehype/remark плагинов нужно явно установить `@astrojs/markdown-remark` и обернуть конфиг в `unified()`.

### 2. Content Collections API v2 (Astro 7)
- Конфиг в `src/content.config.ts` (не `src/content/config.ts`)
- `z` из `astro/zod`, не из `astro:content`
- Нужен `glob` loader из `astro/loaders`
- `post.id` вместо `post.slug`
- `render()` импортируется из `astro:content`

### 3. Scoped styles не применяются к `<Content />`
Markdown, рендеренный через `<Content />`, не получает scoped-стили Astro. Решение — `.prose :global(h2)` паттерн в BlogPostLayout.

### 4. BASE_URL без trailing slash
`import.meta.env.BASE_URL` возвращает `/landingSMC` (без `/` на конце). При конкатенации путей нужен явный слэш: `${base}/favicon.svg`.

### 5. Внутренние ссылки и base prefix
Astro НЕ переписывает `href` в шаблонах автоматически. Все внутренние ссылки (`/blog/`, ссылки на посты) требуют явного `${base}/...`.

### 6. Zero JS bundle
Весь интерактив (тема, бургер) реализован через `is:inline` скрипты, которые не попадают в бандл Vite. Итог: 0 КБ клиентского JS в бандле.

---

## Что осталось для продакшена

- [ ] Заменить placeholder-блоки на реальные фотографии
- [ ] Написать реальные посты блога (снять `draft: true`)
- [ ] Настроить GitHub Pages Source → GitHub Actions в Settings репозитория
- [ ] (Опционально) Подключить кастомный домен soulmindcards.net
- [ ] (Опционально) Провести Lighthouse-аудит на продакшен-URL
