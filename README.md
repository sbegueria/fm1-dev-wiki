# FM-1 Firmware Developers' Wiki (source)

The pages of an unofficial community wiki for M-VAVE FM-1 / SMK-37 firmware developers, written as a
GitHub Wiki: flat `Page-Name.md` files, `Home.md`, `_Sidebar.md`, `_Footer.md`, links as `[[Page Name]]`.

This README and `REPORT.txt` are not wiki pages: don't copy them into the wiki.

## Publishing it as a GitHub Wiki

1. Create (or pick) a repository on GitHub, e.g. `fm1-dev-wiki`. In its Settings, make sure **Wikis** is enabled.
2. Create the wiki's first page in the browser (any text). GitHub only creates the wiki's git repository then.
3. Clone the wiki repository, copy the pages in, and push:

   ```sh
   git clone https://github.com/<owner>/<repo>.wiki.git
   cd <repo>.wiki
   rsync -av --exclude README.md --exclude REPORT.txt --exclude .git /path/to/fm1-wiki/ ./
   git add -A
   git commit -m "Initial pages"
   git push
   ```

4. Check Home, the sidebar and a few `[[links]]` in the browser.

GitHub turns `[[Page Name]]` into a link to `Page-Name`. File names use hyphens for spaces.

## Alternative: pages in a normal repository

If you'd rather take pull requests with review (wikis can't), keep these files in a normal repository's `docs/`
folder and convert `[[Page Name]]` to `[Page Name](Page-Name.md)` links:

```sh
python3 -c 'import re,glob
for f in glob.glob("*.md"):
    s=open(f).read()
    s=re.sub(r"\[\[([^\]|]+)\]\]", lambda m: "[%s](%s.md)" % (m[1], m[1].replace(" ", "-")), s)
    open(f,"w").write(s)'
```

## Licence

Text: CC BY-SA 4.0. Code excerpts keep their own licences. See `License.md`.
