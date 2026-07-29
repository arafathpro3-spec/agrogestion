# Ferme Condé Agro-Pastorale — MVP (Base commune + Finances/Poids)

## Identité de l'entreprise
L'app est maintenant à l'image de **Ferme Condé Agro-Pastorale** : nom dans l'en-tête, l'écran de connexion, les reçus PDF et le bilan, logo en icône PWA. N'oublie pas d'héberger `logo-header.png` et `logo-512.png` au même endroit que `index.html` — ce sont eux qui apparaissent dans l'en-tête et comme icône d'application.

## Audit de la base de données (corrections importantes)
En repassant sur `schema.sql`, deux problèmes réels ont été trouvés et corrigés — **si tu as déjà commencé à exécuter une version précédente du script sur Supabase, réexécute `schema.sql` en entier, il est maintenant correct de bout en bout** :

1. **Bug bloquant** : une instruction modifiait la table `sales` avant même sa création — le script s'arrêtait en erreur à ce moment-là et tout ce qui suivait (ventes, créances, stocks, agro...) n'était jamais créé. Corrigé : l'ordre respecte maintenant les dépendances.
2. **Risque de course en multi-utilisateurs** : les quantités de stock et le statut des paiements étaient recalculés côté app (lire, puis réécrire), ce qui peut perdre la saisie d'un collègue si deux personnes agissent au même moment sur le même article/la même vente. Corrigé avec des triggers côté base (comme celui déjà en place pour les paiements) qui recalculent la valeur exacte de façon atomique, quel que soit le nombre de personnes connectées en même temps. L'app continue d'afficher une valeur immédiate en local (utile hors ligne), mais c'est toujours le calcul côté serveur qui fait foi.
3. **Amélioration** : la mortalité saisie sur un lot de volailles décrémente maintenant automatiquement l'effectif du lot (`current_count`), comme prévu au cahier des charges — avant, il fallait le faire à la main.

