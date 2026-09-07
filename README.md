**README.md**

# joneo-brain

Les décisions qui structurent JONEO.

## Ce qu'il y a ici

`CLAUDE.md` — le contexte de travail, lu automatiquement par Claude Code. Règles de collaboration et rappel de l'architecture.

`positionnement.md` — ce qu'est JONEO, pour qui, avec quels formats, à quel prix. Fait autorité sur toute question éditoriale, de cible ou de modèle économique.

`architecture.md` — le modèle de données, les statuts, les agents, l'orchestration. Fait autorité sur toute question technique.

## La règle

Ce qui est décidé va ici. Ce qui est produit va dans Supabase.

Un contenu, un signal, un transcript n'ont rien à faire dans ce dépôt. Les prompts d'agents vivent dans les agents eux-mêmes, dans Make ou dans les projets Claude.

Aucun agent n'écrit ici. Seul Alain met à jour ces fichiers, à la main.

## La mise à jour

Le dépôt change quand une décision change. Pas quand un contenu est produit.

Toute conversation qui tranche une question d'architecture ou de positionnement se termine par une mise à jour du fichier concerné. Si ce n'est pas commité, ce n'est pas décidé.

## Les autres dépôts

`joneo-supabase` — le schéma de la base.

`joneo-signal` — le template Signal pour Ghost.

`joneo-docs` — ancienne base de connaissance de l'agent NEO. Conservée, plus utilisée.
