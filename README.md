# uictorius.github.io

**Personal blog of [uictorius](https://github.com/uictorius)** — built with [Hugo](https://gohugo.io/) and themed with [PaperMod](https://github.com/adityatelange/hugo-PaperMod).

---

## 📰 About

This repository hosts the source code for my personal tech blog, where I share notes, experiments, and reverse-engineering insights related to software, hardware, and embedded systems.  

The site is generated using the **Hugo static site generator** and styled with the **PaperMod** theme.

---

## ⚙️ Requirements

- [Hugo](https://github.com/gohugoio/hugo/releases)
- [Go](https://go.dev/dl/)

Install Go on Debian/Ubuntu:

```bash
sudo apt update
sudo apt install golang-go -y
```

---

## 🚀 Getting Started

Clone this repository and initialize the theme submodule:

```bash
git clone https://github.com/uictorius/uictorius.github.io.git
cd uictorius.github.io
git submodule update --init --recursive
```

Run the local development server:

```bash
hugo server -D
```

Then open your browser and navigate to:

👉 [http://localhost:1313](http://localhost:1313)

---

## 🧩 Notes

- The blog uses **Hugo modules** and Git submodules for theme management.
- To update the theme, simply run:

```bash
git submodule update --remote --merge
```

---

## 📜 License

This project is licensed under the [MIT License](LICENSE).
