# JONEO — Architecture de production

17 septembre 2026

## Le principe

Un système unique de la veille jusqu'à la diffusion sociale, pas une collection d'agents construits séparément.

Supabase est le dépôt unique de tous les contenus. Toute la logique éditoriale vit dans la base et dans les prompts. Les outils d'exécution sont remplaçables.

L'orchestration se fait par les statuts. Chaque agent lit les lignes dans l'état qui le concerne, agit, écrit le nouvel état. Aucun agent n'en appelle un autre. C'est la base qui orchestre.

## Vocabulaire

Un seul mot par niveau, le même dans la base, dans les documents et dans les scénarios.

**Contenus** : les actus, les nuances, les épisodes. Chaque nature de contenu est une table. La nature n'est donc jamais une donnée.

**Podcasts** : les shows. Un podcast appartient à une **famille**.

**Familles** : `fondamental` et `fil`. Plusieurs podcasts en Fondamentaux, pour se former. Un seul podcast en Fil de l'eau, pour rester informé.

**Tags** : les thèmes transversaux. Ils traversent les familles et les podcasts.

Le mot *format* est banni. Il désignait à la fois une nature de contenu et une famille, ce qui rendait chaque phrase ambiguë.

## Conventions de nommage

**snake_case**, sans exception. Postgres met en minuscules ce qui n'est pas entre guillemets : `datePublication` devient `datepublication`. Le camelCase obligerait à des guillemets dans toutes les requêtes et dans Make.

**Pas d'accents** dans les noms de colonnes. `duree`, pas `durée`. Les accents passent en Postgres mais posent problème dans les outils tiers, Make compris.

**Pas de préfixe de table.** `titre` dans `episodes`, pas `episode_titre`. La désambiguïsation se fait par `episodes.titre` ou par alias dans les vues.

**Exception, les clés étrangères** : `podcast_id` dans `episodes`.

**Français pour l'éditorial, anglais pour la technique.** `titre`, `resume`, `impacts`, `statut`, `consignes_redaction` d'un côté. `ghost_id`, `bunny_url_mp3`, `created_at`, `published_at` de l'autre.

**Valeurs de statut en anglais**, même quand la colonne s'appelle `statut`. `published` est moins ambigu que `publie`, qui se lit aussi bien comme un impératif que comme un participe.

**Pas d'underscore initial** pour marquer le technique. Il est réservé aux objets système Postgres. Les identifiants externes se préfixent par leur système : `ghost_id`, `bunny_url_cover`, `elevenlabs_voice_id`.

## Les deux questions, les deux agents de veille

C'est la distinction structurante de tout l'amont.

**Le veilleur** demande : qu'est-ce qui s'est passé ces dernières vingt-quatre heures. Quotidien. Il écrit dans `actus`.

**Le prospectif** demande : qu'est-ce qui commence à devenir observable, et que faudrait-il voir ensuite pour le prendre au sérieux. Hebdomadaire. Il écrit dans `signaux`.

Les deux cherchent sur le web. Ce qui les distingue n'est pas la source mais la question posée, donc les requêtes, les critères de sélection et le rythme.

Deux agents séparés, pour trois raisons. Les questions prospectives évoluent dans le temps, et ne doivent pas obliger à modifier l'agent qui produit les actus. Un agent quotidien qui cherche des signaux en fabriquera pour remplir. Et un signal se déduit sur des semaines, pas sur une journée.

L'archive n'est pas la source du prospectif, elle est sa mémoire : il consulte `actus` et `signaux` en fin de course, pour ne pas répéter un signal déjà écrit et pour vérifier si un indice nouveau confirme ou fragilise un signal existant.

## Le modèle de données

Deux natures d'objets, à ne pas confondre.

**Les origines** sont la matière première. Elles n'ont pas de cycle de vie, seulement un état de qualification. Elles restent disponibles indéfiniment et peuvent servir plusieurs fois.

**Les contenus** ont un cycle, parce qu'un contenu se produit et se publie.

### La veille

Chaque agent qualifie au moment où il collecte et écrit directement dans sa table. Pas d'entrepôt intermédiaire, pas de tri différé. Ce qui est du bruit n'est écrit nulle part.

