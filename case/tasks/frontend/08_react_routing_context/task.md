# Модуль 8: React Router и Context API — Практическое задание

## Проект: Многостраничная база знаний (KnowledgeHub)

### Цель

Освоить установку и подключение сторонних библиотек, маршрутизацию через `react-router-dom` (`BrowserRouter`, `Routes`, `Route`, `Link`, `NavLink`, вложенные маршруты, `Outlet`, динамические параметры, `useParams`, `useNavigate`, `useSearchParams`, страница 404), реорганизацию структуры проекта под роутинг (`pages/` против `components/`), а также Context API (`createContext`, компонент-провайдер, `useContext`, собственный хук доступа к контексту) как решение проблемы prop drilling.

### Техническое задание

1. **Установка:**

   ```bash
   npm install react-router-dom
   ```

   - В `README.md` поясните разницу между `dependencies` и `devDependencies` применительно к этой библиотеке и зачем нужен `BrowserRouter` против `HashRouter`.

2. **Структура проекта:**

   ```
   src/
   ├── components/
   │   ├── Navbar.jsx
   │   ├── Layout.jsx
   │   ├── ModuleCard.jsx
   │   ├── LessonList.jsx
   │   └── SearchBox.jsx
   ├── pages/
   │   ├── HomePage.jsx
   │   ├── TracksPage.jsx
   │   ├── TrackPage.jsx
   │   ├── ModulePage.jsx
   │   ├── FavoritesPage.jsx
   │   ├── AboutPage.jsx
   │   └── NotFoundPage.jsx
   ├── context/
   │   ├── ContentContext.jsx
   │   └── FavoritesContext.jsx
   ├── data/
   │   └── content.js
   ├── App.jsx
   └── main.jsx
   ```

   - Поясните в `README.md`, чем `pages/` отличается от `components/`: страница — это то, что напрямую сопоставлено маршруту.

3. **Данные (`data/content.js`):**

   - Массив `tracks`: `{ slug, title, description }` — минимум 3 направления (`solidity`, `frontend`, `devops`).

   - Массив `modules`: `{ id, slug, trackSlug, title, level, lessons: [{ id, title, minutes }] }` — минимум 6 модулей.

4. **`ContentContext.jsx` — общий контекст данных:**

   - `createContext` + компонент `ContentProvider`, оборачивающий `children`.

   - В значение контекста передаются: `tracks`, `modules`, а также функции `getTrackBySlug(slug)`, `getModulesByTrack(trackSlug)`, `getModuleBySlug(slug)`.

   - Собственный хук `useContent()`, который вызывает `useContext` и бросает понятную ошибку, если использован вне провайдера.

5. **`FavoritesContext.jsx` — избранное:**

   - Хранит массив id избранных модулей в `useState`.

   - Методы: `toggleFavorite(id)`, `isFavorite(id)`, `clearFavorites()`, поле `favoritesCount`.

   - Хук `useFavorites()`.

   - Значение контекста мемоизируется через `useMemo`, функции — через `useCallback`; поясните комментарием, зачем.

6. **Маршрутизация (`App.jsx`):**

   - `BrowserRouter` подключается на верхнем уровне (в `main.jsx` или `App.jsx`), провайдеры контекста оборачивают роутер.

   - Маршруты:

     | Путь | Страница |
     |---|---|
     | `/` | `HomePage` |
     | `/tracks` | `TracksPage` |
     | `/tracks/:trackSlug` | `TrackPage` |
     | `/modules/:moduleSlug` | `ModulePage` |
     | `/favorites` | `FavoritesPage` |
     | `/about` | `AboutPage` |
     | `*` | `NotFoundPage` |

   - Все маршруты вложены в `Layout` (родительский `Route` с `element={<Layout />}` и `<Outlet />` внутри).

7. **`Layout.jsx`:**

   - Содержит `Navbar`, `<main><Outlet /></main>` и подвал.

   - `Outlet` — место, куда подставляется компонент текущего дочернего маршрута.

8. **`Navbar.jsx`:**

   - Ссылки через `NavLink` с подсветкой активного пункта: `className={({ isActive }) => isActive ? "nav__link nav__link--active" : "nav__link"}`.

   - Пункт «Избранное» показывает счётчик из `FavoritesContext` — **без передачи пропсов через промежуточные компоненты**.

   - Для главной укажите `end`, чтобы `/` не подсвечивалась на всех вложенных маршрутах.

9. **Динамические маршруты:**

   - `TrackPage` читает `trackSlug` через `useParams`, находит направление через `getTrackBySlug`. Если направление не найдено — редирект на `/404` через `<Navigate to="/404" replace />` либо отрисовка `NotFoundPage`.

   - `ModulePage` читает `moduleSlug`, выводит список уроков и кнопку «В избранное».

   - Кнопка «Назад» использует `useNavigate`: `navigate(-1)`.

10. **Параметры строки запроса (`useSearchParams`):**

    - На `TracksPage` реализуйте поиск и фильтр по уровню, сохраняя их в URL: `/tracks?q=erc&level=средний`.

    - При перезагрузке страницы фильтры восстанавливаются из URL.

11. **Программная навигация:**

    - На `HomePage` кнопка «Продолжить обучение» через `useNavigate` переводит на страницу последнего незавершённого модуля.

12. **Страница 404:**

    - Маршрут `*` ловит любой несуществующий путь и показывает `NotFoundPage` со ссылкой на главную.

### Критерии приёмки

- Переход между страницами не перезагружает приложение (проверяется по вкладке Network).
- Прямое открытие адреса `/tracks/solidity` работает корректно.
- Счётчик избранного в шапке обновляется из любой страницы без проброса пропсов.
- `useContent()` вне провайдера бросает понятную ошибку, а не `undefined`-исключение.
- Несуществующий адрес открывает 404, а не пустую страницу.
- Активный пункт меню подсвечен корректно, включая главную.
