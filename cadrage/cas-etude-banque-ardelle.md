# Cas d'étude GRC — Banque Ardelle (DORA)

Oct 16, 2026 · @Kennedy

## Mode d'emploi

Tu joues le consultant GRC mandaté par la direction de la Banque Ardelle pour mesurer son niveau de conformité DORA et bâtir sa feuille de route. L'entreprise est fictive, mais les exigences, la méthode et les livrables sont ceux d'une vraie mission.

1. Lis la fiche d'identité et la cartographie : ce sont tes « entretiens de cadrage ».
2. Saisis le périmètre, les actifs et les prestataires dans CISO Assistant.
3. Mène l'analyse de risques EBIOS RM à partir des éléments fournis, puis complète les ateliers 4 et 5 toi-même.
4. Réalise l'audit DORA en t'appuyant sur les constats de départ.
5. Produis les livrables : rapport d'écarts, plan de remédiation priorisé, synthèse pour le comité des risques.

Règle du jeu : quand une information manque, formule une hypothèse et note-la, comme en mission réelle.

Pour avancer sans te perdre, suis le **Parcours pas à pas** en fin de document : chaque étape a ses cases à cocher.

## Installation de CISO Assistant

L'outil est installé et fonctionne dans la VM Ubuntu du home lab, sur https://localhost:8443. Cette section sert d'aide-mémoire pour le relancer et dépanner.

**Relancer l'outil à chaque séance**

```bash
cd ~/ciso-assistant-community
docker compose up -d
docker compose ps
```

Attends que le backend affiche « healthy » (1 à 3 minutes), puis ouvre https://localhost:8443 dans Firefox.

**Sauvegarder ton travail** (à faire en fin de séance)

```bash
cd ~/ciso-assistant-community
docker compose stop
sudo cp -a db ~/sauvegarde-db-$(date +%F)
docker compose up -d
```

En complément, fais un snapshot de la VM après chaque étape importante du parcours.

**Règle d'or** : ne supprime plus jamais le dossier `db`, il contient tout ton travail.

**Problèmes déjà rencontrés et solutions**

| Message | Cause | Solution |
| --- | --- | --- |
| Could not get lock /var/lib/dpkg/lock-frontend | Mises à jour automatiques d'Ubuntu en cours | Attendre la fin, puis relancer ; ne jamais supprimer le verrou |
| permission denied … docker.sock | Ajout au groupe docker pas encore pris en compte | `newgrp docker`, ou se déconnecter puis se reconnecter |
| The database seems already created | Une tentative précédente a créé `db` | Seulement lors de l'installation initiale : `docker compose rm -fs` puis `sudo rm -rf db` |
| container backend is unhealthy | Premier démarrage long (migrations, référentiels) | Attendre, relancer `docker compose up -d`, puis créer le compte avec `docker compose exec backend python manage.py createsuperuser` |

Amélioration conseillée : passer la VM à 6 Go de RAM pour plus de fluidité.

## Fiche d'identité

Banque Ardelle est une banque régionale de détail basée à Lyon, supervisée directement par l'ACPR, et donc pleinement soumise à DORA depuis janvier 2025.

| Élément | Valeur |
| --- | --- |
| Activités | Banque de détail, crédit immobilier, épargne, paiements, banque en ligne et application mobile |
| Taille | 1 800 salariés, 85 agences en Auvergne-Rhône-Alpes, 900 000 clients |
| Produit net bancaire | 610 M€ par an |
| SI | DSI de 180 personnes, environ 240 applications, 2 datacenters (Lyon et Villeurbanne) et une part croissante de cloud public |
| Autorité compétente | ACPR (Autorité de contrôle prudentiel et de résolution) |

**Gouvernance**

- Directoire de 4 membres, contrôlé par un conseil de surveillance doté d'un comité des risques.
- 1re ligne : métiers et DSI. Le RSSI est rattaché au DSI.
- 2e ligne : direction des risques et direction de la conformité.
- 3e ligne : audit interne, qui a signalé en 2025 un « manque de visibilité sur le risque lié aux prestataires TIC ».

**Contexte de la mission** : l'ACPR annonce un contrôle sur place au 2e trimestre 2027. Le directoire veut un état des lieux DORA et un plan d'action chiffré sous 3 mois.

## Rappel DORA

DORA (règlement (UE) 2022/2554) s'applique depuis le 17 janvier 2025. Il ne demande pas d'éviter tout incident : il demande à la banque de prouver qu'elle tient debout quand un incident survient. On peut le voir comme un code de la route : des règles communes, des contrôles et l'obligation de déclarer les accidents.

| Pilier | Articles | Ce qu'on attend de la Banque Ardelle |
| --- | --- | --- |
| Gestion du risque TIC | 5 à 16 | Un cadre approuvé et porté par le directoire, une fonction de contrôle indépendante, un inventaire des actifs, des sauvegardes et un PCA testés |
| Incidents TIC | 17 à 23 | Classer les incidents selon les critères réglementaires, notifier les incidents majeurs à l'ACPR dans les délais (notification initiale, rapport intermédiaire, rapport final) |
| Tests de résilience | 24 à 27 | Un programme de tests au moins annuel sur les systèmes qui soutiennent les fonctions critiques ; des tests de pénétration fondés sur la menace (TLPT) seulement si l'autorité désigne la banque |
| Prestataires TIC tiers | 28 à 44 | Un registre d'informations de tous les contrats TIC, des clauses contractuelles obligatoires (art. 30), des stratégies de sortie pour les fonctions critiques |
| Partage d'informations | 45 | Une participation volontaire à des dispositifs d'échange sur les menaces |

