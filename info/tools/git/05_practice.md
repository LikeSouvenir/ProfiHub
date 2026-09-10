# Практика: полный цикл работы с Git

Разберём сквозной сценарий — от создания репозитория до слияния готовой ветки с исправлением конфликта, объединяющий все команды и флаги из предыдущих конспектов.

## Шаг 1. Инициализация проекта
```bash
mkdir git-practice && cd git-practice
git init

git config user.name "Алина"
git config user.email "alina@example.com"
```

## Шаг 2. Первый коммит
```bash
echo "# Мой проект" > README.md
git status              # README.md отобразится как untracked (новый, неотслеживаемый файл)

git add README.md
git status               # теперь README.md — в staging area

git commit -m "Первый коммит: добавлен README"
git log --oneline        # проверяем, что коммит появился в истории
```

## Шаг 3. Работа с .gitignore
```bash
mkdir node_modules
touch node_modules/some-lib.js
echo "секретный_ключ=12345" > .env

cat > .gitignore << 'EOF'
node_modules/
.env
EOF

git status    # node_modules и .env НЕ появятся в списке — .gitignore их скрывает
git add .gitignore
git commit -m "Добавлен .gitignore"
```

## Шаг 4. Создание ветки под новую функцию
```bash
git checkout -b feature-about-page
```
Создаём файл новой страницы:
```bash
echo "## О проекте" > ABOUT.md
git add ABOUT.md
git commit -m "Добавлена страница о проекте"
```

## Шаг 5. Параллельные изменения в main (имитация конфликта)
```bash
git checkout main
echo "# Мой проект (обновлённое описание)" > README.md
git commit -am "Обновлено описание в README"
```
Пока мы работали в `main`, ветка `feature-about-page` не менялась — сейчас у обеих веток разная история изменений в `README.md`, если бы мы поменяли его и там (специально создадим конфликт на следующем шаге).

## Шаг 6. Создаём настоящий конфликт для практики
```bash
git checkout feature-about-page
echo "# Мой проект (версия из feature)" > README.md
git commit -am "Изменил README в feature-ветке"

git checkout main
git merge feature-about-page
```
Git выведет сообщение о конфликте в `README.md`. Открываем файл — он будет выглядеть так:
```
<<<<<<< HEAD
# Мой проект (обновлённое описание)
=======
# Мой проект (версия из feature)
>>>>>>> feature-about-page
```

### Решаем конфликт вручную
```bash
cat > README.md << 'EOF'
# Мой проект (объединённое описание из main и feature)
EOF

git add README.md
git commit -m "Слияние feature-about-page: конфликт в README разрешён"
```

## Шаг 7. Проверка истории после слияния
```bash
git log --graph --oneline
```
В выводе будет виден merge-коммит и то, как обе ветки (main и feature-about-page) сошлись в одну точку истории.

## Шаг 8. Публикация на удалённый репозиторий
```bash
git remote add origin https://github.com/username/git-practice.git
git push -u origin main
```

## Шаг 9. Работа со stash (временное сохранение)
Представим, что начали правку, но её нужно срочно отложить, не коммитя:
```bash
echo "// незавершённая правка" >> README.md

git stash                  # откладываем изменения, README.md возвращается к последнему коммиту
git status                 # чисто, изменений нет

git stash pop               # возвращаем отложенные изменения обратно
```

## Шаг 10. Отмена последнего коммита (если он оказался ошибочным)
```bash
git commit -am "Случайный неправильный коммит"

git reset --soft HEAD~1     # отменяем коммит, но изменения остаются в staging area — можно закоммитить заново
git status
```

## Итоговая схема пройденного цикла
```
git init
   ↓
git add + git commit (несколько раз)
   ↓
git checkout -b feature-about-page  (новая ветка)
   ↓
изменения + коммиты в feature и параллельно в main
   ↓
git merge  →  конфликт  →  ручное решение  →  git add + git commit
   ↓
git push -u origin main   (публикация)
   ↓
git stash / git reset   (повседневные вспомогательные операции)
```

Этот сценарий охватывает весь стандартный жизненный цикл работы с Git в реальном проекте: от первого коммита до совместной разработки через ветки, слияния с разрешением конфликтов и синхронизации с удалённым репозиторием.
