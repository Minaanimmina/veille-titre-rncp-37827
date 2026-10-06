# Prompt de scoring Groq, version v1

## Métadonnées

- Version, v1
- Date de mise en service, 2026-07-06
- Modèle cible, `openai/gpt-oss-120b` (Groq), depuis le 2026-10-06, `llama-3.3-70b-versatile` auparavant
- Paramètres d'appel, `temperature: 0`, `response_format: { type: "json_object" }`
- Usage, scoring des sources candidates issues de Tavily selon la grille de fiabilité en cinq critères

## Prompt système

```text
Tu es un evaluateur de sources de veille technique. Tu appliques strictement une grille de fiabilite en cinq criteres reprise du referentiel RNCP37827. Tu retournes exclusivement un JSON valide, sans texte autour. Bareme par critere, 'rempli', 'partiel', 'manque'. Une source est conservee si aucun critere n'est 'manque' ET si au plus deux criteres sont 'partiel'.
```

## Prompt utilisateur, gabarit

Le prompt utilisateur est construit dynamiquement par le workflow N8N. Les variables entre chevrons sont substituées à l'exécution par le contenu produit par les nœuds amont.

```json
Angle de veille, <$json._angle_libelle>. Requete, <$json._angle_requete>. Sources candidates a evaluer, <JSON.stringify($json.results)>. Pour chaque source, evalue les cinq criteres, 1 auteur identifie et qualifie, 2 contenu recent, 3 contenu sourcable, 4 document structure, 5 document accessible et confirmable. Retourne un JSON strict de la forme, {"sources":[{"url":"...","titre":"...","criteres":{"c1":"rempli|partiel|manque","c2":"...","c3":"...","c4":"...","c5":"..."},"justifications":{"c1":"phrase si partiel ou manque, sinon vide","c2":"...","c3":"...","c4":"...","c5":"..."},"conservee":true|false,"raison_rejet":"si non conservee","resume":"deux phrases"}]}
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

## 2026-10-06, alignement sur le nœud en service

- Le texte versionné jusqu'ici dans ce fichier différait de celui en service dans le nœud Scoring Groq depuis le 2026-07-20. C'est désormais ce dernier qui fait foi, le message système et le gabarit du message utilisateur ci-dessus sont recopiés à l'identique du nœud.
- Le modèle passe de `llama-3.3-70b-versatile` à `openai/gpt-oss-120b`, suite à l'arrêt du premier par Groq le 2026-08-16.
- Le nœud Wait est remplacé par l'option de traitement par lots du nœud Scoring Groq, un élément par lot et 65 000 ms entre deux appels, du fait de la limite de 8 000 tokens par minute du palier gratuit.

## Notes de conception

Le prompt système est volontairement court. Le format de sortie est garanti par le paramètre `response_format: json_object` de l'API Groq, il n'a pas besoin d'être détaillé dans le prompt système. Une version antérieure plus verbeuse avait été écartée car elle poussait la consommation de tokens au dessus du plafond gratuit Groq (12 000 TPM sur le modèle 70B, atteint dès trois appels rapprochés).

L'expérimentation initiale a confirmé que ce prompt court produit un JSON strictement conforme au schéma attendu, avec des scorings différenciés selon la qualité éditoriale des sources (sources écartées effectivement rejetées, sources d'éditeurs corporate identifiés marquées `partiel` sur le critère auteur, sources de référence marquées `rempli` sur les cinq critères).