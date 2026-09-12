## Créer le projet : de l’idée à l’environnement de développement

### Choisir la plateforme low‑code

Pour un premier prototype, **Bubble** est souvent recommandé aux PME africaines : il fonctionne dans le navigateur, ne nécessite aucune installation locale (utile quand la connexion internet est intermittente) et propose une version gratuite suffisante pour tester.  
> *Voir chapitre 3* pour la comparaison détaillée des plateformes.

### Créer un compte et un nouveau projet

1. **Inscription** : rendez‑vous sur https://bubble.io, créez un compte avec votre adresse e‑mail professionnelle.  
2. **Nouveau projet** : cliquez sur *New app*, donnez‑lui un nom explicite (ex. `GenTextMarketing`) et choisissez le template *Blank* pour partir d’une page vierge.  
3. **Sauvegarde automatique** : Bubble enregistre chaque modification, mais pensez à activer la fonction *Version control* (disponible dès le plan « Personal ») afin de pouvoir revenir en arrière en cas d’erreur.

### Structurer l’application

| Élément | Rôle | Placement conseillé |
|---------|------|----------------------|
| Input texte | Saisir la description du produit ou service | En haut de la page, largeur 100 % |
| Bouton « Générer » | Déclencher l’appel IA | À droite de l’input ou sous forme de bouton plein écran sur mobile |
| Zone de résultat | Afficher le texte marketing généré | Sous le bouton, avec un fond légèrement différent pour le mettre en valeur |
| Loader (icône) | Indiquer que l’IA travaille | Visible uniquement pendant l’appel API |

## Configurer l’API d’IA générative

### Obtenir les clés d’accès

1. Créez un compte sur **OpenAI** (ou Cohere, selon vos préférences).  
2. Dans le tableau de bord, générez une **API key**. Copiez‑la ; vous en aurez besoin uniquement dans Bubble, jamais dans le code client.

### Stocker la clé en toute sécurité

Dans Bubble :

1. Ouvrez le menu *Settings* → *API* → *API Keys*.  
2. Créez une nouvelle variable d’environnement nommée `OPENAI_API_KEY` et collez votre clé.  
3. Cochez *Private* ; la clé ne sera jamais exposée côté navigateur.

### Ajouter le plugin d’appel API

1. Dans le panneau latéral, cliquez sur *Plugins* → *Add plugins*.  
2. Recherchez **API Connector** et installez‑le.  
3. Ouvrez le plugin, cliquez sur *Add another API* → nommez‑le `OpenAI`.  

#### Définir l’appel « GenerateText »

| Champ | Valeur |
|------|--------|
| **Name** | `GenerateText` |
| **Use as** | Action |
| **Method** | POST |
| **URL** | `https://api.openai.com/v1/completions` |
| **Headers** | `Authorization: Bearer <API_KEY>`<br>`Content-Type: application/json` |
| **Body (JSON)** | ```json { "model": "text-davinci-003", "prompt": "<prompt>", "max_tokens": 150, "temperature": 0.7 }``` |

*Remarque* : le champ `<prompt>` sera remplacé dynamiquement par le texte fourni par l’utilisateur (voir section suivante).

## Concevoir l’interface utilisateur (UI)

### Ajouter les éléments visuels

1. **Input texte** : glissez‑déposez un *Input* depuis le volet *Design*.  
   - *ID* : `product_desc`  
   - *Placeholder* : « Décrivez votre produit (ex. : savon à l’huile d’argan) »  

2. **Bouton** : ajoutez un *Button*.  
   - *ID* : `btn_generate`  
   - *Texte* : « Générer le texte marketing »  

3. **Texte de résultat** : insérez un *Text* vide.  
   - *ID* : `txt_result`  
   - *Style* : police légèrement plus grande, couleur sombre sur fond clair.  

4. **Loader** : ajoutez une icône *Spinner* (ou utilisez le composant *Loading*).  
   - *ID* : `loader`  
   - *Visible only when*: `State is loading` (défini dans le workflow, voir plus bas).

### Organiser avec des groupes réactifs

Pour garantir une bonne expérience sur mobile et sur ordinateur, placez tous les éléments dans un **Group** (`grp_main`) avec la propriété *Responsive* activée. Utilisez les *Margins* et *Padding* de manière cohérente : 10 px autour de chaque composant, 20 px entre le champ d’entrée et le bouton.

## Implémenter le workflow d’appel IA

### Créer le workflow du bouton

1. Sélectionnez le bouton `btn_generate`, puis cliquez sur *Start/Edit workflow*.  
2. Ajoutez une action **Set state** sur le groupe `grp_main` : créez un état nommé `loading` de type *yes/no* et passez‑le à `yes`. Cela déclenchera l’affichage du loader.  

3. **Appel API** : choisissez *Plugins* → *OpenAI – GenerateText*.  

   - **Prompt dynamique** : dans le champ `<prompt>`, insérez la formule suivante :  

     ```text
     Rédige un texte marketing convaincant (max 150 mots) pour le produit suivant : {{Input product_desc's value}}. Met en avant les bénéfices pour le consommateur africain, en insérant éventuellement une référence locale (ex. : marché de Lagos, boutique de Kigali).
     ```

   - **Paramètres** : laissez `max_tokens` à 150, `temperature` à 0.7 (équilibre créativité/contrôle).  

4. **Gestion de la réponse** : ajoutez une action *Set state* sur `grp_main` : créez un état `generated_text` (type texte) et affectez‑lui la valeur `Result of step 2 (GenerateText)'s choices:first item's text`.  

5. **Mise à jour du texte affiché** : ajoutez une action *Element actions* → *Set text* sur `txt_result` : `grp_main's generated_text`.  

