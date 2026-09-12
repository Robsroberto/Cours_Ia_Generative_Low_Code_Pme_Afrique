## Architecture globale du projet

Le fil conducteur d’une application de recommandation repose sur trois briques :

1. **Le catalogue produit** – source de données structurée (nom, catégorie, prix, attributs).  
2. **Le moteur de recommandation IA** – prompt qui transforme le profil client en liste de produits pertinents.  
3. **Le canal de communication** – envoi automatisé du résultat via WhatsApp pour toucher le client où il se trouve.

En low‑code, chaque brique est représentée par un **module** : une base de données (ou un connecteur Google Sheets), un **workflow** qui interroge l’API d’OpenAI (ou d’une IA locale) et un **connecteur WhatsApp Business** (Twilio ou Vonage). Le tout est orchestré par un **trigger** déclenché lorsqu’un client saisit son numéro et quelques réponses à un petit questionnaire.

```
[Formulaire client] → [Workflow “Générer recommandations”] → 
   ├─► [Appel IA] → [Traitement réponse] → [Enregistrement historique]  
   └─► [Envoi WhatsApp] → [Confirmation au client]
```

Cette architecture minimise les déplacements de données : le catalogue reste en lecture seule, le prompt ne transmet que le profil client (nom, préférence, budget) et la réponse IA est immédiatement formatée pour le message.

---

## 1. Préparer le catalogue produit

### 1.1 Format recommandé

Un tableau **CSV** ou une feuille **Google Sheets** suffit pour les PME qui n’ont pas de base de données lourde. La structure minimale :

| id | nom            | catégorie | prix | couleur | taille | tags                     |
|----|----------------|-----------|------|---------|--------|--------------------------|
| 1  | T-shirt coton  | Vêtements | 12   | bleu    | M      | « décontracté », « été » |
| 2  | Sac à main     | Accessoires| 45   | noir    | –      | « cuir », « fête »        |

> **voir chapitre 5** pour les bonnes pratiques de collecte et de sécurisation des données.

### 1.2 Importer le tableau dans la plateforme low‑code

Sur **Bubble** (exemple) :

1. Créez une *Data Type* : `Produit`.
2. Ajoutez les champs correspondant aux colonnes du tableau.
3. Utilisez le plugin *CSV Uploader* pour charger le fichier une fois, puis verrouillez la table en lecture‑seule.

Sur **Microsoft Power Platform** :

1. Créez une *Entity* `Produit` dans le *Dataverse*.
2. Activez l’import depuis Excel/CSV via le *Dataflow*.

---

## 2. Modéliser le profil client

Le questionnaire doit être court (3‑4 questions) afin de ne pas décourager l’utilisateur, tout en fournissant assez d’informations pour personnaliser la recommandation.

| Question                         | Exemple de réponse | Raison |
|----------------------------------|--------------------|--------|
| Quel type de produit cherchez‑vous ? | « vêtements », « accessoires » | Filtre de catégorie |
| Budget moyen (€)                | 20‑30              | Contrainte de prix |
| Couleur ou style préféré         | « bleu », « élégant » | Affinage des tags |
| Occasion d’achat                 | « célébration », « usage quotidien » | Priorisation des tags |

Ces réponses seront stockées dans un objet JSON que le workflow transmettra à l’IA.

```json
{
  "categorie": "Vêtements",
  "budget_min": 20,
  "budget_max": 30,
  "couleur": "bleu",
  "occasion": "célébration"
}
```

---

## 3. Concevoir le prompt de recommandation

Le cœur de la solution repose sur un **prompt** bien structuré. L’objectif : demander à l’IA de choisir parmi le catalogue les produits qui correspondent le mieux au profil.

### 3.1 Prompt de base

