# Git - Система контроля версий

Распределённая система управления версиями исходного кода.

---

## 📚 Основы Git

### Что такое Git

**Git** - распределённая система контроля версий (VCS), созданная Линусом Торвальдсом в 2005.

**Преимущества:**
- **Distributed** - каждый разработчик имеет полную копию репозитория
- **Fast** - большинство операций локальные
- **Branching** - мощная система веток
- **Merging** - умное слияние изменений
- **Integrity** - SHA-1 хеширование, невозможно потерять данные незаметно

### Три состояния файлов

```
Working Directory    →    Staging Area    →    Git Repository
  (modified)              (staged)              (committed)
      
vim file.txt         git add file.txt     git commit -m "..."
      ↓                     ↓                     ↓
   Modified               Staged               Committed
```

**1. Working Directory** - рабочая директория, текущие файлы
**2. Staging Area (Index)** - подготовленные к commit изменения
**3. Git Repository (.git)** - постоянное хранилище commits

---

## 🔧 Базовые команды

### Инициализация и клонирование

```bash
# Создать новый репозиторий
git init

# Клонировать существующий
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git custom-folder

# Клонировать только последний commit (shallow clone)
git clone --depth 1 https://github.com/user/repo.git
```

### Конфигурация

```bash
# Глобальная конфигурация (для всех репозиториев)
git config --global user.name "John Doe"
git config --global user.email "john@example.com"

# Локальная (для текущего репозитория)
git config user.name "John Doe"
git config user.email "john@work.com"

# Редактор по умолчанию
git config --global core.editor "vim"
git config --global core.editor "code --wait"

# Алиасы
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status

# Просмотр конфигурации
git config --list
git config user.name
```

### Состояние и история

```bash
# Статус репозитория
git status
git status -s  # short format

# История коммитов
git log
git log --oneline
git log --graph --oneline --all
git log --author="John"
git log --since="2 weeks ago"
git log --grep="bugfix"
git log -p  # с diff
git log -n 5  # последние 5

# Красивый лог
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit

# История для файла
git log -- file.txt
git log -p -- file.txt  # с изменениями

# Blame (кто изменил строку)
git blame file.txt
git blame -L 10,20 file.txt  # строки 10-20
```

### Добавление и коммит

```bash
# Добавить файл в staging
git add file.txt
git add .  # все файлы
git add *.php  # по маске
git add -A  # все изменения (modified, deleted, new)

# Добавить с интерактивным режимом
git add -p  # выбрать куски изменений

# Убрать из staging
git reset file.txt
git reset  # убрать все

# Commit
git commit -m "Add user authentication"
git commit -am "Fix bug"  # add + commit для tracked files

# Изменить последний commit
git commit --amend -m "New message"
git commit --amend --no-edit  # добавить файлы без изменения сообщения
```

### Отмена изменений

```bash
# Откатить изменения в файле (working directory)
git checkout -- file.txt
git restore file.txt  # новая команда

# Unstage файл (из staging → working)
git reset HEAD file.txt
git restore --staged file.txt  # новая команда

# Откатить последний commit (сохранив изменения)
git reset --soft HEAD~1

# Откатить последний commit (удалив изменения)
git reset --hard HEAD~1

# Откатить к определённому commit
git reset --hard abc123

# Откатить конкретный commit (создаёт новый commit)
git revert abc123
```

---

## 🌿 Ветки (Branches)

### Основы веток

```bash
# Список веток
git branch
git branch -a  # все (включая remote)
git branch -v  # с последним commit

# Создать ветку
git branch feature-login

# Переключиться на ветку
git checkout feature-login
git switch feature-login  # новая команда

# Создать и переключиться
git checkout -b feature-login
git switch -c feature-login  # новая команда

# Удалить ветку
git branch -d feature-login  # safe delete
git branch -D feature-login  # force delete

# Переименовать ветку
git branch -m old-name new-name
git branch -m new-name  # текущую ветку

# Удалить remote ветку
git push origin --delete feature-login
```

### Merge (слияние)

```bash
# Переключиться на целевую ветку
git checkout main

# Слить feature ветку
git merge feature-login

# Типы merge:
# 1. Fast-forward (если нет расхождений)
git merge feature-login
# main просто перемещается вперёд

# 2. Three-way merge (если были commits в обеих ветках)
git merge feature-login
# Создаётся merge commit

# 3. Squash merge (все commits в один)
git merge --squash feature-login
git commit -m "Add login feature"

# Отменить merge
git merge --abort
```

