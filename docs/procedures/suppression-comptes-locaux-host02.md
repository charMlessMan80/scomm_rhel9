# Procédure — suppression des comptes locaux `scom_adm`, `scom_mon` et de `scom-test` sur `host02`

**Ce que ce fichier porte :** des blocs de commandes que **l'opérateur** joue lui-même
sur `host02`, sous **son compte AD avec `sudo`**, dans l'ordre, et la lecture de chaque
sortie. Objet unique (D-023, ouverte par D-026) : retirer de `host02`
1. le fichier `/etc/sudoers.d/scom-test` ;
2. les comptes locaux `scom_adm` et `scom_mon`, avec leur dossier personnel.

**Ce que ce fichier NE porte PAS :** aucun fait, aucune décision, aucun point ouvert —
ils vivent dans `../scomm-certificat-facts.md`. Il n'a **aucune autorité propre**. Ce qui
n'est établi ni par le dossier, ni par le § *Sources* porte `@VERIF`.

**Préalable répondu pour `scom_mon`** `[PILOTE-DÉCLARÉ]`, 2026-11-23 : l'opérateur déclare
qu'**aucun compte *Run As* n'utilise plus `scom_mon`**. Origine de cette confirmation (équipe
SCOM ou constat de l'opérateur dans la console) : non précisée, `@VERIF`.
**Réserve (R-02) : cette déclaration ne nomme pas `scom_adm`**, que la procédure supprime
aussi. F-268 dit que `<COMPTE-SCOM-Q>` porte les deux profils UNIX/Linux, sans rien dire des
autres comptes *Run As* enregistrés. Pour `scom_adm` : `@VERIF`, **à lever avant A2**.

**Hors objet — cités, non traités :**
- les entrées BeyondTrust : geste distinct de l'opérateur (D-023, `comptes-scom-host02.md:169`).
  **À ne faire qu'après V2** : recréer un compte (étape R) suppose son entrée ;
- le dossier personnel de `<COMPTE-SCOM-Q>` créé par la vérification du 09/11 (F-240,
  PO-073 (4)) : sort non décidé ;
- PO-069 et les sauvegardes omi : `/etc/opt/omi/ssl.avant-pfx` **est à garder** ;
- PO-056, dont les fichiers de travail de `comptes-scom-host02.md` dans `~`.

Aucun outil qui fabrique un certificat n'est lancé (R-10).

> **Aucune commande de ce fichier n'a été exécutée sur `host02`** (R-08). Sur le poste :
> lecture de `man userdel`, et enchaînement des blocs dans un bac à sable aux commandes
> système simulées (§ *Sources*). **Réserve de transposition :** Fedora 44, non `host02`.

**R-07 :** `scom_adm`, `scom_mon` et l'empreinte de `scom-test.sudoers` sont déjà publics
dans le dépôt. Le nom de `<COMPTE-SCOM-Q>` est **saisi** en V1, et nulle part ailleurs.
Rien des sorties n'est à recopier dans le dépôt.

## Corrections de PO-073 appliquées ici

| PO-073 | Application dans ce fichier |
|---|---|
| (1) R s'arrête si la sauvegarde manque | Chaque ligne de restauration commence par `sudo test -f … &&`. Les deux écritures sont en plus conditionnées **dans le shell** par `G_A1` et `G_A2`, remises à `KO` en tête de bloc. |
| (2) Seconde session par `sudo -s` | Étape 2S, avec `id -u` capturé. |
| (3) Référence prise juste avant l'action | Compteurs de `/var/log/secure` et de `EventId=20146` relevés dans le bloc qui agit. |
| (4) `sudo -u … sudo -n -l` ouvre une session PAM | Forme **non employée** : seule `sudo -l -U`. |
| (5) S est joué et capturé en premier | Étape S. |

## Ordre de jeu

| # | Étape | Écrit sur `host02` ? | Un échec arrête-t-il la suite ? |
|---|---|---|---|
| S | Témoins, heure de référence | non | **Oui** |
| 0a | Lecture seule : versions, `man userdel` de la cible | non | **Oui** |
| 0b | Lecture seule : état des comptes, références | non | **Oui** |
| 0c | Lecture seule : `scom-test` | non | **Oui** |
| R | **Retour arrière — à lire maintenant, à jouer seulement si besoin** | — | — |
| 2S | Seconde session `root`, ouverte jusqu'à la fin de V2 | non | **Oui** |
| SV | Sauvegardes | `/root/scom-test.avant-d023`, `/root/scom-comptes-locaux.avant-d023.tar.gz` | **Oui** |
| A1 | Retrait de `scom-test` | `/etc/sudoers.d/scom-test` | **Oui** → R |
| A2 | Suppression des deux comptes | comptes, groupes, dossiers, spool | **Oui**, sans retour automatique |
| V1 | Contrôles immédiats | non | **Oui** → décision |
| V2 | Après 15 à 30 minutes : collecte et état SCOM | non | **Oui** → décision |
| N | Clôture : les sauvegardes **restent** | non | — |

**Ajustement au plan du brief :** V est scindée en V1 et V2, car la collecte de SCOM ne se
constate qu'après un intervalle (F-263).

**Toute la procédure se joue dans une seule session** : `G_A1`, `UID_ADM`, `UID_MON` et les
compteurs y sont des variables du shell. Une session perdue reprend à S, et l'on relit
l'état avant de poursuivre.

## Forme des commandes

**Une commande par ligne**, suivie de son code de retour. Aucune continuation, aucun
*heredoc*. Chaque bloc est encadré de `DEBUT` / `FIN` : une capture sans ses deux marqueurs
est incomplète (R-08). Toutes les lignes `sudo` sont des élévations.

**Interdit : afficher le contenu de `/etc/shadow`, `/etc/gshadow`, de leurs copies `-`, ou
de `/etc/security/opasswd`** (`comptes-scom-host02.md:234-236`). La recherche dans `/etc` se
fait donc en deux temps : **noms de fichiers seuls** (`grep -l`) sur tout `/etc`, puis
contenu **en excluant** ces fichiers.

**Journal.** Les recherches dans `/var/log/secure` s'écrivent `sudo tail … | grep`, jamais
`sudo grep motif` : `sudo` journalise sa ligne de commande, et le motif s'y retrouverait.

---

## Étape S — Témoins et heure de référence

```
echo "=== DEBUT ES ==="
date -u; echo "rc_date=$?  (0 attendu : reference UTC)"
unset SCOM_TEMOIN_VIDE; echo "rc_unset=$?"
echo "${SCOM_TEMOIN_VIDE:?vide}"; echo "rc_temoin_vide=$?  (cette ligne NE DOIT PAS s'afficher)"
getent passwd root > /dev/null; echo "rc_getent_root=$?  (0 attendu)"
getent passwd scom_absent; echo "rc_getent_absent=$?  (2 attendu)"
echo "=== FIN ES ==="
```

**Attendu :** message `bash: SCOM_TEMOIN_VIDE: vide`, **aucune** ligne `rc_temoin_vide` ;
`rc_getent_root=0` ; `rc_getent_absent=2`. **Tout écart : arrêt.** Le nom `scom_absent` sert
de témoin dans toute la procédure : il ne doit exister nulle part.

---

## Étape 0a — Lecture seule : versions et `man userdel` de la cible

```
echo "=== DEBUT E0A ==="
rpm -q shadow-utils tar; echo "rc_rpm=$?  (0 attendu)"
LC_ALL=C MANWIDTH=100 man userdel | grep -n -A24 '^EXIT VALUES'; echo "rc_man=${PIPESTATUS[1]}  (0 attendu)"
grep -nE '^(MAIL_DIR|USERGROUPS_ENAB|USERDEL_CMD)' /etc/login.defs; echo "rc_login_defs=$?  (releve)"
echo "=== FIN E0A ==="
```

**Attendu et conditions d'arrêt :**
- `rpm -q` : deux versions, **à recopier au rapport**. **Arrêt** si l'un des paquets manque.
- **`man userdel` lu sur `host02` : c'est lui qui fonde les codes de cette procédure.** La
  page du poste a déjà trompé une fois (F-184, F-200). Sur le poste : `0` succès, `6`
  utilisateur inexistant, `8` utilisateur connecté, `12` dossier non supprimé (S-39). Les
  codes de la cible sont `@VERIF`. **Arrêt** si `6`, `8` ou `12` y a un autre sens, ou si la
  section manque.
- `MAIL_DIR` : `/var/spool/mail` attendu `@VERIF`. **Autre valeur : arrêt**, A2 contrôle ce
  chemin. `USERDEL_CMD` présent : **arrêt**, `userdel` lancerait un script non relu.

---

## Étape 0b — Lecture seule : état des comptes et références

```
echo "=== DEBUT E0B ==="
getent passwd root > /dev/null; echo "rc_temoin_root=$?  (0 attendu)"
getent -s files passwd scom_adm; echo "rc_passwd_adm=$?  (0 attendu)"
getent -s files passwd scom_mon; echo "rc_passwd_mon=$?  (0 attendu)"
getent -s files group scom_adm; echo "rc_group_adm=$?  (0 attendu ; 4e champ vide)"
getent -s files group scom_mon; echo "rc_group_mon=$?  (0 attendu ; 4e champ vide)"
id scom_adm; echo "rc_id_adm=$?  (0 attendu)"
id scom_mon; echo "rc_id_mon=$?  (0 attendu)"
pgrep -a -u scom_adm,scom_mon; echo "rc_pgrep_e=$?  (1 attendu)"
pgrep -a -U scom_adm,scom_mon; echo "rc_pgrep_r=$?  (1 attendu)"
pgrep -u root -c; echo "rc_pgrep_temoin=$?  (0 attendu)"
sudo crontab -l -u scom_adm; echo "rc_cron_adm=$?  (1 attendu, no crontab)"
sudo crontab -l -u scom_mon; echo "rc_cron_mon=$?  (1 attendu, no crontab)"
sudo lastlog -u scom_adm; echo "rc_lastlog_adm=$?  (releve)"
sudo lastlog -u scom_mon; echo "rc_lastlog_mon=$?  (releve)"
sudo last -w -n 5 scom_adm scom_mon; echo "rc_last=$?  (releve)"
sudo ls -la /home/scom_adm /home/scom_mon; echo "rc_ls_home=$?  (0 attendu)"
sudo ls -la /home/scom_adm/.ssh /home/scom_mon/.ssh; echo "rc_ls_ssh=$?  (releve)"
sudo test ! -e /home/scom_adm/.ssh/authorized_keys; echo "rc_sans_cle_adm=$?  (0 attendu)"
sudo test ! -e /home/scom_mon/.ssh/authorized_keys; echo "rc_sans_cle_mon=$?  (0 attendu)"
FS_LOCAUX=$(df --local -x devtmpfs --output=target | tail -n +2); echo "rc_df=$?"; echo $FS_LOCAUX
ORPH=$(sudo find $FS_LOCAUX -xdev -path /home/scom_adm -prune -o -path /home/scom_mon -prune -o -path /var/spool/mail/scom_adm -prune -o -path /var/spool/mail/scom_mon -prune -o \( -user scom_adm -o -user scom_mon \) -print); echo "rc_find=$?  (0 attendu)"
printf '%s' "$ORPH" | head -n 20; printf '%s' "$ORPH" | grep -c .; echo "rc_orphelins=${PIPESTATUS[1]}  (0 affiche, 1 attendu)"
sudo find /home/scom_adm -maxdepth 0 -user scom_adm | grep -c .; echo "rc_find_temoin=${PIPESTATUS[1]}  (1 affiche, 0 attendu)"
L_ETC=$(sudo grep -rlwE 'scom_adm|scom_mon' /etc); echo "rc_l_etc=$?  (0 attendu)"
printf '%s\n' "$L_ETC"
printf '%s\n' "$L_ETC" | grep -cxF /etc/sudoers.d/scom-test; echo "rc_etc_temoin=${PIPESTATUS[1]}  (1 affiche, 0 attendu)"
printf '%s\n' "$L_ETC" | grep -vxE '/etc/(passwd|group|shadow|gshadow|subuid|subgid)-?|/etc/sudoers\.d/scom-test'; echo "rc_hors_liste=${PIPESTATUS[1]}  (1 attendu : rien d'autre)"
sudo grep -rnIwE --exclude='shadow*' --exclude='gshadow*' --exclude='opasswd*' --exclude=scom-test 'scom_adm|scom_mon' /etc; echo "rc_contenu=$?  (releve)"
echo "=== FIN E0B ==="
```

**Attendu et conditions d'arrêt :**
- **Comptes locaux.** Les quatre `getent -s files` à `0`. Dossier personnel exactement
  `/home/scom_adm` et `/home/scom_mon` : `userdel -r` supprime **le chemin inscrit**.
  **Arrêt sinon.** Groupe : 4e champ **vide**, sinon `userdel` peut le conserver : arrêt.
- **Processus.** `rc_pgrep_e=1`, `rc_pgrep_r=1`, témoin `0`. **Arrêt si un processus
  tourne** : `userdel` rendrait `8`. On ne tue rien ; on cherche qui l'a lancé.
- **`crontab`.** `rc=1` et « no crontab » (forme `@VERIF`). **Une crontab : arrêt.**
- **Usage récent** (`lastlog`, `last`) : **relevé, à dire au rapport**. `lastlog` peut ne
  pas exister (`rc=127`, absent du poste) : `last` suffit. Une connexion postérieure à
  D-020 sans explication : **arrêt**, à éclaircir avec l'équipe SCOM.
- **Clé.** **Aucune clé n'a été posée** : la partie B est sans objet depuis D-019 (a).
  `.ssh` peut exister ou non (`rc_ls_ssh` relevé). Un `authorized_keys` présent contredit le
  dossier : **arrêt**.
- **Fichiers hors dossier personnel.** `-xdev` seul depuis `/` manquerait toute partition
  séparée : `find` parcourt donc **chaque système de fichiers local**. `0`, `rc=1`, témoin
  `1`. **Un fichier trouvé : arrêt**, il deviendrait orphelin.
- **Références dans `/etc`.** Attendu : `passwd`, `group`, `shadow`, `gshadow`, leurs copies
  `-`, `subuid`, `subgid` (`@VERIF`) et `sudoers.d/scom-test`. `rc_etc_temoin=0`,
  `rc_hors_liste=1`. **Toute autre ligne : arrêt**, tant qu'elle n'est pas expliquée. Si
  c'est `opasswd`, **ne pas l'afficher** : il peut contenir d'anciens hachages. Le dire au
  rapport ; son sort est une décision de l'opérateur.

---

## Étape 0c — Lecture seule : `scom-test`

```
echo "=== DEBUT E0C ==="
sudo ls -lZ /etc/sudoers.d/scom-test; echo "rc_ls=$?  (0 attendu)"
echo '895682f51137e5c87d16b1b2e3a5f0eb51dc587d033d4d942f8fd6a4f8fd5a07  /etc/sudoers.d/scom-test' | sudo sha256sum -c -; echo "rc_sha=${PIPESTATUS[1]}  (0 attendu)"
echo '0000000000000000000000000000000000000000000000000000000000000000  /etc/sudoers.d/scom-test' | sudo sha256sum -c -; echo "rc_sha_temoin=${PIPESTATUS[1]}  (1 attendu)"
sudo -l -U scom_adm; echo "rc_list_adm=$?  (0 attendu)"
sudo -l -U scom_mon; echo "rc_list_mon=$?  (0 attendu)"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
echo "=== FIN E0C ==="
```

**Attendu :** `-r--r-----`, `root root`, `etc_t`, 5759 octets (F-205, F-230) ;
`/etc/sudoers.d/scom-test: OK`, `rc_sha=0` ; `FAILED`, `rc_sha_temoin=1` ; pour `scom_adm`,
la seule règle `cat` du certificat, pour `scom_mon`, la seule règle `scxlogfilereader -p`
(F-206) ; `visudo -c` `0`. **Arrêt si l'empreinte ne correspond pas** : le fichier a changé
depuis F-201, et la sauvegarde ne restaurerait pas ce que décrit le dossier. **Arrêt** aussi
sur toute autre règle.

---

## Étape R — Retour arrière, écrit avant la première écriture

**À lire maintenant, à ne jouer qu'en cas de besoin.**

**L'asymétrie est à connaître avant A2 :**
- **`scom-test` se restaure** à l'identique, par ER ou ER-ROOT ;
- **un compte supprimé ne se restaure pas.** Sa recréation suit A1 de
  `comptes-scom-host02.md`, avec un **nouveau mot de passe**, celui de son entrée
  BeyondTrust. L'uid peut changer, et l'ancien hachage n'est pas sauvegardé, volontairement ;
- **l'archive de SV sert à récupérer un contenu**, fichier par fichier. Elle ne ramène pas
  à l'état antérieur.

### Avec `sudo` (session de l'opérateur)

```
echo "=== DEBUT ER ==="
sudo test -f /root/scom-test.avant-d023; echo "rc_sauvegarde=$?  (0 attendu ; sinon ARRET, rien n'est restaure)"
sudo test -f /root/scom-test.avant-d023 && sudo install -m 440 -o root -g root /root/scom-test.avant-d023 /etc/sudoers.d/scom-test; echo "rc_install=$?  (0 attendu)"
sudo test -f /root/scom-test.avant-d023 && sudo restorecon -v /etc/sudoers.d/scom-test; echo "rc_restorecon=$?  (0 attendu)"
sudo test -f /root/scom-test.avant-d023 && sudo cmp /root/scom-test.avant-d023 /etc/sudoers.d/scom-test; echo "rc_cmp=$?  (0 attendu)"
sudo ls -lZ /etc/sudoers.d/scom-test; echo "rc_ls=$?"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
echo "=== FIN ER ==="
```

### Sans `sudo` (seconde session, déjà `root`)

```
echo "=== DEBUT ER-ROOT ==="
id -u; echo "rc_id=$?  (0 affiche)"
test -f /root/scom-test.avant-d023; echo "rc_sauvegarde=$?  (0 attendu ; sinon ARRET, rien n'est restaure)"
test -f /root/scom-test.avant-d023 && install -m 440 -o root -g root /root/scom-test.avant-d023 /etc/sudoers.d/scom-test; echo "rc_install=$?  (0 attendu)"
test -f /root/scom-test.avant-d023 && restorecon -v /etc/sudoers.d/scom-test; echo "rc_restorecon=$?  (0 attendu)"
test -f /root/scom-test.avant-d023 && cmp /root/scom-test.avant-d023 /etc/sudoers.d/scom-test; echo "rc_cmp=$?  (0 attendu)"
ls -lZ /etc/sudoers.d/scom-test; echo "rc_ls=$?"
visudo -c; echo "rc_visudo=$?  (0 attendu)"
echo "=== FIN ER-ROOT ==="
```

**Attendu :** tous les codes indiqués. La sauvegarde est en `600` : la restauration passe
donc par `install -m 440`, et non par `cp -a`. Si `rc_sauvegarde` n'est pas `0` : **arrêt et
alerte**. Si `visudo -c` échoue encore après R, **`scom-test` n'était pas la cause** : ne
rien tenter d'autre, garder la seconde session.

---

## Étape 2S — Seconde session, avant la première écriture

Ouvrir **une seconde session SSH** sur `host02`, puis, **chacune seule** :

```
sudo -s
```

```
id -u; echo "rc_id=$?  (0 affiche)"
```

**Pas `sudo -i`** : le shell de connexion de `root` est `nologin` (F-236, PO-073). **Capturer
`id -u`.** Laisser cette session ouverte **jusqu'à la fin de V2**. Sa raison d'être : rester
`root` si le retrait de `scom-test` cassait `sudo`.

---

## Étape SV — Sauvegardes

**Aucune ligne de `/etc/shadow` n'est sauvegardée.** L'archive peut contenir un historique de
commandes : elle reste en `600`, dans `/root`.

```
echo "=== DEBUT ESV ==="
sudo test ! -e /root/scom-test.avant-d023; echo "rc_absente=$?  (0 attendu ; sinon ARRET)"
sudo test ! -e /root/scom-comptes-locaux.avant-d023.tar.gz; echo "rc_archive_absente=$?  (0 attendu ; sinon ARRET)"
sudo test ! -e /root/scom-test.avant-d023 && sudo install -m 600 -o root -g root /etc/sudoers.d/scom-test /root/scom-test.avant-d023; echo "rc_install=$?  (0 attendu)"
sudo test -f /root/scom-test.avant-d023; echo "rc_test_f=$?  (0 attendu)"
sudo cmp /etc/sudoers.d/scom-test /root/scom-test.avant-d023; echo "rc_cmp=$?  (0 attendu)"
sudo cmp /root/scom-test.avant-d023 /dev/null; echo "rc_cmp_temoin=$?  (1 attendu)"
sudo stat -c '%a %U %G %s' /root/scom-test.avant-d023; echo "rc_stat=$?  (600 root root 5759)"
sudo bash -c 'umask 077 && test ! -e /root/scom-comptes-locaux.avant-d023.tar.gz && tar --selinux --acls --xattrs -C /home -czpf /root/scom-comptes-locaux.avant-d023.tar.gz scom_adm scom_mon'; echo "rc_tar=$?  (0 attendu)"
sudo test -f /root/scom-comptes-locaux.avant-d023.tar.gz; echo "rc_test_archive=$?  (0 attendu)"
sudo stat -c '%a %U %G %s' /root/scom-comptes-locaux.avant-d023.tar.gz; echo "rc_stat_archive=$?  (600 root root, taille relevee)"
sudo tar -tzf /root/scom-comptes-locaux.avant-d023.tar.gz | grep -cE '^scom_(adm|mon)/$'; echo "rc_liste=${PIPESTATUS[1]}  (2 affiche, 0 attendu)"
sudo tar -tzf /root/scom-comptes-locaux.avant-d023.tar.gz | grep -c '^scom_absent/$'; echo "rc_liste_temoin=${PIPESTATUS[1]}  (0 affiche, 1 attendu)"
N_TAR=$(sudo tar -tzf /root/scom-comptes-locaux.avant-d023.tar.gz | wc -l); echo "rc_n_tar=$? N_TAR=$N_TAR"
N_HOME=$(sudo find /home/scom_adm /home/scom_mon | wc -l); echo "rc_n_home=$? N_HOME=$N_HOME  (egal a N_TAR)"
echo "=== FIN ESV ==="
```

**Arrêt** sur tout code différent, ou si `N_TAR` et `N_HOME` diffèrent : R et la récupération
d'un contenu ne seraient plus garantis. Une sauvegarde déjà présente n'est **jamais** écrasée.

---

## Étape A1 — Retrait de `scom-test`

```
echo "=== DEBUT EA1 ==="
G_A1=KO; echo "garde_initiale=$G_A1"
date -u; echo "rc_date=$?  (reference UTC de V)"
N_SECURE=$(sudo awk 'END{print NR}' /var/log/secure); echo "rc_ref_secure=$? N_SECURE=$N_SECURE"
N_20146=$(sudo grep -c 'EventId=20146' /var/opt/omi/log/omiserver.log); echo "rc_ref_20146=$? N_20146=$N_20146  (2 selon F-309)"
L_ETC=$(sudo grep -rlwE 'scom_adm|scom_mon' /etc); echo "rc_l_etc=$?  (0 attendu)"
printf '%s\n' "$L_ETC" | grep -vxE '/etc/(passwd|group|shadow|gshadow|subuid|subgid)-?|/etc/sudoers\.d/scom-test'; echo "rc_hors_liste=${PIPESTATUS[1]}  (1 attendu)"
FS_LOCAUX=$(df --local -x devtmpfs --output=target | tail -n +2); echo "rc_df=$?"
ORPH=$(sudo find $FS_LOCAUX -xdev -path /home/scom_adm -prune -o -path /home/scom_mon -prune -o -path /var/spool/mail/scom_adm -prune -o -path /var/spool/mail/scom_mon -prune -o \( -user scom_adm -o -user scom_mon \) -print); RC_ORPH=$?; echo "rc_find=$RC_ORPH  (0 attendu)"
[ "$RC_ORPH" = 0 ] && [ -z "$ORPH" ] && printf '%s\n' "$L_ETC" | grep -qxF /etc/sudoers.d/scom-test && ! printf '%s\n' "$L_ETC" | grep -qvxE '/etc/(passwd|group|shadow|gshadow|subuid|subgid)-?|/etc/sudoers\.d/scom-test' && getent -s files passwd scom_adm > /dev/null && getent -s files passwd scom_mon > /dev/null && ! pgrep -u scom_adm,scom_mon > /dev/null && ! pgrep -U scom_adm,scom_mon > /dev/null && [ "$(sudo sha256sum < /etc/sudoers.d/scom-test | cut -d' ' -f1)" = 895682f51137e5c87d16b1b2e3a5f0eb51dc587d033d4d942f8fd6a4f8fd5a07 ] && sudo cmp -s /root/scom-test.avant-d023 /etc/sudoers.d/scom-test && [ "$(sudo tar -tzf /root/scom-comptes-locaux.avant-d023.tar.gz | grep -cE '^scom_(adm|mon)/$')" = 2 ] && sudo visudo -c > /dev/null && G_A1=OK; echo "garde_a1=$G_A1  (OK attendu ; KO : rien ne sera retire)"
[ "$G_A1" = OK ] && sudo rm /etc/sudoers.d/scom-test; echo "rc_rm=$?  (0 attendu)"
sudo test ! -e /etc/sudoers.d/scom-test; echo "rc_absent=$?  (0 attendu)"
sudo visudo -c; RC_V_A1=$?; echo "rc_visudo=$RC_V_A1  (0 attendu ; sinon R depuis 2S)"
sudo tail -n "+$((N_SECURE+1))" /var/log/secure | grep -F 'rm /etc/sudoers.d/scom-test' | grep -c 'COMMAND='; echo "rc_journal_rm=${PIPESTATUS[2]}  (1 affiche, 0 attendu)"
echo "=== FIN EA1 ==="
```

**Attendu et conditions d'arrêt :**
- **`G_A1`** : `OK`, placée **avant** le `rm` (R-13). `KO` : le fichier **n'est pas retiré**,
  `rc_rm=1` : **arrêt, sans R**. La garde recalcule ses conditions au lieu de faire
  confiance aux étapes précédentes, car un bloc peut être collé hors séquence.
- `rm` est **sans `-f`** : un fichier déjà absent se signale.
- **`visudo -c` différent de `0` : R immédiatement**, depuis la seconde session si `sudo`
  refuse.
- Journal : `1`, `rc=0`. La forme `COMMAND=` est celle de F-275 ; le chemin affiché de `rm`
  sur `host02` est `@VERIF`. Écart : à dire, sans R si `rc_absent=0`.

---

## Étape A2 — Suppression des deux comptes

**Point de non-retour de cette procédure** (étape R). `G_A1` doit venir d'A1, dans cette
même session.

```
echo "=== DEBUT EA2 ==="
G_A2=KO; echo "garde_initiale=$G_A2"
N_SECURE2=$(sudo awk 'END{print NR}' /var/log/secure); echo "rc_ref_secure=$? N_SECURE2=$N_SECURE2"
UID_ADM=$(id -u scom_adm); echo "rc_uid_adm=$?  (0 attendu)"
UID_MON=$(id -u scom_mon); echo "rc_uid_mon=$?  (0 attendu)"
getent passwd scom_absent; echo "rc_absent_temoin=$?  (2 attendu)"
sudo userdel scom_absent; RC_UD_T=$?; echo "rc_userdel_temoin=$RC_UD_T  (6 attendu)"
[ "${G_A1:-KO}" = OK ] && [ "${RC_V_A1:-1}" = 0 ] && [ "$RC_UD_T" = 6 ] && sudo test ! -e /etc/sudoers.d/scom-test && sudo visudo -c > /dev/null && [ "$(getent -s files passwd scom_adm | cut -d: -f6)" = /home/scom_adm ] && [ "$(getent -s files passwd scom_mon | cut -d: -f6)" = /home/scom_mon ] && [ -z "$(getent -s files group scom_adm | cut -d: -f4)" ] && [ -z "$(getent -s files group scom_mon | cut -d: -f4)" ] && [ "$UID_ADM" -gt 0 ] && [ "$UID_MON" -gt 0 ] && ! pgrep -u scom_adm,scom_mon > /dev/null && ! pgrep -U scom_adm,scom_mon > /dev/null && sudo test -f /root/scom-comptes-locaux.avant-d023.tar.gz && sudo test -f /root/scom-test.avant-d023 && G_A2=OK; echo "garde_a2=$G_A2  (OK attendu ; KO : aucun compte ne sera supprime)"
[ "$G_A2" = OK ] && sudo userdel -r scom_adm; RC_UD_ADM=$?; echo "rc_userdel_adm=$RC_UD_ADM  (0 attendu)"
[ "$G_A2" = OK ] && [ "$RC_UD_ADM" = 0 ] && sudo userdel -r scom_mon; RC_UD_MON=$?; echo "rc_userdel_mon=$RC_UD_MON  (0 attendu)"
getent passwd root > /dev/null; echo "rc_root=$?  (0 attendu : temoin)"
getent group root > /dev/null; echo "rc_group_root=$?  (0 attendu : temoin)"
getent passwd scom_adm; echo "rc_passwd_adm=$?  (2 attendu)"
getent passwd scom_mon; echo "rc_passwd_mon=$?  (2 attendu)"
getent group scom_adm; echo "rc_group_adm=$?  (2 attendu)"
getent group scom_mon; echo "rc_group_mon=$?  (2 attendu)"
sudo test ! -e /home/scom_adm; echo "rc_home_adm=$?  (0 attendu)"
sudo test ! -e /home/scom_mon; echo "rc_home_mon=$?  (0 attendu)"
sudo test ! -e /var/spool/mail/scom_adm; echo "rc_mail_adm=$?  (0 attendu)"
sudo test ! -e /var/spool/mail/scom_mon; echo "rc_mail_mon=$?  (0 attendu)"
sudo tail -n "+$((N_SECURE2+1))" /var/log/secure | grep -E 'userdel\[' ; echo "rc_journal_userdel=${PIPESTATUS[1]}  (releve)"
echo "=== FIN EA2 ==="
```

**Attendu et conditions d'arrêt :**
- **Témoin `userdel scom_absent`** : `rc=6`, sans `-r`, sur un nom absent. Il prouve sur
  `host02` le sens du code, et **n'écrit rien**. `G_A2` l'exige.
- **`G_A2`** : `OK`, **avant** tout `userdel`. `KO` : rien n'est supprimé, `rc_userdel_adm=1`
  **vient alors de la garde**, pas de `userdel` : **arrêt**, sans R.
- **`userdel` en échec.** Le second n'est lancé que si le premier a rendu `0`. `8` : un
  processus est apparu, **on ne force pas** (`-f` interdit). `12` : le compte est retiré
  mais son dossier reste, **arrêt et relevé**. **Ne pas rejouer le bloc** : `G_A1` et les
  `id` ne décrivent plus l'état. Reprendre par une lecture de 0b.
- **Contrôles d'absence** : `2` pour les quatre `getent`, témoins `root` à `0`, dossiers et
  spool absents. Le message éventuel sur un spool inexistant et les lignes `userdel[` du
  journal sont `@VERIF` : les relever.

---

## Étape V1 — Contrôles immédiats

Le nom de `<COMPTE-SCOM-Q>` est saisi ici, **seul, interactif** ; c'est son unique emploi :

```
read -r -p 'Compte de supervision AD (nom court, minuscules) : ' COMPTE
```

```
echo "=== DEBUT EV1 ==="
printf '%s\n' "${COMPTE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (1 affiche, 0 attendu)"
sudo -l -U scom_adm; echo "rc_list_adm=$?  (1 attendu, utilisateur inconnu)"
sudo -l -U scom_mon; echo "rc_list_mon=$?  (1 attendu, utilisateur inconnu)"
sudo -l -U "${COMPTE:?vide}"; echo "rc_list=$?  (0 attendu : une seule regle)"
sudo -l -U "${COMPTE:?vide}" /opt/microsoft/scx/bin/scxlogfilereader -p; echo "rc_U_permis=$?  (0 attendu : PERMIS)"
sudo -l -U "${COMPTE:?vide}" /opt/microsoft/scx/bin/tools/scxadmin -status; echo "rc_U_refus=$?  (1 attendu, sans message : REFUSE)"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
FS_LOCAUX=$(df --local -x devtmpfs --output=target | tail -n +2); echo "rc_df=$?"
ORPH2=$(sudo find $FS_LOCAUX -xdev \( -uid "${UID_ADM:?vide}" -o -uid "${UID_MON:?vide}" \) -print); echo "rc_find=$?  (0 attendu)"
printf '%s' "$ORPH2" | head -n 20; printf '%s' "$ORPH2" | grep -c .; echo "rc_orphelins=${PIPESTATUS[1]}  (0 affiche, 1 attendu)"
sudo grep -c 'EventId=20146' /var/opt/omi/log/omiserver.log; echo "rc_20146=$?  (affiche = ${N_20146:?vide})"
echo "=== FIN EV1 ==="
```

**Attendu et lecture :**
- **Anciens comptes** : `sudo` les dit inconnus, `rc=1`. **La forme exacte du message est
  `@VERIF` : la relever.** Une liste de règles affichée : **arrêt**.
- **`<COMPTE-SCOM-Q>`** : **une seule règle**, `scxlogfilereader -p` (D-022, F-300) ; permis
  `0`, refus `1` sans message. **Tout écart : arrêt et alerte**, c'est le compte de
  production de la supervision sur cet hôte.
- **Orphelins par uid** : `0`, `rc=1`. Un fichier trouvé : à dire, **ne rien supprimer**.
- **`EventId=20146`** : égal à `N_20146`. **Inférieur** : le journal a pu tourner (F-259),
  **non concluant**. **Supérieur** : un nouveau refus, à dire en tête du rapport.

---

## Étape V2 — Après 15 à 30 minutes : collecte et état SCOM

**Ne fermer aucune session.** Les connexions de SCOM sont espacées d'environ 15 minutes
(F-263, `[PILOTE-DÉCLARÉ]`, avec l'écart non expliqué de F-310).

```
echo "=== DEBUT EV2 ==="
date -u; echo "rc_date=$?"
sudo tail -n 8 /var/opt/omi/log/omiserver.log; echo "rc_tail=$?  (0 attendu)"
sudo grep -c 'EventId=20146' /var/opt/omi/log/omiserver.log; echo "rc_20146=$?  (affiche = ${N_20146:?vide})"
echo "=== FIN EV2 ==="
```

**Attendu :** au moins une ligne **postérieure à la référence UTC d'A1** (`omiserver.log` est
en UTC, F-254), de la forme relevée en F-263 ; `EventId=20146` inchangé. **Puis faire
confirmer *Healthy* par l'équipe SCOM, à consigner `[TIERS-DÉCLARÉ]`.**

**Portée :** ce contrôle montre que la collecte se poursuit sans les comptes locaux. Il ne
couvre pas le chemin privilégié, non observé (PO-059). Un écart : **décision de
l'opérateur**.

---

## Étape N — Clôture

Il n'y a **aucun fichier de travail** à retirer. **Les sauvegardes de SV restent** : leur sort
est à décider (PO-056). La seconde session se ferme après la confirmation de V2.

```
echo "=== DEBUT EN ==="
sudo ls -l /root/scom-test.avant-d023 /root/scom-comptes-locaux.avant-d023.tar.gz; echo "rc_ls=$?  (0 attendu : conservees)"
unset COMPTE G_A1 G_A2 RC_V_A1 UID_ADM UID_MON ORPH ORPH2 L_ETC; echo "rc_unset=$?"
echo "=== FIN EN ==="
```

---

## Ce que l'opérateur rapporte au pilote

Chaque bloc **entier**, marqueurs compris, en `[ÉCRAN-AAAA-MM-JJ]` (D-003), et en particulier :
l'origine de la déclaration sur `scom_mon` et la réponse sur `scom_adm` (*Run As*) ; les versions et la section
*EXIT VALUES* de 0a ; les lignes de 0b hors liste ; `id -u` de la seconde session ; `garde_a1`,
`garde_a2` ; les codes de `userdel` ; les messages de `sudo -l -U` en V1 ; V2 ; la confirmation
*Healthy* ; toute branche de R jouée.

**Aucune valeur réelle n'est à recopier dans le dépôt** (R-07).

---

## Gardes, branches et ce qui reste non démontré (R-04, R-12)

| Contrôle | Placé avant l'effet ? | Sens de l'échec |
|---|---|---|
| arrêt par `${…:?}` | oui | témoin joué en S |
| `getent` absent / présent | oui | témoins `root` et `scom_absent`, joués en S |
| codes de `userdel` (0a) | oui, avant A2 | `6` exercé en A2 par le témoin ; `8` et `12` **écrits, non démontrés** sur `host02` |
| processus (0b, `G_A1`, `G_A2`) | oui | `KO` exercé en bac à sable |
| fichiers hors dossier (0b, `G_A1`) | oui | témoin `find` en 0b ; `KO` exercé en bac à sable |
| références dans `/etc` (0b, `G_A1`) | oui | témoin `scom-test` en 0b ; `KO` exercé en bac à sable |
| empreinte de `scom-test` (0c, `G_A1`) | oui, avant `rm` | témoin à empreinte nulle, joué en 0c ; `KO` exercé en bac à sable |
| sauvegarde et archive (SV) | oui, avant A1 | témoins `/dev/null` et `scom_absent`, joués en SV ; non-écrasement exercé en bac à sable |
| `G_A1` | oui, avant `rm` | `KO` exercé en bac à sable : rien n'est retiré |
| `G_A2` | oui, avant `userdel` | `KO` exercé en bac à sable : aucun compte supprimé |
| second `userdel` subordonné au premier | oui | exercé en bac à sable (premier à `8`) |
| absences après A2 | **non** — constat | témoins `root` joués en A2 |
| règle de `<COMPTE-SCOM-Q>` (V1) | **non** — constat | permis et refusé côte à côte (R-13) |
| `EventId=20146` (V1, V2) | **non** — constat | compteur **non forcé** |

**Branches non exercées par un jeu nominal :** R et R-ROOT ; tout cas d'arrêt de 0a à 0c ;
`userdel` à `8` ou `12` sur `host02`.

---

## Sources lues pour cette procédure

Lues le **2026-11-23** sur le poste de rédaction, sans élévation.

- **`man userdel`** — `shadow-utils-4.19.0-7.fc44`, *EXIT VALUES* : `0`, `1`, `2`, `6`, `8`,
  `10`, `12` ; *CAVEATS* : refus en cas de processus en cours, et « You should manually check
  all file systems to ensure that no files remain owned by this user. » Au dossier : **S-39**.
- **Bac à sable sur le poste** (P-89), noms de comptes simulés, chemins redirigés vers un
  répertoire temporaire ; `sudo`, `rm` (déplacement vers une corbeille), `userdel`, `getent`,
  `id`, `pgrep`, `crontab`, `visudo`, `install`, `restorecon`, `df` et les critères
  d'utilisateur de `find` simulés ; `tar`, `sha256sum`, `cmp` et `grep` réels. Il éprouve
  **l'enchaînement du shell**, pas `sudo`, `userdel`, SELinux ni l'état réel de `host02`.
  Le bloc S n'y a pas été joué.

**Ce qui reste `@VERIF` :**
- l'origine de la déclaration sur `scom_mon`, et l'absence de *Run As* sur `scom_adm` ;
- les codes de `userdel` dans la page de `host02`, et `MAIL_DIR` ;
- la liste exacte des fichiers de `/etc` qui citent les comptes (`subuid`, `subgid`) ;
- la forme de « no crontab » ; la présence et la sortie de `lastlog` ;
- le chemin de `rm` dans la ligne `COMMAND=` et les lignes `userdel[` du journal ;
- le message éventuel de `userdel -r` sur le spool ;
- la forme du message de `sudo -l -U` pour un utilisateur inconnu.
