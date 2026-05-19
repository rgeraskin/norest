---
name: review-blog-post
description: Review a blog post
argument-hint: ["post-path"]
---

Сделай ревью статьи $ARGUMENTS

1) ВАЖНО: сохрани авторский стиль
2) Исправь опечатки, грамматические ошибки, пунктуацию, тавтологию, прочие косяки
3) Замени дефисы на тире там, где это необходимо
4) Дай стилистические советы по посту
5) Если к посту есть приложенные картинки, убедись, что их размер оптимизирован для веб
6) Сделай краткий summary и помести его в начало статьи под заголовком TL;DR

## Работа с картинками

Картинки лежат в `static/assets/<slug>/`. Hugo раздаёт всё из `static/` от корня сайта, поэтому в markdown путь — `/assets/<slug>/<file>` (с ведущим слешем, без `static/`).

Ширина контента в bearblog-теме — **720px** (`--width` в `themes/hugo-bearblog/layouts/partials/style.html`). Source-картинки шире 720px не дают визуального выигрыша на обычных DPI; на Retina полезен 2× = ~1440px. Для UI-скриншотов часто хватает 500-720px source.

### Инструменты (macOS)

- `sips -g pixelWidth -g pixelHeight -g format <file>` — быстрая инспекция размера/формата
- `cwebp` — кодек WebP (`brew install webp`)
- `magick` (ImageMagick 7) — crop, resize, конвертация форматов

### Формат: WebP lossless для UI-скриншотов

WebP lossless жмёт скриншоты на **40-60%** против PNG без потери качества. Поддержка: Safari 14+ (2020), все остальные браузеры с 2018-19. Для IT-блога — без вопросов.

```bash
cwebp -lossless -quiet input.png -o output.webp
```

Lossy WebP (q85-q90) даёт **−70%**, но на тонком тексте появляются микроартефакты. Для скриншотов с подписями полей — берём lossless.

### Crop (срезать пиксели по краю)

```bash
magick input.webp -crop WIDTHxHEIGHT+X+Y +repage /tmp/cropped.png
cwebp -lossless -quiet /tmp/cropped.png -o output.webp
```

`+repage` обязателен — без него WebP/PNG сохранит исходные размеры в virtual canvas, и некоторые viewers будут рендерить пустую область вокруг.

### Resize

```bash
magick input.webp -filter Lanczos -resize 600x /tmp/resized.png
cwebp -lossless -quiet /tmp/resized.png -o output.webp
```

- **Filter Lanczos** для скриншотов с текстом — лучше сохраняет резкость краёв чем Mitchell/bicubic
- **Всегда ресайзить от source с максимальным разрешением**, не от уже уменьшенной версии. Каскадный resize (1000 → 600 → 500) добавляет потери на каждом шаге, Lanczos их не восстанавливает
- **Через промежуточный PNG**: lossless WebP → PNG → magick → cwebp lossless WebP. Прямая webp→webp перекодировка через magick может терять резкость на entropy coding'е

### Бэкап перед деструктивной правкой

Всегда `cp <file> /tmp/<backup>.webp` перед crop/resize — оригинал в `static/` обычно не закоммичен, восстановить будет неоткуда.

### Алгоритм проверки картинок при ревью

1. Прогнать `sips -g pixelWidth -g pixelHeight` — оценить размер
2. Если source шире 720px и не очень нужен Retina-resolution — ресайзить до ~600-720px
3. Если формат PNG — конвертнуть в WebP lossless
4. Если в markdown расширение `.png`, после конвертации не забыть поменять на `.webp`