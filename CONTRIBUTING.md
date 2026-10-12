# Внесення змін

Створюйте окрему гілку для кожної зміни:

```bash
git checkout -b content/module-06-permissions
```

Перед commit:

```bash
mkdocs build --strict
```

Потім:

```bash
git add .
git commit -m "Add module 06 permissions"
git push -u origin content/module-06-permissions
```

Зміни об'єднуються до `main` через Pull Request.
