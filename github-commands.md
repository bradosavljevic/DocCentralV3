# GitHub komande za upload ažurirane dokumentacije

Pretpostavka: ZIP je raspakovan u root folder lokalnog repository-ja `DocCentralV3`.

```bash
git status

git add README.md   configuration/power-platform-object-names.md   configuration/environment-variables.md   deployment/deployment-model.md   power-automate/flow-inventory.md   power-automate/register-document-flow.md   prompts/master-claude-code-prompt.md   templates/claude-code-development-brief.md   claude-code/README.md   skills/*.md

git status

git commit -m "Update DocCentral V3 documentation with created solution object names"

git push
```

Ako želiš da dodaš sve fajlove iz paketa odjednom:

```bash
git add .
git commit -m "Update DocCentral V3 documentation with created object names"
git push
```
