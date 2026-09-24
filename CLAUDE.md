# Règles de travail — `scomm_rhel9`

Contexte du projet : faire obtenir à des machines RHEL 9 un certificat destiné à SCOM,
délivré par une ADCS via CEP/CES derrière un reverse proxy F5, et maintenu par
certmonger.

Ce fichier ne porte **que** des règles, chacune avec son motif. Une règle sans motif se
fait « optimiser » par la session suivante.

Les **faits**, les **points ouverts** et les **décisions** ne vivent pas ici. Depuis la
séparation du 2026-09-21 (D-006), ils se répartissent par **nature** sur trois fichiers,
et chacun n'existe qu'à un seul endroit :

| Fichier | Ce qu'il porte | Quand le lire |
|---|---|---|
| `docs/scomm-certificat-facts.md` | Contraintes, faits sourcés, faits déclarés, points ouverts, décisions, convention de marquage | **À chaque session.** |
| `docs/scomm-depot-actuel.md` | Ce que le code du dépôt fait aujourd'hui — brouillon sans autorité (F-044), périmable en bloc (D-007) | Avant de toucher au code existant. |
| `docs/scomm-journal-preuves.md` | Les sorties de commande `P-xx` | Quand un fait est contesté, ou pour rejouer une mesure. |

`[RÉVISÉE le 2026-09-21]` **ÉTAIT :** « ils vivent dans `docs/scomm-certificat-facts.md`,
qui est le dépôt unique ».

Une règle vit à un seul endroit ; ne recopiez pas, renvoyez.

**Ce fichier ne doit contenir aucune occurrence du marqueur de vérification** défini dans
`docs/scomm-certificat-facts.md` § *Convention de marquage* — y compris dans une phrase
explicative. *Motif :* le contrôle automatique qui cherche ce marqueur dans ce fichier
n'a de valeur que si toute occurrence est une vraie violation. Une seule mention
pédagogique suffit à rendre le contrôle inexploitable, donc ignoré.

---

## R-01 — Rien n'est affirmé sans source lue

Toute affirmation technique doit provenir d'un artefact effectivement lu : un fichier
(cité `chemin:ligne`), une sortie de commande (citée avec la commande **et** son code de
retour), du code source à la version installée, ou une documentation nommée. Une
documentation externe faisant autorité est recevable si elle est **nommée** et étiquetée
`[SOURCE-EXTERNE]`. À défaut, l'affirmation porte le marqueur de vérification et indique
où la confirmer.

Restent interdites : **la mémoire** et **la plausibilité**. Un chiffre non sourcé ne se
pose pas — ni une taille de clé, ni un chemin, ni un mode de fichier, ni un numéro de
version.

*Motif :* ce projet touche une chaîne (AD, ADCS, CEP/CES, F5, certmonger, agent SCX) où
tout est vraisemblable et où presque rien n'est vérifiable de tête. Une valeur inventée
qui « ressemble » à la bonne coûte plus cher qu'une case vide, parce qu'elle ne se
signale pas.

## R-02 — La règle vaut contre l'opérateur et contre l'énoncé

Si une prémisse d'un énoncé est fausse, la vérifier et la contredire, en produisant la
sortie qui l'établit. Si une consigne est impossible ou en contredit une autre :
s'arrêter et le dire, plutôt que de choisir silencieusement.

*Motif :* cette règle a déjà servi. L'énoncé du livrable 1 supposait `rhel_post_install`
présent sur le poste ; il ne l'est pas, et toute une étape en dépendait. Choisir en
silence aurait produit un document d'inventaire fondé sur rien.

## R-03 — Échec bruyant

Une commande sans sortie n'établit rien : relever le code de retour. **Une variable
commentée est une variable absente.** Avant de prescrire un contrôle, se demander s'il
peut échouer — un contrôle qui ne peut pas échouer ne contrôle rien.

*Motif :* l'inventaire de ce dépôt en donne l'illustration : `inventory/hosts.ini` a
l'air rempli, mais toutes ses lignes d'hôtes sont commentées et le groupe est vide
(`docs/scomm-depot-actuel.md`, F-010 — `[RÉVISÉ le 2026-09-21]` : ce fait vivait en
`docs/scomm-certificat-facts.md` avant la séparation D-006). Deux tâches de
`register.yml` sont sautées aux valeurs par défaut (F-018, même fichier). Rien de tout
cela ne se voit sans relever un code de retour ou lire la condition.
*Le motif survit à la réécriture du code qui l'illustre :* même si `register.yml`
disparaît (D-007), ce qui est démontré ici — un artefact qui a l'air rempli et ne l'est
pas — reste vrai.

