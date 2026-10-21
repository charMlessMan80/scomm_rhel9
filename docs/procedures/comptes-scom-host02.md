# Procédure — comptes SCOM de test sur `host02`

**Ce que ce fichier porte :** des blocs de commandes que **l'opérateur** joue lui-même
sur `host02`, sous **son compte AD avec `sudo`**, dans l'ordre, et la lecture de chaque
sortie. Objet unique : créer à la main les deux comptes locaux demandés par l'équipe
SCOM, `scom_adm` et `scom_mon`, et le fichier sudoers qui les accompagne (D-017).
`[RÉVISÉE le 2026-10-21, D-019]` Les **deux** comptes s'authentifient **par mot de
passe**, pris dans BeyondTrust (F-191) ; aucune clé SSH n'est posée. **ÉTAIT :**
`scom_adm` par clé SSH (partie B), `scom_mon` par mot de passe.

**Ce que ce fichier NE porte PAS :** aucun fait, aucune décision, aucun point ouvert —
ils vivent dans `../scomm-certificat-facts.md`. Il n'a **aucune autorité propre** : tout
ce qu'il affirme y renvoie (`F-xxx`, `PO-xxx`, `D-xxx`).

**Ce que cette procédure ne fait pas, et c'est une propriété :** elle ne touche **ni**
au certificat, **ni** à `omiserver.conf`, **ni** à `sshd`, **ni** au pare-feu, **ni** au
rôle. Elle ne lance aucun outil qui fabrique un certificat (R-10).

> **Aucune commande de ce fichier n'a été exécutée par l'agent qui l'a écrite** (R-08).
> Ce qui l'a été, sur le poste de rédaction : `visudo -cf` sur le fichier sudoers, ses
> témoins, `readlink`, et les lectures de pages de manuel (F-185 à F-189, P-69).
> **Réserve de transposition :** ces pages sont celles de Fedora 44 (`sudo` 1.9.17,
> `shadow-utils` 4.19.0) ; les versions de `host02` ne sont pas au dossier.

## Ordre de jeu

| # | Étape | Écrit sur `host02` ? | Un échec arrête-t-il la suite ? |
|---|---|---|---|
| 0 | Lecture seule : état initial, empreinte « avant » | non (sauf `~/scom-comptes-avant.fp`) | **Oui** |
| T | Transfert du fichier sudoers, contrôle d'empreinte | `~/scom-test.sudoers` | **Oui** |
| R | **Retour arrière — à lire avant A, à jouer seulement si besoin** | — | — |
| A1 | Création des deux comptes | `/etc/passwd`, `/etc/shadow`, `/home` | **Oui** |
| A2 | Installation du sudoers | `/etc/sudoers.d/scom-test` | **Oui** → R |
| A3 | Vérification des droits, sans rien exécuter | non | **Oui** → R |
| V | Connexion SSH de `scom_adm` par mot de passe — **interactive, facultative** | `~/.ssh/known_hosts` de l'opérateur, si la clé d'hôte est acceptée | Non — rapporter |
| B | **Sans objet (D-019 a)** | — | — |
| F | Contrôles finaux | `~/scom-comptes-apres.fp` | **Oui**, et alerte |
| N | Nettoyage — **après validation par l'équipe SCOM** | suppressions dans `~` | — |

`[RÉVISÉE le 2026-10-21, D-019]` Ligne V ajoutée ; ligne B passée à « sans objet ».
**ÉTAIT :** B, « Clé de `scom_adm` — à réception de la clé seulement », écrivait
`/home/scom_adm/.ssh` et arrêtait la suite en cas d'échec.

## Forme des commandes

