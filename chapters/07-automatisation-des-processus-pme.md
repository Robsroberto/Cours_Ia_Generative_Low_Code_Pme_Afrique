## 1. Principes de l’automatisation IA‑générative  

### 1.1. Déclencheurs (triggers) low‑code  
Dans un environnement low‑code, le **déclencheur** est le point d’entrée d’un workflow : création d’un enregistrement, réception d’un e‑mail, horaire programmé, ou message WhatsApp. La plupart des plateformes (Bubble, Microsoft Power Automate, n8n, Adalo) proposent un catalogue de triggers pré‑configurés :  

| Plateforme | Exemple de trigger | Usage typique en PME |  
|------------|-------------------|----------------------|  
| **Bubble** | “When Button is clicked” ou “When Data is added” | Lancement d’une campagne marketing dès qu’un nouveau client est enregistré. |  
| **Power Automate** | “When a new row is added in Excel Online” | Générer automatiquement une facture dès qu’une commande apparaît dans le tableau de suivi. |  
| **n8n** | “Cron” (planification) ou “Webhook” | Envoi quotidien de bulletins météo aux agriculteurs. |  

Le trigger ne fait que **déclencher** le flux ; il ne contient aucune logique métier. La logique est ajoutée sous forme d’**actions** qui peuvent appeler des API IA, manipuler des données ou envoyer des notifications.

### 1.2. Actions IA texte  
Les actions IA texte reposent sur les modèles de génération de langage (GPT‑4, Claude, LLaMA). Dans un workflow low‑code, elles sont généralement exposées via un **HTTP request** ou un **connector natif**. Le point crucial est le **prompt** : il doit être concis, fournir le contexte nécessaire et indiquer le format de sortie attendu (JSON, texte brut, markdown).  

> **Astuce** : inclure dans le prompt un exemple de sortie.  
> ```text
> Prompt : « Rédige un e‑mail de relance pour le client {{client_name}} qui a commandé le produit {{product_name}} le {{order_date}}. Utilise un ton chaleureux et termine par « Merci de votre confiance ». Retourne uniquement le corps du message, sans balises HTML. »
> ```

### 1.3. Actions IA image  
La génération d’images (DALL‑E, Stable Diffusion, Midjourney) fonctionne de la même façon : on envoie un **prompt visuel** et on reçoit une URL ou un fichier binaire. Les images générées sont souvent utilisées pour :

* Illustrer un catalogue produit avec des visuels personnalisés.  
* Créer des infographies météo pour les agriculteurs.  
* Produire des QR‑codes décorés (facture, paiement mobile).  

Dans un workflow low‑code, il faut veiller à **stocker** l’image dans un service accessible (Google Drive, Azure Blob, S3) avant de l’attacher à un e‑mail ou de la publier sur un site.

---

## 2. Construire un workflow d’envoi d’e‑mail marketing personnalisé  

### 2.1. Schéma du processus  
1. **Trigger** : ajout d’un nouveau prospect dans Google Sheets.  
2. **Action IA texte** : génération du corps de l’e‑mail à partir du profil du prospect.  
3. **Action IA image** : création d’une bannière personnalisée (ex. : produit phare + nom du prospect).  
4. **Action d’envoi** : envoi du mail via SendGrid ou Outlook.  
5. **Log** : sauvegarde du statut dans la feuille de suivi.

### 2.2. Implémentation sur Microsoft Power Automate  

```yaml
# 1. Trigger – Google Sheets – When a new row is added
# 2. HTTP – OpenAI Completion
method: POST
uri: https://api.openai.com/v1/chat/completions
headers:
  Authorization: Bearer {{env.OPENAI_KEY}}
  Content-Type: application/json
body: |
  {
    "model": "gpt-4o-mini",
    "messages": [
      {"role":"system","content":"You are a friendly marketing copywriter."},
      {"role":"user","content":"Write a 150‑word email to {{FirstName}} {{LastName}} promoting our solar lamp. Use a warm tone and end with a call‑to‑action."}
    ],
    "temperature":0.7,
    "max_tokens":300
  }
# 3. HTTP – DALL·E 3 (image generation)
method: POST
uri: https://api.openai.com/v1/images/generations
headers:
  Authorization: Bearer {{env.OPENAI_KEY}}
  Content-Type: application/json
body: |
  {
    "model":"dall-e-3",
    "prompt":"A bright solar lamp on a rustic African village night sky, with the text 'Bonjour {{FirstName}}!' in elegant script",
    "size":"1024x1024",
    "response_format":"url"
  }
# 4. Send email – Outlook
To: {{Email}}
Subject: Découvrez notre lampe solaire, {{FirstName}} !
Body: {{outputs('HTTP_-_OpenAI_Completion')?['choices'][0]['message']['content']}}
Attachments: {{outputs('HTTP_-_DALL·E_3')?['data'][0]['url']}}
```

### 2.3. Prompt de génération de texte (exemple concret)  
```text
« Rédige un e‑mail de 120 mots pour le client {{FirstName}} {{LastName}} qui possède une petite boutique à {{City}}. Le produit à mettre en avant est la lampe solaire « SunPower ». Mentionne le prix 15 USD, la garantie 12 mois et propose un code promo « AFRICA10 ». Termine par « À très vite ». »
```  

### 2.4. Test et validation  
* **Mode test** : la plupart des plateformes offrent un “Run once” qui exécute le flux avec des données factices.  
* **Vérifier le rendu** : le texte doit être exempt de balises HTML inattendues, l’image doit être correctement affichée dans le client de messagerie.  
* **Boucle de feedback** : intégrer une étape qui enregistre les taux d’ouverture (via le pixel de suivi SendGrid) afin d’ajuster le prompt dans le temps.

---