## R-04 — Une branche non exercée est non prouvée

Énumérer explicitement, dans le compte rendu, les branches conditionnelles exercées et
celles qui ne l'ont pas été. Ne jamais présenter comme « fait » ce qui n'a été que
« écrit ».

*Motif :* les `when:` de ce dépôt (`register.yml:21-23`, `register.yml:33`,
`tasks/main.yml:12`) sont exactement les endroits où le code semble faire quelque chose
et ne fait rien. **Un `when:` relève de cette règle et non de R-12** (D-011) : il oriente,
il ne garde pas — et s'il peut sauter là où l'on attend un effet, R-12 exige qu'une garde
vérifie cet effet.

## R-05 — Aucune écriture, aucune publication d'initiative

- Pas de `git commit`, pas de `git push` sans demande explicite.
- Si une authentification échoue : s'arrêter et rapporter. **Ne pas** chercher de jeton,
  **ne pas** modifier l'URL d'un remote, **ne pas** configurer de `credential.helper`.
- Édition **ciblée** uniquement sur un fichier suivi. Jamais de `cat >>`, jamais de
  `rm` + recréation, jamais de script qui réécrit un fichier entier.

*Motif :* le push est le seul acte irréversible de ce dépôt, et la configuration d'accès
appartient à l'opérateur. La réécriture globale d'un fichier suivi détruit un travail
concurrent sans laisser de trace lisible dans le diff.

## R-06 — Le poste est personnel et ne s'installe pas

Rien ne s'installe sur ce poste : ni paquet, ni venv, ni `pip`. Un venv
`ansible-lint` / `yamllint` existe déjà et partage l'interpréteur système — ne jamais en
créer un second. `sudo` est sans mot de passe ici : **toute** élévation doit être
rapportée, et précédée d'une tentative sans privilège quand une voie non privilégiée est
plausible.

*Motif :* un `sudo` qui ne demande rien ne se remarque pas ; seul le compte rendu le rend
visible. Un second venv diverge silencieusement du premier et rend les résultats de lint
non reproductibles.

## R-07 — Le dépôt distant est public : aucune valeur réelle dans les fichiers versionnés

Mesuré, pas supposé (`docs/scomm-certificat-facts.md`, F-024). Aucun fichier versionné ne
doit contenir de nom d'hôte réel, d'URL interne, de realm Kerberos, de nom de VIP,
d'adresse IP interne ni de secret. Employer des valeurs d'exemple : `ca.example.com`,
`EXAMPLE.COM`, `host01`. Cela vaut aussi pour les sorties collées dans le journal de
preuves.

**Étendu le 2026-09-18 :** cela vaut également pour les **identifiants personnels** — nom
du compte local, chemins absolus qui le contiennent, nom civil, adresse de courriel.
Écrire `/home/<user>/…`.

*Motif de l'extension* `[RÉVISÉ le 2026-09-21, voir D-008]` **— la portabilité, et elle
seule.** Un chemin absolu qui contient un compte local n'est portable nulle part : ni
vers le nœud de contrôle, ni vers une cible RHEL 9, ni vers la machine d'un autre
opérateur. `/home/<user>/…` se transpose ; le chemin réel se copie-colle et échoue. Cet
argument ne dépend d'aucun arbitrage en cours.

**ÉTAIT :** « la formulation initiale n'énumérait que de la topologie d'entreprise ;
l'identifiant du compte local est passé au travers et s'est retrouvé 13 fois dans un
document destiné à un dépôt public (D-005). Une règle qui énumère laisse toujours passer
ce qu'elle n'a pas nommé. » Ce motif reste **vrai comme constat** mais il était un motif
de *confidentialité*, et il appliquait un côté d'une tension que **PO-012 déclare non
arbitrée**. Il est donc **signalé comme provisoire, pas supprimé** : si PO-012 se tranche
en faveur de la position inverse, c'est cette justification qui tombe — la règle, elle,
tient sur la portabilité. Ce livrable n'a pas arbitré PO-012.

*Ce que l'extension n'a pas résolu :* substituer un identifiant dans un document remplace
aussi le jeton là où il était un **motif de recherche** et non un chemin, ce qui produit
des commandes qui n'ont jamais été exécutées. Relevé et corrigé en P-18 ; c'est une des
raisons d'être de R-15.

*Attention à la règle elle-même :* le nombre « 13 » ci-dessus est une mesure datée du
2026-09-18. Ne pas le reprendre comme s'il était à jour (R-01).