## Ce qui est livré
- `index.html` — l'app complète (PWA offline-first, un seul fichier HTML/CSS/JS)
- `schema.sql` — schéma Supabase (à exécuter dans l'éditeur SQL de ton projet Supabase)
- `manifest.json` + `sw.js` — installabilité PWA et app-shell hors ligne
- `logo-header.png` (120×120) + `logo-512.png` (512×512) — le logo Ferme Condé, utilisé dans l'en-tête et comme icône d'application

## Nouveautés de cette passe
- **Reçu PDF** : bouton "🧾 Reçu PDF" dans le détail d'une vente
- **Relance WhatsApp** : bouton dans le détail d'une vente à crédit (si le client a un numéro de téléphone) — ouvre WhatsApp avec un message de rappel prérempli
- **Courbe de poids** : graphique automatique dès qu'un animal a 2 pesées ou plus
- **Recherche** : champ de recherche dans les onglets Animaux et Stocks
- **Stock d'aliments relié aux rations** : en enregistrant une ration (bovins/équidés) ou l'aliment distribué à un lot de volailles, tu peux choisir de déduire automatiquement du stock correspondant
- **Carte globale des parcelles** : bouton "🗺️ Voir la carte" dans l'onglet Agro (OpenStreetMap, gratuit, sans clé) — affiche toutes les parcelles géolocalisées
- **Multi-devise d'affichage** : sélecteur FCFA/USD/EUR/CNY dans l'en-tête. Tout reste stocké en FCFA — c'est juste un affichage converti (taux récupérés en ligne via une API gratuite, pas de clé requise). Les exports PDF/reçus restent en FCFA (valeur légale).

Non couvert dans cette passe (voir la section suivante) : vraie pagination sur de très gros volumes, validation côté serveur, onboarding guidé, indicateurs de chargement.

## QR Code / NFC
- Chaque fiche animale a un bouton "Générer le QR code" — le scanner (bouton caméra dans l'onglet Animaux) ouvre directement la fiche correspondante, même chose si le lien est ouvert dans un navigateur.
- Sur les appareils/navigateurs compatibles (Chrome Android essentiellement), un bouton "Écrire sur un tag NFC" apparaît en plus du QR — approche-le d'un tag NFC pour l'associer à la fiche.

## LA vraie cause trouvée : "updated_at" manquait sur 20 tables sur 24
Grâce à tes captures de la console développeur (message exact : `Could not find the 'updated_at' column of 'stock_movements' in the schema cache`), la cause réelle et définitive est maintenant claire — et elle explique tous les soucis de synchronisation qu'on a chassés ces derniers jours.

L'app ajoute automatiquement un champ `updated_at` à **chaque** enregistrement avant de l'envoyer à Supabase. Mais je n'avais créé cette colonne que sur 3 tables (`animals`, `plots`, `stock_items`) — les 20 autres (ventes, clients, pesées, mouvements de stock, événements de santé, récoltes, etc.) ne l'avaient pas. Résultat : **Supabase rejetait silencieusement (erreur 400) l'enregistrement de quasiment tout, depuis le début du projet.**

**Corrigé** : `schema.sql` ajoute maintenant cette colonne partout où elle manquait, et force Supabase à recharger son cache de schéma immédiatement.

**Étapes, dans cet ordre :**
1. **Ne vide surtout pas le cache/les données du site pour l'instant** — ta file d'attente locale contient les mouvements bloqués, et ils vont se rattraper tout seuls une fois le schéma corrigé
2. Recolle `schema.sql` en entier dans Supabase
3. Recharge simplement l'app (F5) — les éléments en attente vont se renvoyer automatiquement et, cette fois, réussir
4. Vérifie "son de blé" et quelques autres articles : les quantités devraient enfin correspondre à l'historique des mouvements
5. **N'exécute PAS `reparation-stock-unique.sql`** cette fois — les mouvements en attente vont déjà se rattraper tout seuls à l'étape 3 ; lancer la réparation en plus compterait tout en double. Il ne reste utile que si, après tout ça, un article précis affiche encore un chiffre visiblement faux.

## Bug critique corrigé : les échecs de synchronisation étaient invisibles et bloquaient tout derrière eux
Cause profonde trouvée suite à "ça ne se sauvegarde pas après actualisation" : la bibliothèque Supabase ne lève pas d'erreur automatiquement quand une écriture échoue — elle renvoie un objet `{error}` qu'il faut vérifier soi-même. Deux endroits du code ne le faisaient pas (la suppression d'un élément, et — plus gênant — rien ne vérifiait qu'un envoi avait vraiment réussi avant de le retirer de la file d'attente). Résultat : un envoi pouvait échouer silencieusement côté Supabase tout en étant considéré comme réussi côté app, qui l'effaçait de la file d'attente comme si de rien n'était. Au rechargement suivant, l'app retélécharge l'état réel du serveur — qui, lui, n'avait jamais reçu la donnée — et l'ancienne valeur revient.

En plus de ça : si un élément de la file d'attente échouait vraiment (au lieu d'être ignoré à tort), **tout ce qui était derrière lui dans la file restait bloqué indéfiniment**, sans aucun signal visible dans l'app.

**Corrigé sur les deux fronts :**
1. Chaque écriture vers Supabase vérifie maintenant vraiment si elle a réussi
2. Un élément qui échoue reste en attente et sera retenté, mais **ne bloque plus** les éléments suivants dans la file
3. **Nouveau** : si des éléments restent en attente de synchronisation, un bandeau orange apparaît maintenant en haut du tableau de bord, avec le détail de la dernière erreur si elle existe — ce qui était invisible avant se voit maintenant directement dans l'app

## Bug critique corrigé : les quantités de stock (et potentiellement paiements/mortalité) ne se recalculaient jamais réellement
Signalé par Bio : une entrée de stock isolée ne changeait pas la quantité affichée. Cause trouvée : les fonctions automatiques qui recalculent une valeur après une action (stock après un mouvement, statut d'une vente après un paiement, effectif d'un lot après une mortalité) tournaient avec les droits de l'utilisateur connecté — donc soumises aux règles de sécurité (RLS) de la base. Dans ce contexte, leur écriture pouvait être **silencieusement ignorée** (zéro ligne modifiée, aucune erreur visible) selon la façon dont Supabase évalue ces règles pour du code déclenché automatiquement.

**Corrigé** : les trois fonctions concernées tournent maintenant avec des droits élevés (`security definer`), qui contournent ce problème par construction — la même technique qui faisait déjà fonctionner correctement l'attribution du rôle "employé" à l'inscription.

**Étapes à suivre, dans l'ordre :**
1. Recolle `schema.sql` en entier dans Supabase (comme d'habitude)
2. **Exécute ensuite `reparation-stock-unique.sql` séparément, une seule fois** — il rattrape les mouvements déjà enregistrés mais jamais pris en compte dans la quantité affichée jusqu'ici. Ne le relance pas une deuxième fois, sinon il recompterait les mêmes mouvements en double.
3. Vérifie quelques articles pour confirmer que les quantités sont maintenant correctes

## Correctif : quantité de stock parfois figée à l'ancienne valeur
Bug de course trouvé dans la synchronisation : chaque enregistrement déclenche une tentative d'envoi vers Supabase, mais rien n'empêchait deux synchronisations de tourner en même temps (ex. : créer un article puis faire tout de suite une entrée dessus). Si l'envoi de la création de l'article n'était pas encore terminé quand celui du mouvement de stock partait, ce dernier pouvait arriver au serveur avant l'article lui-même — provoquant un rejet silencieux côté base (l'article n'existe pas encore), et le mouvement restait en attente indéfiniment. La quantité affichée localement restait donc bloquée à l'ancienne valeur dès qu'une resynchronisation ultérieure re-tirait l'état réel (toujours à l'ancienne valeur) depuis Supabase.

**Corrigé** : un verrou garantit maintenant qu'une seule synchronisation tourne à la fois, dans l'ordre exact des actions, avec relance automatique si de nouvelles actions arrivent pendant qu'une synchro est en cours. Si le problème persistait avant ce correctif, il ne devrait plus se reproduire — dis-le si ça arrive encore.

## Correctif important : le journal d'activité (et potentiellement d'autres modules) ne se créait pas chez les utilisateurs déjà actifs
Bug de fond trouvé : la base locale (IndexedDB) ne relit sa liste de "tables" que lorsqu'un numéro de version augmente. Ce numéro était resté bloqué à `1` depuis le début du projet, alors que de nombreuses tables ont été ajoutées en cours de route (Agro, Stocks, puis le journal d'activité). Résultat : sur un téléphone/navigateur qui avait déjà ouvert l'app avant l'ajout d'une table, cette table n'était jamais créée localement — d'où le journal d'activité invisible ou en erreur.

**Corrigé définitivement** : la version se calcule maintenant automatiquement à partir du nombre de tables (`DB_VERSION = STORES.length`). Ajouter une table à l'avenir déclenchera systématiquement la mise à jour chez tout le monde, sans plus jamais pouvoir oublier ce détail.

**Si le journal d'activité (ou autre chose) ne marchait toujours pas avant ce correctif** : il suffit de recharger la nouvelle version de `index.html` — la mise à niveau se fait automatiquement au premier chargement, aucune donnée existante n'est perdue. En dernier recours seulement (si un souci persiste), vider le cache/les données du site dans le navigateur force une base neuve — mais synchronise d'abord avec Supabase pour ne rien perdre si tu fais ça.

## Logo sur les exports
Le logo apparaît maintenant partout où c'est possible :
- **Reçu de vente (PDF)** — déjà en place
- **Export ventes & créances (PDF)** et **Bilan (PDF)** — logo ajouté en en-tête
- **Export Excel** — la bibliothèque Excel gratuite utilisée ne permet pas d'intégrer une image (fonctionnalité réservée aux versions payantes) ; à la place, le nom de l'entreprise apparaît en toutes lettres sur les deux premières lignes de la feuille

## Journal d'activité (date et heure de chaque action)
Bouton "🕒 Journal d'activité" sur le tableau de bord : chaque création, modification ou suppression — animal, vente, mouvement de stock, parcelle, etc. — est maintenant enregistrée automatiquement avec :
- La date **et l'heure précises** (ex. "26/07/2026 à 20:54")
- Le type d'action (➕ création, ✏️ modification, 🗑️ suppression)
- Une description lisible (ex. "Entrée de stock — 50 Sac de Maïs")
- L'email de la personne connectée qui a fait l'action (si l'app est connectée à Supabase)

C'est stocké dans une table à part (`activity_log`) qui ne peut être ni modifiée ni effacée via l'API une fois écrite — même un utilisateur mal intentionné avec un accès direct à la base ne peut pas trafiquer l'historique après coup, il peut seulement le consulter.

## Reçu de vente — corrections suite à ton retour
Tous les points de ta liste ont été traités :
- **Bug des montants corrigé** : "125/000" venait de la police par défaut de la bibliothèque PDF qui n'affiche pas correctement l'espace fine utilisée par le formatage français. Les montants s'affichent maintenant "125 000 FCFA" avec un espace normal, partout (reçu, export ventes, bilan).
- **Numéro de reçu unique** — déjà présent, inchangé (ex. `REC-20260721-A3F9C2`)
- **Échéance** mise en évidence dans un encadré rouge si la vente n'est pas soldée
- **Détails de l'animal/lot vendu** : espèce, nom/lot, race, n° oreille/puce (si renseignés sur la fiche animale)
- **Adresse complète du client** — affichée si renseignée sur sa fiche
- **Mode de paiement** (espèces / chèque / virement / mobile money) — à choisir à chaque vente comptant et à chaque acompte ; agrégé et affiché sur le reçu
- **Mention TVA** configurable (`COMPANY_VAT_MENTION` en haut du fichier — adapte-la à ton régime fiscal réel)
- **Ligne de signature et de cachet** en bas du reçu
- **Pied de page structuré** : adresse, téléphone, RCCM, IFU, mention TVA, regroupés en bas comme sur une facture normalisée

## Reçu de vente — facture normalisée (adaptée à une petite imprimante)
Le bouton "🧾 Reçu PDF" (dans le détail d'une vente) génère un vrai ticket au format 80mm (largeur standard des petites imprimantes de caisse/reçus), avec :
- Logo de l'entreprise en en-tête
- Nom, adresse, téléphone, **RCCM** et **IFU** (à renseigner une fois pour toutes, voir ci-dessous)
- Numéro de reçu unique (ex : `REC-20260721-A3F9C2`)
- Nom de l'animal/lot vendu (si la vente y est rattachée)
- Détail poids / prix au kilo / prix total / montant payé / reste à payer / échéance

**À renseigner une fois** en haut de `index.html`, juste après `SUPABASE_URL` :
```js
const COMPANY_ADDRESS = '';   // ex: 'Bohicon, Bénin'
const COMPANY_PHONE = '';     // ex: '+229 00 00 00 00'
const COMPANY_RCCM = '';      // ex: 'RB/BOH/23 B 1234'
const COMPANY_IFU = '';       // ex: '3202312345678'
```
Tant que ces champs sont vides, la ligne correspondante n'apparaît simplement pas sur le reçu — rien ne casse si tu ne les remplis pas tout de suite.

Le format est pensé pour une imprimante 80mm ; si la vôtre est en 58mm, cherche `const pageWidth = 80;` dans `index.html` et remplace par `58`.

## Export PDF / Excel
- Onglet Finances → Ventes & créances : boutons "Export PDF" et "Export Excel" pour le relevé complet des ventes/créances
- Tableau de bord : bouton "Exporter le bilan (PDF)" pour un résumé synthétique (CA, créances, trésorerie, effectifs, surface)

## Notifications de rappel
- Bouton "Activer les notifications" sur le tableau de bord (demande la permission du navigateur)
- Dès qu'une alerte urgente apparaît (créance en retard de +30j, stock bas, péremption dépassée), une notification s'affiche — **tant que l'app est ouverte dans le navigateur**
- Limite importante : ce ne sont pas de vraies notifications push en arrière-plan (app fermée). Pour ça, il faudrait un service côté serveur (clés VAPID + tâche planifiée, par exemple une Supabase Edge Function déclenchée par cron) — à prévoir si le besoin est confirmé

## Rôles & permissions
- Deux rôles : **gérant** et **employé** (table `user_roles`, un rôle "employé" est attribué automatiquement à chaque inscription)
- Seul un gérant peut supprimer une fiche animale, une parcelle ou un article de stock — la restriction est appliquée à la fois dans l'interface et dans la base (policies RLS), pas seulement côté écran
- **Pour passer le tout premier compte en gérant** : va dans Supabase → Table Editor → `user_roles`, trouve la ligne de ton compte et change `role` en `gerant`

## Météo (par parcelle)
- Ajoute les coordonnées GPS d'une parcelle (bouton "📍 Utiliser ma position actuelle" ou saisie manuelle) pour voir les prévisions à 5 jours (températures, pluie) directement dans sa fiche
- Utilise l'API gratuite Open-Meteo (aucune clé requise) — fonctionne uniquement en ligne

## Multi-utilisateurs (plusieurs personnes connectées en même temps)
- **Connexion par email/mot de passe** (Supabase Auth) — écran de connexion/inscription au démarrage.
- **Temps réel** : dès qu'une personne enregistre une pesée, une vente, un paiement..., les autres utilisateurs connectés voient la mise à jour apparaître automatiquement (Supabase Realtime), sans recharger la page.
- **Traçabilité** : les fiches animales et les ventes gardent une référence à l'utilisateur qui les a créées (`created_by`).
- Toute l'équipe partage la même base (pas d'isolement par utilisateur) — adapté à une ferme avec plusieurs employés/collaborateurs, comme StockPro. Si tu gères plusieurs fermes/clients distincts sur une même installation plus tard, il faudra ajouter une colonne `farm_id` et un filtre par équipe (une note est laissée dans `schema.sql` à cet endroit).
- Sans Supabase configuré, l'app reste utilisable en local (mono-utilisateur, sans écran de connexion) — pratique pour tester.

## Fonctionnalités du MVP
- **Fiches animales/lots** : identité, race, sexe, localisation, statut (actif/vendu/décédé), effectifs pour les lots (volailles), poids cible de vente
- **Pesées** : historique par animal, poids comme donnée centrale
- **Dépenses liées à un animal** → calcul automatique du coût de production/kg
- **Ventes** : poids + prix total → calcul auto du prix/kg
- **Créances clients** : ventes à crédit, échéances, paiements partiels, statut payée/partielle/impayée
- **Tableau de bord** : CA total, créances, trésorerie réelle, effectifs par espèce, alertes retard >30j, rappels santé, alerte poids de rentabilité

## Suivi spécifique par espèce (ouvre la fiche d'un animal pour y accéder)
- **🦆 Volailles** : relevé quotidien (œufs pondus, mortalité, aliment distribué)
- **🐑 Petits ruminants** (chèvre/mouton) : lait & santé mammaire, vermifuges/analyses fécales, poids au sevrage
- **🐄 Bovins** : ration alimentaire journalière, soins des sabots, alerte automatique au poids de rentabilité
- **🐴 Équidés** : soins dentaires (flottage), maréchalerie (ferrage), charge de travail (heures travail/repos)
- **🐕 Canidés** : suivi vétérinaire (vaccin/vermifuge/puce), évaluation comportementale (obéissance/garde), cycle des chaleurs (femelles)

Non couvert dans cette passe (à voir ensuite si besoin) : programme lumineux paramétrable pour pondeuses, calcul automatique de l'indice de consommation (IC) volailles — les données brutes (aliment, poids) sont déjà collectées, seul le calcul reste à ajouter.

## Photos
- Capture directe depuis l'appareil (ou choix dans la galerie) sur : fiche animale, observations de parcelle, événements de santé (blessures, maladies, soins)
- Compression automatique (largeur max ~900px, JPEG) avant envoi pour rester léger sur une connexion faible
- Si connecté : upload vers Supabase Storage (bucket `agro-photos`, public), l'URL est enregistrée sur la fiche
- Si hors ligne : la photo est gardée en local (base64) directement sur l'enregistrement — fonctionne, mais évite d'accumuler beaucoup de photos hors ligne avant de resynchroniser (chaque photo alourdit l'enregistrement local)

## Module Stocks & intrants (onglet "Stocks")
- **Aliments, médicaments/vaccins, engrais, semences, matériel** dans un seul référentiel filtrable par catégorie
- **Seuil d'alerte** par article → badge "stock bas" dès que la quantité passe en dessous
- **Péremption** (médicaments/vaccins) → alerte dès 30 jours avant l'échéance
- **Mouvements d'entrée/sortie** avec historique, quantité en stock mise à jour automatiquement
- **Matériel** : état (bon état / à réviser / hors service) + historique de maintenance avec prochaine échéance
- Le tableau de bord remonte automatiquement les stocks bas et les péremptions proches

## Module Agro (onglet "Agro")
- **Parcelles** : nom, superficie (ha), type de sol, culture en cours, statut (active/en jachère)
- **Itinéraire technique** : labour, semis, fertilisation, irrigation, traitement phytosanitaire — avec intrants utilisés et coût
- **Observations** : état des cultures, ravageurs, maladies, stress hydrique
- **Récoltes** : date, superficie moissonnée, poids récolté → rendement (t/ha) calculé automatiquement
- **Post-récolte** : lieu de stockage, état de séchage, pertes
- Le tableau de bord affiche la surface active totale et le nombre de récoltes enregistrées

Non couvert dans cette passe : géolocalisation GPS visuelle des parcelles sur une carte, prévisions météo intégrées (nécessite de brancher une API météo externe), photos jointes aux observations (le champ `photo_url` existe déjà côté base, l'upload depuis l'appareil reste à câbler).

## Mise en route (5 min)
1. Crée un projet sur [supabase.com](https://supabase.com) (gratuit).
2. Dans l'éditeur SQL du projet, colle et exécute le contenu de `schema.sql`.
3. Dans Supabase → Authentication → Providers, vérifie qu'Email est activé (c'est le cas par défaut). Tu peux désactiver "Confirm email" pour aller plus vite en test.
4. Dans Supabase → Project Settings → API, récupère `Project URL` et `anon public key`.
5. Ouvre `index.html`, renseigne `SUPABASE_URL` et `SUPABASE_ANON_KEY` en haut du `<script>`.
6. Héberge les 3 fichiers (index.html, manifest.json, sw.js) sur un hébergement statique (Netlify, Vercel, GitHub Pages) ou teste en local.
7. Chaque membre de l'équipe crée son compte (email + mot de passe) depuis l'écran de connexion — tout le monde partage ensuite la même base de données en direct.

## Comment fonctionne le mode hors ligne
- Toute saisie est écrite immédiatement dans IndexedDB (donc utilisable sans réseau, aux champs).
- Chaque écriture est aussi ajoutée à une file d'attente (`outbox`).
- Dès que le réseau revient (ou via le bouton "Synchroniser"), la file est envoyée vers Supabase.
- ⚠️ Limite du MVP : la synchronisation est "dernier écrit gagne", sans résolution fine des conflits. Suffisant pour un usage mono-utilisateur ; à revoir si plusieurs personnes saisissent en même temps sur des appareils différents.

## Prochaines étapes suggérées (hors MVP)
Tous les modules du cahier des charges initial sont couverts dans une première version, ainsi que plusieurs extras (reçu, WhatsApp, courbe de poids, carte, multi-devise). Ce qui reste, par ordre de priorité réaliste :
1. **Tester réellement l'app** sur le terrain (caméra, NFC, géoloc, notifications se comportent différemment selon les téléphones) — rien n'a encore été testé en conditions réelles
2. **Vraies notifications push** en arrière-plan (nécessite un backend : VAPID + Supabase Edge Function planifiée)
3. **Validation côté serveur** (aujourd'hui la validation est uniquement côté app — suffisant pour une petite équipe de confiance, mais à muscler si l'app grandit)
4. **Programme lumineux** paramétrable pour pondeuses, **indice de consommation (IC)** calculé automatiquement pour les volailles
5. **Isolation multi-fermes** (colonne `farm_id`) si l'app doit un jour servir plusieurs exploitations distinctes
6. **Rôles plus fins**, onboarding guidé, indicateurs de chargement pendant les appels réseau
