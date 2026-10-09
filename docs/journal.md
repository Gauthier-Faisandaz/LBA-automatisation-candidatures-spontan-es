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
- Première version : liste de 4 codes ROME en dur → Split Out → HTTP Request → mapping → table. 550 entreprises, dont 291 en candidature simplifiée.
- Choix « insérer seulement si le SIRET est absent » plutôt qu'un upsert : un upsert remettrait `statut` à `a_traiter` à chaque collecte et ferait perdre le suivi.
- Colonne `distance_km` supprimée : l'API ne la renvoie pas.
- Les codes ROME sont ensuite lus depuis la table déjà utilisée par mon projet JobDataStudy (26 codes) → 1 475 entreprises.
- Limite connue : une entreprise remontée sur plusieurs codes garde le premier rencontré. C'est pour ça que des entreprises informatiques se retrouvent classées en comptabilité, et que le filtre « informatique » combine code ROME **et** code NAF.

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

### Reste à faire
- Obtenir l'habilitation `applications:write`, puis construire le workflow d'envoi (traitement des `valide`, 10 envois/min max, enregistrement de `application_id`).
- Choisir un fournisseur LLM au débit suffisant pour classer quelques centaines d'entreprises.
- Branche `jobs` (offres déposées sur LBA) dans la collecte.
- Phase 3 : recherche de contact pour les entreprises sans candidature simplifiée.
- CV et prompts pour les autres filières (réseau/support, maintenance, électricité, comptabilité).
