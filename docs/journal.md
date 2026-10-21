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

### Workflow 04 — Rédaction
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

### Relecture par une seconde IA
Objectif à terme : ne plus relire chaque message à la main. Après la rédaction, un second modèle, d'un autre éditeur pour ne pas reproduire les mêmes biais, relit le texte avec la même fiche de profil.

- Il vérifie : surévaluation des compétences, faits inventés sur l'entreprise, répétitions (notamment « ce message est automatisé » dit deux fois), artefacts (listes à puces, formules de politesse en trop, signature, lien).
- Il rend un verdict `ok`, `corrige` (retouches minimales) ou `rejete`, avec la liste des problèmes. Le texte d'origine est conservé pour comparer.
- Premier lot de 8 : 5 `ok`, 3 `corrige`, à chaque fois pour la même répétition. Deux réponses vides au départ : DeepSeek manquait de tokens pour sa réflexion (limite relevée à 12 000).
- La fiche de profil précise mon niveau réel : prototypeur plutôt que développeur, SQL surtout via des interfaces graphiques, certaines compétences du CV seulement côtoyées. Par défaut, le message ne cite aucun langage ni outil.

### Toutes les branches informatiques
Un seul workflow de rédaction pour toutes les spécialités. Ce qui change d'une branche à l'autre (CV joint, cible d'alternance, angle et niveau réel dans le domaine) vit dans une table `lba_branches`, éditée depuis un petit workflow de configuration.

- Développement, data/IA, cybersécurité, systèmes et réseaux : un CV et des consignes chacun. Les consignes data et cyber reprennent la démarche de lettres que j'avais écrites à la main (le réflexe de chercheur qui questionne la donnée avant de l'analyser ; la dimension d'enquête de la cybersécurité).
- Test sur une entreprise data et une entreprise cyber : messages cohérents avec la branche. La relecture a corrigé une surévaluation (« plusieurs années d'UX research » au lieu d'environ un an) et remis en prose une liste à puces.
- Quand la recherche web est floue (homonymes, résultats sans rapport), le message reste général sur l'entreprise au lieu d'inventer : la consigne fonctionne.

### Refus de l'habilitation LBA : un seul canal, l'email
LBA réserve la route d'envoi aux plateformes partenaires et m'a refusé l'habilitation `applications:write`. Plutôt que de contourner ce refus en automatisant le bouton de leur site, le pipeline contacte les entreprises directement. LBA ne sert plus qu'à les repérer.

- Conséquence positive : il n'y a plus deux canaux (simplifiée / spontanée) mais un seul, et toutes les entreprises deviennent des cibles, plus seulement celles qui ont un `recipient_id`.
- Nouveau workflow 03, recherche du contact, placé avant la rédaction (qui devient 04) pour que les numéros suivent l'ordre d'exécution : Tavily récupère le contenu brut des pages, DeepSeek extrait l'email de recrutement, à défaut l'email de contact général, et le formulaire. Garde-fou : un email n'est retenu que s'il apparaît mot pour mot dans les pages récupérées, ce qui écarte les adresses inventées ou devinées.
- Les entreprises avec seulement un formulaire sont traitées à la main, avec le message rédigé.
- Nouveau workflow 05, envoi depuis mon Gmail, en texte brut, CV de la branche en pièce jointe, une candidature par créneau, 20 créneaux par jour en semaine. Un envoi à froid trop rapide depuis une même adresse finit en spam : le rythme est volontairement bas.
- Validation manuelle tant que la calibration n'est pas terminée : seules les lignes que je passe en `valide` partent.
- Le CV est enregistré avec le brouillon (`cv_drive_id`) : l'envoi joint exactement le CV qui a servi à rédiger le message, sans repasser par la table des branches.

### Dédoublonnage en amont
Une même entreprise remonte souvent sous plusieurs établissements (un SIRET par site) qui partagent la même adresse de recrutement. Un premier contrôle placé à l'envoi évitait le doublon, mais trop tard : les lettres étaient déjà rédigées et des créneaux d'envoi perdus. Le dédoublonnage se fait donc le plus tôt possible :
- à la collecte, un seul établissement par SIREN (le premier rencontré, donc le plus proche) ; le contrôle en table compare le début du SIRET, ce qui couvre aussi les lignes déjà présentes ;
- à la recherche du contact, les entreprises sont enregistrées une à une ; si l'adresse trouvée est déjà attribuée à une autre ligne (filiale d'un même groupe, SIREN différent), la ligne passe en `doublon` et n'est jamais rédigée.
Les lignes en double sont marquées plutôt que supprimées : supprimées, elles reviendraient à la collecte suivante.

