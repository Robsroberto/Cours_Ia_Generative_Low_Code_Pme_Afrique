## Collecte de données locales : des sources à portée de main  

Les petites entreprises africaines disposent souvent d’une multitude de points de contact : caisse enregistreuse, SMS de commande, réseaux sociaux, enquêtes papier, etc. En low‑code, la première étape consiste à **centraliser** ces flux dans un format exploitable.  

| Source | Méthode low‑code recommandée | Format d’entrée |
|--------|-----------------------------|-----------------|
| **Formulaires papier** | Scanner + OCR via *Microsoft Power Automate* ou *n8n* | CSV ou texte brut |
| **Google Forms / Typeform** | Connecteur natif (Google Sheets, Webhook) | Tableur en ligne |
| **SMS / USSD** | API de téléphonie (Twilio, Africas Talking) → webhook → Google Sheet | JSON → ligne de tableur |
| **Caisse / POS** | Export CSV manuel ou API du POS (ex. : *Mobicash*, *PayDunya*) | CSV ou JSON |
| **Réseaux sociaux** | Connecteurs Facebook/Instagram ou scraping via *Apify* | JSON |

> **Astuce low‑code** : la plupart des plateformes (Bubble, Power Apps, Adalo) offrent un **trigger “When a new row is added”** qui démarre automatiquement un workflow de nettoyage ou de sauvegarde dès qu’une donnée arrive.

### Exemple : récupérer les réponses d’un Google Form dans Google Sheets  

1. Créez le formulaire et choisissez « Enregistrer les réponses dans une feuille ».  
2. Dans la plateforme low‑code (ex. : Power Automate), ajoutez le trigger **“When a new response is submitted”**.  
3. Mappez chaque champ du formulaire aux colonnes de la feuille.  

Le résultat : chaque réponse apparaît immédiatement dans une table exploitable, prête à être nettoyée.

---

## Stockage adapté aux réalités des PME africaines  

Le choix du **backend** dépend de trois contraintes majeures : bande passante, coût et souveraineté des données.  

### 1. Stockage local (CSV, SQLite)  

- **CSV** : simple, lisible à la main, idéal pour les prototypes ou les petites bases (< 5 000 lignes).  
- **SQLite** : base de données embarquée, aucune installation serveur, fonctionne même hors‑ligne sur un smartphone ou un PC.  

