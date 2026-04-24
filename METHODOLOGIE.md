# METHODOLOGIE - CallCoach

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Claude](https://img.shields.io/badge/Claude_API-D97757?style=for-the-badge&logo=anthropic&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)
![Slack](https://img.shields.io/badge/Slack-4A154B?style=for-the-badge&logo=slack&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-34A853?style=for-the-badge&logo=google-sheets&logoColor=white)
![Notion](https://img.shields.io/badge/Notion-000000?style=for-the-badge&logo=notion&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)

> Un agent IA qui analyse automatiquement chaque appel commercial, génère un email de suivi personnalisé et coache l'agent en temps réel - pour ne plus jamais rater une opportunité ou laisser un client sans réponse.
## Auteur : **Déhollin HOLLAT**
---

## Table des matières

- [1. Pourquoi ce projet](#1-pourquoi-ce-projet)
- [2. Pipeline technique](#2-pipeline-technique)
- [3. Stack & outils](#3-stack--outils)
- [4. Étapes détaillées](#4-étapes-détaillées)
- [5. Intelligence de l'agent IA](#5-intelligence-de-lagent-ia)
- [6. Résultats & limites](#6-résultats--limites)
- [7. Lexique](#7-lexique)
- [8. Coût de déploiement en entreprise](#8-coût-de-déploiement-en-entreprise)
- [9. Ce que j'ai appris](#9-ce-que-jai-appris)

---

## 1. Pourquoi ce projet

Dans un centre d'appels ou une équipe commerciale, deux problèmes reviennent systématiquement : les agents oublient d'envoyer un email de suivi après un appel, et le feedback sur la qualité des échanges arrive trop tard ou pas du tout. Un commercial qui a promis d'envoyer une proposition "d'ici jeudi" sans relance automatique, c'est une opportunité perdue. Un agent qui répète les mêmes erreurs sans coaching structuré, c'est une montée en compétence ralentie.

CallCoach automatise ces deux tâches. Après chaque appel, le système analyse le contenu de l'échange, rédige un email de suivi prêt à envoyer directement depuis Gmail, et envoie un rapport de coaching à l'agent sur Slack - avec un score, des points forts, des axes d'amélioration et la liste des engagements pris. Si l'appel a été particulièrement mal géré, le manager reçoit une alerte immédiate sur un channel dédié. Le tout sans aucune intervention humaine.

L'objectif métier est clair : réduire le temps de suivi post-call de 20 à 30 minutes à moins de 30 secondes, et accélérer la montée en compétence des agents grâce à un feedback structuré et systématique.

---

## 2. Pipeline technique

```
[Transcript simulé (Edit Fields n8n)]
              |
              v
[Mémoire client - lecture historique Google Sheets]
              |
              v
[Claude API - génération email follow-up]
              |
              v
[Gmail - création du brouillon]
              |
              v
[Claude API - analyse coaching + scoring]
              |
              v
[Code node - parsing JSON + enrichissement]
              |
              v
[Google Sheets - stockage objections + historique]
              |
              v
        [IF score < 4 ?]
        /              \
      true            false
        |                |
[Slack #alertes-manager] [Slack #coaching-calls]
                         |
                   [IF score > 7 ?]
                   /            \
                 true           false
                  |               |
           [Notion - base       (fin)
           de connaissances]
```

---

## 3. Stack & outils

| Outil | Rôle dans le projet | Pourquoi ce choix |
|---|---|---|
| n8n (local) | Orchestration de l'ensemble du workflow | Open-source, self-hosted, zéro coût de plateforme |
| Claude API (Haiku) | Génération email + analyse coaching | Modèle rapide, économique, excellente qualité en français |
| Gmail (OAuth2) | Création de brouillons de suivi | Intégration native n8n, brouillon = validation humaine avant envoi |
| Slack | Envoi des rapports coaching + alertes manager | Outil de communication d'équipe standard |
| Google Sheets | Stockage historique appels + bibliothèque objections | Accessible, sans infrastructure, lisible par un non-développeur |
| Notion | Base de connaissances des meilleurs appels | Structuration et consultation facile par l'équipe |
| JavaScript (Code node) | Parsing JSON, gestion des cas vides, enrichissement | Natif n8n, léger, suffisant pour la logique métier |

---

## 4. Étapes détaillées

### Étape 1 - Simulation du transcript

En production, le transcript proviendrait d'un outil comme Fireflies (appels vidéo) ou Whisper/OpenAI (appels téléphoniques via fichier audio). Pour ce projet portfolio, le transcript est simulé via un node **Edit Fields** dans n8n qui injecte le texte de l'échange ainsi que le nom de l'agent et du client.

Trois scénarios ont été testés : appel faible (score < 4), appel moyen (score 5–6), bon appel (score > 7).

---

### Étape 2 - Mémoire client

Avant de générer l'email, n8n consulte l'onglet `historique` du Google Sheets pour vérifier si le client a déjà appelé. Si un historique existe, il est injecté dans le prompt email afin que Claude puisse y faire référence naturellement. Si aucun historique n'existe, un message par défaut "Aucun appel précédent" est utilisé.

---

### Étape 3 - Génération de l'email de suivi

Un appel à l'API Anthropic (Claude Haiku) avec un prompt orienté rédaction commerciale. Claude reçoit le transcript + l'historique client et génère un email professionnel, court, personnalisé, avec un next step clair. Le résultat est envoyé directement en **brouillon Gmail** via OAuth2 - l'agent valide et envoie lui-même.

---

### Étape 4 - Analyse coaching

Un second appel Claude avec un prompt distinct, orienté analyse comportementale. Le modèle retourne un JSON structuré contenant :
- `score` : note sur 10
- `points_forts` : liste des comportements positifs détectés
- `axes_amelioration` : liste des comportements à corriger
- `tonalite` : qualité de l'échange (fluide, tendu, hésitant…)
- `type_appel` : prospection, support, relance ou closing
- `engagements` : promesses concrètes faites par l'agent
- `objections` : objections exprimées par le client
- `resume` : synthèse en 2 phrases

Un **Code node JavaScript** parse ce JSON, nettoie les éventuels backticks markdown retournés par le modèle, et gère les cas vides (tableaux vides remplacés par des messages explicites).

---

### Étape 5 - Stockage Google Sheets

Deux onglets sont alimentés automatiquement après chaque appel, quel que soit le score :
- `objections` : date, agent, client, type d'appel, objections détectées
- `historique` : date, agent, client, type d'appel, score, tonalité, résumé

Un identifiant unique `call_id` (combinaison agent + client + date) est généré pour éviter les doublons.

---

### Étape 6 - Routage conditionnel

Deux IF nodes gèrent le routage :

**IF 1 - Alerte risque** (score < 4) : notification immédiate sur `#alertes-manager` avec le résumé de l'échange et une recommandation de revue manager.

**IF 2 - Base de connaissances** (score > 7) : création automatique d'une page dans la database Notion "CallCoach - Base de connaissances" avec le résumé du meilleur appel.

Tous les appels (quel que soit le score) envoient un rapport structuré sur `#coaching-calls`.

---

### Étape 7 - Reporting hebdomadaire (workflow séparé)

Un second workflow n8n déclenché chaque lundi matin lit les données Google Sheets de la semaine, envoie à Claude un prompt de synthèse, et distribue :
- Un email individuel à chaque agent avec l'évolution de ses scores sur 4 semaines
- Un résumé manager sur Slack avec vue globale équipe

---

## 5. Intelligence de l'agent IA

CallCoach utilise **deux prompts distincts** vers Claude Haiku, chacun optimisé pour une tâche différente.

**Prompt email** : orienté rédaction commerciale. La consigne demande explicitement un email court, professionnel, avec un seul next step clair. L'injection de l'historique client dans le contexte permet à Claude de personnaliser la référence aux échanges passés.

**Prompt coaching** : orienté analyse comportementale et retour structuré. La consigne impose un format JSON strict avec des types de valeurs explicites (liste, entier, string parmi des valeurs définies). Cette contrainte évite les hallucinations de format et garantit un parsing fiable côté n8n.

**Pourquoi Claude Haiku et pas GPT-4 ou un modèle local ?**
- Haiku est 5 à 10x moins cher que GPT-4 pour des tâches de rédaction et d'analyse conversationnelle
- La qualité en français est excellente, sans prompt d'optimisation spécifique
- L'API Anthropic était déjà maîtrisée (FlowReport, SmartDesk) - pas de courbe d'apprentissage supplémentaire
- Un modèle local (Ollama) serait une alternative gratuite, mais la qualité sur les prompts JSON structurés est moins fiable sans fine-tuning

---

## 6. Résultats & limites

| Élément | Statut | Détail |
|---|---|---|
| Génération email brouillon Gmail | ✅ | Brouillon créé automatiquement après chaque appel simulé |
| Analyse coaching JSON | ✅ | Score, points forts, axes, tonalité, type, engagements, objections |
| Alerte manager score < 4 | ✅ | Notification Slack sur channel dédié avec résumé de l'échange |
| Stockage Google Sheets | ✅ | Objections et historique alimentés sur tous les appels |
| Base de connaissances Notion | ✅ | Entrée créée automatiquement si score > 7 |
| Mémoire client | ✅ | Historique injecté dans le prompt email si client connu |
| Reporting hebdo agent + manager | ✅ | Workflow cron séparé, même logique que FlowReport |
| Source transcript réelle | ⚠️ | Simulation en portfolio - en production : Fireflies ou Whisper |
| Anonymisation RGPD | ⚠️ | Non implémentée dans cette version - prévoir un workflow cron mensuel |
| Intégration CRM | ⚠️ | Prévue en Étape 8 (HubSpot Free) - non développée dans cette version |
| Déduplication appels identiques | ⚠️ | call_id basé sur date + noms - deux appels le même jour entre les mêmes personnes seraient fusionnés |

---

## 7. Lexique

**Transcript** : retranscription textuelle d'un appel téléphonique ou vidéo, ligne par ligne avec le nom du speaker.

**Webhook** : mécanisme qui permet à une application externe (ex. Fireflies) d'envoyer automatiquement des données à n8n dès qu'un événement se produit (ex. : transcription terminée).

**Prompt** : instruction envoyée à un modèle de langage (Claude) pour lui dire quoi faire et dans quel format répondre.

**JSON** : format de données structurées utilisé pour faire communiquer les nodes entre eux. Ressemble à un dictionnaire clé-valeur.

**OAuth2** : protocole d'authentification sécurisé qui permet à n8n d'accéder à Gmail ou Google Sheets sans stocker ton mot de passe.

**Scoring** : attribution d'une note numérique (ici /10) à un appel sur la base d'une analyse qualitative automatisée.

**RAG** : Retrieval-Augmented Generation - technique qui enrichit le contexte d'un modèle IA avec des données externes avant de générer une réponse. Utilisé ici de façon simplifiée via l'historique client injecté dans le prompt.

**Cron** : déclencheur temporel dans n8n qui exécute un workflow automatiquement à une heure et un jour définis (ex. chaque lundi à 8h).

**Brouillon Gmail** : email préparé mais non envoyé, en attente de validation humaine. Choix délibéré pour garder un contrôle sur la communication client.

**Escalade** : mécanisme automatique qui détecte une situation à risque (mauvais score, client mécontent) et notifie immédiatement un niveau hiérarchique supérieur.

---

## 8. Coût de déploiement en entreprise

| Poste | Estimation | Remarques |
|---|---|---|
| Infrastructure | 0 – 20 €/mois | n8n self-hosted sur un VPS basique (ex. Hetzner 4€/mois) ou n8n Cloud Starter (~20€/mois) |
| APIs & licences | 5 – 80 €/mois | Claude Haiku (~5 – 15€ pour 50 appels/jour) + Fireflies Business si appels vidéo (~40–80€) |
| Transcription téléphonique | 10 – 30 €/mois | Whisper API OpenAI (~0.006$/min) ou Ringover avec enregistrement natif |
| Développement initial | 3 000 – 6 000 € | 10 – 20 jours/homme profil junior à confirmé (setup, tests, documentation) |
| Maintenance & évolutions | 200 – 500 €/mois | Monitoring, ajustements prompts, nouvelles intégrations CRM |

**Synthèse :** pour une équipe de 5 à 10 agents avec 50 appels/jour, le coût récurrent tourne entre **65 et 170 €/mois** hors développement initial. Ce coût est largement compensé par le gain de productivité : un commercial qui ne passe plus 20 minutes à rédiger un email de suivi récupère environ 2h/jour. Sur une équipe de 10 agents à 35k€/an, 1% de productivité gagnée représente ~3 500 €/an récupérés.

Le coût principal variable est la transcription : Fireflies est indispensable pour les appels vidéo mais onéreux en volume. Pour des appels téléphoniques, Whisper est une alternative quasi gratuite. L'usage d'un modèle local (Ollama + Mistral) à la place de Claude Haiku réduirait la facture API à zéro mais nécessiterait un serveur dédié (~20–50€/mois).


---

## 9. Ce que j'ai appris

- **La gestion du contexte dans n8n est critique** : dans un workflow long, les nodes downstream ne voient que les données du node précédent. Référencer explicitement un node amont avec `$('Nom du node').item.json.champ` est indispensable dès que les données doivent traverser plusieurs branches.
- **Les prompts JSON doivent être très contraints** : Claude retourne parfois des backticks markdown autour du JSON. Un nettoyage systématique avec `.replace(/\`\`\`json|\`\`\`/g, '')` avant le `JSON.parse()` est incontournable en production.
- **Deux prompts valent mieux qu'un** : séparer la rédaction de l'email et l'analyse coaching en deux appels distincts améliore significativement la qualité des deux outputs - chaque prompt reste net et ciblé.
- **Tester les cas limites dès le début** : les cas "tableau vide", "aucun historique", "score limite" sont les premiers à casser le workflow. Les anticiper dans le Code node évite beaucoup de debugging ultérieur.
- **Le positionnement du IF node change tout** : placer le routage conditionnel après le stockage Sheets garantit que tous les appels sont enregistrés, peu importe leur score - une erreur de conception initiale corrigée en cours de projet.

## Auteur
 
**Déhollin HOLLAT**
MBA Big Data & IA
 
[LinkedIn](https://linkedin.com/in/dehollin-hollat) · [GitHub](https://github.com/Dehollinhollat)
