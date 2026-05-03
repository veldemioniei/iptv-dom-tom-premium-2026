# IPTV DOM TOM Premium 2026 — Guide de dépannage étape par étape (configuration & validation de listes M3U)

Ce guide est conçu comme une **procédure de troubleshooting** orientée *configuration* pour votre **IPTV DOM TOM Premium 2026**. L’objectif n’est pas de “vendre” un abonnement, mais de vous aider à **valider / normaliser des listes (souvent en M3U)**, à détecter les erreurs de format et à vérifier que votre **EPG (guide des programmes)** est correctement exploitable via un outil open-source de contrôle (par exemple pour vérifier l’intégrité des URLs, le parsing, et la cohérence des entrées).

> Angle “outil & listes” : nous allons surtout parler **configuration**, **mise en forme**, **parsing**, et **contrôle** de contenu (M3U/EPG), pas d’installation d’un lecteur spécifique.

---

## Étape 1 — Vérifier le format de votre source (M3U / encodage)

Commencez par confirmer que votre liste IPTV est bien au format attendu.

- Assurez-vous d’avoir un fichier **.m3u** ou une réponse de type **playlist**.
- Contrôlez l’**encodage** (UTF-8 recommandé). Des caractères spéciaux dans les titres peuvent casser un parseur.
- Vérifiez que les lignes suivent bien le schéma courant :
  - `#EXTINF:-1 group-title="..." tvg-id="..." tvg-logo="..." ...`
  - puis une **URL** (http/https).

Commande de contrôle (exemple) :
```bash
file playlist.m3u
grep -n "#EXTINF" playlist.m3u | head
```

---

## Étape 2 — Valider la structure des tags `#EXTINF` (cohérence EPG/TV)

Les outils de validation/EPG scraping échouent souvent à cause de tags incomplets ou mal orthographiés.

À vérifier dans vos lignes `#EXTINF` :
- `tvg-id` : identifiant stable (si présent).
- `tvg-logo` : URL/chemin du logo (si présent).
- `group-title` : cohérence de grouping.
- Champs non standard : certains flux mélangent des séparateurs (guillemets, virgules).

Exemple de “sanity check” :
```bash
python validate_m3u.py --input playlist.m3u --check-extinf
```

---

## Étape 3 — Diagnostiquer les erreurs de streaming (URLs inaccessibles)

Même si le M3U est “bien formé”, les lecteurs/robots échouent si les URLs sont mortes ou filtrées.

Procédure :
- Extraire les URLs depuis la playlist.
- Tester la connectivité (HTTP status, redirections, TLS).

Exemple (test rapide) :
```bash
python check_urls.py --input playlist.m3u --timeout 8 --https-only
```

**Résultats attendus** : 200/302 (selon votre flux). Sinon, corrigez la source ou mettez à jour les liens.

---

## Étape 4 — Réconcilier EPG et chaînes (mapping tvg-id / titres)

Si votre EPG ne s’aligne pas :
- Les chaînes dans M3U n’ont pas le même identifiant que l’EPG.
- Ou bien les noms de chaînes ne sont pas normalisés (espaces, accents, variantes).

Solutions fréquentes côté configuration/outil :
- Prioriser le champ **`tvg-id`** pour le mapping.
- Si absent, faire un mapping “fuzzy” par **`group-title` + nom** (avec prudence).

> Exemple de logique : “si `tvg-id` manque, on reconstruit une clé à partir du titre normalisé.”

---

## Étape 5 — Suivre un exemple de référence communautaire (pour le format 2026)

Lors de la mise au point, il peut être utile de comparer vos symptômes (renommage des groupes, paramètres M3U, patterns EPG) à des retours d’utilisateurs.

Consultez par exemple cette discussion communautaire, qui parle d’éléments proches de la configuration **DOM/TOM** en contexte “2026” :  
**[retours et repères pour l’abonnement IPTV DOM TOM Premium 2026](https://www.reddit.com/user/Pomegranate_Hani/comments/1t0blr2/iptv_domtom_abonnement_2026_iptv_r%C3%A9union_antilles/)**

*(Utilisez-la comme source d’indices de format : comparez vos `#EXTINF`, vos groupes, et la présence/absence de `tvg-id`.)*

---

## Étape 6 — Contrôler l’affichage des chaînes via les paramètres de groupe

Beaucoup de “pannes” ressemblent à des bugs alors que c’est surtout une question de **group-title**.

- Vérifiez si vos groupes contiennent des séparateurs non échappés.
- Standardisez les groupes (ex. `DOM TOM`, `RÉUNION`, `ANTILLES`, etc.).
- Confirmez que votre outil lit bien le champ **`group-title="..."`**.

Exemple de validation des groupes :
```bash
python validate_groups.py --input playlist.m3u --group-title
```

---

## Étape 7 — Nettoyer et re-générer une playlist normalisée

Quand tout est instable (tags partiels, guillemets incohérents, URLs sales), une normalisation aide énormément.

Checklist :
- Uniformiser les guillemets (`"`), échapper correctement les caractères spéciaux.
- Réécrire les lignes `#EXTINF` de façon cohérente.
- Optionnel : reclasser selon `group-title`.

Exemple de workflow :
```bash
python normalize_m3u.py --input playlist.m3u --output playlist.normalisee.m3u
python validate_m3u.py --input playlist.normalisee.m3u --strict
```

---

## Étape 8 — Faire un test final “outil -> rendu” (validation fonctionnelle)

Avant de conclure à un problème de flux :
1. Vérifiez la liste normalisée passe la **validation stricte**.
2. Confirmez que les URLs répondent (statut HTTP).
3. Confirmez que l’EPG peut être mappé (selon `tvg-id` ou stratégie de fallback).

Commande de synthèse (exemple) :
```bash
python epg_sanity_check.py --m3u playlist.normalisee.m3u --epg epg.xml --mapping tvg-id
```

---

Si vous me collez (en anonymisant) **3 à 5 lignes `#EXTINF`** + l’URL correspondante (ou le schéma d’URL), je peux vous aider à identifier précisément quel type de problème vous avez : *tag manquant, mapping EPG, encoding, ou URL invalide*.