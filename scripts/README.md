# Auxiliary scripts

## `to_latex.py`

The `to_latex.py` script transforms the wiki into a basic LaTeX document, using [Pandoc](https://pandoc.org/).

To use it, run (in the project's root folder):
```
bun tex OUT_DIR
```

It will output a `main.tex` file as well as other folders.

As the text contains Unicode, in its current form it won't compile unless using [XeTeX](https://xetex.sourceforge.net/)/[LuaTeX](https://www.luatex.org/).

Things that need to be cleaned up in the resulting LaTeX files:
- [ ] Alerts (`!NOTE` / `!INFO` / `!TIP`)
- [ ] Multi-line code blocks
- [ ] Variables (`{{book.appUrl}}` / `{{book.repo}}` / `{{book.webpageUrl}}`)
- [ ] Footnotes (`[^1]`)
- [ ] Enumerated / non-enumerated lists
- [ ] Emojis (Unicode)
- [ ] links to `whatever/stuff.md`
- [ ] Figures (`\pandocbounded{\includegraphics[keepaspectratio]{...}`)
    - [ ] Figure captions