*Motif :* publier la topologie d'une PKI d'entreprise et le nom de ses points d'entrée
est un renseignement offert. Le risque n'est pas hypothétique : le dépôt répond déjà
`HTTP 200` sans aucune authentification.

**Portée de cette règle — ce qu'elle ne peut pas faire.** Elle n'agit que sur ce qui
n'est **pas encore poussé**. Ce qui est publié l'est définitivement, et ne se répare pas
par une réécriture d'historique : celle-ci casse tous les SHA et ne touche ni les forks,
ni les caches, ni les vues web, ni les clones tiers. Constater, ne pas réécrire —
la décision appartient à l'opérateur seul (PO-012, PO-013).

## R-08 — Les machines cibles ne sont pas accessibles ; les mesures s'y délèguent

Aucune commande ne s'exécute ailleurs que sur ce poste. Ce qui doit être mesuré sur une
cible devient une **procédure** que l'opérateur joue lui-même : courte, délimitée par des
marqueurs de début et de fin, affichant son code de retour, et tenant dans une capture
d'écran. Le résultat revient étiqueté `[ÉCRAN-AAAA-MM-JJ]` (cf. D-003).

*Motif :* une procédure longue est tronquée par la capture, et une procédure sans
marqueurs ne permet pas de savoir si on voit tout. L'étiquette datée rappelle que ces
mesures sont non rejouables, périmables et exposées à une erreur de transcription.

## R-09 — `rhel_post_install` est en lecture seule, sans exception

Si ce dépôt devient accessible : aucune écriture, aucune tâche, aucun commit.
Et **ne pas le cloner d'initiative** — rapporter son absence et demander.

**Depuis le 2026-09-22, ce dépôt est cloné en lecture seule** sur le poste de rédaction,
à `/home/<user>/dev/rhel_post_install` (F-063, D-004 amendée). **La lecture seule
s'applique inchangée** : il est devenu lisible, il n'est pas devenu modifiable.

*Motif :* ce dépôt provisionne le parc. Une écriture y met un mécanisme non prouvé sur le
chemin de tous les hôtes. C'est aussi ce qui tient la décision D-001 ouverte.

## R-10 — Rien ne régénère un certificat local sur un hôte déjà enrôlé

`[RÉVISÉE le 2026-09-21]` Aucune tâche, aucun script, aucune procédure ne doit appeler
`scxsslconfig` — ni tout autre outil qui (re)fabrique un certificat local — sur un hôte
dont le certificat vient de CEP/CES. Cela vaut quel que soit le nom du fichier, du rôle
ou du tag qui le porte.

*Motif :* `scxsslconfig -f` force la régénération même s'il existe déjà un certificat, et
**écrase** alors la clé et le certificat obtenus de l'ADCS. Conflit lu dans le code et
dans la documentation de l'outil, pas précaution théorique (F-038,
`docs/scomm-depot-actuel.md` ; PO-003, `docs/scomm-certificat-facts.md`).

**ÉTAIT :** « Ne pas exécuter le tag `register` sur une machine enrôlée par CEP/CES —
`roles/scomm_agent/tasks/register.yml:13-24` lance `scxsslconfig -f` … ».
*Pourquoi la règle a changé de cible :* elle désignait **un fichier** d'un brouillon
qui va être réécrit (F-044, D-007). Un garde attaché à un nom de fichier disparaît avec
le fichier, alors que le geste qu'il interdit peut renaître ailleurs sous un autre nom.
PO-003 est requalifié en « défaut de brouillon à ne pas reconduire » ; la règle, elle,
reste permanente parce qu'elle vise désormais le geste.

## R-11 — L'historique se marque, il ne s'efface pas

Une affirmation révisée conserve sa version antérieure, précédée de `ÉTAIT`, et la
révision porte `[RÉVISÉE le AAAA-MM-JJ]`. Les décisions sont numérotées et datées.

*Motif :* sans trace, on ne sait plus si un fait a changé parce qu'une mesure l'a
contredit ou parce que quelqu'un l'a réécrit de mémoire — et c'est précisément cette
distinction qui donne sa valeur au dossier.

---

*Les règles R-12 à R-16 sont ajoutées le **2026-09-21**. R-01 à R-11 gouvernaient un
travail de lecture ; à partir du livrable 4 on produit des procédures, puis du code, donc
des **gardes**. Une garde est une affirmation sur ce qui n'arrivera pas : elle relève de
R-01 comme une autre, et se prouve de la même façon — par un artefact, pas par le fait
qu'elle soit écrite.*

## R-12 — Une garde se démontre dans les deux sens

