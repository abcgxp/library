# 📚 ABCGXP's Library

🔗 **Live site:** https://abcgxp.github.io/library/

This repository powers my personal knowledge library — a living notebook where I keep:

- 📖 Study notes
- 🧰 Work notes & runbooks
- 🧪 Experiments, labs, and references
- 🧠 Things I *will* forget if I don’t write them down

It’s optimized for fast lookup, practical examples, and future-me sanity.

---

## 🗂️ Where the Content Lives

All actual notes live here:

```
source/content/
```

This folder is edited directly using **Obsidian** or any text editor.  
Each Markdown file becomes a page on the site.

Typical organization:
- Topic-based folders (ex: `kubernetes/`, `linux/`, `networking/`)
- Short, focused notes
- Commands + explanations > essays

---

## 🧱 How the Site Is Built

- Static site generated using **Quartz**
- Deployed automatically via **GitHub Actions**
- Hosted on **GitHub Pages**

The build process converts the Markdown notes into a searchable HTML site.

---

## 🧪 Local Development (Optional)

To preview the site locally:

```bash
cd source
npx quartz build --serve
```

This starts a local dev server so you can preview changes before pushing.

---

## 🧩 Raw HTML Pages

There is a special folder for hosting **standalone HTML pages**:

```
source/raw_html/
```

Anything placed here is copied directly to the deployed site *outside* of Quartz.

Use cases:
- Custom HTML tools
- Generated UIs
- Archived web pages
- One-off experiments

Example URL pattern:
```
/raw-html-page-name.html
```

---

## ⚙️ Customization & Configuration

Quartz is highly configurable. Key files:

- **`quartz.config.ts`** – site metadata, plugins, behavior
- **`quartz.layout.ts`** – layout & page structure

Official Quartz docs:
👉 https://quartz.jzhao.xyz/configuration

---

## 📝 Notes

This repo started from [a Quartz-based template](https://github.com/DefenderOfBasic/obsidian-quartz-template), but it has since been adapted
to serve as a personal, evolving knowledge base.

If you’re reading this from the future:  
yes, this is why things are organized the way they are 😄
