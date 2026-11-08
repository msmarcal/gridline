# Website

Source for [gridline-desktop.org](https://gridline-desktop.org): two pages of
static HTML and one stylesheet. No build step, no JavaScript, no external
fonts or requests; the pages are deployed by copying this directory to the
web server as-is.

| File | Serves |
| --- | --- |
| `index.html` | `/` |
| `install.html` | `/install` |
| `style.css` | stylesheet, light and dark via `prefers-color-scheme` |
