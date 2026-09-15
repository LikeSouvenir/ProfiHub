# Практические кейсы: Frontend

Практические задания по направлению Frontend, построенные по той же схеме, что и кейсы `case/tasks/*` для Solidity: в каждой папке `task.md` (техническое задание с критериями приёмки) и `solution.md` (эталонное решение с комментариями и разбором).

Нумерация модулей соответствует конспектам в `info/frontend/`.

| № | Кейс | Материалы | Проект |
|---|---|---|---|
| 01 | `01_html_css_layout` | `info/frontend/01_html-css` | TeamLanding — лендинг команды |
| 02 | `02_js_basics` | `info/frontend/02_js` | ScoreBoard — зачётная ведомость |
| 03 | `03_dom_events` | `info/frontend/03_conect html-css js` | TaskTracker — интерактивный трекер |
| 04 | `04_objects_promises` | `info/frontend/04_object and promise` | TaskApiClient — клиент учебного API |
| 05 | `05_web3_connect` | `info/frontend/05_async and blockchain conect` | TaskLogClient — Web3.js / Ethers.js / Viem |
| 06 | `06_react_intro` | `info/frontend/06_Introduction react` | ModuleShowcase — витрина модулей |
| 07 | `07_react_hooks` | `info/frontend/07_base react` | PrepTracker — трекер с useState / useEffect |
| 08 | `08_react_routing_context` | `info/frontend/08_react-routing` | KnowledgeHub — роутинг и Context API |
| 09 | `09_react_bootstrap` | `info/frontend/09_bootstrap` | TaskStore — витрина на React-Bootstrap |

## Как проходить

Кейсы выстроены по нарастающей и опираются друг на друга: проект из модуля 7 продолжает идеи модуля 6, модуль 8 добавляет роутинг, модуль 9 объединяет всё вместе с UI-библиотекой. Рекомендуемый порядок — последовательный.

1. Прочитать конспекты соответствующего модуля в `info/frontend/`.
2. Выполнить `task.md` самостоятельно, не заглядывая в решение.
3. Сверить с `solution.md`, обратив внимание на раздел «Разбор решения» — там разобраны типичные ошибки.

## Требования к окружению

- Node.js 18+ и npm (модули 5–9)
- Локальная нода Geth и Remix IDE (модуль 5)
- Браузер с инструментами разработчика (все модули)
