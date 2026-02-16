# Flipper Documentation

## Quick Start

```bash
npm install
npm start      # Development server on http://localhost:3000
npm run build  # Production build
```

## Documentation Structure

The docs are located in `/docs`. The structure is auto-generated in the sidebar.

```
docs/
├── intro.md              # Landing page (displays at /)
├── use-cases/
├── uml/
├── frontend/
├── backend/
│   └── iot/
└── infra/
```

## Adding New Content

### Create a New Folder
Simply create a folder in `/docs`. It will automatically appear in the sidebar.

**Optional**: Add `_category_.json` to customize:
```json
{
  "label": "My Section",
  "position": 2,
  "link": {
    "type": "generated-index",
    "description": "Section description"
  }
}
```

### Create a New Page
Create a `.md` file with frontmatter:
```markdown
---
sidebar_position: 1
title: My Page Title
---

# My Page Title
Your content here...
```

## Markdown Features

- **Headings**: `# H1`, `## H2`, `### H3`
- **Code blocks**: Use triple backticks with language
- **Images**: `![alt text](./images/screenshot.png)`
- **Links**: `[Link text](/docs/other-page)`
- **Admonitions**: `:::note`, `:::tip`, `:::warning`, `:::danger`

Example:
```markdown
:::tip Pro Tip
This is a helpful tip!
:::
```

Visit [Docusaurus docs](https://docusaurus.io/docs/markdown-features) for more features.
