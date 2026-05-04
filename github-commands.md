# GitHub komande za upload

Pretpostavka: nalaziš se u root folderu lokalnog GitHub repo-a, npr. `DocCentralV3`.

## 1. Raspakuj ZIP

Raspakuj sadržaj ZIP fajla direktno u root repo folder.

## 2. Proveri status

```bash
git status
```

## 3. Dodaj sve fajlove

```bash
git add README.md \
  claude-code/ \
  templates/ \
  checklists/ \
  skills/ \
  data-model/ \
  business/ \
  architecture/ \
  power-apps/ \
  power-automate/ \
  security/ \
  testing/ \
  deployment/ \
  operations/ \
  prompts/ \
  github-commands.md
```

Ako želiš jednostavnije:

```bash
git add .
```

## 4. Commit

```bash
git commit -m "Update DocCentral V3 documentation and Claude Code skills"
```

## 5. Push

```bash
git push origin main
```

Ako ti je branch `master`:

```bash
git push origin master
```

## 6. Ako git prijavi obrisane fajlove

Prvo proveri:

```bash
git status
```

Ako su fajlovi namerno zamenjeni novima, koristi:

```bash
git add -A
git commit -m "Refresh DocCentral V3 documentation package"
git push origin main
```

Ako brisanje nije namerno, vrati fajlove iz git-a:

```bash
git restore <putanja-do-fajla>
```