Pas de table de dédoublonnage non plus. Un agent conversationnel produit une synthèse, pas une liste d'items bruts : le recoupement se fait à l'intérieur d'une exécution, c'est son travail. Entre exécutions, la lecture des contenus récents suffit.

`actus` — ce qui s'est passé, publié sur joneo.ai. Table de contenu, pas d'origine.

`signaux` — une ligne par **indice observable**, daté, rattaché à une **question prospective**. Plusieurs lignes par question dans le temps racontent son évolution : un indice en septembre, un autre en novembre, la question progresse. Reste en réserve indéfiniment, alimente le podcast Fil de l'eau.

Un signal n'est pas une tendance déjà nommée. Si le sujet peut se dire en reprenant un terme du discours ambiant, ce n'est plus un signal, c'est du mainstream. La colonne `maturite` n'accepte que `faible` et `emergent`.

Trois règles portées par la structure de la table.

Le fait et l'interprétation sont dans deux colonnes distinctes. Une annonce de laboratoire ne vaut pas une évaluation indépendante.

Le contre-indice est obligatoire. Sans lui, la veille devient un moulin à confirmations : un agent laissé libre confirme toujours ce qu'il cherche.

Le prochain indice à surveiller transforme le signal en enquête ouverte plutôt qu'en constat.

### Les autres origines

`nuance_questions` — la réserve de questions Nuance, alimentée à la main.

`concepts` — la liste des Fondamentaux à produire, alimentée à la main.

`sujets_expert` — la réserve de sujets d'articles personnels, alimentée à la main.

### Podcasts et épisodes

Deux tables distinctes. Un podcast est une entité, pas un tag : il porte une voix, des consignes de rédaction, une cover, un gabarit de cover et une ligne éditoriale propre.

`podcasts` — le show. Titre, sous-titre, teaser, description, `famille`, `voix_id`, `consignes_redaction`, cover, gabarit de cover, couleur, visibilité, SEO, statut.

`episodes` — rattaché à un podcast par `podcast_id`. Brief, script, `numero`, durée, URL des fichiers, `ghost_id`, statut, date de publication prévue.

Cette séparation rend la solution évolutive : ajouter un podcast est une ligne, pas une refonte.

**Il n'y a pas de flux RSS.** Les Fondamentaux et le Fil de l'eau sont payants, donc ils ne sont pas distribués sur les plateformes de podcast. L'écoute se fait sur joneo derrière le paywall Ghost. Le mot podcast désigne ici un objet éditorial, pas un canal de distribution. Castos est supprimé.

### Les deux familles

**Fondamentaux.** Plusieurs podcasts, chacun avec une promesse autonome et un parcours ordonné. Un podcast naît d'épisodes qui existent, jamais d'une intention à remplir : aucun nombre d'épisodes n'est annoncé, le podcast est un contenant ouvert.

Un Fondamental est un livre daté. Il ne se met pas à jour en fonction de l'actualité. Il s'enrichit d'épisodes complémentaires, et son actualité vit dans le Fil de l'eau. Il n'y a donc **aucun mécanisme de versionnement** : pas de `version`, pas de `reviewed_at`, pas de saison. En contrepartie, `published_at` est visible sur la page de l'épisode, comme la date d'édition d'un livre.

**Fil de l'eau.** Un seul podcast continu. Épisodes autonomes, publiés au gré des événements, classés chronologiquement, tagués par thème.

### La règle de routage

Le critère n'est ni le thème ni la durabilité de l'épisode, mais **l'existence d'un corps stable à apprendre**.

Le sujet en a un ? Un Fondamental existe, et le Fil de l'eau en suit l'actualité. L'AI Act relève de ce cas : une série ordonnée qui explique le cadre, plus des épisodes ponctuels sur ce qui bouge.

Le sujet n'en a pas ? Fil de l'eau seul, tagué. Une demande de ralentissement de la recherche relève de ce cas : c'est un moment, pas un socle.

Ce lien est porté par `episodes.podcast_fondamental_id`, nullable. Rempli, il affiche « pour approfondir » côté Fil de l'eau, et la liste des épisodes d'actualité sur la page du Fondamental. Le Fondamental reste figé, sa page reste vivante.