Textes à garder sous la main : le règlement sur [EUR-Lex](https://eur-lex.europa.eu/eli/reg/2022/2554/oj), le RTS sur le cadre de gestion du risque TIC (2024/1774) et celui sur la classification des incidents (2024/1772). Ces références viennent de ma connaissance et non d'une recherche : vérifie les numéros et les délais de notification sur EUR-Lex avant de les citer.

## Cartographie

La banque a identifié 6 fonctions critiques ou importantes (FCI). Ce sont elles qui fixent le périmètre prioritaire de l'audit et de l'analyse de risques.

**Fonctions critiques ou importantes**

| ID | FCI | Durée maximale d'interruption tolérable (MTD) | Actifs supports principaux | Prestataire clé |
| --- | --- | --- | --- | --- |
| FCI-01 | Tenue de comptes (core banking) | 4 h | Plateforme core banking, base clients, batch de fin de journée | NovaCore (SaaS) |
| FCI-02 | Paiements (SEPA, virements instantanés) | 2 h | Passerelle paiements, HSM, lien interbancaire | PayLink Services |
| FCI-03 | Banque en ligne et application mobile | 4 h | Front web, API mobile, authentification forte | Kumo Digital (éditeur) |
| FCI-04 | Octroi de crédit | 48 h | Outil de scoring, gestion électronique de documents | Interne |
| FCI-05 | Lutte anti-blanchiment | 24 h | Outil de filtrage et de détection | Veridex (SaaS) |
| FCI-06 | Reporting réglementaire | 72 h | Entrepôt de données, chaîne de reporting | Interne, hébergé chez Stratus Cloud |

**Prestataires TIC tiers à saisir**

| Prestataire | Service | Soutient une FCI | Localisation des données |
| --- | --- | --- | --- |
| NovaCore | Core banking en SaaS | Oui | France |
| Stratus Cloud | Cloud public (IaaS) | Oui | UE, deux régions |
| PayLink Services | Passerelle de paiement | Oui | France |
| Kumo Digital | Développement et maintenance de l'application mobile | Oui | France et Maroc |
| Veridex | Filtrage anti-blanchiment | Oui | Irlande |
| Sentinelle SOC | SOC externalisé (supervision 24/7) | Indirectement | France |
| Opalis | Infogérance des postes de travail | Non | France |

Hypothèse à noter : le recours au Maroc pour la maintenance de l'application mobile implique un accès distant hors UE. C'est un point à examiner dans l'analyse de risques et dans les clauses contractuelles.

## Analyse de risques EBIOS RM

Les ateliers 1 à 3 sont amorcés ci-dessous. Les ateliers 4 et 5 sont à produire par toi. EBIOS RM se lit comme une enquête à l'envers : on part de ce qu'on ne veut pas perdre, puis on cherche qui voudrait nous le prendre et par quel chemin.

**Atelier 1 : valeurs métier et événements redoutés** (gravité de 1 à 4)

| Valeur métier | Événement redouté | Gravité |
| --- | --- | --- |
| Paiements | Indisponibilité des virements pendant plus de 2 h | 4 |
| Tenue de comptes | Altération des soldes clients | 4 |
| Données clients | Fuite massive de données personnelles et bancaires | 3 |
| Banque en ligne | Prise de contrôle de comptes clients, fraude | 3 |
| Reporting réglementaire | Retard ou erreur dans un reporting à l'ACPR | 2 |

**Atelier 2 : sources de risque et objectifs visés**

| Source de risque | Objectif visé | Pertinence |
| --- | --- | --- |
| Groupe cybercriminel (rançongiciel) | Extorsion par chiffrement et vol de données | Élevée |
| Fraudeurs organisés | Détournement de fonds via la banque en ligne | Élevée |
| Hacktivistes | Déni de service sur les canaux clients | Moyenne |
| Initié malveillant | Revente de données, fraude interne | Moyenne |
| Prestataire défaillant (non malveillant) | Panne prolongée d'un service critique | Élevée |

**Atelier 3 : scénarios stratégiques, par l'écosystème**

1. Le groupe cybercriminel compromet Kumo Digital via un poste de maintenance au Maroc, pénètre l'API mobile, puis se déplace vers le SI de la banque et chiffre le core banking.
2. Les fraudeurs mènent du phishing ciblé sur les clients, contournent l'authentification forte par SIM swapping et détournent des fonds via la banque en ligne.
3. NovaCore subit une panne majeure de 3 jours. Faute de stratégie de sortie, la tenue de comptes est interrompue bien au-delà des 4 h tolérées.
4. Un administrateur de Sentinelle SOC, disposant d'accès étendus, exfiltre la base clients.

**À toi de jouer (ateliers 4 et 5)** : détaille les modes opératoires des scénarios 1 et 3, estime leur vraisemblance, puis propose les mesures de traitement et le risque résiduel.

## État des lieux : constats de départ

Ces 10 constats sont issus des « entretiens » de cadrage. Utilise-les pour noter chaque exigence de l'audit DORA comme conforme, partiellement conforme ou non conforme, en justifiant ton choix.

| N° | Constat | Pilier DORA |
| --- | --- | --- |
| C1 | La politique de gestion du risque TIC date de 2022, n'a pas été revue depuis et n'a pas été approuvée par le directoire | Gestion du risque TIC |
| C2 | Le RSSI rend compte au DSI : la fonction de contrôle du risque TIC n'est pas indépendante | Gestion du risque TIC |
| C3 | Aucun membre du directoire n'a suivi de formation sur le risque TIC depuis 2024 | Gestion du risque TIC |
| C4 | L'inventaire des actifs couvre environ 85 % des applications ; le reste relève du shadow IT métier | Gestion du risque TIC |
| C5 | Les sauvegardes du core banking ne sont pas isolées du réseau de production | Gestion du risque TIC |
| C6 | La classification des incidents repose sur une grille interne antérieure à DORA ; la chaîne de notification à l'ACPR n'a jamais été testée | Incidents TIC |
| C7 | Le dernier test du PCA date de 2024 et ne couvrait pas la perte de NovaCore | Tests de résilience |
| C8 | Le registre d'informations ne recense qu'environ 60 % des contrats TIC | Prestataires TIC |
| C9 | Les contrats de Kumo Digital et de Veridex ne prévoient ni droit d'audit ni clauses de sortie | Prestataires TIC |
| C10 | Il n'existe aucune stratégie de sortie documentée pour NovaCore ni pour Stratus Cloud | Prestataires TIC |

Points positifs à valoriser aussi : authentification forte en place pour tous les clients, SOC 24/7 opérationnel, tests d'intrusion annuels sur l'application mobile.

## Parcours pas à pas dans CISO Assistant

Chaque étape suit le même rythme : **Comprendre** (la théorie et le pourquoi), **Faire** (les manipulations dans l'outil), **Vérifie ta maîtrise** (les questions auxquelles tu dois savoir répondre sans notes avant de passer à la suite). Une étape n'est terminée que lorsque tu sais l'expliquer à voix haute en 2 minutes, comme en entretien. Les menus sont ceux de l'interface en français ; compte environ 8 séances de 2 h.

### Étape 1 : préparer le terrain

**Comprendre**

Un outil GRC ressemble à un classeur bien rangé. Le *domaine* est le classeur (l'organisation), le *périmètre* est l'intercalaire (ce qu'on évalue), et les *bibliothèques* sont les règles du jeu qu'on y range : référentiels, matrices, catalogues de menaces.

DORA est un **règlement** européen : il s'applique tel quel dans tous les États membres, sans loi de transposition. NIS2, au contraire, est une **directive** que chaque pays transpose. Pour une banque, DORA est la règle spécialisée (*lex specialis*) : sur la cybersécurité et les incidents, c'est lui qui prime sur NIS2. DORA applique aussi un principe de **proportionnalité** : les exigences s'adaptent à la taille et au profil de risque de l'entité.

La **matrice de risques** se choisit avant toute analyse, comme on choisit l'unité de mesure avant de mesurer. Elle croise la gravité (l'impact si ça arrive) et la vraisemblance (la probabilité que ça arrive). Si deux analystes utilisent deux grilles différentes, leurs résultats ne sont pas comparables.

Enfin, CISO Assistant sépare les **exigences** (ce que dit le texte) des **mesures** (ce que l'entreprise met en place). Une même mesure, par exemple des sauvegardes isolées, peut répondre à DORA, ISO 27001 et NIS2 à la fois. C'est le principe du découplage.

**Le mot juste** : référentiel, exigence, mesure, périmètre, matrice de risques, gravité, vraisemblance, proportionnalité, lex specialis.

**Vérifie ta maîtrise**

- Pourquoi DORA s'applique-t-il de la même façon en France et en Allemagne, alors que NIS2 peut varier ?
- Une banque est-elle soumise à DORA, à NIS2, ou aux deux ? Lequel prime en cas de recoupement ?
- Pourquoi choisir la matrice de risques avant de commencer l'analyse ?

**Faire**

- [x] Charger la bibliothèque DORA (Gouvernance → Bibliothèques)
- [ ] Charger la « Matrice 4x4 EBIOS-RM » (onglet Matrices de risque, icône flèche à droite de la ligne)
- [x] Créer le domaine « Banque Ardelle » (Organisation → Domaines)
- [ ] Vérifier que le périmètre « FCI DORA » apparaît bien dans Organisation → Périmètres

### Étape 2 : saisir les 6 fonctions critiques ou importantes

**Comprendre**

DORA ne demande pas de tout protéger au même niveau : il demande de savoir **ce qui compte vraiment**. C'est l'idée des fonctions critiques ou importantes (FCI). En substance, une fonction est critique ou importante si son arrêt ou son mauvais fonctionnement nuirait sérieusement à la santé financière de la banque, à la continuité de ses services ou à sa capacité à respecter ses obligations réglementaires. Tout le reste de DORA (tests, contrats, stratégies de sortie) est plus exigeant pour ces fonctions.

On distingue les **actifs primaires** (ce que fait la banque : une activité, un processus, une information) des **actifs supports** (ce sur quoi elle s'appuie : logiciels, serveurs, prestataires). EBIOS RM parle de *valeurs métier* et de *biens supports*. On protège les seconds parce qu'ils portent les premiers.

Deux chiffres traduisent l'exigence de continuité. Imagine un magasin :

- le **RTO** (Recovery Time Objective) est la durée maximale pendant laquelle le magasin peut rester fermé ;
- le **RPO** (Recovery Point Objective) est la quantité de ventes qu'on accepte de perdre du carnet, c'est-à-dire l'ancienneté maximale de la dernière sauvegarde utilisable.

Un RPO de 15 min impose des sauvegardes ou une réplication au moins toutes les 15 min. Ces valeurs se fixent normalement dans un **bilan d'impact sur l'activité** (BIA) mené avec les métiers, pas par l'informatique seule.

Dans l'outil, chaque actif est décrit par deux attributs qui répondent à deux questions différentes :

- le **type** (primaire ou support) indique son *rôle* dans la chaîne de valeur : est-ce ce qu'on protège, ou ce sur quoi on s'appuie ?
- la **classe** indique sa *nature* : processus métier, données, logiciel, appareil, réseau, locaux, utilisateurs.

Une base de données clients a donc la classe « Données » mais le type « Support », car elle porte des FCI.

Chaque actif doit aussi avoir un **propriétaire** (champ « Assigné à ») : la personne qui répond de sa protection et arbitre les risques qui le concernent. Pour une FCI, c'est un responsable métier, pas le DSI. ISO 27001 exige d'ailleurs un inventaire des actifs avec leurs propriétaires.

**Les sous-classes de « Processus métier »** reprennent un découpage classique des processus d'une entreprise. Imagine un restaurant : les **opérations principales** (processus de réalisation) produisent ce que le client achète, c'est la cuisine et le service en salle ; la **gestion et gouvernance** (processus de pilotage) fixe les règles et rend des comptes, c'est le gérant, l'hygiène et la comptabilité légale ; les **fonctions de support** (RH, achats, informatique) font tourner le reste sans être vendues au client ; les **relations externes** gèrent les échanges avec les partenaires, fournisseurs et autorités. Tenue de comptes, paiements, banque en ligne et crédit sont ce que la banque vend : opérations principales. La lutte anti-blanchiment et le reporting répondent à des obligations de conformité : gestion et gouvernance.

**Les cibles de sécurité** traduisent le besoin de protection de chaque actif selon plusieurs critères. Les quatre classiques sont le **DICP** : Disponibilité (le service est accessible quand il le faut), Intégrité (les données sont exactes et non altérées), Confidentialité (seules les personnes autorisées y accèdent) et Preuve, ou traçabilité (on peut prouver qui a fait quoi). L'outil ajoute l'Authenticité (on est sûr de l'identité de l'émetteur), la Vie privée (protection des données personnelles) et la Sûreté (protection des personnes physiques, pertinente en industrie ou dans le ferroviaire, pas pour une banque de détail). On ne met pas 4 partout : un besoin surévalué coûte cher à protéger et dilue les priorités.

**Trois durées, à ne pas confondre.** Reprenons le magasin :

- le **MTD** (Maximum Tolerable Downtime, en français DMIA) est la limite fixée par le métier : au-delà, les dégâts deviennent inacceptables ;
- le **RTO** est le délai de redémarrage que l'on vise. Il est toujours inférieur au MTD, pour garder une marge si la reprise se passe mal ;
- le **RPO** est la perte de données maximale acceptable.

Précision : la version précédente de ce document assimilait la durée maximale tolérée au RTO. C'est une simplification courante, mais un recruteur attentif fera la différence.

**La section « Spécifique à DORA »** reprend les informations du **registre d'informations** DORA sur les fonctions : l'activité réglementée concernée, l'évaluation de la criticité, sa justification et l'impact d'un arrêt. Remplir ces champs revient donc à préparer le registre que l'ACPR peut demander.

**Pourquoi la liste « Activité sous licence » est si longue.** Elle couvre toutes les entités financières soumises à DORA, et rassemble donc les activités définies par plusieurs textes européens : banques (directive CRD), services de paiement (DSP2), assurance (Solvabilité II), services d'investissement (MiFID II), dépositaires centraux (CSDR), crypto-actifs (MiCA), gestion de fonds, indices de référence, financement participatif. Une banque est un établissement de crédit : elle ne choisit que parmi les activités bancaires et de paiement. Piège à éviter : « Providing regulatory reporting » ou « regulatory compliance monitoring » semblent correspondre à FCI-05 et FCI-06, mais ces libellés viennent d'autres régimes (dépositaires centraux, gestion de fonds). La lutte anti-blanchiment et le reporting sont des **obligations** de la banque, pas des activités agréées : on choisit « supporting function ». Un libellé qui « sonne juste » mais vient du mauvais régime est une erreur classique dans un registre d'informations.

**Le mot juste** : FCI, actif primaire, actif support, valeur métier, bien support, type, classe, propriétaire d'actif, RTO, RPO, BIA.

**Vérifie ta maîtrise**

- Pourquoi les paiements ont-ils un RTO plus court que le reporting réglementaire ?
- Qui doit fixer le RTO d'une fonction : le DSI ou le métier ? Pourquoi ?
- Quelle conséquence technique concrète a un RPO de 5 min ?

**Faire**

Menu : Gestion des actifs → Actifs → bouton violet « + ». Pour chaque FCI :

1. **ID** et **Nom** : selon le tableau A.
2. **Description** : celle du tableau A.
3. **Domaine** : Banque Ardelle.
4. **Classe** : déplier « Processus métier » et choisir la sous-classe indiquée dans le tableau A.
5. **Assigné à** : vide pour l'instant (le propriétaire métier sera créé plus tard comme utilisateur).
6. **Type** : Primaire.
7. **Est une dépendance de** et **Dépend de** : vides (ils se rempliront depuis les actifs supports à l'étape 3).
8. Section **Cibles de sécurité** : activer les critères avec l'interrupteur et choisir le niveau, selon le tableau B. Laisser « Sûreté » désactivé.
9. Section **Objectifs de reprise d'activité** : MTD, RTO et RPO du tableau A (en heures et minutes).
10. Section **Spécifique à DORA** : cocher « Est une fonction métier », puis remplir les champs selon le tableau C.
11. Section **Plus** : dans **Observation**, noter « RTO et RPO = hypothèses, à valider par le métier lors du BIA » ; dans **Étiquettes**, créer et ajouter « DORA » et « FCI ». Laisser Lien, Mesures appliquées, Exceptions et Solutions vides (étape 7).

**Tableau A : identité et objectifs de reprise**

| ID | Nom | Classe (chemin complet) | Description à copier | MTD | RTO | RPO |
| --- | --- | --- | --- | --- | --- | --- |
| FCI-01 | Tenue de comptes (core banking) | Processus métier › Opérations Principales › Livraison de Produits ou Services | Fonction critique ou importante au sens de DORA. Tenue des comptes clients et batch de fin de journée. | 4 h | 2 h | 15 min |
| FCI-02 | Paiements (SEPA, virements instantanés) | Processus métier › Opérations Principales › Livraison de Produits ou Services | Fonction critique ou importante au sens de DORA. Émission et réception des virements SEPA et instantanés. | 2 h | 1 h | 5 min |
| FCI-03 | Banque en ligne et application mobile | Processus métier › Opérations Principales › Livraison de Produits ou Services | Fonction critique ou importante au sens de DORA. Accès des clients à leurs comptes et opérations en ligne. | 4 h | 2 h | 1 h |
| FCI-04 | Octroi de crédit | Processus métier › Opérations Principales › Livraison de Produits ou Services | Fonction critique ou importante au sens de DORA. Instruction, décision et mise en place des crédits. | 48 h | 24 h | 24 h |
| FCI-05 | Lutte anti-blanchiment | Processus métier › Gestion et Gouvernance › Gestion de la Conformité | Fonction critique ou importante au sens de DORA. Filtrage des opérations et détection des transactions suspectes. | 24 h | 12 h | 4 h |
| FCI-06 | Reporting réglementaire | Processus métier › Gestion et Gouvernance › Gestion de la Conformité | Fonction critique ou importante au sens de DORA. Production et transmission des états réglementaires à l'ACPR. | 72 h | 48 h | 24 h |

Le MTD vient de la fiche d'identité de la banque. Le RTO et le RPO sont des hypothèses, à justifier en entretien.

**Tableau B : cibles de sécurité** (1 = faible, 4 = critique)

| ID | Confidentialité | Intégrité | Disponibilité | Preuve | Authenticité | Vie privée |
| --- | --- | --- | --- | --- | --- | --- |
| FCI-01 | 3 | 4 | 4 | 4 | 3 | 3 |
| FCI-02 | 3 | 4 | 4 | 4 | 4 | 2 |
| FCI-03 | 3 | 3 | 3 | 3 | 4 | 3 |
| FCI-04 | 3 | 3 | 2 | 3 | 3 | 3 |
| FCI-05 | 4 | 3 | 2 | 4 | 3 | 3 |
| FCI-06 | 2 | 4 | 2 | 4 | 3 | 2 |

**Tableau C : section Spécifique à DORA**

| ID | Est une fonction métier | Activité sous licence (libellé exact dans l'outil) | Évaluation de la criticité | Justification de la criticité | Impact de l'arrêt |
| --- | --- | --- | --- | --- | --- |
| FCI-01 | ☑ Cochée | Taking deposits and other repayable funds | Oui | Un arrêt prive les clients de l'accès à leurs fonds et peut fausser les soldes : atteinte directe à la continuité des services et à la solidité financière. | Haut |
| FCI-02 | ☑ Cochée | Execution of payment transactions, including transfers of funds on a payment account… | Oui | Un arrêt bloque les virements des particuliers et des entreprises, avec un impact immédiat sur les clients et sur le système de paiement. | Haut |
| FCI-03 | ☑ Cochée | Payment services | Oui | Canal principal d'accès des clients : un arrêt entraîne une perte de service massive et accroît le risque de fraude. | Haut |
| FCI-04 | ☑ Cochée | Lending activities | Oui | Activité génératrice de revenus : un arrêt prolongé pèse sur le PNB et sur la relation client. | Moyen |
| FCI-05 | ☑ Cochée | supporting function | Oui | Obligation légale : un arrêt expose la banque à des sanctions et menace le respect de ses conditions d'agrément. | Haut |
| FCI-06 | ☑ Cochée | supporting function | Oui | Obligation envers l'ACPR : des défauts répétés remettent en cause la conformité réglementaire de la banque. | Moyen |
|  |  |  |  |  |  |

« Évaluation de la criticité : Oui » signifie que la fonction est jugée critique ou importante. « Supporting function » désigne une fonction qui soutient les activités agréées sans en être une : c'est le cas de la lutte anti-blanchiment et du reporting.

Les libellés exacts des listes déroulantes peuvent différer : choisis l'option la plus proche, et si le niveau « Moyen » n'existe pas, prends le niveau intermédiaire.

- [ ] FCI-01 à FCI-03 saisies
- [ ] FCI-04 à FCI-06 saisies

### Étape 3 : saisir les actifs supports

**Comprendre**

Une fonction métier ressemble à un arbre : on voit le tronc et les branches (le service rendu au client), mais il tient grâce à ses racines (applications, bases de données, prestataires). Si une racine pourrit, l'arbre tombe. DORA exige donc que la banque identifie ses actifs et **cartographie leurs interdépendances**, en particulier pour les FCI.

Cette cartographie a deux effets pratiques :

- un actif support **hérite de la criticité** de la fonction qu'il soutient : la base clients devient critique parce qu'elle porte FCI-01 et FCI-03 ;
- elle révèle les **points uniques de défaillance** : si plusieurs FCI dépendent d'un même actif ou d'un même prestataire, une seule panne peut tout arrêter. C'est le **risque de concentration**.

**Le mot juste** : cartographie, interdépendance, héritage de criticité, point unique de défaillance, risque de concentration.

**Vérifie ta maîtrise**

- Pourquoi la base de données clients est-elle plus critique que l'outil de scoring ?
- Dans ce cas, quel actif ou quel prestataire constitue un point unique de défaillance pour plusieurs FCI ?
- Que perdrait-on si les actifs supports étaient saisis sans les relier aux FCI ?

**Faire**

Même menu, Type : Support. Dans le champ « Est une dépendance de », sélectionne la ou les FCI que l'actif soutient : c'est ce lien qui permettra à l'outil de remonter un risque technique jusqu'à la fonction métier.

| ID | Nom | Classe (chemin complet) | Champ « Est une dépendance de » |
| --- | --- | --- | --- |
| SUP-01 | Plateforme core banking (NovaCore) | Logiciels › Applications › SaaS | FCI-01 |
| SUP-02 | Base de données clients | Données › Données Sensibles | FCI-01, FCI-03 |
| SUP-03 | Sauvegardes du core banking | Données › Données Sensibles | FCI-01 |
| SUP-04 | Passerelle de paiements (PayLink) | Logiciels › Applications › Services | FCI-02 |
| SUP-05 | HSM (modules de sécurité matériels) | Appareils › Actifs d'Entreprise › Serveurs | FCI-02 |
| SUP-06 | Front web et API mobile (Kumo Digital) | Logiciels › Applications › APIs | FCI-03 |
| SUP-07 | Authentification forte | Logiciels › Applications › Services | FCI-03 |
| SUP-08 | Outil de scoring et GED crédit | Logiciels › Applications | FCI-04 |
| SUP-09 | Outil de filtrage anti-blanchiment (Veridex) | Logiciels › Applications › SaaS | FCI-05 |
| SUP-10 | Entrepôt de données et chaîne de reporting (Stratus Cloud) | Appareils › Actifs d'Entreprise › Infrastructure Cloud | FCI-06 |

Laisse le champ « Dépend de » vide sur les actifs supports. Un lien ne se saisit qu'une fois, d'un seul côté : après enregistrement, ouvre FCI-01 et vérifie que SUP-01, SUP-02 et SUP-03 apparaissent dans son champ « Dépend de ».

Règle de classement : choisis la sous-classe la plus précise qui décrit vraiment l'actif. Si aucune ne convient, reste au niveau parent (SUP-08) plutôt que de forcer une case ; pour le HSM, qui n'a pas de classe dédiée, « Serveurs » est l'équipement le plus proche, à signaler en Observation.

- [ ] 10 actifs supports saisis et reliés à leurs FCI

### Étape 4 : déclarer les prestataires TIC

**Comprendre**

Principe clé de DORA : **on peut externaliser un service, jamais la responsabilité**. Si NovaCore tombe, c'est la Banque Ardelle qui en répond devant l'ACPR et devant ses clients. C'est comme un propriétaire qui fait construire par une entreprise : si la maison s'effondre, il doit pouvoir prouver qu'il avait choisi, encadré et contrôlé son constructeur.

Concrètement, DORA demande :

- un **registre d'informations** qui recense tous les contrats de services TIC, transmis à l'autorité sur demande ;
- des **clauses contractuelles obligatoires** (article 30) : description des services, lieux de traitement des données, droits d'accès et d'audit, niveaux de service, conditions de résiliation ;
- pour les FCI, une **stratégie de sortie** : savoir comment changer de prestataire ou reprendre le service en interne sans interruption inacceptable ;
- une analyse du **risque de concentration**, quand trop de fonctions dépendent d'un même acteur.

Les prestataires jugés critiques pour l'ensemble du secteur financier, typiquement les grands fournisseurs de cloud, peuvent être désignés **prestataires tiers critiques** et sont alors supervisés directement par les autorités européennes. Cette supervision ne dispense pas la banque de ses propres obligations.

**Prestataire ou fournisseur ?** Un *fournisseur* vend un bien ou une licence (des ordinateurs, un logiciel installé chez soi). Un *prestataire* rend un service dans la durée (héberger, exploiter, maintenir, superviser). Les 7 tiers du cas rendent tous un service TIC : ce sont des prestataires au sens de DORA.

**Le niveau d'un tiers** se déduit des fonctions qu'il soutient et de la difficulté à le remplacer. Retiens la règle : un tiers qui soutient une FCI est au moins « Important », et il devient « Critique » quand son arrêt bloquerait une FCI sans solution de repli rapide (pas de stratégie de sortie, ou plusieurs FCI concentrées chez lui).

**L'identifiant légal de référence** dans le registre DORA est le **LEI** (Legal Entity Identifier), un code international de 20 caractères qui identifie une entité juridique de façon unique. Les prestataires du cas sont fictifs : on laisse ce champ vide. En mission réelle, un LEI manquant est un écart du registre.

**L'évaluation rapide EBIOS RM** mesure le niveau de menace que représente une partie prenante, avec quatre critères :

- **Dépendance** (0 à 4) : la relation est-elle vitale pour la banque ?
- **Pénétration** (0 à 4) : jusqu'où le tiers accède-t-il au SI ?
- **Maturité** (1 à 4) : quelles sont ses capacités de sécurité ?
- **Confiance** (1 à 4) : ses intérêts pourraient-ils s'opposer à ceux de la banque ?

La criticité vaut (Dépendance × Pénétration) ÷ (Maturité × Confiance). Intuitivement : plus on dépend d'un tiers et plus il est entré dans la maison, plus il est dangereux ; plus il est mature et digne de confiance, moins il l'est. Un tiers très intégré mais peu mature devient le maillon faible par lequel passe l'attaquant : c'est exactement le scénario stratégique 1 (Kumo Digital).

**Le mot juste** : prestataire de services TIC, registre d'informations, clauses contractuelles, droit d'audit, stratégie de sortie, risque de concentration, prestataire tiers critique.

**Vérifie ta maîtrise**

- Pourquoi le constat C10 (pas de stratégie de sortie pour NovaCore) est-il plus grave que l'absence de clause d'audit chez Opalis ?
- Que fais-tu si un grand fournisseur cloud refuse un droit d'audit sur site ?
- Le maintien applicatif de Kumo Digital depuis le Maroc est-il interdit par DORA ? Que faut-il vérifier ?

**Faire**

Menu : Gestion des tiers → Entités → Ajouter. Pour chaque prestataire :

1. **ID**, **Nom**, **Relation**, **Niveau** et **Description** : selon le tableau D.
2. **Domaine** : Banque Ardelle. **Actif** : coché.
3. **Identifiants légaux** : vide (prestataires fictifs, voir le LEI dans Comprendre).
4. Section **Plus** : **Mission** = le service rendu (tableau D) ; **Entité parente** et **Adresse** : vides ; **Pays** : tableau D ; **Devise** : EUR ; **Type de personne** : personne morale ; **Étiquettes** : « DORA » et « Prestataire TIC ».
5. Section **Spécifique à DORA** : laisser vide pour l'instant (ces champs décrivent surtout les entités financières elles-mêmes ; on y reviendra).
6. Section **Évaluation rapide de criticité (EBIOS RM)** : selon le tableau E.

**Tableau D : identité des prestataires**

| ID | Nom | Relation | Niveau | Pays | Description et mission |
| --- | --- | --- | --- | --- | --- |
| PRE-01 | NovaCore | Prestataire | Critique | France | Core banking en SaaS. Soutient FCI-01. Aucune stratégie de sortie (C10). |
| PRE-02 | Stratus Cloud | Prestataire | Critique | Irlande (hypothèse : siège UE, deux régions) | Cloud public IaaS. Héberge l'entrepôt de données (FCI-06) ; risque de concentration. |
| PRE-03 | PayLink Services | Prestataire | Critique | France | Passerelle de paiement. Soutient FCI-02 (MTD 2 h). |
| PRE-04 | Kumo Digital | Prestataire | Important | France | Développement et maintenance de l'application mobile, accès distant depuis le Maroc. Soutient FCI-03. Ni droit d'audit ni clause de sortie (C9). |
| PRE-05 | Veridex | Prestataire | Important | Irlande | Filtrage anti-blanchiment en SaaS. Soutient FCI-05. Ni droit d'audit ni clause de sortie (C9). |
| PRE-06 | Sentinelle SOC | Prestataire | Important | France | SOC externalisé, supervision 24/7, accès privilégiés étendus. Soutient indirectement toutes les FCI. |
| PRE-07 | Opalis | Prestataire | Standard | France | Infogérance des postes de travail. Ne soutient pas de FCI. |

**Tableau E : évaluation rapide de criticité EBIOS RM**

| ID | Dépendance | Pénétration | Maturité | Confiance | Criticité calculée |
| --- | --- | --- | --- | --- | --- |
| PRE-01 NovaCore | 4 | 4 | 3 | 3 | 1,8 |
| PRE-02 Stratus Cloud | 4 | 3 | 4 | 3 | 1,0 |
| PRE-03 PayLink | 4 | 3 | 3 | 3 | 1,3 |
| PRE-04 Kumo Digital | 3 | 4 | 2 | 2 | 3,0 |
| PRE-05 Veridex | 3 | 2 | 3 | 3 | 0,7 |
| PRE-06 Sentinelle SOC | 3 | 4 | 3 | 2 | 2,0 |
| PRE-07 Opalis | 2 | 3 | 2 | 3 | 1,0 |

À observer : le niveau « métier » (tableau D) et la criticité EBIOS (tableau E) ne classent pas les tiers dans le même ordre. NovaCore est le plus critique pour l'activité, mais Kumo Digital et Sentinelle SOC sont les plus dangereux du point de vue de la menace. Les deux lectures sont complémentaires.

- [ ] 7 prestataires créés
- [ ] Contrats et registre d'informations renseignés

### Étape 5 : réaliser l'audit DORA

**Comprendre**

Un audit de conformité ressemble au contrôle technique d'une voiture : on vérifie point par point une liste d'exigences, et chaque point est validé ou non, **preuve à l'appui**. On ne cherche pas encore à savoir si la voiture va tomber en panne demain : ça, c'est l'analyse de risques de l'étape 6. Une banque peut être conforme sur le papier et rester vulnérable, ou l'inverse. Les deux exercices se complètent.

Bonnes pratiques d'auditeur :

- **Pas de preuve, pas de conformité.** Une politique non approuvée (C1) ne vaut pas « conforme », même si le document existe.
- **« Partiellement conforme »** décrit un dispositif qui existe mais est incomplet : l'inventaire couvre 85 % des applications (C4).
- **« Non applicable »** doit toujours être justifié. Un auditeur qui coche N/A sans explication perd sa crédibilité.
- Un **constat** décrit un écart factuel, son exigence de référence et son impact. Il ne propose pas encore de solution.

DORA place la responsabilité au sommet : l'**organe de direction** (ici le directoire) définit, approuve et répond du cadre de gestion du risque TIC. C'est pour cela que C1, C2 et C3 sont des écarts de gouvernance majeurs, même sans aucune faille technique. La fonction de contrôle du risque TIC doit être indépendante de ceux qu'elle contrôle : c'est la logique des **trois lignes de maîtrise** (opérationnels, contrôle, audit interne).

**Le mot juste** : audit de conformité, évaluation des écarts, preuve, constat, non applicable, organe de direction, trois lignes de maîtrise.

**Vérifie ta maîtrise**

- Quelle différence entre un audit de conformité et une analyse de risques ? Pourquoi faire les deux ?
- Pourquoi un RSSI rattaché au DSI pose-t-il un problème d'indépendance ?
- Comment justifies-tu « non applicable » pour le chapitre VII de DORA ?

**Faire**

**Comprendre la fiche d'audit**

La fiche d'audit est l'équivalent numérique de la **lettre de mission** : avant de poser la moindre question, l'auditeur écrit ce qu'il évalue (périmètre), par rapport à quoi (référentiel), comment (méthode), pour quand (échéance) et avec quelles limites. Un audit sans lettre de mission claire finit toujours en débat sur ce qui était « dans le périmètre ».

L'outil distingue deux rôles. L'**auditeur** évalue et conclut ; le **répondant**, côté audité, apporte des réponses et des preuves. Les réglages « Visibilité des champs » décident qui voit quoi : par exemple, le répondant ne voit pas le statut de conformité tant que l'auditeur n'a pas conclu, pour ne pas orienter ses réponses. C'est la **séparation des rôles**, garante de l'objectivité.

Le **relecteur** applique le principe des **quatre yeux** : aucune conclusion ne sort sans une seconde lecture. Le **verrouillage** final protège l'intégrité du rapport : une fois validé, il ne doit plus pouvoir être modifié sans trace.

**Faire**

Menu : Conformité → Audits → Ajouter (« Démarrer l'audit »). Remplis la fiche selon le tableau F.

**Tableau F : fiche de l'audit**

| Champ | Valeur | Pourquoi |
| --- | --- | --- |
| ID | AUD-DORA-01 | Numérotation qui permettra de suivre les audits successifs |
| Nom | Audit de conformité DORA – Banque Ardelle 2026 | Référentiel, entité et année : le triplet qu'on cherche dans un historique d'audits |
| Description | Évaluation de la conformité de la Banque Ardelle au règlement (UE) 2022/2554 (DORA) sur le périmètre des fonctions critiques ou importantes, en préparation du contrôle ACPR du T2 2027. Méthode : revue documentaire et entretiens de cadrage (constats C1 à C10). | Un audit commence toujours par sa lettre de mission : objectif, périmètre, référentiel, méthode |
| Domaine | Banque Ardelle |  |
| Périmètre | FCI DORA |  |
| Référentiel cible | Règlement sur la résilience opérationnelle numérique (DORA) |  |
| Version | 1.0 | La version augmentera à chaque révision du rapport |
| Statut | En cours |  |
| ETA (date estimée) | 08/01/2027 | Fin prévue, avec une semaine de marge |
| Date d'échéance (section Plus) | 16/01/2027 | Les 3 mois demandés par le directoire |
| Visibilité des champs | Valeurs par défaut | Tu joues seul : le partage auditeur / répondant n'a pas d'effet ici |
| Actifs | Les 6 FCI (FCI-01 à FCI-06) | Les actifs supports sont rattachés aux FCI par leurs dépendances |
| Preuves | Vide | Elles s'ajoutent exigence par exigence pendant l'évaluation |
| Auteurs | Toi |  |
| Relecteurs | Vide | En réel, un manager relit avant diffusion (principe des quatre yeux) |
| Observation | Audit pédagogique sur une entreprise fictive. Hypothèses : constats issus d'entretiens de cadrage, aucun test technique réalisé. | Les limites d'un audit doivent toujours être écrites |
| Verrouillé | Non coché | On verrouille seulement à la fin, une fois le rapport validé |
| Synchronisation automatique | Non cochée |  |

**Évaluer les exigences**

Ouvre l'audit (icône œil). Pour chaque exigence, l'outil propose deux champs à ne pas confondre :

- **Statut** : l'avancement de *ton travail* (à faire, en cours, en revue, fait) ;
- **Résultat** : ta *conclusion* (conforme, partiellement conforme, non conforme, non applicable, non évalué).

Dans le commentaire ou l'observation, cite toujours le constat (« C5 : sauvegardes non isolées »). Les exigences sont découpées par paragraphe : applique le résultat de l'article à ses paragraphes, et ajuste si un paragraphe est clairement satisfait ou s'adresse aux autorités.

**Tableau G : grille d'évaluation par article**

| Article | Thème | Résultat | Justification |
| --- | --- | --- | --- |
| 1 à 4 | Dispositions générales | Non applicable | Définitions et champ d'application : pas d'obligation à vérifier |
| 5 | Gouvernance | Non conforme | C1 (politique non approuvée par le directoire), C3 (direction non formée) |
| 6 | Cadre de gestion du risque TIC | Partiellement conforme | Le cadre existe, mais il n'est pas revu depuis 2022 et la fonction de contrôle n'est pas indépendante (C1, C2) |
| 7 | Systèmes, protocoles et outils | Non évalué | Aucune preuve collectée : on ne conclut pas sans preuve |
| 8 | Identification | Partiellement conforme | C4 : inventaire limité à 85 % des applications |
| 9 | Protection et prévention | Partiellement conforme | Authentification forte en place ; pas de preuve sur le reste (segmentation, gestion des accès) |
| 10 | Détection | Conforme | SOC 24/7 opérationnel (preuve attendue : rapports du SOC) |
| 11 | Réponse et rétablissement | Partiellement conforme | C7 : PCA testé en 2024, sans le scénario de perte de NovaCore |
| 12 | Sauvegarde et restauration | Non conforme | C5 : sauvegardes du core banking non isolées |
| 13 | Apprentissage et évolution | Partiellement conforme | Tests d'intrusion annuels, mais pas de formation de la direction (C3) |
| 14 | Communication | Non évalué | Plan de communication de crise non examiné |
| 15 et 16 | Harmonisation, cadre simplifié | Non applicable | Art. 15 : mandat donné aux autorités européennes ; art. 16 : réservé aux petites entités, ce qui exclut la banque |
| 17 | Processus de gestion des incidents | Partiellement conforme | C6 : processus existant mais antérieur à DORA |
| 18 | Classification des incidents | Non conforme | C6 : grille interne, sans les critères réglementaires |
| 19 | Notification des incidents majeurs | Non conforme | C6 : chaîne de notification à l'ACPR jamais testée |
| 20 à 22 | Harmonisation, centralisation, retour des autorités | Non applicable | Obligations des autorités, pas de la banque |
| 23 | Incidents liés aux paiements | Non conforme | S'applique aux établissements de crédit ; mêmes lacunes que C6 |
| 24 | Exigences générales des tests | Partiellement conforme | Tests d'intrusion sur l'application mobile seulement ; pas de programme couvrant toutes les FCI (C7) |
| 25 | Tests des outils et systèmes | Partiellement conforme | Même constat que l'article 24 |
| 26 et 27 | Tests de pénétration fondés sur la menace (TLPT) | Non applicable | Hypothèse : la banque n'a pas été désignée par l'ACPR |
| 28 | Principes généraux (tiers) | Non conforme | C8 : registre à 60 % ; C10 : aucune stratégie de sortie |
| 29 | Risque de concentration | Non conforme | Aucune analyse ; l'audit interne a signalé le manque de visibilité |
| 30 | Clauses contractuelles | Partiellement conforme | C9 : clauses absentes chez Kumo Digital et Veridex |
| 31 à 44 | Supervision des prestataires critiques | Non applicable | Concerne les autorités européennes et les prestataires désignés |
| 45 | Partage d'informations | Non applicable | Disposition facultative ; à recommander comme bonne pratique |
| 46 à 64 | Autorités, actes délégués, dispositions finales | Non applicable | Ne créent pas d'obligation pour la banque |

Dans l'outil, l'audit compte 260 exigences évaluables : 126 au chapitre II, 33 au III, 34 au IV, 61 au V, 5 au VI et 1 au IX. Les chapitres I, VII et VIII n'en contiennent aucune : l'éditeur de la bibliothèque les a déjà marqués comme non évaluables, il n'y a rien à saisir. Pour aller vite, utilise le **Mode tableau** (une ligne par exigence, résultat en un clic) et avance article par article, dans l'ordre du tableau G. Compte environ 2 séances pour le chapitre II et 2 pour les chapitres III à VI.

**Méthode pour chaque sous-article : 3 questions, dans l'ordre**

Le tableau G donne la tendance de chaque article, mais l'outil découpe les articles en sous-articles (5.1, 5.2, 5.2 a…), et chacun se juge sur son propre texte. Pour chaque ligne :

1. **Ce sous-article impose-t-il quelque chose à la banque ?** S'il donne une définition, ou s'il s'adresse aux autorités → *Non applicable*.
2. **Ai-je une information sur ce sujet ?** Cherche dans les constats C1 à C10 et les points positifs. Si rien n'en parle → *Non évalué*.
3. **Que dit cette information ?** Exigence entièrement remplie → *Conforme* ; remplie en partie → *Partiellement conforme* ; pas remplie → *Non conforme*.

L'observation reprend en une ou deux phrases ce que tu as trouvé à la question 2, en citant le constat.

**Le résultat étendu** qualifie la gravité d'un écart, avec la nomenclature des audits ISO :

- **Non-conformité majeure** : l'exigence n'est pas remplie du tout, ou l'écart est systémique ;
- **Non-conformité mineure** : le dispositif existe mais présente une lacune ponctuelle ;
- **Observation / point sensible** : pas encore d'écart, mais un risque d'en avoir un ;
- **Piste d'amélioration** : conforme, mais perfectible ;
- **Bonne pratique** : au-delà de l'exigence, à valoriser.

**Exemple détaillé : l'article 5, paragraphe par paragraphe**

| Ligne | Résultat | Résultat étendu | Observation à copier |
| --- | --- | --- | --- |
| 5.1 | Partiellement conforme | Non-conformité mineure | Un dispositif de gouvernance existe (directoire, comité des risques, trois lignes de maîtrise), mais la fonction de contrôle du risque TIC n'est pas indépendante : le RSSI rend compte au DSI (C2). |
| 5.2 | Non conforme | Non-conformité majeure | Le directoire n'a ni approuvé ni revu la politique de gestion du risque TIC, datée de 2022 (C1). |
| 5.2.a | Non conforme | Non-conformité majeure | Faute d'approbation et de revue de la politique (C1), le directoire n'exerce pas dans les faits sa responsabilité ultime sur le risque TIC. |
| 5.2.b | Partiellement conforme | Non-conformité mineure | Des mesures existent (authentification forte, SOC 24/7), mais sans stratégie approuvée par le directoire (C1) ; les sauvegardes du core banking ne sont pas isolées (C5). |
| 5.2.c | Partiellement conforme | Non-conformité mineure | Les fonctions TIC sont identifiées (DSI, RSSI, risques, conformité), mais le rattachement du RSSI au DSI crée une confusion entre exploitation et contrôle (C2). |
| 5.2.d | Non évalué | (aucun) | Aucune stratégie de résilience opérationnelle numérique ni niveau de tolérance au risque TIC présentés lors des entretiens. Preuve à demander. |
| 5.2.e | Partiellement conforme | Non-conformité mineure | Un PCA existe et a été testé en 2024, mais sans le scénario de perte de NovaCore (C7) ; son approbation par le directoire n'est pas démontrée. |
| 5.2.f | Non évalué | (aucun) | L'audit interne est actif, mais l'approbation de ses plans d'audit TIC par le directoire n'a pas été vérifiée. Preuve à demander. |
| 5.2.g | Non évalué | (aucun) | Budget de résilience opérationnelle numérique non examiné. Preuve à demander. |
| 5.2.h | Non évalué | Observation / point sensible | Existence et approbation d'une politique sur les prestataires TIC non vérifiées ; les faiblesses constatées (C8 à C10) laissent penser qu'elle est absente ou non appliquée. |
| 5.2.i | Non conforme | Non-conformité majeure | L'audit interne a signalé en 2025 un manque de visibilité sur le risque lié aux prestataires TIC : aucun canal ne remonte cette information au directoire. |
| 5.2.i.i | Non conforme | Non-conformité majeure | Le registre d'informations ne recense qu'environ 60 % des contrats TIC (C8) : le directoire ne peut pas être informé de tous les accords. |
| 5.2.i.ii | Non conforme | Non-conformité majeure | Faute de canal de remontée (voir 5.2.i), les changements significatifs chez les prestataires ne sont pas portés à la connaissance du directoire. |
| 5.2.i.iii | Non évalué | (aucun) | Remontée au directoire des incidents majeurs et de leurs impacts sur les FCI non examinée. Preuve à demander. |
| 5.3 | Non conforme | Non-conformité majeure | Aucun rôle désigné pour suivre les accords avec les prestataires TIC ; l'audit interne a signalé en 2025 un manque de visibilité sur ce risque. |
| 5.4 | Non conforme | Non-conformité majeure | Aucun membre du directoire n'a suivi de formation sur le risque TIC depuis 2024 (C3). |

La ligne « Aux fins du premier alinéa, l'organe de direction : » n'est qu'une phrase d'introduction : rien à évaluer.

**Une cause racine, plusieurs exigences.** Les lignes 5.2, 5.2.a et 5.2.b remontent toutes au même problème : la politique n'est pas approuvée (C1). Dans le rapport, on en fera **un seul constat** relié à plusieurs exigences, et non trois écarts distincts. Sinon le directoire croit avoir trois problèmes à traiter, alors qu'une seule décision les règle.

**Exemple détaillé : l'article 6, ligne par ligne**

| Ligne | Résultat | Résultat étendu | Observation à copier |
| --- | --- | --- | --- |
| 6.1 | Partiellement conforme | Non-conformité mineure | Un cadre de gestion du risque TIC existe, mais sa politique date de 2022, n'a jamais été revue ni approuvée par le directoire (C1). |
| 6.2 | Partiellement conforme | Non-conformité mineure | Le cadre ne couvre pas tous les actifs : l'inventaire n'atteint qu'environ 85 % des applications, le reste relevant du shadow IT (C4). |
| 6.3 | Partiellement conforme | Non-conformité mineure | Des mesures réduisent le risque (authentification forte, SOC 24/7), mais des lacunes subsistent (sauvegardes non isolées, C5). Capacité à fournir ces informations à l'ACPR non vérifiée. |
| 6.4 | Non conforme | Non-conformité majeure | La fonction de contrôle du risque TIC n'est pas indépendante : le RSSI rend compte au DSI, qui pilote l'exploitation qu'il est censé contrôler (C2). |
| 6.5 | Non conforme | Non-conformité majeure | Le cadre n'a pas été réexaminé depuis 2022, alors qu'un réexamen au moins annuel est exigé (C1). |
| 6.6 | Non évalué | (aucun) | L'audit interne a traité le risque TIC en 2025 (prestataires) ; le plan d'audit, la fréquence et les compétences TIC des auditeurs restent à vérifier. Preuve à demander. |
| 6.7 | Non conforme | Non-conformité mineure | Le constat de l'audit interne de 2025 sur les prestataires TIC n'a pas été corrigé (registre à 60 %, C8 ; aucun rôle de suivi, 5.3) : le suivi des recommandations n'est pas efficace. |
| 6.8 et 6.8.a à 6.8.h | Non évalué | (aucun) | Aucune stratégie de résilience opérationnelle numérique présentée lors des entretiens (voir 5.2.d). Preuve à demander. |
| 6.9 | Non applicable | Piste d'amélioration | Faculté, et non obligation (« peuvent »). Recommandé au vu de la dépendance à Stratus Cloud et à NovaCore. |
| 6.10 | Non applicable | (aucun) | Faculté non utilisée : la banque n'a pas externalisé la vérification de sa conformité. |

Trois enseignements de l'article 6 :

- **« Peuvent » n'est pas « doivent ».** Les paragraphes 6.9 et 6.10 ouvrent une faculté : on ne peut pas reprocher à la banque de ne pas l'utiliser. On peut en revanche la recommander.
- **Même constat, gravité différente.** C2 donne une non-conformité mineure au 5.1 (exigence large, le cadre de gouvernance existe) mais majeure au 6.4, dont l'indépendance est l'objet même.
- **Chaque fait s'évalue là où se trouve son exigence.** Les sous-points 6.8.e et 6.8.g parlent de détection et de tests, mais ils demandent que la *stratégie* les décrive. Le SOC et les tests seront jugés aux articles 10 et 24 ; ici, faute de stratégie présentée, c'est « Non évalué ».

## Plan de finition en une journée

Pour boucler la mission, on applique une règle d'audit courante quand le temps est compté : **l'audit « borné dans le temps »**. On accepte une granularité moindre, et on le déclare comme limite dans le rapport.

**Articles 7 et 8**

| Ligne | Résultat | Résultat étendu | Observation à copier |
| --- | --- | --- | --- |
| 7 (introduction) et 7.a à 7.d | Non évalué | (aucun) | Aucune information sur la capacité, la fiabilité et la résilience des systèmes TIC (dimensionnement, tests de charge). Preuve non fournie à la date de clôture. |
| 8.1 | Partiellement conforme | Non-conformité mineure | Les FCI sont identifiées, mais l'inventaire des actifs qui les soutiennent est incomplet (C4) et sa revue annuelle n'est pas démontrée. |
| 8.2 | Non évalué | Observation / point sensible | Aucune analyse de risques ni revue annuelle des scénarios présentée ; l'analyse EBIOS RM de la mission semble être la première. |
| 8.3 | Non évalué | (aucun) | Évaluation des risques lors des changements importants non examinée. Preuve non fournie à la date de clôture. |
| 8.4 | Partiellement conforme | Non-conformité mineure | Environ 15 % des applications échappent à l'inventaire (shadow IT, C4) ; les interdépendances n'étaient pas cartographiées avant la mission. |
| 8.5 | Non conforme | Non-conformité majeure | Les processus dépendant de prestataires TIC ne sont pas tous documentés : le registre d'informations ne couvre qu'environ 60 % des contrats (C8). |
| 8.6 | Partiellement conforme | Non-conformité mineure | Les inventaires existent mais sont incomplets et ne sont pas tenus à jour (C4). |
| 8.7 | Non évalué | (aucun) | Évaluation des systèmes hérités non examinée. Preuve non fournie à la date de clôture. |

**Articles 9 à 64 : méthode rapide**

En Mode tableau, pour chaque article, applique à toutes ses lignes le résultat du tableau G et colle en observation la justification du tableau G. Deux exceptions à vérifier en lisant chaque ligne : une ligne qui commence par « peuvent » → Non applicable ; une ligne qui s'adresse aux autorités → Non applicable. Statut : « Terminé » partout, y compris pour les « Non évalué » (avec la mention « Preuve non fournie à la date de clôture »).

**Étape 7 express : le plan de remédiation**

Depuis l'audit, bouton « Plan d'action » : crée une mesure par ligne ci-dessous et relie-la aux exigences concernées.

**Tableau H : les 11 mesures, dans l'ordre du formulaire « Ajouter une mesure appliquée »** (Opérations → Mesures appliquées). Un sous-tableau par section du formulaire : H1 (champs principaux), H2 (Gestion de projet), H3 (Coût), H4 (Relations).

**H1 : champs principaux**

Pour toutes les mesures : **Mesure de référence** vide, **Domaine** Banque Ardelle, **Assigné à** vide, **Statut** À faire, **Progression** 0, **Preuves** vide.

| ID | Nom | Description | Catégorie | ETA |
| --- | --- | --- | --- | --- |
| M01 | Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle | Responsable : Directeur des risques. Mettre à jour la politique de 2022, la présenter au directoire pour approbation et inscrire une revue annuelle à son calendrier. | Politique | 31/12/2026 |
| M02 | Rattacher le RSSI à la direction des risques (2e ligne) | Responsable : Directoire. Modifier l'organigramme pour que la fonction de contrôle du risque TIC soit indépendante de la DSI. | Processus | 31/01/2027 |
| M03 | Former les membres du directoire au risque TIC | Responsable : DRH et RSSI. Session annuelle de formation du directoire au risque TIC et à DORA. | Processus | 28/02/2027 |
| M04 | Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration | Responsable : DSI. Mettre en place une copie des sauvegardes hors ligne ou immuable, séparée de la production, et tester la restauration complète. | Technique | 31/03/2027 |
| M05 | Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR | Responsable : RSSI. Refondre la grille de classification selon l'article 18 et ses RTS, intégrer les modèles de notification et réaliser un exercice de notification. | Processus | 31/03/2027 |
| M06 | Compléter le registre d'informations à 100 % des contrats TIC | Responsable : Achats et conformité. Recenser les 40 % de contrats manquants et distinguer ceux qui soutiennent des FCI. | Processus | 31/03/2027 |
| M07 | Compléter l'inventaire des actifs, shadow IT compris | Responsable : DSI. Identifier les applications hors inventaire, les classer et les rattacher aux FCI. | Processus | 30/06/2027 |
| M08 | Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus | Responsable : Responsable PCA. Formaliser le BIA, ajouter le scénario NovaCore au PCA et tester chaque année les systèmes des FCI. | Processus | 30/06/2027 |
| M09 | Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie) | Responsable : Juridique. Ajouter les clauses de l'article 30 : droit d'audit, lieux de traitement, résiliation et période de transition. | Processus | 30/06/2027 |
| M10 | Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration | Responsable : DSI et risques. Documenter les solutions de repli, les plans de transition et l'analyse de substituabilité, puis les tester. | Processus | 30/09/2027 |
| M11 | Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires | Responsable : Directoire. Nommer un membre de la direction chargé du suivi des prestataires et faire approuver une stratégie dédiée. | Politique | 31/01/2027 |

**H2 : section Gestion de projet**

| ID | Priorité | Date de début | Date d'expiration | Effort | Impact | Observation |
| --- | --- | --- | --- | --- | --- | --- |
| M01 | P1 | 02/11/2026 | 31/12/2027 | Petit | Haut | Gain rapide. Prérequis de M03. |
| M02 | P1 | 02/11/2026 | Vide | Petit | Haut | Gain rapide ; décision du directoire. |
| M03 | P1 | 04/01/2027 | 28/02/2028 | Petit | Moyen | Démarre après l'approbation de la politique (M01). À renouveler chaque année. |
| M04 | P1 | 16/11/2026 | Vide | Moyen | Haut | Réduit le risque rançongiciel sur FCI-01 (scénario 1). |
| M05 | P1 | 01/12/2026 | Vide | Moyen | Haut | Indispensable avant le contrôle ACPR du T2 2027. |
| M06 | P1 | 02/11/2026 | Vide | Moyen | Haut | Sur le chemin critique : prérequis de M09 et M10. |
| M07 | P2 | 04/01/2027 | Vide | Moyen | Moyen | Peut s'appuyer sur les résultats de M06. |
| M08 | P2 | 01/02/2027 | Vide | Grand | Haut | Démarre après le lancement de M04. |
| M09 | P2 | 01/02/2027 | Vide | Moyen | Moyen | Démarre après M06 et M11. |
| M10 | P2 | 01/03/2027 | Vide | Très grand | Haut | Chemin critique ; ne sera pas achevée avant le contrôle ACPR : présenter le plan et l'avancement. |
| M11 | P1 | 02/11/2026 | Vide | Petit | Moyen | Gain rapide ; prérequis de M09. |

**H3 : section Coût** (estimations fictives, pour l'exercice)

| ID | Build : coût fixe (€) | Build : jours-personnes | Durée d'amortissement (années) | Run : coût fixe annuel (€) | Run : jours-personnes annuels |
| --- | --- | --- | --- | --- | --- |
| M01 | 0 | 10 | 1 | 0 | 3 |
| M02 | 0 | 5 | 1 | 0 | 0 |
| M03 | 15 000 | 5 | 1 | 8 000 | 3 |
| M04 | 120 000 | 40 | 5 | 20 000 | 15 |
| M05 | 20 000 | 30 | 1 | 0 | 10 |
| M06 | 0 | 60 | 1 | 0 | 20 |
| M07 | 30 000 | 40 | 3 | 10 000 | 10 |
| M08 | 50 000 | 80 | 1 | 30 000 | 30 |
| M09 | 25 000 | 20 | 1 | 0 | 0 |
| M10 | 60 000 | 100 | 1 | 0 | 15 |
| M11 | 0 | 10 | 1 | 0 | 55 |
| Total | 320 000 | 400 |  | 68 000 | 161 |

Une durée de 1 an signifie que le coût est passé en charge dans l'année : c'est le cas des prestations (conseil, formation, avocat) et du temps interne. Seuls les achats durables s'amortissent sur plusieurs années : le stockage immuable de M04 (5 ans) et l'outil de découverte de M07 (3 ans).

**H4 : section Relations**

Pour toutes les mesures : **Objectifs**, **Exceptions** et **Lien** vides.

| ID | Fonction | Actifs | Étiquettes | Exigences à relier ensuite dans l'audit |
| --- | --- | --- | --- | --- |
| M01 | Gouverner | (aucun) | DORA, C1 | 5.2, 5.2.a, 6.1, 6.5, 6.7, 9.4.a |
| M02 | Gouverner | (aucun) | DORA, C2 | 5.1, 5.2.c, 6.4 |
| M03 | Gouverner | (aucun) | DORA, C3 | 5.4, 13.6 |
| M04 | Récupérer | FCI-01, SUP-03 | DORA, C5 | 9.3.b, 9.3.c, 12.2, 12.3 |
| M05 | Répondre | FCI-02 | DORA, C6 | 17.1, 17.3.b, 17.3.f, 18.1, 19.1, alinéas modèles et contenu, 19.4, 19.4.a à c, 23 |
| M06 | Identifier | SUP-01, SUP-04, SUP-06, SUP-09, SUP-10 | DORA, C8 | 5.2.i.i, 8.5, 28.3 et ses alinéas |
| M07 | Identifier | (aucun) | DORA, C4 | 6.2, 8.1, 8.4, 8.6, 9.1 |
| M08 | Récupérer | FCI-01, SUP-01 | DORA, C7 | 11.1, 11.2, 11.4, 11.5, 11.6.a, alinéa scénarios, 12.6, 24.1, 24.2, 24.6, 25.1 |
| M09 | Gouverner | SUP-06, SUP-09 | DORA, C9 | 28.6, 28.7.a à d, 30.2.h, 30.3.e, e.i, e.iii, e.iv, 30.3.f, f.i, f.ii |
| M10 | Gouverner | SUP-01, SUP-10 | DORA, C10 | 10.1, 11.2.a, 28.4.c, 28.8 et ses sous-lignes, 29.1.a, 29.1.b, alinéas solutions alternatives et insolvabilité |
| M11 | Gouverner | (aucun) | DORA | 5.2.i, 5.2.i.ii, 5.3, 28.1, 28.1.b, 28.1.b.i, 28.1.b.ii, 28.2 |

Les dates de début suivent les dépendances entre mesures : M03 démarre après l'approbation de la politique (M01) ; M09 après le registre complet (M06) et la stratégie prestataires (M11) ; M08 après le lancement des sauvegardes isolées (M04) ; M10 en dernier, car il s'appuie sur le registre (M06) et le PCA étendu (M08). La date d'expiration ne concerne que les mesures qui doivent être renouvelées : la politique (revue annuelle) et la formation du directoire (recyclage annuel).

H1 à H4 se remplissent dans le formulaire « Ajouter » ; la dernière colonne de H4 se traite ensuite dans l'audit, exigence par exigence (bouton « Mesure appliquée »). La colonne « Objets associés » de la liste se remplit alors toute seule.

**Tableau I : relier les mesures aux exigences de l'audit**

Dans l'audit AUD-DORA-01, ouvre chaque exigence ci-dessous, clique sur le bouton « Mesure appliquée » et sélectionne la mesure. Coche la case une fois la liaison enregistrée. Le résultat entre parenthèses t'aide à vérifier que tu es sur la bonne ligne.

**M01 – Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle**

- [ ] 5.2 (non conforme)
- [ ] 5.2.a (non conforme)
- [ ] 6.1 (partiellement conforme)
- [ ] 6.5 (non conforme)
- [ ] 6.7 (non conforme)
- [ ] 9.4.a (partiellement conforme)

**M02 – Rattacher le RSSI à la direction des risques (2e ligne)**

- [ ] 5.1 (partiellement conforme)
- [ ] 5.2.c (partiellement conforme)
- [ ] 6.4 (non conforme)

**M03 – Former les membres du directoire au risque TIC**

- [ ] 5.4 (non conforme)
- [ ] 13.6 (non conforme)

**M04 – Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration**

- [ ] 9.3.b (partiellement conforme)
- [ ] 9.3.c (partiellement conforme)
- [ ] 12.2 (non conforme)
- [ ] 12.3 (non conforme)

**M05 – Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR**

- [ ] 17.1 (partiellement conforme)
- [ ] 17.3.b (non conforme)
- [ ] 17.3.f (partiellement conforme)
- [ ] 18.1 (non conforme)
- [ ] 19.1 (non conforme)
- [ ] Article 19, alinéa « modèles de notification » (non conforme)
- [ ] Article 19, alinéa « informations nécessaires » (non conforme)
- [ ] 19.4 (non conforme)
- [ ] 19.4.a (non conforme)
- [ ] 19.4.b (non conforme)
- [ ] 19.4.c (non conforme)
- [ ] Article 23, alinéa unique (non conforme)

**M06 – Compléter le registre d'informations à 100 % des contrats TIC**

- [ ] 5.2.i.i (non conforme)
- [ ] 8.5 (non conforme)
- [ ] 28.3 (non conforme)
- [ ] Article 28.3, alinéa « distinction des contrats FCI » (non conforme)
- [ ] Article 28.3, alinéa « mise à disposition du registre » (non conforme)

**M07 – Compléter l'inventaire des actifs, shadow IT compris**

- [ ] 6.2 (partiellement conforme)
- [ ] 8.1 (partiellement conforme)
- [ ] 8.4 (partiellement conforme)
- [ ] 8.6 (partiellement conforme)
- [ ] 9.1 (partiellement conforme)

**M08 – Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus**

- [ ] 11.1 (partiellement conforme)
- [ ] 11.2 (partiellement conforme)
- [ ] 11.4 (non conforme)
- [ ] 11.5 (partiellement conforme)
- [ ] 11.6.a (non conforme)
- [ ] Article 11.6, alinéa « scénarios de cyberattaque et de basculement » (non conforme)
- [ ] 12.6 (partiellement conforme)
- [ ] 24.1 (partiellement conforme)
- [ ] 24.2 (partiellement conforme)
- [ ] 24.6 (non conforme)
- [ ] 25.1 (partiellement conforme)

**M09 – Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)**

- [ ] 28.6 (non conforme)
- [ ] 28.7.a (partiellement conforme)
- [ ] 28.7.b (partiellement conforme)
- [ ] 28.7.c (partiellement conforme)
- [ ] 28.7.d (partiellement conforme)
- [ ] 30.2.h (partiellement conforme)
- [ ] 30.3.e (partiellement conforme)
- [ ] 30.3.e.i (partiellement conforme)
- [ ] 30.3.e.iii (partiellement conforme)
- [ ] 30.3.e.iv (partiellement conforme)
- [ ] 30.3.f (partiellement conforme)
- [ ] 30.3.f.i (partiellement conforme)
- [ ] 30.3.f.ii (partiellement conforme)

**M10 – Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration**

- [ ] 10.1 (partiellement conforme)
- [ ] 11.2.a (partiellement conforme)
- [ ] 28.4.c (non conforme)
- [ ] 28.8 (non conforme)
- [ ] Article 28.8, alinéa « se retirer des accords sans… » (non conforme)
- [ ] 28.8.a (non conforme)
- [ ] 28.8.b (non conforme)
- [ ] 28.8.c (non conforme)
- [ ] Article 28.8, alinéa « plans de sortie complets et testés » (non conforme)
- [ ] Article 28.8, alinéa « solutions alternatives et plans de transition » (non conforme)
- [ ] Article 28.8, alinéa « mesures d'urgence » (non conforme)
- [ ] 29.1.a (non conforme)
- [ ] 29.1.b (non conforme)
- [ ] Article 29.1, alinéa « solutions alternatives » (non conforme)
- [ ] Article 29.2, alinéa « insolvabilité » (non conforme)

**M11 – Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires**

- [ ] 5.2.i (non conforme)
- [ ] 5.2.i.ii (non conforme)
- [ ] 5.3 (non conforme)
- [ ] 28.1 (non conforme)
- [ ] 28.1.b (non conforme)
- [ ] 28.1.b.i (non conforme)
- [ ] 28.1.b.ii (non conforme)
- [ ] 28.2 (non conforme)

**Contrôle final de couverture** : une fois toutes les cases cochées, filtre l'audit sur les résultats « Non conforme » et « Partiellement conforme ». Il reste quelques lignes partiellement conformes sans mesure (par exemple 5.2.b, 5.2.e, 6.3, 9.2, 9.3, 11.2.b, 11.2.e, 12.1) : relie-les à la mesure la plus proche (M01 pour 6.3 et 9.2, M04 pour 5.2.b, 9.3 et 12.1, M05 pour 11.2.b et 11.2.e, M08 pour 5.2.e). Un écart sans mesure est un écart oublié.

**Tableau I bis : liaisons dans l'ordre de l'audit (à utiliser de préférence)**

Une ligne par exigence, dans l'ordre où « Enregistrer et suivant » les fait défiler. Le nom de la mesure est exactement celui du tableau H1, donc celui de la liste de l'outil. Les exigences absentes de cette liste n'ont pas de mesure à relier (conformes, non applicables ou non évaluées) : passe-les directement.

**Article 5**

- [ ] 5.1 → Rattacher le RSSI à la direction des risques (2e ligne)
- [ ] 5.2 → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle
- [ ] 5.2.a → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle
- [ ] 5.2.b → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 5.2.c → Rattacher le RSSI à la direction des risques (2e ligne)
- [ ] 5.2.e → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 5.2.i → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 5.2.i.i → Compléter le registre d'informations à 100 % des contrats TIC
- [ ] 5.2.i.ii → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 5.3 → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 5.4 → Former les membres du directoire au risque TIC

**Article 6**

- [ ] 6.1 → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle
- [ ] 6.2 → Compléter l'inventaire des actifs, shadow IT compris
- [ ] 6.3 → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle
- [ ] 6.4 → Rattacher le RSSI à la direction des risques (2e ligne)
- [ ] 6.5 → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle
- [ ] 6.7 → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle

**Article 8**

- [ ] 8.1 → Compléter l'inventaire des actifs, shadow IT compris
- [ ] 8.4 → Compléter l'inventaire des actifs, shadow IT compris
- [ ] 8.5 → Compléter le registre d'informations à 100 % des contrats TIC
- [ ] 8.6 → Compléter l'inventaire des actifs, shadow IT compris

**Article 9**

- [ ] 9.1 → Compléter l'inventaire des actifs, shadow IT compris
- [ ] 9.2 → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle
- [ ] 9.3 → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 9.3.b → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 9.3.c → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 9.4.a → Faire approuver la politique de gestion du risque TIC par le directoire et instaurer sa revue annuelle

**Article 10**

- [ ] 10.1 → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration

**Article 11**

- [ ] 11.1 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 11.2 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 11.2.a → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] 11.2.b → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 11.2.e → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 11.4 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 11.5 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 11.6.a → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] Alinéa « scénarios de cyberattaques et de basculement » (après 11.6.b) → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus

**Article 12**

- [ ] 12.1 → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 12.2 → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 12.3 → Isoler les sauvegardes du core banking (copie hors ligne ou immuable) et tester la restauration
- [ ] 12.6 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus

**Article 13**

- [ ] 13.6 → Former les membres du directoire au risque TIC

**Articles 17 à 23**

- [ ] 17.1 → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 17.3.b → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 17.3.f → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 18.1 → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 19.1 → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] Alinéa « modèles de notification » (après 19.1) → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] Alinéa « informations nécessaires » (après 19.1) → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 19.4 → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 19.4.a → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 19.4.b → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] 19.4.c → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR
- [ ] Article 23, alinéa unique → Aligner la classification des incidents sur les critères réglementaires et tester la notification à l'ACPR

**Articles 24 et 25**

- [ ] 24.1 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 24.2 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 24.6 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus
- [ ] 25.1 → Étendre le PCA, le BIA et le programme de tests aux FCI, scénario de perte de NovaCore inclus

**Article 28**

- [ ] 28.1 → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 28.1.b → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 28.1.b.i → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 28.1.b.ii → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 28.2 → Désigner un responsable du suivi des prestataires TIC et adopter une stratégie relative aux prestataires
- [ ] 28.3 → Compléter le registre d'informations à 100 % des contrats TIC
- [ ] Alinéa « distinction des contrats FCI » (après 28.3) → Compléter le registre d'informations à 100 % des contrats TIC
- [ ] Alinéa « mise à disposition du registre » (après 28.3) → Compléter le registre d'informations à 100 % des contrats TIC
- [ ] 28.4.c → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] 28.6 → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 28.7.a → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 28.7.b → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 28.7.c → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 28.7.d → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 28.8 → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] Alinéa « se retirer des accords sans… » (après 28.8) → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] 28.8.a → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] 28.8.b → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] 28.8.c → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] Alinéa « plans de sortie complets et testés » → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] Alinéa « solutions alternatives et plans de transition » → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] Alinéa « mesures d'urgence » → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration

**Article 29**

- [ ] 29.1.a → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] 29.1.b → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] Alinéa « solutions alternatives » (après 29.1.b) → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration
- [ ] Alinéa « insolvabilité » (après 29.2) → Rédiger les stratégies de sortie NovaCore et Stratus Cloud et analyser le risque de concentration

**Article 30**

- [ ] 30.2.h → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.e → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.e.i → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.e.iii → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.e.iv → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.f → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.f.i → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)
- [ ] 30.3.f.ii → Renégocier les contrats Kumo Digital et Veridex (droit d'audit, clauses de sortie)

M10 ne sera pas achevée avant le contrôle de l'ACPR (T2 2027). Ce n'est pas un échec : un contrôleur attend un plan crédible, daté et suivi, pas une conformité totale.

**Clôture de la mission**

- [ ] Toutes les lignes de l'audit au statut « Terminé »
- [ ] Plan d'action saisi (M01 à M10)
- [ ] Statut de l'audit : « Terminé » ; export du rapport (bouton Exporter)
- [ ] Audit verrouillé
- [ ] Sauvegarde du dossier `db` et snapshot de la VM
- [ ] Étape 6 (EBIOS RM) reportée : à faire comme exercice séparé si le temps le permet

Pour chaque ligne terminée, passe le **Statut** à « Terminé ». Le résultat de l'article (« non conforme » dans le tableau G) est la synthèse de ses paragraphes : il suffit qu'un paragraphe essentiel soit non conforme pour que l'article le soit.

Piège classique illustré par les articles 7 et 14 : **« non évalué » n'est pas « conforme »**. Sans preuve, on ne conclut pas ; on le signale comme une limite de l'audit.

1. Chapitres II à VI : évaluer chaque exigence (conforme, partiellement conforme, non conforme) à partir des constats C1 à C10, avec un commentaire qui cite le constat.
2. Chapitres I, VII, VIII et IX : marquer « non applicable » et justifier en une phrase.
3. Créer un constat (Gouvernance → Constats) pour chaque écart important.

- [ ] Chapitre II évalué
- [ ] Chapitres III à VI évalués
- [ ] Chapitres non applicables justifiés

### Étape 6 : mener l'analyse EBIOS RM

**Comprendre**

EBIOS Risk Manager est la méthode d'analyse de risques de l'ANSSI. Elle se lit comme une enquête menée à l'envers : on part de ce qu'on ne veut pas perdre, on se demande qui voudrait nous le prendre, puis par quel chemin il passerait. Elle se déroule en 5 ateliers :

1. **Cadrage et socle de sécurité** : périmètre, valeurs métier, biens supports, événements redoutés et leur gravité, référentiels applicables.
2. **Sources de risque** : qui pourrait attaquer (cybercriminel, initié, hacktiviste…) et pour quel objectif.
3. **Scénarios stratégiques** : les grands chemins d'attaque, en passant souvent par l'**écosystème** (prestataires, partenaires), le maillon faible le plus fréquent.
4. **Scénarios opérationnels** : le détail technique d'un chemin, étape par étape, avec sa vraisemblance.
5. **Traitement du risque** : les mesures retenues, le risque résiduel et son acceptation.

Le **niveau de risque** combine la gravité de l'événement redouté et la vraisemblance du scénario, placés dans la matrice 4x4. Le **risque résiduel** est celui qui reste après les mesures : il n'est jamais nul, et c'est à la direction de l'accepter formellement.

**Le mot juste** : événement redouté, source de risque, objectif visé, partie prenante, scénario stratégique, scénario opérationnel, vraisemblance, risque résiduel.

**Vérifie ta maîtrise**

- Pourquoi EBIOS RM insiste-t-il autant sur l'écosystème ? Relie ta réponse au scénario 1 du cas.
- Le scénario 3 (panne de NovaCore) n'a pas d'attaquant : a-t-il sa place dans une analyse EBIOS RM ? Pourquoi est-il essentiel sous DORA ?
- Qui doit accepter le risque résiduel, et pourquoi pas le RSSI ?

**Faire**

Menu : Risque → EBIOS RM → nouvelle étude, avec la matrice 4x4 EBIOS-RM et le périmètre FCI DORA.

- [ ] Ateliers 1 à 3 saisis à partir de la section Analyse de risques
- [ ] Ateliers 4 et 5 rédigés par toi (scénarios 1 et 3 au minimum)

### Étape 7 : construire le plan de remédiation

**Comprendre**

Face à un risque, il existe quatre réponses : le **réduire** (mettre une mesure), le **transférer** (assurance, contrat), l'**éviter** (arrêter l'activité concernée) ou l'**accepter** (en connaissance de cause, décision de la direction). Un plan de remédiation traduit ces choix en projet. C'est là que ta casquette de chef de projet PMP prend toute sa valeur : chaque mesure a un responsable, une échéance, une charge et des dépendances.

