# joneo-brain

> **Ce README est écrit pour les agents, pas pour les humains.**
> Si tu es un agent IA (rédacteur en chef, agent Watch, agent de déclinaison, instance Claude), lis ce fichier en premier : il définit ce que contient ce dépôt, dans quel ordre le lire, et les conventions à respecter en écriture.

## Rôle du dépôt

`joneo-brain` est la **base de connaissance canonique de JONEO**, média IA francophone porté par deux avatars : **JO** (animateur podcast) et **NEO** (consultant stratégie IA). Ce dépôt est la source de vérité versionnée. Notion sert de tampon opérationnel (pipelines Make), Mintlify de couche de publication pour NEO. En cas de conflit entre une donnée de ce repo et une donnée externe, **le repo fait foi**.

## Ordre de lecture pour un agent

1. `charte/` — l'identité éditoriale : ton, positionnement, interdits. À charger à chaque run.
2. `decisions/` — le journal des décisions structurantes (ADR). Ne jamais contredire une décision active sans la référencer.
3. `catalogue/` — l'existant : podcasts et contenus publiés. Sert au contrôle anti-redondance.
4. `strategie/` — vision, priorités du trimestre, positionnement marché.
5. Le dossier spécifique à ta mission (`veille/`, `contenus/`, `agents/`).

## Arborescence

| Dossier | Contenu | Qui écrit |
|---|---|---|
| `charte/` | Charte éditoriale, ton, interdits, personas JO/NEO | Alain (humain) |
| `decisions/` | Décisions structurantes, une par fichier, format ADR | Alain + Claude |
| `catalogue/` | Catalogue des podcasts et contenus publiés | Pipelines Make |
| `agents/configs/` | Instructions canoniques de chaque agent (Watch, rédac chef, etc.) | Alain + Claude |
| `agents/declinaisons/` | Règles de déclinaison multicanal (LinkedIn, Instagram, vidéo, audio) | Alain + Claude |
| `veille/` | Sorties JSON des agents Watch, classées par date `AAAA-MM-JJ/` | Agents Watch (via Make) |
| `contenus/propositions/` | Propositions d'épisodes/articles émises par le rédacteur en chef | Agent rédac chef |
| `contenus/episodes/` | Épisodes validés et produits (frontmatter YAML riche) | Pipelines Make |
| `contenus/articles/` | Articles blog validés et produits | Pipelines Make |
| `transcripts/` | Transcripts de réunions et d'enregistrements | Fathom → Make |
| `strategie/` | Vision, roadmap, pricing, positionnement | Alain (humain) |
| `livre/` | Matériau du livre *Jeune par l'Algo* | Alain + Claude |

## Conventions d'écriture

- **Format** : Markdown avec frontmatter YAML. Un objet de contenu = un fichier.
- **Nommage** : `kebab-case`, préfixe date `AAAA-MM-JJ-` pour tout ce qui est daté.
- **Langue** : français. Les clés YAML restent en anglais (`title`, `status`, `family`...).
- **Un contenu, un fichier source** : les déclinaisons (LinkedIn, Instagram, vidéo, audio) dérivent toutes du frontmatter de l'objet source, jamais l'inverse.
- **Statuts** : `proposition` → `validé` → `produit` → `publié`. Le passage `proposition → validé` est **exclusivement humain** (Alain, via Notion).
- Ne jamais supprimer un fichier : passer `status: archivé` dans le frontmatter.

## Principes d'architecture (rappels)

- **Un seul rédacteur en chef** — pas un par thème — pour garantir la cohérence de la vision.
- Les agents Watch font du **jugement éditorial** (anti-redondance, sélection qualitative, génération d'angles), pas de la collecte déterministe.
- Chaîne éditoriale : veille → décision éditoriale → validation humaine → production → déclinaison multicanal.