## 3. Automatiser la facturation pour un commerce de détail  

### 3.1. Sources de données  
* **Google Sheets** : tableau `Commandes` contenant `ClientID`, `Produit`, `Quantité`, `Date`.  
* **Base locale** : SQLite embarquée dans l’application mobile (Adalo).  
* **API comptable** (ex. : **Paystack**, **Flutterwave**) pour récupérer le statut de paiement.

### 3.2. Génération de facture texte + QR‑code image  
1. **Action IA texte** : créer le corps de la facture (tableau, total, conditions).  
2. **Action IA image** : générer un QR‑code contenant l’URL de paiement (via **Stable Diffusion** ou un service dédié).  

```javascript
// n8n Function node – création du prompt
const { clientName, orderId, items, total } = $json;
return [
  {
    json: {
      prompt: `Write a French invoice for ${clientName}, order #${orderId}. Include a table with items: ${items.join(', ')}, total amount ${total} USD. End with "Merci pour votre confiance". Return only plain text.`
    }
  }
];
```

### 3.3. Envoi automatisé au client  
* **Trigger** : mise à jour du statut `Paid` dans la base comptable.  
* **Action** : appel à l’API SendGrid avec le texte de la facture en pièce jointe PDF (conversion via **PDFShift**) et le QR‑code en image.  

```yaml
# PDFShift HTTP request
method: POST
uri: https://api.pdfshift.io/v3/convert/
headers:
  Content-Type: application/json
body: |
  {
    "source": "{{outputs('HTTP_-_OpenAI_Completion')?['choices'][0]['message']['content']}}",
    "landscape": false,
    "margin": "1cm"
  }
```

### 3.4. Gestion des erreurs et limites d’API  
| Situation | Action corrective | Exemple de mise en œuvre |
|-----------|-------------------|--------------------------|
| **Quota API dépassé** | Implémenter un **cache** des factures déjà générées (Redis ou Google Sheet). | Avant d’appeler l’API, vérifier si `orderId` existe déjà dans le cache. |
| **Erreur de génération d’image** | Fallback vers un QR‑code générique pré‑généré. | Utiliser un `Try/Catch` dans le nœud Function de n8n. |
| **E‑mail non délivré** | Enregistrement du statut `failed` et relance après 24 h. | Ajouter une étape “Delay” puis “Retry”. |

---

## 4. Support client IA : chatbot et réponses par e‑mail  

### 4.1. Capture du ticket  
* **Formulaire web** (Bubble) ou **WhatsApp Business API** (via Twilio).  
* Chaque soumission crée un enregistrement `Ticket` contenant `Subject`, `Description`, `CustomerID`.  

### 4.2. Classification avec IA texte  
Utiliser un modèle de **zero‑shot classification** pour router le ticket :  

```json
{
  "model": "gpt-4o-mini",
  "messages": [
    {"role":"system","content":"You are a ticket routing assistant. Classify the following request into one of: [Facturation, Livraison, Produit, Technique]. Return only the label."},
    {"role":"user","content":"{{Ticket.Description}}"}
  ],
  "temperature":0
}
```

Le label détermine le **chemin** du workflow : facturation → génération de facture, technique → appel à un technicien, etc.

### 4.3. Génération de réponse et image explicative  
* **Réponse texte** : prompt qui résume la demande et propose une solution.  
* **Image** : si le ticket concerne un produit défectueux, générer une illustration montrant le montage correct.  

```text
« Explique en français, en 2 paragraphes, comment remplacer le filtre du réfrigérateur SunCool modèle X200. Ajoute une illustration simple montrant le filtre en place. »
```

L’image est ensuite **encodée en base64** et insérée dans le corps du mail (compatible avec SendGrid).

### 4.4. Escalade humaine  
Lorsque le score de confiance du modèle est inférieur à 0,7, le workflow crée automatiquement une tâche dans **Microsoft Teams** ou **Slack** et notifie le responsable du support.  

```yaml
# Condition node (Power Automate)
if: {{outputs('Classification')?['choices'][0]['message']['content']}} == "Facturation" && {{confidence}} < 0.7
then: Create Teams task → "Vérifier le ticket #{{Ticket.ID}}"
else: Send AI‑generated reply
```

---

## 5. Cas d’usage agriculture : rapport météo et recommandations visuelles  

### 5.1. Collecte de données météo  
Utiliser l’API publique **Open-Meteo** (gratuit, sans clé) :  

```http
GET https://api.open-meteo.com/v1/forecast?latitude=9.08&longitude=7.48&daily=weathercode,temperature_2m_max,temperature_2m_min&timezone=Africa/Lagos
```

Les paramètres `latitude`/`longitude` sont récupérés depuis la base de données des parcelles (`FarmPlot`).

### 5.2. Prompt pour rapport texte  
```text
« Rédige un rapport météo de 3 paragraphes pour la région de {{RegionName}} du {{StartDate}} au {{EndDate}}. Indique les températures maximales/minimales, le risque de pluie et les conseils agronomiques (irrigation, protection des cultures). Utilise un ton pédagogique. »
```

### 5.3. Création d’infographie IA  
Prompt d’image :  

```text
« Create a colorful infographic for farmers in {{RegionName}} showing daily max/min temperature, rain probability, and a simple icon for each recommended action (e.g., water, shade). Use a hand‑drawn African style. »
```

Le résultat est stocké sur **Cloudinary** et le lien partagé via **WhatsApp** (Twilio) ou **SMS** (via **Africa's Talking**).

### 5.4. Diffusion via SMS ou WhatsApp  
* **Trigger** : planificateur quotidien à 06 h00.  
* **Action** : appel à l’API d’envoi (WhatsApp Business) avec le texte du rapport et le lien de l’infographie.  

```yaml
method: POST