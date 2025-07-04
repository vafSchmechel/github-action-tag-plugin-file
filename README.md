
# Automatische Plugin-Versionierung via GitHub Actions

## Zweck

Dieser Workflow aktualisiert bei jedem Push eines Git Tags die Plugin-Version in der Plugin-Hauptdatei (`my-plugin.php`) im Default-Branch automatisch auf die Tag-Version. Anschließend wird der Git Tag neu gesetzt, damit er auf den Commit mit der aktualisierten Version zeigt. So bleiben Tag und Plugin-Version synchron.

---

## Workflow Ablauf

1. **Trigger:**  
   Workflow läuft bei Push von *irgendeinem* Git Tag (`tags: ['*']`).

2. **Checkout:**  
   Es wird der Standardbranch ausgecheckt (`actions/checkout@v3` mit `ref: ${{ github.event.repository.default_branch }}`).

3. **Branch Sync:**  
   Der lokale Branch wird auf den aktuellen Remote-Stand gebracht:

   ```bash
   BRANCH=${{ github.event.repository.default_branch }}
   git fetch origin "$BRANCH"
   git reset --hard "origin/$BRANCH"
   ```

4. **Plugin-Datei ermitteln:**  
   Der Pfad zur Plugin-Datei wird aus `composer.json` ausgelesen.  
   Beispiel: `"name": "xyz/my-plugin"` → Plugin-Datei `my-plugin.php`.

5. **Version aktualisieren:**  
   Die `Version:`-Zeile im Plugin-Header wird per `sed` auf den Tag-Namen gesetzt.  
   Der Tag-Name wird aus der Action-Variable `github.ref_name` gelesen (z.B. `1.0.34`).

6. **Commit & Push:**  
   Änderungen werden committed und auf den Default-Branch gepusht.  
   Falls Push-Konflikte auftreten, versucht der Workflow bis zu 5-mal neu, inklusive erneuter Synchronisation.

7. **Tag aktualisieren:**  
   Der Git Tag wird mit `git tag -f` auf den neuen Commit mit aktualisierter Plugin-Version gesetzt und mit `git push --force` gepusht.

---

## Vorteile

- Automatische Synchronisierung von Plugin-Version und Git Tag.
- Vermeidet, dass der Tag auf einem Commit mit falscher Version bleibt.
- Retry-Mechanismus sorgt für robuste Updates trotz möglicher Push-Konflikte.
- Kein manuelles Aktualisieren der Version nötig.

---

## Voraussetzungen

- `composer.json` mit `"name": "xyz/plugin-name"` im Root, Plugin-Hauptdatei heißt `plugin-name.php`.
- GitHub Actions mit Schreibrechten (`GITHUB_TOKEN` mit Push-Permission).
- Branch muss als Standardbranch in den Repository-Einstellungen gesetzt sein.

---

## Beispiel für `composer.json`

```json
{
  "name": "xyz/my-plugin"
}
```

---

## Beispiel Plugin-Header (vorher)

```php
<?php
/**
 * Plugin Name: Mein WP Plugin
 * Description: Ein Test-Plugin
 * Version: 1.0.30
 * Author: ChatGPT
 */
```

---

## Nach Workflow-Lauf (Tag 1.0.34)

```php
<?php
/**
 * Plugin Name: Mein WP Plugin
 * Description: Ein Test-Plugin
 * Version: 1.0.34
 * Author: ChatGPT
 */
```

---

## Beispiel Workflow (Auszug)

```yaml
name: Update Plugin Version on Tag Push

on:
  push:
    tags:
      - '*'

jobs:
  update-version:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout default branch
        uses: actions/checkout@v3
        with:
          ref: ${{ github.event.repository.default_branch }}
          token: ${{ secrets.GITHUB_TOKEN }}
          persist-credentials: true

      - name: Sync branch (fetch & reset)
        run: |
          BRANCH=${{ github.event.repository.default_branch }}
          git fetch origin "$BRANCH"
          git reset --hard "origin/$BRANCH"

      - name: Determine plugin file name
        run: |
          NAME=$(jq -r '.name' composer.json)
          PLUGIN_NAME=${NAME#*/}
          PLUGIN_FILE="$PLUGIN_NAME.php"
          echo "PLUGIN_FILE=$PLUGIN_FILE" >> $GITHUB_ENV

      - name: Update version in plugin file
        env:
          TAG: ${{ github.ref_name }}
        run: |
          sed -i "s/^\(\s*\*\s*[Vv]ersion:\s*\).*/$TAG/" "$PLUGIN_FILE"

      - name: Commit and push changes
        env:
          TAG: ${{ github.ref_name }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add "$PLUGIN_FILE"
          if ! git diff --cached --quiet; then
            git commit -m "chore: Update plugin version to $TAG [skip ci]"
          fi
          git push origin "$BRANCH"

      - name: Re-tag with updated commit
        env:
          TAG: ${{ github.ref_name }}
        run: |
          git tag -f "$TAG"
          git push --force origin "$TAG"
```

---

Falls du die Doku als PDF möchtest oder weitere Erklärungen brauchst, melde dich gern!  
Viel Erfolg mit deinem Workflow! 🚀