`[RÉVISÉE le 2026-09-24 — texte antérieur et motif en D-011]` Une **garde** est ce qui
**s'arrête bruyamment** : `assert`, `fail`, `failed_when:`, ou un relevé préalable suivi
d'un `assert`. Un `when:` n'en est pas une : il **oriente** et peut sauter en silence —
il relève de R-04. Toute garde est démontrée par **deux exécutions relevées**, et non par
une :

1. le **cas nominal**, où elle laisse passer et la suite s'exécute ;
2. l'**échec forcé**, où la condition est délibérément mise en défaut, et où l'on relève
   le **code de retour** **et** le **message effectivement affiché**.

Le compte rendu porte les deux sorties. Une garde dont seul le cas nominal est montré est
consignée **écrite, non démontrée** — jamais « en place ». **Une garde modifiée perd sa
démonstration** : changer la condition, le message, une variable qu'elle lit, ou déplacer
la tâche invalide les deux exécutions ; il faut les rejouer, une démonstration ne
s'hérite pas.

*Motif :* une garde qui ne casse jamais ne prouve rien — elle prouve seulement qu'on ne
l'a pas essayée. L'aiguillage a sa propre leçon, tirée du même dépôt : les `when:` de
`register.yml:21-23` et `:33` sautent les deux tâches aux valeurs par défaut, **sans rien
dire** (F-018, `docs/scomm-depot-actuel.md`). **Un aiguillage dont le saut pourrait passer
pour un succès doit être doublé d'une garde qui vérifie l'état attendu.**

## R-13 — L'échec forcé doit discriminer, et survenir avant le dommage

Un échec ne prouve la garde que s'il est **imputable à elle**. Deux exigences, chacune
suffisante pour invalider une démonstration :

- **Discrimination.** Un échec survenu pour une autre raison — collection absente, hôte
  injoignable, faute de frappe dans le nom du module, droit manquant — ne démontre rien,
  même si le code de retour est non nul. Le message relevé doit être **celui de la
  garde**, cité. Si l'on ne peut pas distinguer les deux causes, l'échec forcé est **mal
  construit** : il faut le refaire, pas l'interpréter.
- **Antériorité.** La garde doit se déclencher **avant** l'action qu'elle protège, pas
  après. Un contrôle placé derrière l'écriture constate le dommage au lieu de l'empêcher,
  et passe pourtant tous les tests de la même façon. Le démontrer suppose de vérifier
  qu'à l'échec forcé, **l'effet protégé n'a pas eu lieu** — fichier non modifié, service
  non redémarré, certificat non écrasé.

