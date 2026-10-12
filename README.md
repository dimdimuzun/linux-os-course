# linux-os-course

Навчальний курс **«Операційні системи — Основи Linux»**.

Репозиторій є єдиним джерелом актуальних лекційних матеріалів курсу. Статичний сайт будується за допомогою **MkDocs + Material for MkDocs** та публікується через **GitHub Pages**.

## Структура курсу

- 16 навчальних модулів
- 8 лабораторних робіт
- 2 підсумкові квізи
- довідник команд та основних понять
- глосарій
- додаткові ресурси

## Архітектура

```text
GitHub repository → GitHub Actions → GitHub Pages → навчальний сайт
                         │
                         └── CI: mkdocs build --strict

Moodle → лабораторні роботи, квізи та оцінювання
```

## Локальний запуск

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve --livereload
```

Відкрити `http://127.0.0.1:8000/`.

Перевірка production-збірки:

```bash
mkdocs build --strict
```

## GitHub Pages

Перед першим push замініть `USERNAME` у `mkdocs.yml` на ваш GitHub username.

У GitHub відкрийте **Settings → Pages → Build and deployment → Source → GitHub Actions**.

Після merge до `main` workflow `deploy.yml` побудує та опублікує сайт.

## Робочий процес

```text
branch → Markdown → mkdocs serve → mkdocs build --strict → commit/push → PR → CI → merge → GitHub Pages
```

## Модель навчання

GitHub відповідає за зберігання та публікацію навчального контенту. Moodle використовується для виконання лабораторних робіт, квізів та оцінювання.

## Ліцензія

Замініть шаблон `LICENSE` на ліцензію, обрану правовласником матеріалів.