Il donne aussi un signal de production : si plusieurs épisodes du Fil de l'eau accumulent le même tag sans qu'un Fondamental existe, c'est peut-être qu'un corps stable a émergé. Le Fondamental naît alors du fil, pas d'une décision a priori.

### La numérotation

`numero` est obligatoire et unique par podcast, dans les deux familles. En Fondamentaux il porte l'ordre du parcours. En Fil de l'eau il est un compteur continu, incrémenté par rapport au dernier épisode produit.

C'est aussi ce qui s'affiche sur la cover.

### Les voix

`voix` — une table à part, qui porte l'identité technique d'une voix : label, `elevenlabs_voice_id`, email, description. Rien d'autre.

L'email correspond à un auteur Ghost et produit la signature affichée sur la page de l'épisode.

Une voix est partagée par plusieurs podcasts. Ce qui est propre au show ne vit donc pas sur la voix mais sur le podcast.

`podcasts.consignes_redaction` — les consignes de traitement propres au show, par exemple un registre journalistique. Elles sont lues par l'agent rédacteur **en amont** de l'écriture. Elles ne sont jamais envoyées à ElevenLabs.

Les balises audio du modèle v3 sont ajoutées par le workflow de production, après le script.

Pour Nuance, la voix est portée par `nuance_positions`, une par intervenant.

### Les tags

`tags`, plus deux tables de liaison, parce que les tags ne jouent pas au même niveau.

`podcast_tags` — qualifie un podcast. Le tag `adoption` marque les cinq shows de registre organisationnel, qui restent en Fondamentaux mais forment un sous-ensemble identifiable. C'est aussi ce qui permettra de les basculer un jour vers l'offre entreprise sans toucher à la structure.

`episode_tags` — qualifie un épisode, principalement dans le Fil de l'eau : réglementation, emploi, éthique, souveraineté.

### Les autres tables de contenu

`nuances` — avec sa table fille `nuance_positions` : une ligne par IA participante, avec le modèle, la thèse défendue, le texte, l'ordre de passage, la voix. C'est ce qui permet de boucler pour produire l'audio.

`articles_expert` — texte long publié sur alainfargeon.com, sous la signature d'Alain. Toujours en relecture. Hors périmètre du positionnement JONEO.

`publications_sociales` — une ligne par publication sur un réseau, avec sa plateforme, son texte, sa date prévue.

### Table technique

`journal` — en ajout seul, alimenté par trigger.

Il n'y a pas de table `formats`. La nature du contenu est portée par la table elle-même, le régime de publication par le statut, et la configuration par le podcast.

### Les jointures remplacent les rollups

Aucun champ rollup. Un rollup Notion contourne l'absence de jointure ; Postgres en a nativement.

Une vue SQL, par exemple `episodes_complets`, joint `episodes`, `podcasts` et `voix`, et expose côté épisode le nom du podcast, la famille, l'`elevenlabs_voice_id`, les consignes de rédaction, la cover du podcast et sa couleur. Make lit cette vue en un seul appel.

Avantage : la donnée n'est jamais dupliquée. Changer la voix d'un podcast se répercute immédiatement sur tous ses épisodes. Ajouter une information à exposer, c'est une colonne dans la vue, pas un champ dans la table.

Une vue est en lecture seule : lecture dans la vue, écriture dans les tables sous-jacentes.

### Les liens

Un épisode du Fil de l'eau référence le ou les indices qui l'ont déclenché, et le cas échéant son Fondamental. Un Nuance référence une question, et éventuellement un indice quand une actualité l'a réveillée. Un article expert référence un sujet, ou un contenu JONEO qu'il prolonge.

Ce qui a déjà servi se lit dans ces liens, jamais dans un statut. Ne pas créer de statut pour une information qu'une relation exprime déjà.

## Les statuts

Un statut décrit **l'état atteint**, jamais l'action à venir. Les contenus sont donc au participe passé, les origines à l'adjectif.

**`signaux`, `nuance_questions`, `concepts`, `sujets_expert`** — `available`, `discarded`. Une origine reste disponible indéfiniment et peut alimenter plusieurs contenus. Une question déjà traitée reste sélectionnable : elle peut être rejouée quand l'actualité l'a fait bouger. Un champ `derniere_utilisation`, mis à jour par trigger, permet de voir ce qui a déjà été joué et depuis quand.