*Motif :* ce projet a déjà produit un `rc` non nul qui ne disait pas ce qu'on croyait
(`ansible-playbook --syntax-check` → `rc=4` **parce qu'une collection manque**, F-041) et
un `rc=1` qui n'était pas un échec du tout (`ssh -T git@github.com`, P-25). Un code de
retour ne nomme pas sa cause. Et l'antériorité n'est pas un détail : sur cette chaîne,
l'action protégée est l'écrasement d'une clé privée (R-10), qui ne se rattrape pas.

## R-14 — Une garde qui compare deux sources s'exerce avec des valeurs divergentes

Quand une garde compare deux valeurs issues de **sources indépendantes** — une valeur
attendue et une valeur constatée, une valeur de vault et une valeur lue sur la machine —
elle doit être exercée **au moins une fois avec des valeurs qui diffèrent**, et pas
seulement dans le cas où elles coïncident.

*Motif, et il est vécu :* une comparaison de chemins protégeant un fichier critique
n'avait été exercée que dans le cas **convergent** — les deux chemins étaient égaux, la
garde passait, tout paraissait correct. Le cas **divergent** aurait montré que la
comparaison désarmait la protection au lieu de l'armer. Le défaut a été trouvé par
**relecture humaine, après que tous les contrôles automatisés soient passés** : aucune
exécution ne l'avait signalé, parce qu'aucune exécution ne l'avait mis en défaut.

**Ce projet comparera au moins une paire de ce type**, et elle est identifiée d'avance :
**l'émetteur attendu du certificat contre son émetteur réel** — la valeur déclarée
(gabarit, CA d'entreprise) contre la sortie de `openssl x509 -noout -issuer`. Une garde
qui n'accepte que le cas où les deux coïncident laisserait passer, sans rien dire, un
certificat auto-signé ou contresigné par le serveur d'administration (F-035, F-037).
L'exercer en divergence signifie : présenter volontairement un certificat d'un autre
émetteur, et constater que la garde refuse **et dit pourquoi**.

## R-15 — Les commandes se relèvent au moment de l'exécution

La commande, sa sortie et son code de retour sont consignés **quand ils se produisent**,
pas reconstitués en fin de session. Ce qui n'a pas été relevé sur le moment n'entre pas au
journal : il est rejoué, ou il est absent.

**Tout basculement d'outil après échec se déclare immédiatement** : outil abandonné,
raison, code de retour de l'échec, outil retenu. Un résultat obtenu au deuxième outil ne
se présente jamais comme s'il l'avait été au premier.

*Motif :* une commande recopiée de mémoire est indiscernable d'une commande exécutée —
c'est exactement ce qui interdit R-01. Ce dépôt en porte la preuve : la substitution du
livrable 2 a laissé en P-18 une commande **que personne n'a jamais exécutée sous cette
forme**, et personne ne l'a vu pendant un livrable entier parce qu'elle était
vraisemblable. Le contre-exemple positif existe aussi : P-12b déclare un `rc=143` et
l'abandon de `dnf repoquery -l`, ce qui permet aujourd'hui de savoir que les faits cepces
reposent sur S-1/S-2 et non sur le paquet.

## R-16 — Une décision amendée impose de rechercher ses renvois dans tout le dépôt

Amender, réviser ou clore une décision (`D-xxx`), un point ouvert (`PO-xxx`) ou une règle
(`R-xx`) n'est terminé qu'après avoir **cherché ses renvois dans l'ensemble du dépôt** —
les trois documents, `README.md`, et les fichiers de code — et statué sur chacun :
corrigé, ou laissé tel quel **avec le motif**. La recherche et son résultat figurent au
compte rendu ; un renvoi laissé cassé se déclare comme tel.

*Motif :* une décision ne vit qu'à un seul endroit (R-01, et l'en-tête de ce fichier),
mais elle est **citée** à plusieurs. Amender la source sans reprendre les citations
produit un dépôt qui se contredit lui-même, et la contradiction est invisible depuis
l'endroit qu'on vient de corriger — c'est toujours l'autre fichier qui ment. La
séparation du 2026-09-21 (D-006) a déplacé des dizaines de faits : sans cette recherche,
chaque renvoi « § 1.2 » ou « § 5 » serait devenu muet sans que rien ne le signale.

---

## Ce que ces règles n'incluent PAS, et pourquoi

Une règle inerte finit par être ignorée avec les autres. Les points suivants
appartenaient au cadre du **livrable 1** et **ne sont pas** des règles permanentes de ce
projet. Ils sont écartés explicitement pour qu'une session ultérieure ne se croie pas
bloquée par eux.

- **« Ne produire aucun code, lecture seule intégrale. »** — Écartée. C'était l'objet du
  seul livrable 1. La suite du projet consiste précisément à écrire des tâches Ansible.
  *Ce qui subsiste :* R-01 (rien sans source) et R-09 (`rhel_post_install` reste en
  lecture seule).

- **« N'écrire que dans `CLAUDE.md`, `docs/scomm-certificat-facts.md`, `README.md`,
  `.gitignore`. »** — Écartée. Périmètre du livrable 1 uniquement. *Ce qui subsiste :*
  R-05, l'édition ciblée plutôt que la réécriture.

- **« Pas de `git checkout`, pas de `git stash`. »** — Écartée comme interdiction
  permanente : travailler sur une branche deviendra nécessaire, et l'interdire
  pousserait à travailler directement sur `main`, ce qui est pire. *Ce qui subsiste :*
  R-05, l'interdiction de commiter ou pousser d'initiative, et le relevé de l'état du
  dépôt avant toute modification — le retour arrière existe avant la première écriture,
  pas après.

- **« Arrêter le travail si l'arbre n'est pas propre. »** — Conservée comme **contrôle
  d'entrée**, pas comme interdit. Relever `git status --porcelain -uall` (l'option
  `-uall` est obligatoire : sans elle, un répertoire non suivi est replié en une seule
  ligne) et rapporter ce qui est sale, avant d'écrire. Un arbre sale appartient à
  l'opérateur ; il se signale, il ne se contourne pas et ne se nettoie pas.

- **« Consigner la question de l'acceptation d'un certificat externe par l'agent SCOM
  comme point ouvert non instruit. »** — Écartée sous cette forme : la question **a** été
  instruite sur documentation Microsoft nommée (F-035 à F-037). *Ce qui subsiste :*
  PO-001, réduit à ce qui reste non mesuré — le comportement de **cette** installation,
  qu'aucune documentation ne peut établir.
