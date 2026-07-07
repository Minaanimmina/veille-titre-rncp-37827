# Prompt de scoring Groq, version v1

## Métadonnées

- Version, v1
- Date de mise en service, 2026-07-06
- Modèle cible, `llama-3.3-70b-versatile` (Groq)
- Paramètres d'appel, `temperature: 0`, `response_format: { type: "json_object" }`
- Usage, scoring des sources candidates issues de Tavily selon la grille de fiabilité en cinq critères

## Prompt système

```text
Tu es un evaluateur de sources de veille. Applique strictement la grille en cinq criteres, 1 auteur qualifie, 2 recent, 3 sourcable, 4 structure, 5 accessible. Barème, rempli, partiel, manque. Une source est conservee si aucun critere n'est manque et au plus deux sont partiels. Retourne uniquement le JSON demande.
```

## Prompt utilisateur, gabarit

Le prompt utilisateur est construit dynamiquement par le workflow N8N. Les variables entre chevrons sont substituées à l'exécution par le contenu produit par les nœuds amont.

```json
Angle de veille, <libelle_angle>. Requete, <requete_tavily>. Sources candidates a evaluer, <json_stringify_des_resultats_tavily>. Pour chaque source, evalue les cinq criteres, 1 auteur identifie et qualifie, 2 contenu recent, 3 contenu sourcable, 4 document structure, 5 document accessible et confirmable. Retourne un JSON strict de la forme, {"sources":[{"url":"...","titre":"...","criteres":{"c1":"rempli|partiel|manque","c2":"...","c3":"...","c4":"...","c5":"..."},"justifications":{"c1":"phrase si partiel ou manque, sinon vide","c2":"...","c3":"...","c4":"...","c5":"..."},"conservee":true|false,"raison_rejet":"si non conservee","resume":"deux phrases"}]}
```

## Format de sortie attendu

Un objet JSON avec une clé `sources`, tableau d'objets, un par source évaluée. Champs de chaque objet.

- `url`, chaîne, URL de la source telle que fournie par Tavily.
- `titre`, chaîne, titre de la source.
- `criteres`, objet, cinq clés `c1` à `c5`, chacune valant `rempli`, `partiel` ou `manque`.
- `justifications`, objet, cinq clés `c1` à `c5`, chacune contenant une phrase de justification si le critère est partiel ou manqué, vide sinon.
- `conservee`, booléen, vrai si aucun critère n'est manqué et au plus deux sont partiels.
- `raison_rejet`, chaîne, motif du rejet si la source n'est pas conservée, vide sinon.
- `resume`, chaîne, résumé en deux phrases de la source.

## Historique des versions

- v1, 2026-07-06, version initiale mise en service pour la première trace hebdomadaire.

## Notes de conception

Le prompt système est volontairement court. Le format de sortie est garanti par le paramètre `response_format: json_object` de l'API Groq, il n'a pas besoin d'être détaillé dans le prompt système. Une version antérieure plus verbeuse avait été écartée car elle poussait la consommation de tokens au dessus du plafond gratuit Groq (12 000 TPM sur le modèle 70B, atteint dès trois appels rapprochés).

L'expérimentation initiale a confirmé que ce prompt court produit un JSON strictement conforme au schéma attendu, avec des scorings différenciés selon la qualité éditoriale des sources (sources écartées effectivement rejetées, sources d'éditeurs corporate identifiés marquées `partiel` sur le critère auteur, sources de référence marquées `rempli` sur les cinq critères).