**`podcasts`** — `to_create`, `created`, `to_update`. Exception assumée au formalisme : un podcast n'est pas un pipeline mais un cycle de vie, qui revient toujours à l'état stable `created`.

**`actus`** — `drafted`, `published`. Régime automatique, pas d'arrêt.

**`episodes`, `nuances`, `articles_expert`** — `briefed`, `drafted`, `approved`, `produced`, `published`.

**`publications_sociales`** — `drafted`, `approved`, `distributed`.

Un champ `verrou` séparé marque une ligne prise par un scénario, sans polluer la liste des statuts métier. Sans lui, un select planifié traite deux fois la même ligne. Il n'est utile que sur les étapes déclenchées par select avec une exécution longue, donc `drafted` et `produced`.

## Le cycle d'un épisode

Chaque statut décrit ce qui existe. Chaque agent prend les lignes dans l'état qui précède ce qu'il fait, et écrit l'état qui décrit ce qu'il vient de produire.

### `briefed` — le brief existe

Écrit par l'agent éditorial ou à la main. En Fondamentaux à partir d'un concept, en Fil de l'eau à partir d'un ou plusieurs signaux, avec le lien vers eux.

Renseigne `podcast_id`, `numero`, `sujet`, `angle`, `intention`, les tags, et `podcast_fondamental_id` le cas échéant. Rien d'autre : c'est un brief, pas un contenu.

### `drafted` — le script existe

AGENT Rédacteur, un Make AI Agent. Pose le verrou, lit le brief plus la ligne éditoriale et les consignes de rédaction du podcast par la vue, écrit le script à la longueur imposée.

Renseigne `script`, `titre` définitif, `resume`, `points_cles`. Libère le verrou.

Le pipeline s'arrête là. Les épisodes attendent la relecture, contrairement aux actus.

### `approved` — la validation humaine existe

Alain, à la main. Relecture dans l'éditeur Ghost en brouillon, ou avec l'agent éditorial dans Claude qui réécrit en base.

Renseigne le statut et, si la publication est différée, `date_publication_prevue`.

C'est le seul endroit où un humain décide. Tout ce qui suit est mécanique.

### `produced` — les fichiers existent

WF Production, sans IA. Il ne décide de rien.

1. ElevenLabs v3, voix du podcast lue par la vue, balises audio ajoutées au script
2. Sound design et merge
3. Upload du MP3 sur Bunny
4. Rendu de la cover : gabarit HTML, `numero`, fond du podcast, sortie WebP
5. Upload de la cover sur Bunny
6. Calcul de la durée

Renseigne `bunny_url_mp3`, `bunny_url_sound_design`, `bunny_url_cover`, `duree`.

C'est la raison d'être de la séparation entre `approved` et `produced` : cette étape appelle plusieurs services externes et peut échouer. Elle se relance sans revalidation.

**Idempotence obligatoire.** Les fichiers sont nommés de façon déterministe, `{podcast_slug}-{numero}.mp3` et `.webp`. Un nom horodaté ferait accumuler un orphelin sur Bunny à chaque relance.

### `published` — le post Ghost existe

WF Publication. Prend les lignes en `produced` dont la date prévue est atteinte, ou sans date.

Envoie le MP3 à Ghost, crée le post avec la carte audio, l'image de une, les tags famille plus podcast plus thèmes, et la visibilité payante.

Renseigne `ghost_id`, `ghost_url_mp3`, `ghost_updated_at`, `published_at`.

`ghost_updated_at` est indispensable : l'API Admin de Ghost exige l'`updated_at` courant pour modifier un post. À stocker à la création et à rafraîchir après chaque modification.

## Les covers

Une cover par épisode, générée au carré, avec le `numero` en grand au centre sur le fond du podcast.

Le carré est le format contraint : il s'affiche en 16:9 par recadrage CSS avec `object-fit: cover`, et un numéro centré n'est jamais coupé. Une seule génération, deux affichages.

Le texte est incrusté dans le fichier parce que la cover est consommée par le lecteur audio, où aucune superposition CSS n'est possible. Le titre de l'épisode n'y figure pas : il est illisible à cette taille et déjà affiché en HTML à côté.