```
Tu es un assistant commercial pour un petit magasin de Kigali. 
Voici le catalogue produit sous forme JSON (max 200 lignes) :
{{catalogue}}
Le client a les préférences suivantes :
{{profil}}
Propose trois produits du catalogue qui correspondent le mieux, en respectant le budget et les préférences. 
Retourne le résultat sous ce format JSON :
[
  {"id": <id>, "nom": "<nom>", "prix": <prix>, "raison": "<court texte>"}
]
```

### 3.2 Astuces d’optimisation (voir chapitre 6)

| Astuce | Pourquoi |
|--------|----------|
| Limiter le catalogue à 200 lignes | Réduit le coût d’appel API et évite le dépassement de token. |
| Utiliser des *tags* pré‑normalisés | Facilite le matching par l’IA. |
| Inclure la contrainte de prix dans le prompt | L’IA ne propose pas de produits hors budget. |

Dans la plupart des plateformes low‑code, le prompt est stocké dans une **variable** ou un **template** que l’on injecte avec les valeurs `{{catalogue}}` et `{{profil}}`.

---

## 4. Créer le workflow de génération

### 4.1 Déclencheur

- **Événement** : soumission du formulaire client.
- **Action** : appeler la fonction *Generate Recommendations*.

### 4.2 Étapes du workflow

| Étape | Action | Détails |
|------|--------|---------|
| 1 | **Récupérer le catalogue** | Lecture de la table `Produit`, conversion en JSON (limité à 200 lignes). |
| 2 | **Construire le prompt** | Remplacer les placeholders par le catalogue et le profil. |
| 3 | **Appeler l’API OpenAI** | POST `/v1/chat/completions` avec le prompt en `messages`. |
| 4 | **Parser la réponse** | Extraire le tableau JSON, vérifier la validité (`try/catch`). |
| 5 | **Enregistrer l’historique** | Table `Recommandation` : client, timestamp, résultat. |
| 6 | **Envoyer le message WhatsApp** | Utiliser le connecteur Twilio (voir § 5). |

#### Exemple de requête API (format low‑code “Custom Code”)

```javascript
// Bubble – API Connector
let body = {
  model: "gpt-4o-mini",
  messages: [{role: "system", content: prompt}],
  temperature: 0.2,
  max_tokens: 500
};

return {
  method: "POST",
  url: "https://api.openai.com/v1/chat/completions",
  headers: {
    "Authorization": "Bearer " + process.env.OPENAI_KEY,
    "Content-Type": "application/json"
  },
  body: JSON.stringify(body)
};
```

> **voir chapitre 6** pour la gestion des clés API et le traitement des réponses.

---

## 5. Intégrer WhatsApp Business via Twilio

### 5.1 Prérequis

- Compte Twilio avec le **sandbox WhatsApp** (gratuit pour les tests).  
- Numéro de téléphone du client (collecté dans le formulaire).  
- Clé d’API Twilio (`ACCOUNT_SID`, `AUTH_TOKEN`).

### 5.2 Construction du message

Le message doit être **concise**, **personnalisé** et **lisible** sur mobile :

```
Bonjour {{nom_client}} ! 🎉
Voici 3 produits qui pourraient vous plaire :
1️⃣ {{produit1.nom}} – {{produit1.prix}} € ({{produit1.raison}})
2️⃣ {{produit2.nom}} – {{produit2.prix}} € ({{produit2.raison}})
3️⃣ {{produit3.nom}} – {{produit3.prix}} € ({{produit3.raison}})
Répondez « 1 », « 2 » ou « 3 » pour plus de détails ou visitez https://votre-boutique.africa.
```

### 5.3 Appel à l’API Twilio (exemple Power Automate)

1. **Action** : *HTTP – POST*  
2. **URL** : `https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages.json`  
3. **Headers** : `Authorization: Basic <base64(AccountSid:AuthToken)>`  
4. **Body (x-www-form-urlencoded)**  

```
From=whatsapp:+14155238886
To=whatsapp:+{numero_client}
Body=Bonjour {{nom}}! Voici vos recommandations…
```

Le workflow attend la réponse HTTP 200 pour confirmer l’envoi, sinon il consigne l’erreur dans la table `Log`.