Pour prioriser, croise trois critères :

- le **niveau de risque** couvert : on traite d'abord ce qui menace les FCI ;
- l'**urgence réglementaire** : ici, le contrôle de l'ACPR au 2e trimestre 2027 ;
- l'**effort** : les gains rapides (faire approuver la politique par le directoire, former la direction) montrent vite des progrès.

Un bon plan se présente au directoire dans son langage : risques pour l'activité, coûts, décisions attendues. Pas de jargon technique.

**Le mot juste** : réduction, transfert, évitement, acceptation du risque, plan de remédiation, gain rapide, lot, feuille de route.

**Vérifie ta maîtrise**

- Quelles mesures du cas sont des gains rapides, et lesquelles demandent plus de 6 mois ?
- Pourquoi la stratégie de sortie NovaCore ne peut-elle pas être bouclée en quelques semaines ?
- Comment présenterais-tu en 3 phrases au directoire le risque lié à C5 (sauvegardes non isolées) ?

**Faire**

Pour chaque écart, crée une mesure avec un responsable, une échéance et une priorité, puis regroupe-les en lots sur 12 mois.

- [ ] Mesures créées et priorisées
- [ ] Rapport d'audit et plan de traitement exportés
- [ ] Snapshot de la VM et sauvegarde du dossier `db`

## Livrables et entretien

**Livrables à produire et à publier (anonymisés) sur ton GitHub**

1. Un rapport d'écarts DORA : taux de conformité par pilier et top 5 des écarts.
2. Un plan de remédiation sur 12 mois, en lots priorisés, avec charge estimée en jours-homme. C'est là que ta casquette PMP fait la différence.
3. Une note de 2 pages pour le comité des risques, dans un registre de consultant : décisions attendues du directoire, risques résiduels acceptés.

**Questions d'entretien auxquelles ce cas te prépare**

- Comment distinguez-vous une fonction critique ou importante d'une fonction ordinaire, et pourquoi cette distinction change-t-elle tout ?
- Que contient un registre d'informations, et comment le tenez-vous à jour ?
- Un prestataire cloud refuse le droit d'audit : que proposez-vous ?
- Comment articulez-vous DORA, NIS2 et ISO 27001 sans faire trois fois le travail ?
- Comment présentez-vous un risque TIC à un directoire qui n'est pas technique ?