### Rebase (перебазирование)

```bash
# Переместить commits на новый base
git checkout feature-login
git rebase main

# Процесс:
# 1. Git находит общего предка (main и feature)
# 2. Временно сохраняет commits из feature
# 3. Переключается на main
# 4. Применяет commits из feature поверх main

# Интерактивный rebase (редактирование истории)
git rebase -i HEAD~3

# Опции:
# pick   - использовать commit
# reword - изменить сообщение
# edit   - остановиться для изменения
# squash - объединить с предыдущим
# fixup  - как squash но без сообщения
# drop   - удалить commit

# Продолжить после разрешения конфликтов
git rebase --continue

# Отменить rebase
git rebase --abort
```

### Merge vs Rebase

**Merge:**
```
main:    A---B---C---F (merge commit)
              \     /
feature:       D---E

✅ Сохраняет полную историю
✅ Понятно когда feature была влита
❌ История становится сложной (много веток)
```

**Rebase:**
```
main:    A---B---C
                  \
feature:           D'---E'

✅ Линейная история
✅ Чистый git log
❌ Переписывает историю (опасно для public веток)
```

**Golden Rule:**
```
❌ НИКОГДА не rebase public ветки (main, develop)
✅ Rebase только свои feature ветки перед merge
```

---

## 🔀 Стратегии ветвления

### Git Flow

```
main (production)
  ├─ develop (integration)
  │   ├─ feature/user-auth
  │   ├─ feature/payment
  │   └─ feature/notifications
  ├─ release/v1.2.0
  └─ hotfix/critical-bug

Ветки:
- main: production, только releases
- develop: integration, объединение features
- feature/*: новые функции
- release/*: подготовка релиза
- hotfix/*: срочные исправления production
```

**Workflow:**
```bash
# Feature
git checkout develop
git checkout -b feature/user-auth
# ... работа ...
git checkout develop
git merge --no-ff feature/user-auth
git branch -d feature/user-auth

# Release
git checkout develop
git checkout -b release/v1.2.0
# ... финальные правки, версия ...
git checkout main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0
git checkout develop
git merge --no-ff release/v1.2.0
git branch -d release/v1.2.0

# Hotfix
git checkout main
git checkout -b hotfix/critical-bug
# ... исправление ...
git checkout main
git merge --no-ff hotfix/critical-bug
git tag -a v1.2.1
git checkout develop
git merge --no-ff hotfix/critical-bug
git branch -d hotfix/critical-bug
```

### GitHub Flow (упрощённый)

```
main (production + development)
  ├─ feature/user-auth
  ├─ feature/payment
  └─ bugfix/login-error

Workflow:
1. Создать ветку от main
2. Commit изменения
3. Открыть Pull Request
4. Code review
5. Merge в main
6. Deploy
```

**Workflow:**
```bash
# 1. Создать feature ветку
git checkout main
git pull origin main
git checkout -b feature/user-auth

# 2. Работа
git add .
git commit -m "Add user authentication"

# 3. Push
git push origin feature/user-auth

# 4. Открыть PR на GitHub

# 5. После review и approval:
# Merge на GitHub (merge commit или squash)

# 6. Обновить локальный main
git checkout main
git pull origin main
git branch -d feature/user-auth
```

### Trunk-Based Development

```
main (trunk)
  └─ короткоживущие feature ветки (1-2 дня)

Принципы:
- Все commit часто в main (или короткие feature ветки)
- Feature flags для неготовых функций
- Continuous Integration
- Быстрые итерации
```

---

## 🏷️ Tags (теги)

```bash
# Lightweight tag
git tag v1.0.0

# Annotated tag (рекомендуется)
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag для старого commit
git tag -a v0.9.0 abc123

# Список тегов
git tag
git tag -l "v1.*"

# Информация о теге
git show v1.0.0

# Push тег
git push origin v1.0.0
git push origin --tags  # все теги

# Удалить тег
git tag -d v1.0.0
git push origin --delete v1.0.0

# Checkout тега (detached HEAD)
git checkout v1.0.0
```

---

## 🌐 Remote (удалённые репозитории)

### Основы

```bash
# Список remote
git remote
git remote -v  # с URL

# Добавить remote
git remote add origin https://github.com/user/repo.git

# Изменить URL
git remote set-url origin git@github.com:user/repo.git

# Удалить remote
git remote remove origin

# Информация о remote
git remote show origin
```

