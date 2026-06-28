# AGENTS.md — Правила синхронизации изменений

> Этот файл описывает места, которые **обязательно** нужно обновить при добавлении, перемещении или удалении разделов/статей, а также при изменении версии проекта.

---

## 📁 Структура нумерации

| Диапазон | Группа | Назначение |
|----------|--------|------------|
| `001`–`099` | Программирование | .NET, Web, БД, архитектура, языки, фреймворки |
| `101`–`199` | Развитие и инфраструктура | Стартапы, здоровье, DevOps, софт-скиллы |
| `201`–`299` | Наука и AI | Наука XX-XXI века, AI/ML |
| `301`–`399` | Гуманитарные науки | История, литература, мифы, религии, психология |

Номер раздела — трёхзначный (`019`, `021`, `104`).
Директория на диске: `{NNN}-{kebab-case-name}/`.
В `id` JS-данных — строка без ведущих нулей (`"19"`, `"21"`, `"104"`).

---

## 🔄 При добавлении/перемещении/удалении раздела

| # | Где обновить | Что менять |
|---|-------------|------------|
| 1 | `[README.md](README.md)` — навигация (группы) | Строки с секциями в `<details>` для соответствующей группы |
| 2 | `[README.md](README.md)` — полный список разделов | Добавить/убрать секцию `### [...]` со списком статей |
| 3 | `[README.md](README.md)` — таблица Итого | Строка в секции `## 🎯 Итого` |
| 4 | `[README.md](README.md)` — счётчик разделов | Строка `**N раздела, 200+ статей**` и блок-цитата в шапке |
| 5 | `[index.html](index.html)` — JS-данные (sections) | Массив `sections` внутри группы; проверить `id`, `dir`, `title`, `articles` |
| 6 | `[index.html](index.html)` — статистика на welcome screen | Цифры `stat-num` (HTML + JS-шаблон `showWelcomeScreen`) |
| 7 | `[index.html](index.html)` — meta description | Строка 7: `<meta name="description" content="...N раздела...">` |
| 8 | Физически: `mv` директорию | Переименовать папку раздела |

### Пример: перенос раздела (104 → 021)

```
README.md:
  - навигация: 001–018 → добавить 021, убрать 104 в старой группе
  - полный список: 021 между 018 и 101
  - итого: добавить строку 021, обновить 104
  - счётчик: 42→43

index.html:
  - sections: добавить 021 в programming group, заменить 104
  - stats: проверить stat-num
  - meta description: обновить число разделов
```

---

## 📝 При добавлении/удалении статьи внутри раздела

| # | Где обновить | Что менять |
|---|-------------|------------|
| 1 | `[README.md](README.md)` — полный список раздела | Добавить/убрать строку в таблице статей |
| 2 | `[index.html](index.html)` — JS-данные | Добавить/убрать `{ num, title, file }` в массиве `articles` |
| 3 | `[README.md](README.md)` — навигация (группы) | Обновить число статей в колонке «Статей» |
| 4 | `[README.md](README.md)` — таблица Итого | Обновить число статей и описание |
| 5 | `README.md` раздела (например `021-avalonia-xaml-mobile/README.md`) | Добавить строку в таблицу содержимого |

---

## 🔖 При изменении версии

Версия указывается в формате `v{major}.{minor}`.

| # | Где обновить | Что менять |
|---|-------------|------------|
| 1 | `[README.md](README.md:1)` | Суффикс `<sup>vX.Y</sup>` в заголовке |
| 2 | `[index.html](index.html)` — sidebar header | `<span class="version">vX.Y</span>` после названия |
| 3 | `[index.html](index.html)` — breadcrumb (HTML) | `<span>vX.Y</span>` в breadcrumb |
| 4 | `[index.html](index.html)` — `loadArticle()` (JS) | `breadcrumb.textContent = '🏠 DensePedia vX.Y'` |
| 5 | `[index.html](index.html)` — `showWelcomeScreen()` (JS) | `breadcrumb.textContent = '🏠 DensePedia vX.Y'` |

---

## 🔍 Поиск и проверка

Перед коммитом выполнить:

```bash
# 1. Проверить, что нет битых ссылок на несуществующие директории
ls -d */ | sort > /tmp/dirs_on_disk.txt
grep -oP 'dir: "\K[^"]+' index.html | sort > /tmp/dirs_in_html.txt
diff /tmp/dirs_on_disk.txt /tmp/dirs_in_html.txt

# 2. Проверить, что нет китайских символов в .md
python3 -c "
import os, re
cjk = re.compile(r'[\u4e00-\u9fff\u3400-\u4dbf\uf900-\ufaff]')
for r, _, fs in os.walk('.'):
    for f in fs:
        if f.endswith('.md'):
            with open(os.path.join(r,f), encoding='utf-8') as fp:
                for i, line in enumerate(fp, 1):
                    if cjk.search(line):
                        print(f'{os.path.join(r,f)}:{i}: {line.strip()[:80]}')
"

# 3. Убедиться, что число разделов в README и index.html совпадает
grep -c 'id: "'[0-9] index.html
grep -c '| \[' README.md  # приблизительно
```

---

## 📋 Чек-лист перед коммитом

- [ ] README.md: навигация, полный список, итого, счётчик
- [ ] index.html: JS-данные (sections), stats, meta description
- [ ] index.html: breadcrumb version (`textContent` x2 + HTML)
- [ ] README.md: версия в заголовке
- [ ] Все ссылки на файлы (`dir`, `file`) ведут в существующие пути
- [ ] Нет китайских символов в .md
- [ ] Число разделов совпадает в README и index.html
