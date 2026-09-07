**architecture.md**

# JONEO — Architecture de production

7 septembre 2026

## Le principe

Un système unique de la veille jusqu'à la diffusion sociale, pas une collection d'agents construits séparément.

Supabase est le dépôt unique de tous les contenus. Toute la logique éditoriale vit dans la base et dans les prompts. Les outils d'exécution sont remplaçables.

L'orchestration se fait par les statuts. Chaque agent lit les lignes dans l'état qui le concerne, agit, écrit le nouvel état. Aucun agent n'en appelle un autre. C'est la base qui orchestre.

## Le modèle de données

Deux natures d'objets, à ne pas confondre.

**Les origines** sont la matière première. Elles n'ont pas de cycle de vie, seulement un état de qualification. Elles restent disponibles indéfiniment et peuvent servir plusieurs fois.

**Les contenus** ont un cycle, parce qu'un contenu se produit et se publie.

### La veille

L'agent de veille lit ses sources, qualifie, et écrit directement dans la bonne table. Pas d'entrepôt intermédiaire, pas de tri différé. Ce qui est du bruit n'est écrit nulle part.

`actus` — ce qui relève de l'actualité et sera publié sur joneo.ai. C'est une table de contenu, pas d'origine.

`signaux` — ce qui relève du signal, c'est-à-dire ce qui pourrait se passer. Reste en réserve, alimente le fil d'épisodes.

Un signal n'est pas toujours collecté. Il peut être déduit en relisant plusieurs actus : trois annonces qui racontent la même bascule. Reste à trancher si c'est le veilleur quotidien ou un agent d'exploitation qui écrit ces signaux déduits.

`vus` — table technique de dédoublonnage. URL, empreinte, embedding. Aucune logique éditoriale, jamais consultée à la main. Remplace le Data Store Make actuel, en plus fin, et permet de comparer un nouvel item à tout ce qui a déjà été traité, y compris ce qui a été écarté.

### Les autres origines

`nuance_questions` — la réserve de questions Nuance, alimentée à la main.

`concepts` — la liste des Fondamentaux, alimentée à la main.

`sujets_expert` — la réserve de sujets d'articles personnels, alimentée à la main.

### Les tables de contenu

`actus` — texte, publié sur joneo.ai. Titre, corps, tags, URL de publication.

`episodes` — podcasts, fil et Fondamentaux. Script, durée, voix, URL du fichier, identifiants Castos et Ghost, tags. Un champ distingue les deux formats, qui partagent la chaîne de production audio.

`nuances` — avec sa table fille `nuance_positions` : une ligne par IA participante, avec le modèle, la thèse défendue, le texte, l'ordre de passage, la voix. C'est ce qui permet de boucler pour produire l'audio.

`articles_expert` — texte long publié sur alainfargeon.com, sous la signature d'Alain. Toujours en relecture.

`publications_sociales` — une ligne par publication sur un réseau, avec sa plateforme, son texte, sa date prévue.

### Tables techniques

`formats` — la configuration : voix, template, destination, canal par défaut. Remplace l'ancienne base technique Notion.

`journal` — en ajout seul, alimenté par trigger.

### Les liens

Un épisode du fil référence le ou les signaux qui l'ont déclenché. Un Nuance référence une question, et éventuellement un signal quand une actualité l'a réveillée. Un article expert référence un sujet, ou un contenu JONEO qu'il prolonge.

Ce qui a déjà servi se lit dans ces liens, jamais dans un statut. Ne pas créer de statut pour une information qu'une relation exprime déjà.

## Les statuts

**`signaux`** — `réserve`, `écarté`. Un signal reste disponible indéfiniment et peut alimenter plusieurs contenus.

**`nuance_questions`, `concepts`, `sujets_expert`** — `réserve`, `écarté`. Une question déjà traitée reste sélectionnable : elle peut être rejouée plus tard, quand l'actualité l'a fait bouger. Un champ `dernière_utilisation`, mis à jour par trigger, permet de voir d'un coup d'œil ce qui a déjà été joué et depuis quand.

**`actus`** — `rédigé`, `publié`. Régime automatique, pas d'arrêt.

**`episodes`, `nuances`, `articles_expert`** — `à produire`, `rédigé`, `validé`, `produit`, `publié`. La distinction entre `validé` et `produit` sépare le geste de relecture de la fabrication technique, qui peut échouer et être relancée sans revalidation.

**`publications_sociales`** — `rédigé`, `validé`, `diffusé`.

Un champ `verrou` séparé marque une ligne prise par un scénario, sans polluer la liste des statuts métier. Sans lui, un select planifié traite deux fois la même ligne.

Ces listes seront ajustées après les premiers contenus réels.

## La publication planifiée

Un champ `date_publication_prévue` sur la ligne. Le statut autorise la publication, la date la déclenche.

Le select planifié cherche les lignes validées dont la date est atteinte. Une ligne sans date part immédiatement.

Même mécanisme partout. Pour les publications sociales, ça permet de préparer une séquence en une fois et de la laisser se dérouler sur plusieurs jours.

## Les régimes

Le format porte le régime.

Actus 360 est automatique : de `rédigé` à `publié` sans arrêt.

Les épisodes, les Nuance et les articles expert s'arrêtent à `rédigé` et attendent la relecture.

## Les agents

Trois natures, selon la tâche.