Ces images ne sortent pas de joneo. Une image Open Graph statique suffit pour le partage.

**Le gabarit vit dans `joneo-brain`**, en HTML et CSS, versionné avec les prompts. `podcasts.cover_template_id` porte son chemin. Le fond est un actif Bunny référencé en `background-image`.

**Le générateur n'est pas tranché.** Deux candidats, tous deux synchrones, tous deux avec sortie WebP native, donc sans étape de conversion et sans webhook.

`0CodeKit`, endpoint `POST /image/html`. Déjà payé pour le merge audio, donc aucun fournisseur supplémentaire. Avec `getAsUrl: false` la réponse est le binaire, donc deux modules Make au lieu de trois. Aucun paramètre d'attente des ressources externes documenté.

`htmlcsstoimage`, app Make native. Documente explicitement l'attente du chargement des CSS et images externes, plus `ms_delay` et `render_when_ready` en garde-fous, et un paramètre de polices Google. Rend une URL, donc un module de téléchargement en plus. Palier gratuit de 50 images par mois.

**Le test qui tranche** : le fond Bunny se charge-t-il au rendu, et la typographie JONEO s'applique-t-elle. Repli commun en cas d'échec : encoder le fond en base64 dans le HTML, ce qui supprime toute requête réseau pendant le rendu.

## Le stockage des fichiers

Bunny est le magasin d'actifs. Les MP3 et les covers y vivent, et ce sont ces URL qui sont écrites en base.

Le MP3 est ensuite envoyé à Ghost pour la lecture sur le site. `ghost_url_mp3` conserve cette seconde URL.

Une URL rendue par un générateur d'images n'est jamais l'actif. Elle est téléchargée puis réuploadée sur Bunny, ce qui rend le changement de générateur indolore.

## La publication planifiée

Un champ `date_publication_prevue` sur la ligne. Le statut autorise la publication, la date la déclenche.

Le select planifié cherche les lignes en `produced` dont la date est atteinte. Une ligne sans date part immédiatement.

Même mécanisme partout. Pour les publications sociales, ça permet de préparer une séquence en une fois et de la laisser se dérouler sur plusieurs jours.

## Les régimes

Les actus sont automatiques : de `drafted` à `published` sans arrêt.

Les épisodes, les Nuance et les articles expert s'arrêtent à `drafted` et attendent la relecture.

## Les agents

Trois natures, selon la tâche.

**Agents conversationnels planifiés.** Claude ou ChatGPT, connectés à Supabase et à Feeder par MCP. Le veilleur et le prospectif sont de cette nature : ils cherchent sur le web, qualifient, et écrivent en base. L'agent éditorial l'est aussi, mais son intérêt est l'échange plutôt que la planification.

**Agents Make.** Les Make AI Agents : un LLM avec des instructions en français et des outils. Le rédacteur des cas automatiques, l'agent social. Les outils sont des modules directs ou des outils MCP, sans construire de scénario.

**Workflows Make.** Sans IA. Création de podcast, publication Ghost, image, audio. Ils ne décident de rien, ils exécutent.

Un agent dédié pour les articles expert, avec un prompt qui porte la voix personnelle d'Alain et non celle de JONEO. C'est la raison de le séparer : ce n'est pas le même auteur.

**Convention de nommage des scénarios Make** : `AGENT` pour ce qui décide, `WF` pour ce qui exécute, `TOOL` pour ce qui sert un agent, `CRON` pour ce qui déclenche. Le nom dit ce que le scénario fait, jamais quand il tourne. Le dossier porte le domaine.

## Feeder, la source choisie

Feeder expose un serveur MCP hébergé, connecté en OAuth. L'agent peut lister les flux et dossiers, tirer les articles récents, filtrer sur les non lus, ouvrir le texte complet, et faire une recherche plein texte sur tout l'archive avec plage de dates.

Ordre de collecte pour le veilleur : recherche web large sur le spectre 360 d'abord, pour ne pas laisser Feeder cadrer le champ. Puis Feeder, articles et newsletters du jour, pour l'apport propriétaire. Puis recherche ciblée pour recouper.

Feeder est une matière première, pas un sommaire. On n'y résume pas les items un par un.

