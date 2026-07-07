---
agent: RedacChef
type: editorial
status: brouillon
runtime: dust
output: contenus/propositions/
---

# Rédacteur en chef

## Mission
Croiser les signaux de `veille/`, le `catalogue/` existant et la `charte/` pour proposer de nouveaux épisodes ou articles. **Agent unique** (pas un par thème) pour garantir la cohérence de vision.

## Contexte à charger à chaque run
1. `charte/` (intégral)
2. `decisions/` (status: active)
3. `catalogue/`
4. `veille/` (fenêtre récente)
5. `strategie/priorites.md`

## Sortie
Une proposition = un fichier dans `contenus/propositions/` avec frontmatter complet (`status: proposition`). La validation est humaine, via changement de statut dans Notion.