**Agents conversationnels planifiés.** Claude ou ChatGPT, connectés à Supabase, dans un projet. Ils lisent la base et interagissent avec Alain. Ils peuvent être planifiés, mais leur intérêt est l'échange : discuter un angle, ajuster, valider. C'est l'agent éditorial.

**Agents Make.** Les Make AI Agents : un LLM avec des instructions en français et des outils. Le veilleur, le rédacteur des cas automatiques, l'agent social. Ils tournent sans intervention. Les outils sont des modules directs ou des outils MCP, sans construire de scénario.

**Workflows Make.** Sans IA. Publication Ghost, image, audio. Ils ne décident de rien, ils exécutent.

Un agent dédié pour les articles expert, avec un prompt qui porte la voix personnelle d'Alain et non celle de JONEO. C'est la raison de le séparer : ce n'est pas le même auteur.

## Les déclencheurs

Trois options, un seul principe.

**Select planifié**, recommandé par défaut. Contrôle du débit, rattrapage des ratés, aucun déclenchement sur une correction mineure, et c'est le seul compatible avec la publication planifiée.

**Database Webhook Supabase** sur changement de ligne. Immédiat mais sans réessai, et se déclenche sur toute mise à jour.

**Trigger natif `watchEvents`** du module Supabase de Make. À tester en premier, plus simple que les deux autres.

## La validation et la relecture

Deux gestes différents, deux endroits.

**La décision se prend dans Supabase Studio.** Voir la file, changer un statut, corriger un champ court. Le Table Editor avec un filtre sur le statut suffit. Une vue SQL rassemblant les lignes en attente des différentes tables donne un écran unique.

**La relecture de fond se fait ailleurs.** Un script de deux mille mots ne se relit pas dans une interface de base de données. Deux options : publier en brouillon dans Ghost et relire dans son éditeur, ou relire avec l'agent éditorial dans Claude, qui réécrit en base après discussion.

Directus a été envisagé puis écarté. Directus Cloud ne peut pas se connecter à une base Supabase externe, seul l'auto-hébergement le permet, ce qui ajoute un serveur à maintenir pour un gain marginal.

## L'observabilité

Table `journal` en ajout seul, alimentée par un trigger Postgres sur les tables de contenu. À chaque changement de statut : horodatage, acteur, action, objet, statut avant, statut après, contexte.

L'avantage du trigger : personne ne peut oublier de logger.

Chaque agent a son propre rôle Postgres, cantonné à ce qu'il écrit. Le trigger capte l'identité de l'acteur sans que l'agent la déclare. Un agent ne peut pas mentir sur son identité. L'agent JONEO ne doit pas pouvoir écrire sous la signature d'Alain, et inversement.

Second niveau, écrit par les agents eux-mêmes : modèle utilisé, tokens, outils appelés, éléments traités et rejetés, justification des décisions.

Alerte quotidienne : un select planifié qui cherche les lignes bloquées dans un statut intermédiaire depuis plusieurs heures.

Ce triptyque reprend les principes de sécurité des agents publiés par Google : contrôleur humain, pouvoirs limités, actions observables.

## Configuration technique

Ce qui dépend du format va dans `formats` : voix, template, destination, canal par défaut.

Ce qui dépend de la ligne va sur la ligne : URL du fichier, identifiants de publication, dates. Écrit par Make, jamais à la main.

Pour Nuance, la voix est portée par `nuance_positions`, une par intervenant.

Aucun identifiant en dur dans un module Make. Si un scénario contient une chaîne qui ressemble à un identifiant, elle devrait être en base.

## Points vérifiés

Le module Supabase de Make expose douze modules, dont `searchRows`, `createARow`, `upsertARecord`, `deleteRows`, `getRowsCount` et `makeAnApiCall`. Il existe un trigger natif `watchEvents`.

Il n'y a pas de module de mise à jour simple, seulement `upsertARecord`. Reste à vérifier s'il préserve les colonnes non renseignées. Point structurant, puisque chaque étape ne modifie qu'un champ.

Le MCP Supabase fonctionne avec des permissions de développeur. Supabase recommande de ne pas l'exposer à des utilisateurs finaux et de privilégier le mode lecture seule pour les routines non supervisées. Conséquence : l'agent éditorial lit par ce canal, il n'écrit pas par ce canal.

Aucun module vectoriel dans Make. La recherche de similarité passe par une fonction SQL en base, appelée via `makeAnApiCall`. Il faut un fournisseur d'embeddings tiers, Anthropic n'en propose pas.

Une app Make expose `Execute inline Python Code`, `Generate PNG (from HTML)` et `Run Puppeteer`. Profil correspondant à 0CodeKit, déjà utilisé pour le merge audio. À confirmer.

Directus Cloud n'accepte pas de base externe. Confirmé par la documentation Supabase et par la communauté Directus.

## Outils supprimés

Dust, Notion, Bannerbear. Plus de 1000 euros par an et trois dépendances en moins.

Les covers d'épisodes sont supprimées. Une image Open Graph statique par format suffit pour le partage.

## Ce qui reste à faire

Le schéma SQL, à confronter au premier jet présent dans `joneo-supabase`.

Trancher qui écrit les signaux déduits : le veilleur quotidien ou un agent d'exploitation.

Le périmètre de lancement : quels formats sortent, lesquels attendent.

La première chaîne de bout en bout, sur `actus`, avec le veilleur et le rédacteur.

Le catalogue d'agents et le fichier de contraintes techniques, à écrire au fur et à mesure de la construction.