```python
# Exemple d’insertion dans SQLite depuis un fichier CSV (exécuté dans un bloc “Run JavaScript” de Bubble)
const fs = require('fs');
const sqlite3 = require('sqlite3').verbose();

let db = new sqlite3.Database('clients.db');
db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS clients (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    telephone TEXT,
    email TEXT,
    ville TEXT
  )`);

  const rows = fs.readFileSync('clients.csv','utf8')
                 .split('\n')
                 .slice(1) // sauter l’en‑tête
                 .map(l=>l.split(','));

  const stmt = db.prepare('INSERT INTO clients (nom, telephone, email, ville) VALUES (?,?,?,?)');
  rows.forEach(r=> stmt.run(r[0], r[1], r[2], r[3]));
  stmt.finalize();
});
db.close();
```

### 2. Stockage cloud léger  

- **Google Sheets** : déjà utilisé pour la collecte, il sert de base de données « serverless ». Limite : 5 M de cellules, pas de requêtes complexes.  
- **Supabase** : PostgreSQL hébergé, API REST et GraphQL, gratuit jusqu’à 500 Mo. Bon pour les apps qui doivent évoluer.  
- **Firebase Realtime Database / Firestore** : très répandu, mais attention aux **réglementations** (voir section suivante).  

### 3. Décision en fonction du contexte  

| Situation | Solution recommandée |
|-----------|----------------------|
| Connectivité intermittente, besoin d’accès hors‑ligne | SQLite + export CSV périodique |
| Petit commerce, budget ultra‑limité | Google Sheets (gratuit) |
| Application qui doit croître rapidement, besoin d’authentification | Supabase (plan gratuit) |
| Données sensibles (données de santé, finances) | Serveur local ou fournisseur certifié (ex. : *DataCentric* au Kenya) |

---

## Nettoyage et transformation des données sans écrire une ligne de code (ou presque)  

Même les meilleures collectes contiennent des **incohérences** : espaces superflus, formats de téléphone différents, doublons. Les plateformes low‑code proposent des **blocs de transformation** qui remplacent les scripts traditionnels.

### 1. Nettoyage de texte avec des fonctions intégrées  

| Plateforme | Fonction de nettoyage | Exemple d’usage |
|------------|----------------------|-----------------|
| Power Automate | `trim()`, `replace()` | Supprimer les espaces avant/après un nom |
| Bubble | “:lowercase”, “:replace” | Uniformiser les adresses e‑mail |
| n8n | “Set” node + JavaScript | Convertir `+225 07 12 34 56` → `07123456` |

#### Exemple n8n : uniformiser les numéros de téléphone  

```javascript
// Node "Set" – field "tel_clean"
$set({
  tel_clean: $json["telephone"]
               .replace(/\s+/g, "")       // supprime les espaces
               .replace(/^(\+?225|0)/, "0") // force le préfixe local
});
```

### 2. Déduplication  

- **Power Apps** : ajouter une règle « If(CountRows(Filter(Clients, Email = ThisItem.Email)) > 1, Notify("Doublon", NotificationType.Error)) ».  
- **Bubble** : workflow “Search for Clients” → “Only when count > 1 → Show alert”.

### 3. Enrichissement (lookup)  

Utilisez des **API de géocodage** (ex. : *OpenCage*, *Mapbox*) pour transformer une ville en coordonnées GPS, puis stockez les lat/lng dans la même table. La plupart des plateformes offrent un **connector HTTP** où il suffit de coller l’URL et de mapper les champs de réponse.

---

## Confidentialité : bonnes pratiques à appliquer dès le premier enregistrement  

### 1. Masquage et chiffrement des champs sensibles  

| Donnée | Technique | Implémentation low‑code |
|--------|-----------|------------------------|
| Numéro de téléphone | **Hash SHA‑256** (non réversible) | Power Automate : `hash('SHA256', triggerBody()?['telephone'])` |
| Adresse e‑mail | **Chiffrement symétrique** (AES) | Bubble : plugin “CryptoJS” → `CryptoJS.AES.encrypt(email, secretKey).toString()` |
| Données personnelles (PII) | **Tokenisation** (remplacer par un identifiant) | n8n : stocker le mapping dans une table « Tokens » séparée |

#### Exemple CryptoJS (Bubble)  

```javascript
// Encryption of email
let secret = "mySuperSecretKey123"; // à placer dans les variables d’environnement
let encrypted = CryptoJS.AES.encrypt(currentUser.email, secret).toString();
```

### 2. Gestion des accès  

- **Rôles** : créez au minimum deux profils : *Administrateur* (accès complet) et *Opérateur* (lecture/écriture limitée).  
- **Principle of Least Privilege** : chaque utilisateur ne voit que les colonnes dont il a besoin.  
- **Audit logs** : activez le suivi des modifications (ex. : “Who changed what and when” dans Power Apps).

### 3. Stockage sécurisé des clés API  

Ne jamais hard‑coder les clés dans le UI. Utilisez les **Secret Managers** intégrés :  

- **Power Platform** : *Azure Key Vault* ou *Dataverse Secrets*.  
- **Bubble** : “App Settings → Private Variables”.  
- **Adalo** : “Custom Actions → Environment Variables”.

---

## Conformité aux réglementations africaines  

### 1. Panorama des lois majeures  

| Pays | Loi principale | Points clés |
|------|----------------|------------|
| **Nigeria** | *Nigeria Data Protection Regulation* (NDPR) | Consentement explicite, notification de violation, droit à la portabilité. |
| **Kenya** | *Data Protection Act* (2019) | Responsable du traitement, registre des activités, évaluation d’impact. |
| **Afrique du Sud** | *POPIA* (Protection of Personal Information Act) | Sécurité renforcée, notification dans les 72 h, droit à l’oubli. |
| **Rwanda** | *Law No 24/2016 on Data Protection* | Consentement préalable, stockage local recommandé. |
| **Côte d’Ivoire** | *Loi n° 2021‑102* sur la protection des données personnelles | Obligations de déclaration auprès de l’ANPD. |

> **À retenir** : la plupart de ces lois partagent trois exigences communes : **consentement**, **sécurité** et **droit d’effacement**.

### 2. Checklist de conformité low‑code  

1. **Collecte** : afficher un bandeau de consentement avant chaque formulaire (ex. : composant “Checkbox – J’accepte les conditions”).  
2. **Minimisation** : ne collecter que les champs strictement nécessaires (ex. : pas de date de naissance si le service ne le requiert pas).  
3. **Sécurisation** : chiffrer les colonnes sensibles, limiter les accès, activer le TLS sur les appels API.  
4. **Durée de conservation** : créer un workflow qui supprime automatiquement les enregistrements après X mois (ex. : “Delete rows older than 12 months”).  
5. **Droit à l’oubli** : fournir un bouton “Supprimer mes données” qui déclenche un “Delete” sur toutes les tables liées.  
6. **Registre des traitements** : exporter périodiquement la structure des tables (CSV) et les conserver dans un dossier partagé sécurisé.  

### 3. Implémentation d’un bouton “Supprimer mes données” (Power Apps)  

```powerapps
// OnSelect du bouton
RemoveIf(Clients, Email = User().Email);
Notify("Vos données ont été supprimées.", NotificationType.Success);
```

Le même principe s’applique dans Bubble : “Delete a thing” → “Current User”.

---

## Sécurisation des API d’IA générative  

Lorsque l’on intègre **OpenAI**, **Cohere**, **Stability AI**, les clés d’API sont le point d’entrée le plus sensible.

1. **Stockage** : utilisez le **Secret Manager** de votre plateforme (voir section précédente).  
2. **Rotation** : planifiez une rotation trimestrielle des clés et mettez à jour le secret sans toucher au code.  
3. **Limitation d’usage** : configurez des **quotas** au niveau du compte fournisseur (ex. : 10 000 tokens/jour).  
4. **Audit** : conservez les logs d’appels (heure, utilisateur, prompt) dans une table séparée, sans stocker le texte complet si cela représente des données personnelles.  

#### Exemple d’appel sécurisé avec n8n  

```json
{
  "nodes": [
    {
      "parameters": {
        "url": "https://api.openai.com/v1/chat/completions",
        "method": "POST",
        "jsonParameters": true,
        "options": {
          "bodyContentType": "json"
        },
        "headerParametersJson": [
          {
            "name": "Authorization",
            "value": "Bearer {{$env.OPENAI_KEY}}"
          }
        ],
        "bodyParametersJson": [
          {
            "name": "model",
            "value": "gpt-4o-mini"
          },
          {
            "name": "messages",
            "value": [
              {"role":"system","content":"You are a helpful assistant."},
              {"role":"user","content":"{{ $json.prompt }}"}
            ]
          }
        ]
      },
      "name": "Appel OpenAI",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 1,
      "position": [400,300]
    }
  ],
  "connections": {}
}
```

Le secret `OPENAI_KEY` est récupéré depuis le **Secret Manager** d’n8n, jamais exposé dans le workflow.

---

## Cas pratique : Application de gestion de clientèle pour un petit commerce à Abidjan  

1. **Collecte** : un formulaire Google Formulaire « Inscription client » (nom, téléphone, email, quartier).  
2. **Stockage** : réponses dirigées vers Google Sheets, puis synchronisées chaque nuit vers une base SQLite locale via n8n.  
3. **Nettoyage** : n8n exécute un workflow qui  
   - supprime les espaces,  
   - convertit les numéros en format national (`0XXXXXXXXX`),  
   - crée un hash SHA‑256 du numéro pour le comparer à des listes de blocage.  
4. **Chiffrement** : l’email est chiffré avec AES avant d’être inséré dans SQLite (clé stockée dans Azure Key Vault).  
5. **Consentement** : le formulaire inclut une case à cocher “J’accepte le traitement de mes données selon la loi ivoirienne sur la protection des données”.  
6. **Droit à l’oubli** : un bouton dans l’app low‑code (