Point de vigilance : une tâche planifiée tourne sans surveillance. Si la connexion OAuth demande une réautorisation, elle échoue silencieusement. D'où l'alerte quotidienne.

## Les déclencheurs

Trois options, un seul principe.

**Select planifié**, recommandé par défaut. Contrôle du débit, rattrapage des ratés, aucun déclenchement sur une correction mineure, et c'est le seul compatible avec la publication planifiée.

**Database Webhook Supabase** sur changement de ligne. Immédiat mais sans réessai, et se déclenche sur toute mise à jour.

**Trigger natif `watchEvents`** du module Supabase de Make. À tester en premier, plus simple que les deux autres.

## La validation et la relecture

Deux gestes différents, deux endroits.

**La décision se prend dans Supabase Studio.** Voir la file, changer un statut, corriger un champ court. Le Table Editor avec un filtre sur le statut suffit. Une vue SQL rassemblant les lignes en attente des différentes tables donne un écran unique.

**La relecture de fond se fait ailleurs.** Un script ne se relit pas dans une interface de base de données. Deux options : publier en brouillon dans Ghost et relire dans son éditeur, ou relire avec l'agent éditorial dans Claude, qui réécrit en base après discussion.

Directus a été envisagé puis écarté. Directus Cloud ne peut pas se connecter à une base Supabase externe, seul l'auto-hébergement le permet, ce qui ajoute un serveur à maintenir pour un gain marginal.

## Le site

Ghost Pro, plan Publisher, thème Signal versionné sur GitHub et déployé automatiquement. Les modifications sont faites par Claude Code.

**Deux portes d'entrée UX seulement**, correspondant aux deux familles. Se former mène au catalogue des podcasts Fondamentaux. Rester informé mène au flux du Fil de l'eau. Les thèmes ne sont jamais au même rang que les familles.

**Traduction en tags Ghost**, qui n'a qu'une taxonomie mais distingue public et interne. Les familles sont des tags internes, invisibles, qui servent au filtrage. Les podcasts sont des tags publics, ce qui leur donne nativement une page et une identité. Les thèmes sont des tags internes.

Conséquence utile : le `primary_tag` d'un épisode est son podcast, ce qui affiche le nom du show sur les cartes sans code.

**Le filtrage par thème passe par `routes.yaml`**, une entrée par thème, pas par du JavaScript côté client qui casserait la pagination et l'indexation.

Point de vigilance : `routes.yaml` vit dans l'admin Ghost, pas dans le thème. Il n'est donc pas versionné par Claude Code, et une erreur de syntaxe casse le routing du site entier. Il est copié à la main dans le dépôt, avec une sauvegarde avant chaque upload.

**Limites Ghost Pro vérifiées.** Stockage et transfert illimités, nombre de fichiers non limité, 100 Mo par fichier sur le plan Publisher. Les limites de bande passante ne s'appliquent qu'aux configurations headless. Le fair use cite explicitement l'hébergement de podcast comme usage prévu.

Ghost ne gère pas ses médias : supprimer un post n'efface pas le MP3, et il n'existe pas de vue sur ce qui est stocké. Les URL en base sont le seul inventaire.

## L'observabilité éditoriale

Table `journal` en ajout seul, alimentée par un trigger Postgres sur les tables de contenu. À chaque changement de statut : horodatage, acteur, action, objet, statut avant, statut après, contexte.

L'avantage du trigger : personne ne peut oublier de logger.

Chaque agent a son propre rôle Postgres, cantonné à ce qu'il écrit. Le veilleur écrit dans `actus` seulement. Le prospectif lit `actus` et écrit dans `signaux` seulement. Le trigger capte l'identité de l'acteur sans que l'agent la déclare. L'agent JONEO ne doit pas pouvoir écrire sous la signature d'Alain, et inversement.

Alerte quotidienne : un select planifié qui cherche les lignes bloquées dans un statut intermédiaire depuis plusieurs heures, et qui vérifie qu'une actu a bien été écrite dans les dernières vingt-quatre heures.

Ce triptyque reprend les principes de sécurité des agents publiés par Google : contrôleur humain, pouvoirs limités, actions observables.

## L'observabilité des coûts LLM

