---
title: joneo-brain comme base de connaissance canonique
date: 2026-07-07
status: active
---

# Décision

## Contexte
Deux besoins mémoire distincts : continuité conversationnelle autour de la vision JONEO, et base de connaissance structurée alimentée par les pipelines Make (transcripts, veille, stratégie).

## Décision
Architecture à trois couches : **Notion** = tampon opérationnel des pipelines, **GitHub (`joneo-brain`)** = base canonique en Markdown versionné, **Mintlify** = couche de publication pour NEO. Obsidian écarté (connaissance principalement générée et consommée par des machines).

## Conséquences
- Le rédacteur en chef charge ce repo à chaque run comme pivot mémoire.
- Un seul rédacteur en chef, un objet contenu unique avec frontmatter YAML riche, le blog est un paramètre du même pipeline.