6. **Arrêt du loader** : ajoutez une dernière action *Set state* : `grp_main's loading = no`.

### Gestion des erreurs

Ajoutez un *Event* : *When API GenerateText fails*.  
- Action : *Alert* : « Impossible de contacter le service IA ; vérifiez votre connexion internet ou la validité de la clé API. »  
- Réinitialisez `loading` à `no` pour éviter que le spinner reste bloqué.

## Tester rapidement le prototype

### Mode prévisualisation

1. Cliquez sur le bouton *Preview* en haut à droite.  
2. Dans la fenêtre qui s’ouvre, saisissez une courte description de produit (ex. : « T-shirt en coton bio fabriqué à Accra »).  
3. Cliquez sur *Générer le texte marketing*. Le loader doit apparaître pendant quelques secondes, puis le texte généré s’affichera dans la zone prévue.

### Vérifier la pertinence du résultat

- **Clarté** : le texte doit être lisible et sans jargon technique.  
- **Localisation** : assurez‑vous que le prompt a bien intégré une référence locale (ex. : « parfait pour les journées ensoleillées de Nairobi »).  
- **Longueur** : respectez la contrainte de 150 mots ; si le texte dépasse, ajustez `max_tokens` dans le workflow.

### Itération rapide

- Modifiez le **prompt** pour tester différentes tonalités (plus formel, plus humoristique).  
- Changez le **temperature** : une valeur plus élevée (0.9) donnera des réponses plus créatives, tandis qu’une valeur basse (0.3) les rendra plus précises.  
- Enregistrez chaque version dans le *Version control* pour pouvoir comparer les performances.

## Enrichir le prototype : persistance et partage

### Sauvegarder les résultats dans Google Sheets

1. **Plugin Google Sheets** : installez‑le via le Marketplace.  
2. Créez une feuille avec les colonnes `Date`, `Produit`, `TexteMarketing`.  
3. Dans le workflow, après l’étape d’affichage du texte, ajoutez une action *Create a new row* : remplissez les champs avec `Current date/time`, `Input product_desc's value`, `grp_main's generated_text`.  
4. Cette persistance permet aux équipes de suivi commercial d’accéder aux contenus générés sans passer par l’interface Bubble.

### Partager le prototype avec les parties prenantes

- **Mode public** : dans *Settings* → *Domain / URL*, activez le sous‑domaine gratuit (`yourapp.bubbleapps.io`).  
- **Accès restreint** : activez la fonction *Password protection* pour que seuls les collaborateurs de la PME puissent tester l’app.  

## Optimiser le prototype avant la mise en production

### Limiter les appels inutiles

- **Déclencheur conditionnel** : ajoutez une condition `Only when Input product_desc's value is not empty` avant l’appel API.  
- **Déduplication** : créez un état `last_prompt`. Avant d’appeler l’API, comparez le nouveau prompt avec `last_prompt` ; si identiques, réutilisez le texte déjà généré.

### Gestion du coût d’API

OpenAI facture à la tokenisation ; pour éviter les dépassements :

- **Baisser `max_tokens`** à 100 si les résultats restent pertinents.  
- **Mettre en cache** les réponses fréquentes : créez une table *Cache* dans Bubble (type *Data* → *Things*) avec les champs `Prompt` et `Response`. Avant chaque appel, interrogez cette table ; si le prompt existe, renvoyez la réponse stockée.

### Tester la robustesse hors ligne

- **Mode hors‑connexion** : activez le *Offline mode* de Bubble (voir *Settings* → *General*) pour que l’interface reste utilisable même si la connexion se coupe. Le loader affichera alors un message d’erreur clair et proposera de réessayer.

## Déployer votre première version

1. **Vérifier les quotas** : dans le tableau de bord OpenAI, assurez‑vous que le plan choisi (Free tier ou payant) correspond à la fréquence d’utilisation prévue.  
2. **Passer en mode production** : dans *Settings* → *Domain / URL*, ajoutez votre propre domaine (ex. `genmarketing.entreprise.africa`).  
3. **Configurer le certificat SSL** : Bubble fournit automatiquement le certificat ; activez simplement le bouton *Enable SSL*.  
4. **Informer les utilisateurs** : préparez un court e‑mail (ou SMS) contenant le lien, le mot de passe d’accès et une courte vidéo de démonstration (30 s) montrant comment saisir le produit et récupérer le texte.

## Points clés

- **Sélection de la plateforme** : Bubble offre un environnement complet, accessible même avec une connexion intermittente.  
- **Gestion sécurisée des clés API** : stockez les clés dans les variables d’environnement de la plateforme, jamais côté client.  
- **Prompt dynamique** : construisez le texte de l’appel IA à partir de la saisie utilisateur, en incluant des références locales pour renforcer la pertinence.  
- **Workflow structuré** : séparez clairement les étapes de chargement, d’appel API, de traitement de la réponse et de mise à jour UI pour faciliter le débogage.  
- **Gestion des coûts** : limitez `max_tokens`, mettez en cache les réponses et surveillez les quotas d’OpenAI.  
- **Persistance et partage** : enregistrez chaque génération dans Google Sheets (ou une base Bubble) pour créer un historique exploitable par les équipes marketing.  
- **Déploiement progressif** : commencez par un sous‑domaine gratuit, puis migrez vers votre propre domaine avec SSL dès que le prototype a fait ses preuves.  

Ce protocole vous permet de passer rapidement d’une idée à une application fonctionnelle, tout en respectant les contraintes techniques et budgétaires typiques des PME africaines. Vous disposez désormais d’une base solide pour enrichir le prototype (intégration d’images, automatisation de campagnes e‑mail, etc.) dans les chapitres suivants.