### Fetch, Pull, Push

```bash
# Fetch - скачать изменения без merge
git fetch origin
git fetch origin main

# Pull - fetch + merge
git pull origin main

# Pull с rebase
git pull --rebase origin main

# Push
git push origin main
git push origin feature-login

# Push all branches
git push origin --all

# Force push (опасно!)
git push --force origin feature-login
git push --force-with-lease origin feature-login  # безопаснее

# Upstream tracking (связать local ветку с remote)
git push -u origin feature-login
# Теперь можно просто:
git push
git pull
```

### Upstream и Origin

```bash
# Fork workflow:
# origin - ваш fork
# upstream - оригинальный репозиторий

git remote add upstream https://github.com/original/repo.git

# Синхронизация с upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## 🔍 Stash (отложить изменения)

```bash
# Сохранить изменения
git stash
git stash save "WIP: working on feature"

# Список stash
git stash list

# Применить последний stash
git stash apply
git stash pop  # apply + drop

# Применить конкретный stash
git stash apply stash@{2}

# Удалить stash
git stash drop stash@{0}

# Очистить все stash
git stash clear

# Stash с untracked files
git stash -u

# Создать ветку из stash
git stash branch feature-login stash@{0}
```

---

## 🔍 Продвинутые команды

### Cherry-pick

**Применить конкретный commit из другой ветки.**

```bash
# Взять commit abc123 из другой ветки
git cherry-pick abc123

# Несколько commits
git cherry-pick abc123 def456

# Range
git cherry-pick abc123..def456

# Только применить без commit
git cherry-pick -n abc123
```

### Reflog

**История всех действий (включая удалённые commits).**

```bash
# Показать reflog
git reflog

# Восстановить удалённый commit
git reflog
# найти SHA commit
git checkout abc123
git checkout -b recovered-branch

# Восстановить удалённую ветку
git reflog
git checkout -b recovered-feature abc123
```

### Bisect

**Бинарный поиск проблемного commit.**

```bash
# Начать bisect
git bisect start
git bisect bad  # текущий commit плохой
git bisect good abc123  # этот commit был хороший

# Git переключает на средний commit
# Тестируем...
git bisect good  # или git bisect bad

# Повторяем пока не найдём проблемный commit

# Закончить bisect
git bisect reset
```

### Worktree

**Несколько рабочих директорий для одного репозитория.**

```bash
# Создать worktree
git worktree add ../feature-hotfix hotfix/critical-bug

# Список worktrees
git worktree list

# Удалить worktree
git worktree remove ../feature-hotfix
```

---

## 🛠️ .gitignore

```bash
# .gitignore

# Vendor
/vendor
/node_modules