---

## 6. Gestion des sessions et de la persistance

### 6.1 Identifier le client

- **Option 1 : numéro WhatsApp** – unique, suffit pour les petites boutiques.  
- **Option 2 : identifiant interne** – créez un champ `client_id` lors de la première interaction.

### 6.2 Historique des recommandations

Une table `HistoriqueRecommandation` stocke :

| client_id | date | recommandations (JSON) | canal |
|-----------|------|------------------------|-------|

Cela permet :

- D’analyser les taux de conversion (voir chapitre 8).  
- De proposer des recommandations récurrentes (« Vous avez aimé X la semaine dernière »).

### 6.3 Gestion de la confidentialité

Ne conservez jamais le texte intégral du prompt contenant les clés d’API. Utilisez les **variables d’environnement** de la plateforme et limitez les logs aux identifiants anonymes (voir chapitre 5).

---

## 7. Test fonctionnel et itérations

### 7.1 Scénarios de test

| Scénario | Étapes | Résultat attendu |
|----------|--------|-------------------|
| 1️⃣ Client avec budget limité | Entrer budget 10 € | Aucun produit > 10 € proposé |
| 2️⃣ Client sans préférence de couleur | Laisser le champ vide | L’IA privilégie les best‑sellers |
| 3️⃣ Erreur d’API | Simuler token expiré | Workflow passe en *catch* et envoie un SMS d’erreur |

Utilisez les **simulateurs** intégrés (Bubble “Run as” ou Power Automate “Test”) pour automatiser ces cas.

### 7.2 Boucle d’amélioration

1. **Collecte de feedback** – Ajoutez un bouton « Ce produit vous plaît ? » dans le message WhatsApp (réponse « Oui » / « Non »).  
2. **Enrichissement du prompt** – Intégrez le taux de satisfaction pour affiner les futures suggestions.  
3. **Réduction du coût** – Si le nombre d’appels dépasse le budget, passez à un modèle moins cher (`gpt-3.5-turbo`) ou à une IA locale (voir chapitre 6).

---

## 8. Déploiement et suivi en production

### 8.1 Hébergement

- **Bubble** : déploiement direct sur le domaine `votre-boutique.africa`.  
- **Power Apps** : publier l’application comme *Canvas App* accessible via navigateur ou mobile.

### 8.2 Monitoring

| Indicateur | Outil | Seuil d’alerte |
|------------|-------|----------------|
| Temps moyen d’appel IA | Dashboard de la plateforme | > 3 s |
| Taux d’erreur API (OpenAI / Twilio) | Logs CloudWatch ou Power Platform Analytics | > 2 % |
| Coût mensuel IA | Tableau de bord OpenAI | > 30 USD (ajuster le modèle) |

### 8.3 Gestion de la bande passante (voir chapitre 8)

- **Compression** : ne transmettre que le JSON nécessaire.  
- **Caching** : mettre en cache le catalogue pendant 24 h pour éviter des lectures répétées.  
- **Pagination** : si le catalogue dépasse 200 produits, charger les 200 les plus pertinents (par catégorie ou popularité) avant d’appeler l’IA.

---

## Points clés

- **Modularité** : séparez catalogue, moteur IA et canal WhatsApp pour faciliter la maintenance.  
- **Prompt efficace** : limitez le catalogue, encodez les contraintes de prix et de préférence, et demandez un format JSON strict.  
- **Gestion des clés** : conservez les secrets dans les variables d’environnement, jamais dans le code visible.  
- **Feedback loop** : exploitez les réponses client (Oui/Non) pour entraîner progressivement le système.  
- **Surveillance des coûts** : suivez le nombre d’appels IA et choisissez le modèle le plus économique selon le volume.  
- **Adaptation locale** : privilégiez les plateformes qui fonctionnent avec une connectivité intermittente et offrent un support francophone, afin d’assurer la pérennité de l’application dans les environnements africains.