### Recherche du contact : explorer le site lui-même
Premier test réel de la recherche du contact : 0 adresse sur 2 entreprises. Sur la première, l'adresse figurait pourtant sur son site, mais Tavily n'en renvoyait qu'un extrait tronqué (`bonjour@`). La recherche web seule ne suffit pas : il faut aller lire le site.

- Le site officiel est pris dans la réponse de l'IA, sinon dans les résultats web (domaine qui contient le nom de l'entreprise), sinon par une seconde recherche dédiée.
- Le workflow télécharge directement la page d'accueil, repère les liens utiles (contact, mentions légales, recrutement, équipe…) et visite une dizaine de pages au plus. Les mentions légales sont une bonne source : la loi impose d'y faire figurer un moyen de contact.
- Les emails sont repérés en clair, dans les liens `mailto:` et sous leurs formes masquées (protection Cloudflare, `[at]`, `&#64;`). Les formulaires de contact sont détectés.
- Choix de l'adresse par ordre de préférence : recrutement@ ou rh@ du domaine de l'entreprise, puis contact@ ou bonjour@, puis une autre adresse du domaine, puis une messagerie gratuite affichée sur le site (fréquent dans les petites structures). Jamais l'hébergeur, l'agence qui a fait le site, le DPO ou un no-reply. Toujours aucune adresse devinée.
- Piège rencontré : l'objet `URL` de JavaScript n'existe pas dans les expressions n8n. Le domaine du site n'était jamais reconnu, et la bonne adresse, pourtant trouvée, était écartée. Les URL sont maintenant découpées par expressions régulières.
- Résultat sur le même échantillon : l'adresse et le formulaire de la première entreprise sont trouvés sur son site, une deuxième entreprise est joignable par email. Celle qui n'a pas de site reste sans contact.

### Recherche du contact : premier passage complet
Sur les 28 entreprises IT retenues, le premier passage donnait 15 emails, 3 formulaires et 10 sans contact. En relisant les adresses une par une, plusieurs n'étaient pas envoyables : une messagerie gratuite sans lien avec une entreprise de 50 salariés, l'adresse d'une autre société, des annuaires pris pour le site officiel, un site sans rapport retenu parce que son nom commençait comme celui de l'entreprise.

- Le site officiel n'est accepté que si son domaine reprend le nom de l'entreprise, y compris quand c'est l'IA qui le propose. Un mot du nom ne compte qu'en début ou en fin de domaine ; une partie du nom doit en couvrir au moins 60 %. Les annuaires sont exclus.
- Les sous-domaines et autres extensions d'un même nom comptent comme le même domaine (`fr.exemple.com` pour `france.exemple.com`, `.com` pour `.fr`). Une adresse RH restait sinon ignorée.
- Messagerie gratuite seulement si elle est affichée sur le site d'une petite structure (moins de 10 salariés). Adresses de service écartées : DPO, support, presse, accessibilité, facturation, achats.
- Un formulaire passe avant l'adresse d'une autre société du groupe ; les formulaires des pages légales (protection des données, cookies) sont ignorés.
- Nouvelle étape : une recherche Tavily limitée au domaine officiel. Elle rattrape les sites dont le contenu est généré en JavaScript et repère les portails carrières (enregistrés comme formulaires, à remplir à la main).
- Résultat : 15 emails, 7 formulaires ou portails, 6 sans contact. Les 6 restants n'ont pas de site identifiable ou n'affichent qu'une adresse de DPO.

### Reste à faire
- Vérifier à la main un échantillon de classifications pour mesurer l'accord avec Jev.
- Branche `jobs` (offres déposées sur LBA) dans la collecte.
- Décider quoi envoyer aux ESN généralistes et aux intégrateurs ERP/conseil (pas encore de branche).
- Calibrer la rédaction sur un échantillon par branche avant de lancer en volume.
