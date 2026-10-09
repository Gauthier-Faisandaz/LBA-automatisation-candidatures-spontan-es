# Journal de bord

## 2026-10-09 — Mise en place

### Objectif initial
Trois phases :
1. Lister les entreprises LBA sur des codes ROME ciblés et les trier selon qu'elles acceptent ou non la candidature simplifiée.
2. Pour celles qui l'acceptent : recherche rapide sur l'entreprise, rédaction d'un message adapté, envoi via l'API.
3. Pour les autres : trouver une adresse ou un formulaire de contact sur leur site, puis candidater manuellement ou par email.

### Exploration de l'API
- L'espace développeurs LBA expose une route de recherche et une route d'envoi de candidature : pas besoin de scraper le site ni de simuler le bouton « candidature simplifiée ».
- Le marqueur fiable est `apply.recipient_id` : s'il est présent, l'entreprise peut recevoir une candidature via l'API.
- Je n'ai pas testé l'envoi avec ma clé : si elle avait déjà le droit d'envoyer, un test aurait produit une vraie candidature. La route exige l'habilitation `applications:write`, à demander pour une clé de production.

### Workflow 01 — Collecte
- Première version : quelques codes ROME en dur → Split Out → HTTP Request → mapping → table.
- Choix « insérer seulement si le SIRET est absent » plutôt qu'un upsert : un upsert remettrait `statut` à `a_traiter` à chaque collecte et ferait perdre le suivi.
- Colonne `distance_km` supprimée : l'API ne la renvoie pas.
- Les codes ROME sont ensuite lus depuis une data table : on peut en ajouter ou en retirer sans toucher au workflow.
- Limite connue : une entreprise remontée sur plusieurs codes ROME garde le premier rencontré. C'est pour ça que le filtre « informatique » de l'étape suivante combine code ROME **et** code NAF.

### Workflow 02 — Classification par spécialité
Besoin : je n'envoie pas le même CV à une ESN réseau, une boîte de cybersécurité ou un éditeur de logiciel. Le CV « concepteur développeur d'applications » ne sert que pour les entreprises orientées développement.

- Recherche web Tavily (nom + adresse) → LLM avec sortie structurée : `specialite`, `resume_activite`, `site_web`.
- Le prompt impose de vérifier que les résultats parlent bien de la même entreprise, sinon `inconnu`, et d'exclure les annuaires pour le site web.
- **Limites de débit rencontrées** :
  - Groq (gratuit) : 8 000 tokens/minute sur `gpt-oss-120b`, dépassé dès le deuxième appel.
  - Mistral : 429 dès le premier appel (quota du compte).
  - Gemini (`gemini-3-flash-preview`) : fonctionne, mais quota gratuit vite atteint (une dizaine d'appels).
- Corrections : délai de 15 s entre deux appels, retry, et `onError: continueRegularOutput` suivi d'un filtre, pour enregistrer les entreprises classées même si certaines échouent. Avant ça, une seule erreur faisait perdre tout le lot.
- Qualité observée sur le premier lot : classifications cohérentes, sites officiels correctement identifiés.

### Workflow 03 — Rédaction
- Le CV est lu directement depuis Google Drive puis converti en texte : le prompt s'appuie sur le vrai CV, pas sur un résumé figé.
- Contraintes du prompt : 120–180 mots, 3 paragraphes, rien d'inventé, pas d'école ni de date si elles ne figurent pas dans le CV.
- Premier essai tronqué en plein milieu : Gemini 3 compte ses tokens de réflexion dans la limite de sortie. Passage de 1 024 à 4 096 tokens.
- Les messages sont enregistrés en `brouillon` : je relis et passe en `valide` ce qui part.

### Passage à OpenRouter et à Jev
Les quotas gratuits bloquant le volume, passage à des modèles payants via OpenRouter, choisis pour le rapport qualité-prix :

- **Classification → Jev (TypeSafe).** Jev n'écrit pas de texte : on lui envoie un état (les infos de l'entreprise) et des questions typées, il renvoie un choix, une distribution de probabilités et une confiance. C'est exactement le besoin d'un tri, sans parsing de JSON ni prompt de format. Appel via l'API Decisions d'OpenRouter (`POST /api/v1/systemone`, modèle `jev-1.13`), avec deux questions :
  - `specialite` (choice) : une des 7 catégories ;
  - `meme_entreprise` (noul, 0–1) : les résultats web parlent-ils bien de cette entreprise ? En dessous de 0,5, la spécialité est forcée à `inconnu`.
  Coût mesuré : environ 0,00006 $ par entreprise. Latence inférieure à 1 s.
- **Rédaction → Claude Haiku 5.5** (0,10 $ / 0,50 $ par million de tokens), qui reste un modèle génératif.

### Ce que la confiance de Jev a révélé
- Premier essai : les résultats Tavily étaient dominés par les annuaires (societe.com, Pappers…) qui ne donnent que le code NAF. Jev répondait alors « développement » avec une confiance de 1 pour toute entreprise en 62.01Z, y compris une société en réalité orientée maintenance matérielle.
- Exclure les annuaires sans changer la requête a produit l'effet inverse pour les petites structures sans site : du bruit (BODACC, homonymes). La question `meme_entreprise` l'a bien détecté (scores de 0,17 à 0,56).
- Requête finale : nom exact entre guillemets + ville, annuaires exclus. Sur le cas ambigu, Jev répartit sa probabilité à 48/49 entre développement et réseau, avec une confiance de 0,4 : c'est exactement le signal voulu pour envoyer la ligne en relecture manuelle.
- La rédaction ne traite donc que les entreprises `developpement` avec une confiance ≥ 0,7.

### Prompt de rédaction v2 : un message qui assume d'être automatisé
Constat : environ 90 candidatures manuelles, classiques, sans retour. Le message change donc de parti pris : il dit ouvertement qu'il est envoyé par cette automatisation, que je l'ai construite moi-même, et renvoie vers ce dépôt. Le message devient une démonstration de compétences plutôt qu'une lettre de plus.

- Le style s'inspire de mes lettres précédentes : « Bonjour, », phrases directes, profil atypique présenté franchement, honnêteté sur le fait que je ne suis pas encore développeur professionnel.
- Le prompt fixe une liste de faits autorisés (parcours, compétences, certification en préparation présentée comme telle) et interdit d'en inventer.
- Une seule phrase sur l'entreprise, tirée de son activité réelle ; si les infos sont floues, le message reste général.
- Modèle : Claude Haiku 5.5. Les 8 premiers brouillons ont coûté moins d'un centime au total.
- Observations sur ce premier lot : la personnalisation est juste (logiciels pour notaires, pour collectivités, pour l'immobilier, agences e-commerce) ; certains messages dépassent la longueur visée et répètent « ce message est automatisé » ; la structure est très proche d'un message à l'autre.

### Reste à faire
- Obtenir l'habilitation `applications:write`, puis construire le workflow d'envoi (traitement des `valide`, 10 envois/min max, enregistrement de `application_id`).
- Vérifier à la main un échantillon de classifications pour mesurer l'accord avec Jev.
- Branche `jobs` (offres déposées sur LBA) dans la collecte.
- Phase 3 : recherche de contact pour les entreprises sans candidature simplifiée.
- CV et prompts pour les autres spécialités informatiques (réseau/infra, cybersécurité, data, QA).