**Une commande par ligne**, suivie sur la même ligne de l'affichage de son code de
retour. Aucune continuation, aucun *heredoc*, aucune valeur à remplacer : chaque ligne
se colle seule. *Motif :* les collages multi-lignes sont abîmés par le terminal
(`diagnostic-enrolement.md`, « Forme des commandes »). Les commandes
**interactives** (les deux `passwd` de A1, la seconde session de A2, la connexion `ssh`
de V) sont isolées et le disent.
`[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** « Les deux commandes interactives
(`passwd`, et la seconde session de A2) ».
Chaque bloc est encadré de marqueurs `DEBUT` / `FIN` : une capture sans ses deux
marqueurs est incomplète (R-08).

**Élévations.** Toutes les lignes `sudo` sont des élévations ; elles sont **toutes**
nécessaires (création de comptes, fichiers sous `/etc`, lecture d'un certificat
`root:omi` 640, F-152). Là où une voie sans privilège est plausible, elle est tentée
d'abord (étape 0).

---

## Étape 0 — Lecture seule

**But :** établir que rien n'existe déjà, que `sudo` est sain, que `sshd` délègue à PAM,
et relever l'ancien chemin du certificat (PO-058) et l'empreinte « avant ».

```
echo "=== DEBUT E0 ==="
getent passwd scom_adm; echo "rc_getent_adm=$?  (2 attendu)"
getent passwd scom_mon; echo "rc_getent_mon=$?  (2 attendu)"
ls -la /etc/sudoers.d/; echo "rc_ls_sudoersd=$?"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
sudo sshd -T | grep -i '^usepam'; echo "rc_sshd=${PIPESTATUS[0]} rc_grep=${PIPESTATUS[1]}  (0 0 attendus)"
ls -l /etc/opt/microsoft/scx/ssl/; echo "rc_ls_scx=$?"
readlink -e /etc/opt/microsoft/scx/ssl/scx.pem; echo "rc_readlink=$?"
sudo openssl x509 -in /etc/opt/omi/ssl/omi.pem -noout -fingerprint -sha256 | tee ~/scom-comptes-avant.fp; echo "rc_openssl=${PIPESTATUS[0]} rc_tee=${PIPESTATUS[1]}  (0 0 attendus)"
echo "=== FIN E0 ==="
```

Si `rc_ls_sudoersd` n'est pas `0` (répertoire non lisible sans privilège), rejouer
seulement :

```
sudo ls -la /etc/sudoers.d/; echo "rc_sudo_ls_sudoersd=$?"
```

Si `rc_ls_scx` n'est pas `0` pour un refus de permission, rejouer seulement :

```
sudo ls -l /etc/opt/microsoft/scx/ssl/; echo "rc_sudo_ls_scx=$?"
```

**Résultat attendu et conditions d'arrêt :**

| Ligne | Attendu | Sinon |
|---|---|---|
| `getent passwd` ×2 | rien d'affiché, `rc=2` (« key could not be found », `man getent`) — couvre aussi les comptes AD vus par SSSD | **Arrêt** : un compte de ce nom existe déjà. `rc=1` n'est pas une absence (« database unknown ») : arrêt aussi. |
| `/etc/sudoers.d/` | ni `scom-test`, ni `scom` (fichier du rôle, F-015) | **Arrêt.** |
| `visudo -c` | `rc=0` | **Arrêt** : on n'ajoute rien à une configuration `sudo` déjà cassée. |
| `sshd -T` | **relevé, pas critère** `[RÉVISÉE le 2026-10-21, D-019]` : dire `usepam yes` ou `usepam no` | — ; le résultat décide de ce que V éprouve : avec `yes`, la pile PAM de `sshd` ; avec `no`, la vérification propre de `sshd`, sans PAM (F-194). Si `rc_sshd` n'est pas `0`, le dire : V ne pourra pas être interprétée. **ÉTAIT :** critère d'arrêt, parce qu'avec `UsePAM no` `sshd` refuse un compte au mot de passe verrouillé (F-184) et que la partie B, connexion par clé sur un compte verrouillé, n'aurait pas pu réussir. |
| `ls -l` et `readlink -e` | **relevé, pas critère** : dire si `scx.pem` existe, si c'est un lien, et vers quoi `readlink -e` le résout (`rc=1` = absent ou lien cassé) | — ; le résultat va à PO-058. **S'il mène à `/etc/opt/omi/ssl/omi.pem`**, la ligne de signature commentée en A2 aurait remplacé le certificat ADCS : le dire en tête du rapport. |
| empreinte | une ligne `sha256 Fingerprint=…`, `0 0` | **Arrêt** si `rc_openssl` n'est pas `0` : le fichier `~/scom-comptes-avant.fp` ne prouverait rien. |

**Ne pas supprimer `~/scom-comptes-avant.fp`** avant la fin du test SCOM : c'est aussi
l'empreinte « avant découverte » (F-169).

---

## Étape T — Transfert du fichier sudoers

**But :** amener `docs/procedures/fichiers/scom-test.sudoers` (ce dépôt) dans le dossier
personnel de l'opérateur sur `host02`, sous le nom `scom-test.sudoers`, **intact**.

**Le chemin du fichier jusqu'à `host02` n'est pas prescrit** : les cibles ne sont pas
joignables depuis le poste de rédaction (R-08). Il passe par SFTP, comme le PFX de F-144,
depuis la machine où l'opérateur dispose du dépôt. **Le contrôle d'empreinte rend le
chemin indifférent.**

**Empreinte attendue**, celle du fichier tel qu'il est versionné avec cette procédure :

`895682f51137e5c87d16b1b2e3a5f0eb51dc587d033d4d942f8fd6a4f8fd5a07`

Côté source, à la racine du dépôt :

```
sha256sum docs/procedures/fichiers/scom-test.sudoers; echo "rc_sha_source=$?"
```

Si cette empreinte **diffère** de celle ci-dessus, le fichier a changé depuis l'écriture
de cette procédure : **arrêt**, les deux doivent être remis d'accord avant de jouer.

Sur `host02` :

```
echo "=== DEBUT ET ==="
cd ~; echo "rc_cd=$?"
echo '895682f51137e5c87d16b1b2e3a5f0eb51dc587d033d4d942f8fd6a4f8fd5a07  scom-test.sudoers' | sha256sum -c -; echo "rc_sha=${PIPESTATUS[1]}  (0 attendu)"
echo '0000000000000000000000000000000000000000000000000000000000000000  scom-test.sudoers' | sha256sum -c -; echo "rc_sha_temoin=${PIPESTATUS[1]}  (1 attendu)"
echo "=== FIN ET ==="
```

**Attendu :** `scom-test.sudoers: OK` et `rc_sha=0` ; puis, pour le témoin,
`FAILED` et `rc_sha_temoin=1` — formes rejouées sur le poste de rédaction (P-69). Le
témoin montre que la comparaison **peut** échouer (R-03, R-12). **Arrêt** si `rc_sha`
n'est pas `0`, ou si le témoin ne rend pas `1`.

---

## Étape R — Retour arrière, écrit avant la première écriture

**À lire maintenant, à ne jouer qu'en cas de besoin.** Il défait A1 et A2.
`[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** « Il défait A1, A2 et B. »

```
echo "=== DEBUT ER ==="
sudo rm /etc/sudoers.d/scom-test; echo "rc_rm=$?"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
sudo userdel -r scom_adm; echo "rc_userdel_adm=$?"
sudo userdel -r scom_mon; echo "rc_userdel_mon=$?"
getent passwd scom_adm; echo "rc_getent_adm=$?  (2 attendu)"
getent passwd scom_mon; echo "rc_getent_mon=$?  (2 attendu)"
echo "=== FIN ER ==="
```

`rm` est **sans `-f`** : s'il n'y a rien à retirer, il le dit. `userdel -r` retire aussi
le dossier personnel. `[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** « …, donc la clé de
B. » Ce bloc ne touche ni aux entrées BeyondTrust ni à l'entrée `localhost` que V a pu
ajouter à `~/.ssh/known_hosts`. **Si `sudo` ne fonctionne plus** (fichier
sudoers cassé malgré A2), seule la seconde session ouverte en A2 permet de jouer ce
bloc : c'est sa raison d'être.

---

## Étape A1 — Création des deux comptes

**But :** créer `scom_adm` et `scom_mon` (D-017 c), fixer la non-expiration des
**deux** comptes (PO-063), donner à chacun le mot de passe pris dans BeyondTrust, saisi
en interactif (D-017 e, D-019 b).
`[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** non-expiration et mot de passe pour
`scom_mon` seul.

```
echo "=== DEBUT EA1 ==="
sudo useradd -m -c 'SCOM maintenance (test)' scom_adm; echo "rc_useradd_adm=$?"
sudo useradd -m -c 'SCOM monitoring (test)' scom_mon; echo "rc_useradd_mon=$?"
getent passwd scom_adm; echo "rc_getent_adm=$?  (0 attendu)"
getent passwd scom_mon; echo "rc_getent_mon=$?  (0 attendu)"
id scom_adm; echo "rc_id_adm=$?"
id scom_mon; echo "rc_id_mon=$?"
sudo getent shadow scom_adm | cut -d: -f1,2; echo "rc_shadow_adm=${PIPESTATUS[0]}"
sudo chage -m 0 -M -1 -I -1 -E -1 scom_adm; echo "rc_chage_adm=$?"
sudo chage -l scom_adm; echo "rc_chage_l_adm=$?"
sudo chage -m 0 -M -1 -I -1 -E -1 scom_mon; echo "rc_chage_mon=$?"
sudo chage -l scom_mon; echo "rc_chage_l_mon=$?"
echo "=== FIN EA1 ==="
```

`[RÉVISÉE le 2026-10-21, D-019]` Les deux lignes `chage` de `scom_adm` sont ajoutées ;
celles de `scom_mon` sont renommées `rc_chage_mon` et `rc_chage_l_mon`. **ÉTAIT :**
`chage` pour `scom_mon` seul, codes `rc_chage` et `rc_chage_l`.

Puis les deux `passwd`, **chacun seul, interactif, isolé** — le mot de passe n'apparaît
jamais sur une ligne de commande (D-017 e). **Avant chacun**, ouvrir l'entrée BeyondTrust
du compte (F-191) : le mot de passe saisi est celui-là, et aucun autre.

```
sudo passwd scom_adm; echo "rc_passwd_adm=$?"
```

```
sudo passwd scom_mon; echo "rc_passwd_mon=$?"
```

`[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** un seul `passwd`, celui de `scom_mon`, code
`rc_passwd`.

**Attendu et conditions d'arrêt :**
- `useradd` ×2 : `rc=0`. Sinon **arrêt**.
- `getent passwd` : `rc=0` et un sixième champ **`/home/scom_adm`** et **`/home/scom_mon`**.
  Ce passage de `rc=2` (étape 0) à `rc=0` est le **sens contraire** du contrôle de
  l'étape 0, exercé ici sans rien forcer. Si le dossier personnel est ailleurs :
  **relever et le dire**, pas d'arrêt. `[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :**
  « **Arrêt** si le dossier personnel est ailleurs : la partie B l'écrit en dur » — la
  partie B est sans objet, plus rien ne l'écrit en dur.
- `id scom_adm` : un groupe primaire **`scom_adm`** — **relever**, pas d'arrêt.
  `[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** « **Arrêt** sinon : B l'emploie. »
- `getent shadow scom_adm`, **avant tout `passwd`** : `scom_adm:` suivi d'un champ qui
  **commence par `!`** — un seul `!` selon `man useradd` du poste, `!!` selon le pilote ;
  **relever la forme exacte** (F-184). Il constate l'état laissé par `useradd`. **Arrêt**
  si le champ ne commence pas par `!` : un mot de passe existerait déjà, qui ne vient pas
  de BeyondTrust.
- **Interdit : afficher le champ `shadow` de l'un ou l'autre compte après un `passwd`**
  — il porterait l'empreinte du mot de passe. Aucune ligne de cette procédure ne le fait
  après ce point ; n'en ajouter aucune.
- `chage` ×2 : `rc=0`. `chage -l` ×2 : expiration du mot de passe, inactivité et
  expiration du compte **sans date** (`-M -1`, `-I -1`, `-E -1`, F-186). La forme exacte
  des libellés n'est pas sourcée (F-186) : relever les deux sorties entières. **Arrêt**
  si une date apparaît sur l'une des trois lignes, pour l'un ou l'autre compte.
- **Si l'entrée BeyondTrust d'un compte n'est pas accessible : arrêt avant son
  `passwd`.** Ne pas improviser de mot de passe. Le compte reste verrouillé par
  `useradd`, ce qui est **sans danger** : le champ verrouillé ne correspond à aucun mot
  de passe, et aucune clé n'est posée (F-195). Reprendre **à ce `passwd`**, sans
  rejouer le bloc EA1 : ses `useradd` porteraient sur des comptes déjà créés.
  `[RÉVISÉE le 2026-10-21, D-019]`
  **ÉTAIT :** « s'il n'est pas encore remis, différer cette seule ligne ».
- `passwd` ×2 : `rc=0`. **Arrêt** sinon ; relever le message affiché, sans le mot de
  passe.

`[RÉVISÉE le 2026-10-21, D-019]` **Les deux comptes ont désormais un mot de passe.** Le
cas de F-184 — `sshd` et un compte au mot de passe verrouillé — ne se présente plus après
les deux `passwd`. La connexion SSH de `scom_adm` par mot de passe est éprouvée en V,
facultative ; celle de `scom_mon` par WS-Man passe par `/etc/pam.d/omi` (F-180) et sera
constatée par l'équipe SCOM (F-169). **ÉTAIT :** « `scom_adm` garde le mot de passe
verrouillé laissé par `useradd` » ; d'après S-19 (F-184), sa connexion par clé y était
dite attendue, pas établie.

---

## Étape A2 — Installation du fichier sudoers

**But :** poser `/etc/sudoers.d/scom-test` (D-017 d), **après** avoir prouvé sur
`host02` que le contrôle de syntaxe sait échouer et que le fichier le passe.

**Avant ce bloc, ouvrir une seconde session SSH sur `host02` et y lancer `sudo -i`.**
Interactive, isolée, et laissée ouverte jusqu'à la fin de A3. *Motif :* si le fichier
posé rendait `sudo` inutilisable, cette session est le seul chemin vers l'étape R.

```
echo "=== DEBUT EA2 ==="
printf 'scom_adm ALL=(root NOPASSWD: /bin/true\n' > ~/scom-temoin.sudoers; echo "rc_printf=$?"
sudo visudo -cf ~/scom-temoin.sudoers; echo "rc_visudo_temoin=$?  (1 attendu)"
rm ~/scom-temoin.sudoers; echo "rc_rm_temoin=$?"
sudo visudo -cf ~/scom-test.sudoers; echo "rc_visudo_fichier=$?  (0 attendu)"
sudo install -m 440 -o root -g root ~/scom-test.sudoers /etc/sudoers.d/scom-test; echo "rc_install=$?"
sudo restorecon -v /etc/sudoers.d/scom-test; echo "rc_restorecon=$?"
sudo ls -lZ /etc/sudoers.d/scom-test; echo "rc_ls=$?"
sudo visudo -c; echo "rc_visudo_global=$?  (0 attendu)"
echo "=== FIN EA2 ==="
```

**Attendu et conditions d'arrêt :**
- Témoin : un message **« syntax error »** et `rc_visudo_temoin=1` — la ligne est
  celle qui échoue sur le poste (F-189). **Arrêt** si elle passe : le contrôle suivant ne
  prouverait rien. *Ne pas « simplifier » le témoin :* une ligne `… NOPASSWD` sans « : »
  **passe** `visudo` (F-189).
- `visudo -cf` du fichier : `parsed OK`, `rc=0`. **Arrêt sinon, avant l'`install`** :
  c'est la garde qui empêche de poser un fichier cassé (R-13, antériorité).
- `install` : `rc=0`. Le nom de destination **ne contient pas de point** : `sudo` ignore
  les fichiers de `/etc/sudoers.d` dont le nom contient « . » (F-185). Un
  `scom-test.sudoers` posé tel quel serait ignoré **en silence**.
- `ls -lZ` : `-r--r-----`, `root root`, un contexte SELinux de la famille `etc_t` ou
  propre à `sudo` — **relever**, pas de valeur sourcée ici.
- `visudo -c` global : `rc=0`. **Sinon : étape R immédiatement**, depuis la seconde
  session si `sudo` refuse.

---

## Étape A3 — Vérifier les droits sans rien exécuter

**But :** établir que **seules** les trois lignes actives de D-017 (d) sont accordées, et
**démontrer la garde dans le sens de l'échec** : la ligne de signature du certificat est
**refusée** par `sudo` (R-10, R-12).

**Pourquoi rien n'est exécuté** (F-185) : avec `-l` suivi d'une commande, `sudo`
**affiche** le chemin complet et les arguments si la commande est permise, et sort avec
`1` sinon. La page ne dit pas en toutes lettres « n'exécute pas » ; elle ne décrit qu'un
affichage. `-n` interdit toute demande de mot de passe.

```
echo "=== DEBUT EA3 ==="
sudo -l -U scom_adm; echo "rc_list_adm=$?"
sudo -l -U scom_mon; echo "rc_list_mon=$?"
sudo -u scom_adm sudo -n -l /bin/sh -c 'cat /etc/opt/microsoft/scx/ssl/scx.pem'; echo "rc_adm_cat=$?  (0 attendu : PERMIS)"
sudo -u scom_adm sudo -n -l /bin/sh -c 'cp /tmp/scx-scom_adm/scx.pem /etc/opt/microsoft/scx/ssl/scx.pem; rm -rf /tmp/scx-scom_adm; /opt/microsoft/scx/bin/tools/scxadmin -restart'; echo "rc_adm_cp=$?  (1 attendu : REFUSE)"
sudo -u scom_adm sudo -n -l /bin/sh -c 'sh /tmp/scx-scom_adm/scx-1.9.3-0.universald.1.s.x64.sh --install --enable-opsmgr; EC=$?; cd /tmp; rm -rf /tmp/scx-scom_adm; exit $EC'; echo "rc_adm_install=$?  (1 attendu : REFUSE)"
sudo -u scom_mon sudo -n -l /opt/microsoft/scx/bin/scxlogfilereader -p; echo "rc_mon_log=$?  (0 attendu : PERMIS)"
sudo -u scom_mon sudo -n -l /bin/sh -c 'cat /etc/opt/microsoft/scx/ssl/scx.pem'; echo "rc_mon_cat=$?  (1 attendu : REFUSE)"
echo "=== FIN EA3 ==="
```

Les commandes testées sont **recopiées du fichier sudoers**, guillemets simples
compris : `sudo` reçoit `-c` puis une seule chaîne, et la compare aux arguments de la
règle. Le nom du *bundle* testé est celui du kit (F-165), qui a la forme de la première
règle d'installation de S-16.

**Attendu :**
- `sudo -l -U scom_adm` : parmi les commandes permises, **une seule** règle, celle de
  `/bin/sh -c cat /etc/opt/microsoft/scx/ssl/scx.pem`, et `!requiretty` parmi les
  réglages `Defaults`. `sudo -l -U scom_mon` : **une seule** règle, celle de
  `/opt/microsoft/scx/bin/scxlogfilereader -p`. La mise en forme exacte de cette liste
  n'est pas sourcée ici : **relever la sortie entière**. **Toute autre règle** — venue
  d'un autre fichier, d'un groupe, d'une règle `ALL` — **arrête** la procédure : les
  comptes auraient plus que D-017 (d).
- Permis : la commande réaffichée, chemin complet, `rc=0`.
- Refusé : `rc=1` et **aucun message**.

**Discrimination du refus (R-13)** — un `rc=1` ne nomme pas sa cause : `sudo` rend aussi
`1` pour un échec d'authentification ou un problème de configuration (F-185). Le refus
n'est imputable **à la règle absente** que si les trois conditions sont réunies :
1. la ligne **permise** du même compte, de même forme (`/bin/sh -c '…'`), a rendu `0`
   **juste avant** — ce n'est donc ni `-n`, ni le `sudo` extérieur, ni la forme de la
   commande ;
2. **aucun message** n'accompagne le `rc=1` — un message de mot de passe requis, ou
   « not allowed to execute … as scom_adm » venu du `sudo` extérieur, désigne une autre
   cause : **arrêt**, l'échec forcé est mal construit, on ne l'interprète pas ;
3. la ligne refusée **n'apparaît pas** dans la liste de `sudo -l -U`.

**Si le `sudo` extérieur refuse `-u scom_adm`** (règle de l'opérateur limitée à
`root`), employer à la place la forme `-U`, permise à qui peut tout exécuter (F-185) ;
par exemple, une ligne :

```
sudo -l -U scom_adm /bin/sh -c 'cp /tmp/scx-scom_adm/scx.pem /etc/opt/microsoft/scx/ssl/scx.pem; rm -rf /tmp/scx-scom_adm; /opt/microsoft/scx/bin/tools/scxadmin -restart'; echo "rc_U_adm_cp=$?  (1 attendu)"
```

et de même pour chacune des cinq lignes, en gardant les trois conditions. La forme `-U`
ne vérifie pas que `scom_adm` lui-même peut interroger `sudo` sans mot de passe ; la
forme `-u … sudo -n` le vérifie. Dire laquelle a été jouée.

**Ce que cette étape démontre, et ce qu'elle ne démontre pas.** Elle démontre que la
politique installée **refuse** la signature, dans le sens de l'échec, sans rien
exécuter. Elle ne démontre pas ce que fera la découverte : si elle demande la
signature, l'équipe SCOM verra une erreur, et `/var/log/secure` la portera (S-16,
« Sudo error log ») — c'est **l'effet recherché** (D-017 d), à relever pour PO-059.

---

## Étape V — Connexion SSH de `scom_adm` par mot de passe (interactive, facultative)

`[AJOUTÉE le 2026-10-21, D-019]`

**But :** éprouver, **pour ce compte**, `sshd` et sa pile d'authentification par mot de
passe, avant que l'équipe SCOM ne s'y connecte pour la maintenance (F-182). À jouer
**après les deux `passwd` de A1 et après A3**. Facultative : un échec ne défait rien et
n'appelle pas R ; il se rapporte.

**Seule, interactive** — `ssh` demande le mot de passe de `scom_adm`, pris dans
BeyondTrust ; il n'apparaît sur aucune ligne de commande :

```
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no scom_adm@localhost true; echo "rc_ssh_adm=$?  (0 attendu)"
```

**Options** (F-193, `man ssh_config` du poste) : `PreferredAuthentications=password` ne
fait essayer que la méthode `password` — ni `gssapi-with-mic`, ni `publickey`, ni
`keyboard-interactive`, qui sont dans la liste par défaut ; `PubkeyAuthentication=no`
écarte en plus toute clé de l'opérateur. Données en ligne de commande, elles passent
**avant** `~/.ssh/config` et `/etc/ssh/ssh_config`. `true` est la commande distante :
`ssh` rend son code de sortie.

**La première connexion peut demander d'accepter la clé d'hôte de `localhost`** :
`StrictHostKeyChecking` vaut `ask` par défaut (F-193), sauf si `/etc/ssh/ssh_config` de
`host02` le change. L'accepter ajoute une entrée `localhost` au `~/.ssh/known_hosts` **de
l'opérateur** — la seule écriture de cette étape.

**Attendu :** rien d'affiché après la saisie, `rc_ssh_adm=0`.
**Lecture d'un échec (R-13) :** `ssh` rend **`255`** pour toute erreur (F-193) ; le code
ne nomme pas sa cause. **Relever le message** : un refus d'authentification désigne le
compte, son mot de passe ou la pile PAM ; un refus de connexion, ou une clé d'hôte
refusée, désigne `sshd` ou le client, **pas** le compte — ce n'est alors pas un échec de
la vérification, qui est à refaire. Ne rien corriger dans `sshd` d'ici : cette procédure
n'y touche pas.

**Ce que V éprouve dépend de l'étape 0** (F-194) : avec `usepam yes`, la pile PAM de
`sshd`, `auth` et `account` — son contenu n'est pas relevé (F-179) ; avec `usepam no`,
la seule vérification du mot de passe par `sshd`.

**Pas de vérification équivalente pour `scom_mon`.** Son canal est WS-Man (F-182), par
`/etc/pam.d/omi` (F-180) ; la seule voie connue de l'éprouver d'ici passerait le mot de
passe en ligne de commande, ce qu'interdit D-017 (e). Le contrôle revient à l'équipe
SCOM (leurs étapes, F-169). **Ne pas éprouver `scom_mon` par SSH :** ce n'est pas son
canal, et une réussite ne dirait rien de WS-Man (PO-064).

---

## Étape B — Sans objet depuis D-019 (a)

`[RÉVISÉE le 2026-10-21, D-019]` **Sans objet depuis D-019 (a)** : `scom_adm`
s'authentifie par mot de passe, aucune clé n'est posée. **À reprendre si une clé SSH
revient (PO-066).** **ÉTAIT :** « Clé de `scom_adm` (à réception de la clé) » — pose de
la clé publique reçue de l'équipe SCOM dans `/home/scom_adm/.ssh/authorized_keys`, après
contrôle du nombre de clés et de l'empreinte (PO-060, clos pour le test) ; ses commandes
sont retirées.

---

## Étape F — Contrôles finaux

**But :** constater que rien de ce qui précède n'a touché le certificat ni l'écoute.

```
echo "=== DEBUT EF ==="
sudo openssl x509 -in /etc/opt/omi/ssl/omi.pem -noout -fingerprint -sha256 | tee ~/scom-comptes-apres.fp; echo "rc_openssl=${PIPESTATUS[0]} rc_tee=${PIPESTATUS[1]}"
diff ~/scom-comptes-avant.fp ~/scom-comptes-apres.fp; echo "rc_diff=$?  (0 attendu)"
diff ~/scom-comptes-avant.fp /dev/null; echo "rc_diff_temoin=$?  (1 attendu)"
sudo ss -tlnp | grep 1270; echo "rc_ss=${PIPESTATUS[0]} rc_grep=${PIPESTATUS[1]}  (0 0 attendus)"
id scom_adm; echo "rc_id_adm=$?"
id scom_mon; echo "rc_id_mon=$?"
echo "=== FIN EF ==="
```

**Attendu :** `rc_diff=0`, rien d'affiché ; le témoin affiche une différence et rend `1`
— il montre que `diff` **sait** échouer. Une ligne d'écoute sur `*:1270` portant
`omiengine` (F-176). **Si `rc_diff` n'est pas `0` : arrêt et alerte** — le certificat a
changé alors que rien ici ne le touche ; ne rien corriger, rapporter.

*Portée :* ce contrôle **constate** après coup ; il ne protège rien. La protection du
certificat pendant la découverte, c'est le refus de A3. Après la découverte, rejouer
les deux premières lignes de ce bloc (empreinte « après » contre
`~/scom-comptes-avant.fp`) donne la comparaison prévue par l'équipe SCOM (F-169).

---

## Étape N — Nettoyage, après validation par l'équipe SCOM

**Pas avant :** `~/scom-comptes-avant.fp` sert à la comparaison après découverte.
Rattaché à PO-056.

```
echo "=== DEBUT EN ==="
rm ~/scom-test.sudoers ~/scom-comptes-avant.fp ~/scom-comptes-apres.fp; echo "rc_rm=$?"
ls -la ~/scom-test.sudoers ~/scom-comptes-avant.fp ~/scom-comptes-apres.fp; echo "rc_ls=$?  (2 attendu)"
echo "=== FIN EN ==="
```

`rm` sans `-f` : un fichier déjà absent se signale.
`[RÉVISÉE le 2026-10-21, D-019]` `~/scom_adm.pub` sort des deux lignes. **ÉTAIT :** il y
figurait, et son absence quand B n'avait pas été jouée rendait `rc_rm=1`, attendu.

---

## Ce que l'opérateur transmet ensuite à l'équipe SCOM

- les noms des deux comptes : `scom_adm` (maintenance), `scom_mon` (supervision), **tous
  deux par mot de passe** ;
- que **les deux mots de passe sont dans BeyondTrust** (F-191) et **ne transitent par
  aucun autre canal** — ni courriel, ni message, ni ce dépôt ;
- l'écoute sur 1270 confirmée (étape F), et le résultat de V si elle a été jouée ;
- les empreintes du certificat — l'opérateur les a ; **elles ne sont écrites ni ici ni
  au dossier** (R-07) ;
- que la ligne de signature du certificat est **refusée par conception** (D-017 d) : si
  la découverte l'exige, l'erreur est attendue et doit être rapportée, pas contournée.

`[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** `scom_adm` « (maintenance, clé SSH) », et
les contraintes de clé de S-17 (F-183) à transmettre — RSA ou DSA, PPK version 2, clé
publique au format OpenSSH et son empreinte par un autre canal.

---

## Gardes, branches et ce qui reste non démontré (R-04, R-12)

| Contrôle | Placé avant l'effet ? | Sens de l'échec |
|---|---|---|
| `getent` absent (E0) | oui, avant A1 | exercé en sens contraire en A1 (`rc=2` → `rc=0`), sans forçage |
| `usepam` (E0) | **relevé, plus une garde** `[RÉVISÉE le 2026-10-21, D-019]` | sans objet : rien ne s'arrête dessus ; il sert à lire V (F-194). **ÉTAIT :** « oui, avant A et B » ; **écrit, non démontré** — le forcer demanderait de modifier `sshd` |
| `sha256sum -c` (T) | oui, avant A2 | témoin à empreinte nulle, joué en T |
| `visudo -cf` (A2) | oui, avant `install` | témoin à parenthèse non fermée, joué en A2 ; démontré sur le poste (F-189) |
| refus `sudo` de la signature (A3) | oui, avant la découverte | **démontré par construction** : permis et refusés côte à côte, trois conditions de R-13 |
| `wc -l` et empreinte de la clé (B) | **sans objet (D-019 a)** `[RÉVISÉE le 2026-10-21]` | **ÉTAIT :** oui, avant `install` ; **écrit, non démontré** — contrôle à l'œil |
| champ `shadow` de `scom_adm` commençant par `!` (A1) | oui, avant les `passwd` | **écrit, non démontré** : son sens de l'échec supposerait un mot de passe déjà posé `[AJOUTÉE au tableau le 2026-10-21]` — la condition d'arrêt existait en A1, la ligne n'y figurait pas |
| entrée BeyondTrust accessible (A1) | oui, avant chaque `passwd` | **écrit, non démontré** — contrôle humain `[AJOUTÉE le 2026-10-21, D-019]` |
| connexion SSH de `scom_adm` (V) | **non** — vérification, pas une garde ; rien ne s'arrête dessus | **écrit, non démontré** : aucun témoin d'échec n'est prescrit ; un `255` se lit par son message (F-193) `[AJOUTÉE le 2026-10-21, D-019]` |
| empreinte avant/après (F) | **non** — constat après coup | témoin `/dev/null`, joué en F |

**Branches :** la partie B est **sans objet** ; un `passwd` peut être **arrêté** faute
d'entrée BeyondTrust (A1), et reprend à cette ligne ; V est facultative ; la forme `-U`
de A3 ne sert que si `-u scom_adm` est refusé. Le compte rendu dit lesquelles ont été
jouées. `[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** « B n'est jouée qu'à réception de la
clé ; la ligne `passwd` peut être différée (A1) ; … ».

**Rapport attendu**, en `[ÉCRAN-AAAA-MM-JJ]` (D-003) : chaque bloc **entier**, marqueurs
compris ; le résultat de `readlink -e` et de `ls -l` de E0 (PO-058) ; la ligne `usepam`
de E0 ; la forme exacte du champ `shadow` de `scom_adm`, **avant** les `passwd` ; les
deux sorties de `chage -l` ; `rc_passwd_adm` et `rc_passwd_mon` ; les deux `sudo -l -U` ;
la forme de A3 jouée ; si V a été jouée, `rc_ssh_adm` et le message éventuel ; toute
élévation hors de ce fichier. `[RÉVISÉE le 2026-10-21, D-019]` **ÉTAIT :** une seule
sortie de `chage -l`, sans `usepam`, ni `passwd`, ni V.
