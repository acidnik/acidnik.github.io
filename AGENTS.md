# AGENTS.md

## Project: acidnik.github.io

### Structure

```
content/
├── _index.md                  # Главная — приветствие + список последних постов
├── blog/
│   ├── _index.md              # Список всех статей блога (cascade: type: blog)
│   └── llm.md                 # Статья
layouts/
└── shortcodes/
    └── recent-posts.html      # Shortcode для списка последних статей на главной
```

### Blog Section

- Статьи хранятся в `content/blog/` — стандарт Hextra
- `content/blog/_index.md` использует `cascade: { type: blog }` для применения blog-шаблона Hextra

### Homepage

- `content/_index.md` содержит приветствие и вызов `{{< recent-posts max=5 >}}`
- Shortcode `recent-posts` выводит список последних статей с датой и ссылкой, внизу «View all posts →»

### Shortcode: `recent-posts`

**Файл:** `layouts/shortcodes/recent-posts.html`

**Параметры:**
| Параметр | Тип | По умолчанию | Описание |
|----------|------|--------|-----------|
| `max` | int | 5 | Количество постов |
| `section` | string | "blog" | Раздел для выборки |

**Пример:**
```markdown
{{< recent-posts max=3 >}}
```

### hugo.toml

```toml
[menu]
  [[menu.main]]
    name = "Blog"
    pageRef = "/blog"
    weight = 1
  [[menu.main]]
    name = "Search"
    weight = 2
    [menu.main.params]
      type = "search"

[params.blog.list]
  displayTags = true
  sortBy = "date"
  sortOrder = "desc"
  pagerSize = 10

[params.theme]
  default = "system"
  displayToggle = true
```
