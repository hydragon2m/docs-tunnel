# Hướng dẫn chạy Documentation Site

Documentation site này được xây dựng với [Docsify](https://docsify.js.org/).

## Option 1: Docsify CLI (Recommended)

### Cài đặt Docsify CLI

```bash
npm install -g docsify-cli
```

### Chạy local server

```bash
# Từ thư mục gốc của project
cd docs
docsify serve

# Hoặc từ thư mục gốc
docsify serve docs
```

Mở browser: `http://localhost:3000`

## Option 2: Python HTTP Server

```bash
cd docs
python -m http.server 3000
```

Mở browser: `http://localhost:3000`

## Option 3: Node.js HTTP Server

```bash
# Cài đặt http-server
npm install -g http-server

# Chạy server
cd docs
http-server -p 3000
```

Mở browser: `http://localhost:3000`

## Option 4: Live Server (VS Code)

1. Cài extension "Live Server" trong VS Code
2. Right-click vào `docs/index.html`
3. Chọn "Open with Live Server"

## Cấu trúc thư mục

```
docs/
├── index.html          # Docsify configuration
├── README.md           # Homepage
├── _sidebar.md         # Sidebar navigation
├── _coverpage.md       # Cover page
│
├── introduction.md     # Giới thiệu
├── quickstart.md       # Quick start guide
├── deployment.md       # Deployment guide
├── monitoring.md       # Monitoring guide
├── security.md         # Security guide
├── faq.md              # FAQ
│
└── _media/             # Images, logos
    └── logo.svg
```

## Customization

### Thay đổi theme

Edit `docs/index.html`:

```html
<!-- Themes available: -->
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css">
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/buble.css">
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/dark.css">
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/pure.css">
```

### Thay đổi màu chủ đề

Edit `docs/index.html` trong tag `<style>`:

```css
:root {
  --theme-color: #42b983;  /* Đổi màu này -->
}
```

## Deploy lên Production

### GitHub Pages

1. Push docs folder lên GitHub
2. Settings → Pages → Source: `main branch /docs folder`
3. Access tại: `https://your-username.github.io/go-tunnel/`

### Netlify

```bash
# netlify.toml
[build]
  publish = "docs"
  command = "echo 'No build needed'"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### Vercel

```json
// vercel.json
{
  "routes": [
    { "handle": "filesystem" },
    { "src": "/.*", "dest": "/index.html" }
  ]
}
```

## Thêm pages mới

1. Tạo file `.md` trong `docs/`
2. Thêm link vào `docs/_sidebar.md`
3. Reload browser

Example:

```bash
# Tạo file mới
echo "# My New Page" > docs/my-page.md

# Thêm vào sidebar
echo "  * [My Page](my-page.md)" >> docs/_sidebar.md
```

## Tips

### Search functionality

Search đã được enable. Nhấn `/` để focus vào search box.

### Code highlighting

Supported languages:
- Bash
- Go
- YAML
- JSON
- Docker
- Nginx

Add more trong `index.html`:
```html
<script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-python.min.js"></script>
```

### Mermaid diagrams

Đã support! Dùng:

\`\`\`mermaid
graph LR
    A --> B
\`\`\`

---

**Enjoy your documentation site! 📚**