# Environment
.env
.env.*
!.env.example

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Laravel
/storage/*.key
/storage/logs/*
/storage/framework/cache/*
/storage/framework/sessions/*
/storage/framework/views/*
/bootstrap/cache/*

# Testing
/coverage
.phpunit.result.cache

# Compiled assets
/public/hot
/public/storage
/public/css
/public/js
/public/mix-manifest.json

# Logs
*.log
npm-debug.log*
```

**Игнорировать уже отслеживаемый файл:**
```bash
# Удалить из Git но оставить в файловой системе
git rm --cached .env

# Добавить в .gitignore
echo ".env" >> .gitignore
git commit -m "Remove .env from tracking"
```

---

## 🔐 Git Hooks

**Скрипты, выполняющиеся на определённых событиях.**

### Типы hooks

**Client-side:**
- `pre-commit` - перед commit
- `prepare-commit-msg` - перед открытием commit message
- `commit-msg` - проверка commit message
- `post-commit` - после commit
- `pre-push` - перед push

**Server-side:**
- `pre-receive` - перед получением push
- `update` - обновление ветки
- `post-receive` - после получения push

### Пример: pre-commit hook

```bash
#!/bin/sh
# .git/hooks/pre-commit

# PHP linting
files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.php$')
if [ -n "$files" ]; then
    for file in $files; do
        php -l "$file"
        if [ $? -ne 0 ]; then
            echo "PHP syntax error in $file"
            exit 1
        fi
    done
fi

# PHP CS Fixer
vendor/bin/php-cs-fixer fix --dry-run --diff
if [ $? -ne 0 ]; then
    echo "Code style errors found. Run php-cs-fixer fix"
    exit 1
fi

# PHPStan
vendor/bin/phpstan analyse
if [ $? -ne 0 ]; then
    echo "PHPStan found errors"
    exit 1
fi

exit 0
```

**Сделать исполняемым:**
```bash
chmod +x .git/hooks/pre-commit
```

### Husky (для Node.js проектов)

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "vendor/bin/php-cs-fixer fix && git add .",
      "pre-push": "vendor/bin/phpunit"
    }
  }
}
```

---

## 🔄 Resolving Conflicts

### Пример конфликта

```php
// file.php
<?php

<<<<<<< HEAD
function getUser($id) {
    return User::find($id);
}
=======
function getUser(int $id): ?User {
    return User::findOrFail($id);
}
>>>>>>> feature-branch
```

**Разрешение:**
```bash
# 1. Открыть файл и выбрать нужную версию
# 2. Удалить маркеры <<<<, ====, >>>>
# 3. Добавить разрешённый файл
git add file.php

# 4. Продолжить merge/rebase
git commit  # для merge
git rebase --continue  # для rebase
```

**Инструменты:**
```bash
# Использовать их версию
git checkout --theirs file.php

# Использовать нашу версию
git checkout --ours file.php

# Merge tool
git mergetool
```

---

## 📊 Laravel + Git Best Practices

### Структура commits

```bash
# Semantic commits
feat: Add user registration
fix: Resolve login redirect issue
refactor: Extract user service
docs: Update API documentation
test: Add user authentication tests
style: Fix code formatting
perf: Optimize database queries
chore: Update dependencies

# С областью
feat(auth): Add two-factor authentication
fix(api): Handle null response in user endpoint
```

### .gitattributes для Laravel

```bash
# .gitattributes

# Auto detect text files and normalize line endings
* text=auto

# PHP
*.php text eol=lf

# JavaScript
*.js text eol=lf
*.json text eol=lf

# Styles
*.css text eol=lf
*.scss text eol=lf

# Documentation
*.md text eol=lf

# Exclude from exports
.gitattributes export-ignore
.gitignore export-ignore
.github/ export-ignore
tests/ export-ignore
.phpunit.* export-ignore
```

### Workflow для Laravel

```bash
# 1. Новая feature
git checkout main
git pull origin main
git checkout -b feature/user-notifications

# 2. Работа
php artisan make:controller NotificationController
git add app/Http/Controllers/NotificationController.php
git commit -m "feat: Add notification controller"

# 3. Тесты
php artisan make:test NotificationTest
git add tests/Feature/NotificationTest.php
git commit -m "test: Add notification tests"

# 4. Перед push - rebase на актуальный main
git fetch origin
git rebase origin/main

# 5. Если конфликты - разрешить
git add .
git rebase --continue

# 6. Push
git push origin feature/user-notifications

# 7. Pull Request на GitHub

# 8. После merge - удалить локальную ветку
git checkout main
git pull origin main
git branch -d feature/user-notifications
```

---

## 🎓 Для собеседования: ключевые точки

1. **Git vs SVN** - Git распределённая (каждый клон = полная копия), SVN централизованная (один сервер)
2. **Три состояния** - Working Directory (modified) → Staging Area (staged) → Repository (committed)
3. **Merge vs Rebase** - Merge сохраняет историю создаёт merge commit, Rebase переписывает историю линейно, НИКОГДА не rebase public ветки
4. **Git Flow** - main (production), develop (integration), feature/*, release/*, hotfix/*
5. **GitHub Flow** - упрощённая: ветка от main → PR → code review → merge → deploy
6. **Fast-forward vs Three-way merge** - FF если нет расхождений (просто перемещение указателя), 3-way если commits в обеих ветках
7. **Rebase interactive** - `git rebase -i HEAD~3` редактировать историю (pick/reword/squash/fixup/drop)
8. **Cherry-pick** - взять конкретный commit из другой ветки `git cherry-pick abc123`
9. **Reflog** - история всех действий включая удалённые commits, восстановление через `git reflog`
10. **Hooks** - pre-commit (linting/tests перед commit), pre-push (тесты перед push), автоматизация проверок

**Главное:** Понимай разницу merge vs rebase (когда использовать), три состояния файлов, стратегии ветвления (Git Flow vs GitHub Flow), никогда не rebase public ветки, semantic commits для читаемой истории.
