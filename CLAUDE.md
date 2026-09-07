**CLAUDE.md**

# JONEO — Contexte de travail

## Le projet

JONEO est une veille à 360 sur l'IA et ses impacts sur la société. Elle aide à se faire une opinion sur ces impacts.

Alain Fargeon la porte seul. Le projet doit rester réaliste pour un solo, porteur de sens, et compatible avec un équilibre pro et perso. Toute proposition qui augmente la charge sans servir cet objectif est à écarter.

Lire `positionnement.md` avant toute question éditoriale, de format, de cible ou de prix. Il fait autorité.

## L'architecture

Supabase est le dépôt unique de tous les contenus.

L'orchestration se fait par les statuts : chaque acteur lit un état, agit, écrit un nouvel état. Aucun acteur n'en appelle un autre.

Trois natures d'acteurs. Les agents Claude connectés à la base pour l'éditorial. Les agents IA Make pour ce qui tourne sans intervention. Les workflows Make pour l'exécution technique.

Notion, Dust et Bannerbear ont été supprimés. Ne pas les proposer.

## Ce dépôt

Il contient les décisions, pas la production.

Ce qui est décidé va ici. Ce qui est produit va dans Supabase. Un contenu, un signal, un transcript n'ont rien à faire dans ce dépôt.

Les prompts d'agents vivent dans les agents eux-mêmes, dans Make ou dans les projets Claude. Ce dépôt en tient le catalogue, pas le contenu.

Aucun agent n'écrit ici. Seul Alain met à jour ces fichiers, à la main.

## Règles de travail

Ne rien affirmer sans vérifier. Une donnée de marché, une capacité produit, un concurrent : chercher avant de l'écrire. En cas de doute, dire ce qui a été vérifié et ce qui ne l'a pas été.

Se méfier des comparatifs publiés par des acteurs du secteur. Ils se classent toujours premiers.

Ne pas combler un blanc par de la formulation générique. Poser la question à la place.

Pas de phrases qui en font trop, pas de formules creuses, pas de conclusions rhétoriques. Si une phrase ne dit rien de vérifiable, la supprimer.

Ne pas reformuler ce qu'Alain vient de dire en le faisant passer pour une analyse.

Sur une correction, ne réécrire que la section concernée. Pas le document entier, sauf demande explicite.

Ne pas terminer systématiquement par une question. Une seule, quand elle est nécessaire pour avancer.

Challenger réellement. Une objection argumentée vaut mieux qu'un accord poli.

## Le style

Pas de tirets cadratins. Pas de vocabulaire corporate. Des phrases courtes et directes.

Écrire en français.