Tous les appels LLM des agents Make passent par Helicone, en proxy.

Helicone fournit le suivi technique : modèle, tokens, latence, coût par appel. Aucune structure à prévoir dans Supabase, et les agents n'écrivent pas leurs tokens en base.

**Convention obligatoire** : chaque appel porte en métadonnée l'identifiant de la ligne Supabase concernée et le type de contenu. Sans ça, impossible de reconstituer un coût par contenu, puisqu'un Nuance consomme un appel par position plus un pour la synthèse.

Cette donnée est celle qui permettra de vérifier que le modèle économique tient au prix d'abonnement visé.

Répartition : Helicone pour le coût technique, le journal Supabase pour les décisions éditoriales.

## Configuration technique

Ce qui dépend du podcast va dans `podcasts` : voix, consignes de rédaction, cover, gabarit, couleur, visibilité.

Ce qui dépend de la ligne va sur la ligne : URL des fichiers, identifiants de publication, dates. Écrit par Make, jamais à la main.

Aucun identifiant en dur dans un module Make. Si un scénario contient une chaîne qui ressemble à un identifiant, elle devrait être en base.

## Points vérifiés

Le module Supabase de Make expose douze modules, dont `searchRows`, `createARow`, `upsertARecord`, `deleteRows`, `getRowsCount` et `makeAnApiCall`. Il existe un trigger natif `watchEvents`.

Il n'y a pas de module de mise à jour simple, seulement `upsertARecord`, qui impose de renseigner tous les champs. **Pour toute mise à jour partielle, utiliser `Make an API Call` en PATCH** sur `/rest/v1/<table>?id=eq.<uuid>`, avec l'en-tête `Prefer: return=representation`. Le `eq.` est obligatoire : sans lui, PostgREST refuse la requête.

PostgREST ne sait pas comparer deux colonnes dans un filtre. Toute condition de ce type, par exemple `updated_at > ghost_updated_at`, passe par une vue.

Le MCP Supabase fonctionne avec des permissions de développeur. Supabase recommande de ne pas l'exposer à des utilisateurs finaux et de privilégier le mode lecture seule pour les routines non supervisées. Arbitrage retenu : les agents de veille écrivent par ce canal, l'accès étant celui d'Alain sur sa propre base.

0CodeKit expose `POST /image/html`, avec sortie png, jpg ou webp, dimensions, facteur d'échelle, et retour en binaire ou en URL.

htmlcsstoimage expose une app Make native avec quatre actions, dont `Create an Image with HTML/CSS`. Le rendu attend l'événement `load` puis le trafic réseau des CSS et images externes.

Directus Cloud n'accepte pas de base externe. Confirmé par la documentation Supabase et par la communauté Directus.

Feeder expose un serveur MCP hébergé, inclus sur les plans Plus, Professional et Enterprise. Sur un compte gratuit, la connexion fonctionne mais les appels d'outils demandent une mise à niveau.

## Outils supprimés

Dust, Notion, BannerBear, Castos, WordPress.

Notion était jusqu'ici la base réelle, Supabase n'en étant qu'un miroir partiel. La migration inverse ce rapport.

La table de dédoublonnage, le calcul d'embeddings et la fonction de similarité pgvector, devenus inutiles depuis que la veille est faite par un agent conversationnel et non par un collecteur mécanique. Une dépendance de moins : plus besoin de fournisseur d'embeddings.

La table `formats` et la colonne `podcasts.format`, qui portaient un mot ambigu.

## Ce qui reste à faire

Le schéma SQL complet, à confronter au premier jet présent dans `joneo-supabase`.

Le choix du générateur de covers, après le test de chargement du fond Bunny.

La migration des 87 épisodes conservés, avec réécriture du cadrage éditorial : la base actuelle s'adresse à des managers et des dirigeants, le positionnement s'adresse à des personnes.

Le périmètre de lancement : quels contenus sortent, lesquels attendent.

La première chaîne de bout en bout, sur `actus`, avec le veilleur et la publication Ghost.

La cadence du Fil de l'eau, jamais testée : aucun épisode n'a jamais été produit dans ce régime.

Le catalogue d'agents et le fichier de contraintes techniques, à écrire au fur et à mesure de la construction.
