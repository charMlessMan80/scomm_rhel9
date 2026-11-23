# Procédure — retour de `scom-kerberos` au contenu de D-022 et fermeture de SSH aux groupes SCOM sur `host02`

**Ce que ce fichier porte :** des blocs de commandes que **l'opérateur** joue lui-même
sur `host02`, sous **son compte AD avec `sudo`**, dans l'ordre, et la lecture de chaque
sortie. Objet unique, sur `host02` seulement :
1. ramener `/etc/sudoers.d/scom-kerberos` au contenu de D-022, en retirant la ligne
   `scxadmin -status` et la ligne vide (D-024, F-278) ;
2. fermer SSH aux deux groupes SCOM par `DenyGroups` (D-025, PO-075).

**Ce que ce fichier NE porte PAS :** aucun fait, aucune décision, aucun point ouvert —
ils vivent dans `../scomm-certificat-facts.md`. Il n'a **aucune autorité propre** : tout
ce qu'il affirme renvoie au dossier, ou aux sources citées au § *Sources*. Ce qui n'est
établi ni par l'un ni par l'autre porte `@VERIF`.

**Hors périmètre — aucune ligne de ce fichier n'y touche :**
- les comptes locaux et `/etc/sudoers.d/scom-test` (D-023, procédure distincte) ;
- la configuration du parc : `rhel_post_install` (R-09, renvoi seul) ;
- le keytab, le certificat, omi. Aucun outil qui fabrique un certificat n'est lancé (R-10).

**Prérequis déclaratifs, à confirmer avant S :**
- l'équipe SCOM et l'interlocuteur de l'équipe AD ont été prévenus avant le jeu ;
- `<ADMIN-TIERS>` a été prévenu : recommandation du pilote sous D-024.

> **Aucune commande de ce fichier n'a été exécutée sur `host02`** (R-08). Ce qui a été
> exécuté sur le poste de rédaction est listé au § *Sources* : pages de manuel, `sshd`
> non privilégié sur la boucle locale, et enchaînement des blocs dans un bac à sable
> dont `sudo`, `systemctl`, `install`, `restorecon` et le client `ssh` sont simulés.
> **Réserve de transposition :** Fedora 44 et code amont, non `host02`. Les versions de
> `host02` sont relevées en 0a.

**R-07 :** aucun nom réel n'apparaît ici. Les trois noms sont **saisis** à l'étape S. Les
sorties affichées en contiennent : **rien n'en est recopié dans le dépôt.**

## Corrections de PO-073 appliquées ici

| PO-073 | Application dans ce fichier |
|---|---|
| (1) R s'arrête si la sauvegarde manque | Chaque ligne de restauration commence par `sudo test -f … &&`. **`sudo` est nécessaire** : sans lui, `test -f /root/…` échoue même si le fichier existe. |
| (2) Seconde session par `sudo -s` | Ouverte avant SV, avec `id -u` capturé. |
| (3) Référence prise juste avant l'action | Compteurs de `/var/log/secure` et de `EventId=20146` relevés dans le bloc qui agit. |
| (4) `sudo -u … sudo -n -l` ouvre une session PAM | Cette forme n'est **pas** employée : seule `sudo -l -U`. |
| (5) S est joué et capturé en premier | Étape S. |

La leçon de (1) va plus loin : **une consigne ne protège pas un bloc collé d'un seul
tenant**. Les deux écritures sont donc conditionnées **dans le shell** par `G_A1` et
`G_A2`, remises à `KO` en tête de leur bloc.

## Ordre de jeu

| # | Étape | Écrit sur `host02` ? | Un échec arrête-t-il la suite ? |
|---|---|---|---|
| S | Témoins, saisie des trois noms | non | **Oui** |
| 0a | Lecture seule : versions, `sshd` | non | **Oui** |
| 0b | Lecture seule : `scom-kerberos` | non | **Oui** |
| 0c | Lecture seule : groupes, opérateur, référence du certificat | non | **Oui** |
| R | **Retour arrière — à lire maintenant, à jouer seulement si besoin** | — | — |
| 2S | Seconde session `root`, ouverte jusqu'à la fin de V | non | **Oui** |
| SV | Sauvegarde de `scom-kerberos` | `/root/scom-kerberos.avant-d024` | **Oui** |
| A1 | Retour du sudoers (D-024) | `~/scom-kerberos-d024.sudoers`, `/etc/sudoers.d/scom-kerberos` | **Oui** → R1 |
| A2 | Fermeture SSH (D-025) | `~/scom-denygroups.conf`, `/etc/ssh/sshd_config.d/10-scom-denygroups.conf`, rechargement de `sshd` | **Oui** → R2 |
| V1 | Nouvelle session SSH de l'opérateur | non | **Oui** → R2 |
| V2 | Refus du compte, témoin, sudo, certificat | non | **Oui** → R |
| V3 | Après 15 à 30 minutes : collecte et état SCOM | non | **Oui** → décision, R si besoin |
| N | Nettoyage, après validation | suppressions | — |

**Toute la procédure se joue dans une seule session** : les noms y sont des variables du
shell. Une session perdue reprend à S, et l'on relit l'état avant de poursuivre.

## Forme des commandes

**Une commande par ligne**, suivie de son code de retour. Aucune continuation, aucun
*heredoc*. Chaque bloc est encadré de `DEBUT` / `FIN` : une capture sans ses deux
marqueurs est incomplète (R-08).

**Noms.** Toute ligne qui emploie un nom l'écrit `${COMPTE:?vide}`, `${GROUPE_Q:?vide}` ou
`${GROUPE_P:?vide}`. Si la variable est vide, le shell interactif abandonne la ligne
entière, `echo "rc_…"` compris (S-26). **Une ligne `rc_…` absente, avec ce message, vaut
arrêt.**

**Chaque bloc commence par `rc_vars`. Exceptions, inévitables :** le témoin initial de S,
joué avant la saisie, et **R** et **N**, qui n'emploient aucun nom. R doit rester jouable
depuis la seconde session, où ces variables n'existent pas : `rc_vars` l'y bloquerait.

**Journal.** Les recherches dans `/var/log/secure` s'écrivent `sudo tail … | grep`, jamais `sudo grep motif` :
`sudo` consigne sa ligne de commande dans `/var/log/secure`, et le motif s'y retrouverait.
Elles ne gardent que les lignes de `sshd` ou `sshd-session`.

**Élévations.** Toutes les lignes `sudo` sont des élévations. `/etc/ssh/sshd_config`, son
répertoire d'inclusion et `/var/log/secure` ne sont pas présumés lisibles sans privilège.

---

## Étape S — Session : témoins, puis saisie des trois noms

```
echo "=== DEBUT ES ==="
unset SCOM_TEMOIN_VIDE; echo "rc_unset=$?"
echo "${SCOM_TEMOIN_VIDE:?vide}"; echo "rc_temoin_vide=$?  (cette ligne NE DOIT PAS s'afficher)"
printf '%s\n' 'x,y' | grep -cxE '[a-z0-9_-]+'; echo "rc_forme_temoin=${PIPESTATUS[1]}  (0 affiché, 1 attendu)"
echo "=== FIN ES ==="
```

Puis, **chacune seule, interactive** :

```
read -r -p 'Compte de supervision (nom court, minuscules) : ' COMPTE
```

```
read -r -p 'Groupe SCOM Qualification (nom court, minuscules) : ' GROUPE_Q
```

```
read -r -p 'Groupe SCOM Production (nom court, minuscules) : ' GROUPE_P
```

Puis :

```
echo "=== DEBUT ES2 ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
[ "${GROUPE_Q:?vide}" != "${GROUPE_P:?vide}" ]; echo "rc_groupes_distincts=$?  (0 attendu)"
echo "=== FIN ES2 ==="
```

**Attendu :** message `bash: SCOM_TEMOIN_VIDE: vide` et **aucune** ligne `rc_temoin_vide` ;
`0` et `rc_forme_temoin=1` ; `3` et `rc_vars=0` ; `rc_groupes_distincts=0`. **Tout écart :
arrêt**, reprendre les `read`.

---

## Étape 0a — Lecture seule : versions et configuration de `sshd`

```
echo "=== DEBUT E0A ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
rpm -q bash sudo openssh-server; echo "rc_rpm=$?  (0 attendu)"
MANWIDTH=100 man sshd_config | grep -n -A8 '^ *DenyGroups$'; echo "rc_man_deny=${PIPESTATUS[1]}  (relevé)"
MANWIDTH=100 man sshd_config | grep -n 'first obtained value'; echo "rc_man_first=${PIPESTATUS[1]}  (0 attendu)"
sudo /usr/sbin/sshd -T | grep -E '^(allowgroups|allowusers|denygroups|denyusers|usepam|passwordauthentication|loglevel|syslogfacility|port|listenaddress) '; echo "rc_sshd_T=${PIPESTATUS[0]}  (0 attendu)"
sudo grep -nE '^[[:space:]]*(Include|Match)[[:space:]]' /etc/ssh/sshd_config; echo "rc_grep_include=$?  (0 attendu)"
sudo grep -rnE '^[[:space:]]*Match[[:space:]]' /etc/ssh/sshd_config.d/; echo "rc_grep_match_d=$?  (1 attendu : aucun Match)"
sudo ls -l --time-style=full-iso /etc/ssh/sshd_config.d/; echo "rc_ls_d=$?  (0 attendu)"
sudo ls -l /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_ls_conf=$?  (2 attendu : absent)"
systemctl show -p ActiveEnterTimestamp -p ExecReload sshd; echo "rc_show=$?  (0 attendu)"
systemctl is-active sshd; echo "rc_active=$?  (0 attendu)"
echo "=== FIN E0A ==="
```

**Attendu et conditions d'arrêt :**

| Ligne | Attendu | Sinon |
|---|---|---|
| `rpm -q` | trois versions : **à recopier au rapport** | **Arrêt** si `openssh-server` manque. |
| `man` | **Relevé** : `DenyGroups`, noms de groupes, liste séparée par des espaces, « each instance appending to the list ». `rc_man_deny` est relevé : sa valeur `1` n'arrête pas. | Relevé, pas critère. `host02` a OpenSSH 8.7 `[PILOTE-DÉCLARÉ]` ; que sa page contienne cette phrase n'est pas établi (`@VERIF`), alors que le code `V_8_7_P1` ajoute bien à la liste (§ *Sources*). La propriété est sans effet ici : la ligne `sshd -T` de cette même étape exige qu'aucune directive `denygroups` n'existe, donc le fichier d'A2 porte la seule. Absence de la phrase : à dire au rapport, sans arrêt. |
| `sshd -T` | **aucune** ligne `allowgroups`, `allowusers`, `denygroups`, `denyusers` (F-178, à reconfirmer) ; `usepam yes` (F-197) ; `loglevel INFO` ; `port 22` ; écoute incluant la boucle locale | **Arrêt** si une de ces directives existe (état inconnu du dossier), si `loglevel` est `QUIET`, `FATAL` ou `ERROR` (le refus ne serait pas journalisé), ou si le port n'est pas 22 (V2 ne viserait pas ce service). |
| `Include` / `Match` | une ligne `Include /etc/ssh/sshd_config.d/*.conf`, **avant** tout `Match` | **Arrêt** sinon : le fichier d'A2 serait ignoré, ou conditionnel. |
| `Match` dans `sshd_config.d` | aucun, `rc=1` | **Arrêt et relecture** : à instruire avant A2. |
| `sshd_config.d` | relevé : noms, modes, dates ; `50-redhat.conf` attendu `@VERIF` | Absence de `50-redhat.conf` : **à signaler, sans arrêt**. |
| `10-scom-denygroups.conf` | absent, `rc=2` | **Arrêt** : on ne pose pas par-dessus un fichier non lu. |
| `ActiveEnterTimestamp`, `ExecReload`, `is-active` | date ; une commande de rechargement ; `active` | **Arrêt** si inactif ou sans `ExecReload`. |

---

## Étape 0b — Lecture seule : état de `scom-kerberos`

```
echo "=== DEBUT E0B ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
sudo ls -l --time-style=full-iso /etc/sudoers.d/scom-kerberos; echo "rc_ls=$?  (0 attendu)"
sudo stat -c '%s' /etc/sudoers.d/scom-kerberos; echo "rc_stat=$?  (348 affiché)"
sudo awk 'END{print NR}' /etc/sudoers.d/scom-kerberos; echo "rc_lignes=$?  (5 affiché)"
sudo grep -c '^$' /etc/sudoers.d/scom-kerberos; echo "rc_vides=$?  (1 affiché, 0 attendu)"
sudo grep -cF "${COMPTE:?vide} ALL=(root) NOPASSWD: /opt/microsoft/scx/bin/tools/scxadmin -status" /etc/sudoers.d/scom-kerberos; echo "rc_scxadmin=$?  (1 affiché, 0 attendu)"
sudo cat -A /etc/sudoers.d/scom-kerberos; echo "rc_cat=$?"
sudo -l -U "${COMPTE:?vide}"; echo "rc_list=$?  (0 attendu)"
sudo ls -l /root/scom-kerberos.avant-d024; echo "rc_ls_sauvegarde=$?  (2 attendu)"
ls -l ~/scom-kerberos-d024.sudoers ~/scom-denygroups.conf; echo "rc_ls_travail=$?  (2 attendu)"
echo "=== FIN E0B ==="
```

**Attendu :** 348 octets, daté du 2026-11-12 16:34:07, `-r--r-----`, `root root` ; trois
lignes de F-237, la ligne `scxadmin -status` avec son commentaire, une ligne vide ; dans
`cat -A`, chaque ligne finit par `$` seul ; `sudo -l -U` montre **deux** règles,
`scxlogfilereader -p` et `scxadmin -status` (F-278, F-279). Sauvegarde et fichiers de
travail **absents**. **Tout écart : arrêt.** Le fichier a changé depuis le dossier, ou
un état précédent existe ; A1 et R ne s'appliqueraient plus à ce qu'ils décrivent.

---

## Étape 0c — Lecture seule : groupes, opérateur, référence du certificat

```
echo "=== DEBUT E0C ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
getent group "${GROUPE_Q:?vide}"; echo "rc_getent_q=$?  (0 attendu)"
getent group "${GROUPE_P:?vide}"; echo "rc_getent_p=$?  (0 attendu ; membres : relevé)"
getent group "${GROUPE_Q:?vide}" | cut -d: -f1 | grep -cxF "${GROUPE_Q:?vide}"; echo "rc_nom_q=${PIPESTATUS[2]}  (1 affiché, 0 attendu)"
getent group "${GROUPE_P:?vide}" | cut -d: -f1 | grep -cxF "${GROUPE_P:?vide}"; echo "rc_nom_p=${PIPESTATUS[2]}  (1 affiché, 0 attendu)"
id -Gn "${COMPTE:?vide}" | tr ' ' '\n' | grep -cxF "${GROUPE_Q:?vide}"; echo "rc_compte_dans_q=${PIPESTATUS[2]}  (1 affiché, 0 attendu)"
id -un; echo "rc_id=$?"
id -Gn "$(id -un)" | tr ' ' '\n' | grep -cxE "${GROUPE_Q:?vide}|${GROUPE_P:?vide}"; echo "rc_operateur_hors_groupes=${PIPESTATUS[2]}  (0 affiché, 1 attendu)"
ls -l ~/scom-kerberos-avant.fp; echo "rc_ls_fp=$?  (0 attendu)"
sudo openssl x509 -in /etc/opt/omi/ssl/omi.pem -noout -fingerprint -sha256 | diff ~/scom-kerberos-avant.fp -; echo "rc_openssl=${PIPESTATUS[0]} rc_diff=${PIPESTATUS[1]}  (0 0 attendus)"
echo "=== FIN E0C ==="
```

**Attendu et conditions d'arrêt :**
- **Nom exact des groupes.** `sshd` compare les noms **à la casse près**, et un nom qui ne
  correspond pas rend la règle **inerte, sans erreur** (mesuré, § *Sources*). Les quatre
  lignes `rc_getent_*` et `rc_nom_*` doivent donc rendre `0`. **Arrêt sinon.**
- **Groupe de production.** Il peut **ne pas avoir de membre** sur `host02` : le dire,
  **sans arrêter**. En revanche, `rc_getent_p=2` **arrête** : on ne peut distinguer une
  faute de frappe d'un groupe non visible (`ldap_id_mapping = False`, F-219). C'est une
  décision de l'opérateur, pas un cas à contourner.
- **Compte dans `<GROUPE-SCOM-Q>`** : `1`, `rc=0` (F-249). **Arrêt sinon** : V2 ne pourrait
  pas montrer le refus.
- **Opérateur hors des deux groupes** : `0`, `rc=1`. **Arrêt sinon** : A2 le fermerait
  dehors. La ligne interroge le **nom** (`id -Gn <nom>`), comme `sshd`, et non les groupes
  figés de la session en cours.
- **Membres relevés** : si un autre administrateur figure dans un de ces groupes, A2 lui
  fermera SSH. **À dire avant de poursuivre.**
- **Référence du certificat** : `0 0`. **Arrêt sinon** : V2 ne pourrait plus imputer un
  écart à ce jeu. Ce fichier appartient à `kerberos-scom-host02.md` ; il n'est ni modifié,
  ni supprimé ici.

---

## Étape R — Retour arrière, écrit avant la première écriture

**À lire maintenant, à ne jouer qu'en cas de besoin.** R1 et R2 sont **indépendants** :
un problème de supervision après A1 ne demande pas de rouvrir SSH, et inversement. R
n'emploie aucun nom.

### Avec `sudo` (session de l'opérateur)

```
echo "=== DEBUT ER1 ==="
sudo test -f /root/scom-kerberos.avant-d024; echo "rc_sauvegarde=$?  (0 attendu ; sinon ARRET, rien n'est restauré)"
sudo test -f /root/scom-kerberos.avant-d024 && sudo cp -a /root/scom-kerberos.avant-d024 /etc/sudoers.d/scom-kerberos; echo "rc_cp=$?  (0 attendu)"
sudo test -f /root/scom-kerberos.avant-d024 && sudo restorecon -v /etc/sudoers.d/scom-kerberos; echo "rc_restorecon=$?  (0 attendu)"
sudo test -f /root/scom-kerberos.avant-d024 && sudo cmp /root/scom-kerberos.avant-d024 /etc/sudoers.d/scom-kerberos; echo "rc_cmp=$?  (0 attendu)"
sudo ls -lZ /etc/sudoers.d/scom-kerberos; echo "rc_ls=$?"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
echo "=== FIN ER1 ==="
```

```
echo "=== DEBUT ER2 ==="
sudo test -f /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_present=$?  (0 si A2 a posé le fichier ; 1 : rien à retirer)"
sudo test -f /etc/ssh/sshd_config.d/10-scom-denygroups.conf && sudo rm /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_rm=$?  (0 si le fichier existait)"
sudo ls -l /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_ls=$?  (2 attendu : absent)"
sudo /usr/sbin/sshd -t && sudo systemctl reload sshd; echo "rc_t_reload=$?  (0 attendu)"
systemctl is-active sshd; echo "rc_active=$?  (0 attendu)"
sudo /usr/sbin/sshd -T | grep -c '^denygroups '; echo "rc_deny=${PIPESTATUS[1]}  (0 affiché, 1 attendu)"
echo "=== FIN ER2 ==="
```

### Sans `sudo` (seconde session, déjà `root`)

**Si `sudo` ne fonctionne plus**, `sudo` relit le fichier cassé et refuserait même à
`root` : jouer ces blocs depuis la seconde session.

```
echo "=== DEBUT ER1-ROOT ==="
id -u; echo "rc_id=$?  (0 affiché)"
test -f /root/scom-kerberos.avant-d024; echo "rc_sauvegarde=$?  (0 attendu ; sinon ARRET, rien n'est restauré)"
test -f /root/scom-kerberos.avant-d024 && cp -a /root/scom-kerberos.avant-d024 /etc/sudoers.d/scom-kerberos; echo "rc_cp=$?  (0 attendu)"
test -f /root/scom-kerberos.avant-d024 && restorecon -v /etc/sudoers.d/scom-kerberos; echo "rc_restorecon=$?  (0 attendu)"
test -f /root/scom-kerberos.avant-d024 && cmp /root/scom-kerberos.avant-d024 /etc/sudoers.d/scom-kerberos; echo "rc_cmp=$?  (0 attendu)"
ls -lZ /etc/sudoers.d/scom-kerberos; echo "rc_ls=$?"
visudo -c; echo "rc_visudo=$?  (0 attendu)"
echo "=== FIN ER1-ROOT ==="
```

```
echo "=== DEBUT ER2-ROOT ==="
id -u; echo "rc_id=$?  (0 affiché)"
test -f /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_present=$?  (0 si A2 a posé le fichier ; 1 : rien à retirer)"
test -f /etc/ssh/sshd_config.d/10-scom-denygroups.conf && rm /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_rm=$?  (0 si le fichier existait)"
ls -l /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_ls=$?  (2 attendu : absent)"
/usr/sbin/sshd -t && systemctl reload sshd; echo "rc_t_reload=$?  (0 attendu)"
systemctl is-active sshd; echo "rc_active=$?  (0 attendu)"
/usr/sbin/sshd -T | grep -c '^denygroups '; echo "rc_deny=${PIPESTATUS[1]}  (0 affiché, 1 attendu)"
echo "=== FIN ER2-ROOT ==="
```

**Attendu :** tous les codes indiqués. Si `rc_sauvegarde` n'est pas `0`, R1 ne restaure
rien : **arrêt et alerte**. R2 **ne recharge pas** `sshd` si `sshd -t` échoue. **Tout
écart : arrêt et alerte**, rien d'autre n'est tenté. Après R2, rouvrir une session SSH
de l'opérateur avant de fermer quoi que ce soit.

---

## Étape 2S — Seconde session, avant la première écriture

Ouvrir **une seconde session SSH** sur `host02`, puis, **chacune seule** :

```
sudo -s
```

```
id -u; echo "rc_id=$?  (0 affiché)"
```

**Pas `sudo -i`** : le shell de connexion de `root` est `nologin` (F-236, PO-073). **Capturer
`id -u`.** Laisser cette session ouverte **jusqu'à la fin de V2**. Garder aussi ouverte
la **session SSH de l'opérateur** pendant A2 : `reload` ne coupe pas, en principe, les
sessions établies (`@VERIF`, constaté en jouant A2 et V1).

---

## Étape SV — Sauvegarde de `scom-kerberos`

```
echo "=== DEBUT ESV ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
sudo test ! -e /root/scom-kerberos.avant-d024; echo "rc_absente=$?  (0 attendu ; sinon ARRET)"
sudo test ! -e /root/scom-kerberos.avant-d024 && sudo cp -a /etc/sudoers.d/scom-kerberos /root/scom-kerberos.avant-d024; echo "rc_cp=$?  (0 attendu)"
sudo cmp /etc/sudoers.d/scom-kerberos /root/scom-kerberos.avant-d024; echo "rc_cmp=$?  (0 attendu)"
sudo cmp /root/scom-kerberos.avant-d024 /dev/null; echo "rc_cmp_temoin=$?  (1 attendu)"
sudo stat -c '%a %U %G %C %s' /root/scom-kerberos.avant-d024; echo "rc_stat=$?  (440 root root ... 348)"
echo "=== FIN ESV ==="
```

**Arrêt** si `rc_absente`, `rc_cp` ou `rc_cmp` n'est pas `0`, ou si le témoin ne rend pas
`1` : R1 ne pourrait pas restaurer. Une sauvegarde déjà présente n'est **jamais** écrasée.

---

## Étape A1 — Retour du sudoers au contenu de D-022 (D-024)

**But :** reposer exactement les trois lignes de F-237. La garde compare le fichier de
travail aux **trois premières lignes du fichier en place** : seule la fin (ligne
`scxadmin -status` et ligne vide) peut disparaître.

```
echo "=== DEBUT EA1 ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
G_A1=KO; echo "garde_initiale=$G_A1"
N_20146=$(sudo grep -c 'EventId=20146' /var/opt/omi/log/omiserver.log); echo "rc_ref_20146=$? N_20146=$N_20146  (0 ; 2 selon F-265)"
printf '%s\n' '# scom-kerberos : regle sudo du compte AD de supervision SCOM (D-022, PO-071).' '# Pose a la main, host02 seulement ; procedures/kerberos-scom-host02.md.' "${COMPTE:?vide} ALL=(root) NOPASSWD: /opt/microsoft/scx/bin/scxlogfilereader -p" > ~/scom-kerberos-d024.sudoers; echo "rc_printf=$?  (0 attendu)"
wc -l < ~/scom-kerberos-d024.sudoers; echo "rc_wc=$?  (3 affiché)"
wc -c < ~/scom-kerberos-d024.sudoers; echo "rc_octets=$?  (232 affiché, F-237)"
grep -cv '^#' ~/scom-kerberos-d024.sudoers; echo "rc_actives=$?  (1 affiché)"
grep -cxF "${COMPTE:?vide} ALL=(root) NOPASSWD: /opt/microsoft/scx/bin/scxlogfilereader -p" ~/scom-kerberos-d024.sudoers; echo "rc_ligne=$?  (1 affiché, 0 attendu)"
grep -cxF "${COMPTE:?vide} ALL=(root) NOPASSWD: /bin/true" ~/scom-kerberos-d024.sudoers; echo "rc_ligne_temoin=$?  (0 affiché, 1 attendu)"
LC_ALL=C grep -c '[^ -~]' ~/scom-kerberos-d024.sudoers; echo "rc_non_ascii=$?  (0 affiché, 1 attendu)"
cat -A ~/scom-kerberos-d024.sudoers; echo "rc_cat=$?"
sudo head -n 3 /etc/sudoers.d/scom-kerberos | cmp - ~/scom-kerberos-d024.sudoers; echo "rc_cmp_3_lignes=${PIPESTATUS[1]}  (0 attendu)"
printf '%s\n' "${COMPTE:?vide} ALL=(root NOPASSWD: /bin/true" > ~/scom-temoin.sudoers; echo "rc_printf_temoin=$?"
sudo visudo -cf ~/scom-temoin.sudoers; RC_VT=$?; echo "rc_visudo_temoin=$RC_VT  (1 attendu)"
rm ~/scom-temoin.sudoers; echo "rc_rm_temoin=$?"
sudo visudo -cf ~/scom-kerberos-d024.sudoers; echo "rc_visudo_fichier=$?  (0 attendu)"
[ "$(printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+')" = 3 ] && [ "$RC_VT" = 1 ] && [ "$(wc -l < ~/scom-kerberos-d024.sudoers)" = 3 ] && [ "$(wc -c < ~/scom-kerberos-d024.sudoers)" = 232 ] && [ "$(grep -cv '^#' ~/scom-kerberos-d024.sudoers)" = 1 ] && [ "$(grep -cxF "${COMPTE:?vide} ALL=(root) NOPASSWD: /opt/microsoft/scx/bin/scxlogfilereader -p" ~/scom-kerberos-d024.sudoers)" = 1 ] && ! LC_ALL=C grep -q '[^ -~]' ~/scom-kerberos-d024.sudoers && [ "$(sudo stat -c %s /etc/sudoers.d/scom-kerberos)" = 348 ] && [ "$(sudo awk 'END{print NR}' /etc/sudoers.d/scom-kerberos)" = 5 ] && sudo head -n 3 /etc/sudoers.d/scom-kerberos | cmp -s - ~/scom-kerberos-d024.sudoers && sudo cmp -s /root/scom-kerberos.avant-d024 /etc/sudoers.d/scom-kerberos && sudo visudo -cf ~/scom-kerberos-d024.sudoers > /dev/null && G_A1=OK; echo "garde_a1=$G_A1  (OK attendu ; KO : rien ne sera posé)"
H_AVANT=$(sha256sum < ~/scom-kerberos-d024.sudoers); echo "rc_sha_avant=$? $H_AVANT"
[ "$G_A1" = OK ] && sudo install -m 440 -o root -g root ~/scom-kerberos-d024.sudoers /etc/sudoers.d/scom-kerberos; echo "rc_install=$?  (0 attendu)"
H_APRES=$(sudo cat /etc/sudoers.d/scom-kerberos | sha256sum); echo "rc_sha_apres=$? $H_APRES"
[ "$H_AVANT" = "$H_APRES" ]; echo "rc_egal=$?  (0 attendu)"
[ "$H_AVANT" = "${H_AVANT}x" ]; echo "rc_egal_temoin=$?  (1 attendu)"
sudo restorecon -v /etc/sudoers.d/scom-kerberos; echo "rc_restorecon=$?"
sudo ls -lZ /etc/sudoers.d/scom-kerberos; echo "rc_ls=$?"
sudo visudo -c; echo "rc_visudo_global=$?  (0 attendu)"
sudo -l -U "${COMPTE:?vide}"; echo "rc_list=$?"
sudo -l -U "${COMPTE:?vide}" /opt/microsoft/scx/bin/scxlogfilereader -p; echo "rc_U_permis=$?  (0 attendu : PERMIS)"
sudo -l -U "${COMPTE:?vide}" /opt/microsoft/scx/bin/tools/scxadmin -status; echo "rc_U_refus=$?  (1 attendu, sans message : REFUSE)"
echo "=== FIN EA1 ==="
```

**Attendu et conditions d'arrêt** — tout écart : **arrêt, puis R1** :
- **Garde `G_A1`** : `OK`, placée **avant** l'`install` (R-13, antériorité). `KO` : le
  fichier en place **n'est pas modifié**, `rc_install=1`, et les lignes suivantes
  décrivent alors l'ancien état : **arrêt, sans R1**.
- **Contenu** : 3 lignes, 232 octets, 1 ligne active trouvée une fois ; témoin de ligne
  voisine `0`, `rc=1` ; aucun caractère non ASCII ; `cat -A` propre ; `rc_cmp_3_lignes=0`.
- **Témoin `visudo`** : « syntax error », `rc=1` ; **ne pas le simplifier** (F-189, F-202).
- **Empreintes** : `rc_egal=0`, témoin `1`. Constat **après** la pose, non une prévention.
- `ls -lZ` : `-r--r-----`, `root root`, `etc_t`, 232 octets (F-237). `visudo -c` : `rc=0`.
  **Sinon R1 immédiatement**, depuis la seconde session si `sudo` refuse.
- **`sudo -l -U`** : **une seule règle**, `scxlogfilereader -p`. Toute autre règle : **arrêt**.

**Discrimination du refus (R-13)** — trois conditions, toutes requises :
1. la ligne **permise** de la même forme vient de rendre `0` ;
2. le refus rend `1` **sans aucun message** ;
3. `scxadmin -status` **n'apparaît plus** dans la liste de `sudo -l -U`.

---

## Étape A2 — Fermeture de SSH aux groupes SCOM (D-025)

**Nom du fichier : `10-scom-denygroups.conf`.**
- `.conf` : requis par la ligne `Include` attendue en 0a.
- `10-` : le place avant `50-redhat.conf`, comme demandé. **La règle de la première valeur
  n'en est pas la justification** : elle ne s'applique pas à `DenyGroups`, dont **chaque
  occurrence s'ajoute à la liste** (`man sshd_config` **du poste**, code `servconf.c` de
  `V_8_7_P1` à `V_10_2_P1`, mesuré sur le poste ; page de `host02` : `@VERIF`, 0a). Le fichier
  serait pris en compte à n'importe quel rang. Le préfixe a seulement l'avantage de le
  faire lire avant tout fichier distribué, sans dépendre de leur contenu.

**Forme de la ligne :** `DenyGroups nom1 nom2`, séparés par une espace, sans guillemets
littéraux : les noms sont de la forme `[a-z0-9_-]+`, donc sans espace ni caractère de motif.

```
echo "=== DEBUT EA2 ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
G_A2=KO; echo "garde_initiale=$G_A2"
printf '%s\n' '# scom-denygroups : SSH ferme aux groupes SCOM (D-025), host02 seulement.' "DenyGroups ${GROUPE_Q:?vide} ${GROUPE_P:?vide}" > ~/scom-denygroups.conf; echo "rc_printf=$?  (0 attendu)"
wc -l < ~/scom-denygroups.conf; echo "rc_wc=$?  (2 affiché)"
grep -cv '^#' ~/scom-denygroups.conf; echo "rc_actives=$?  (1 affiché)"
grep -cxF "DenyGroups ${GROUPE_Q:?vide} ${GROUPE_P:?vide}" ~/scom-denygroups.conf; echo "rc_ligne=$?  (1 affiché, 0 attendu)"
grep -cxF "DenyGroups ${GROUPE_Q:?vide}" ~/scom-denygroups.conf; echo "rc_ligne_temoin=$?  (0 affiché, 1 attendu)"
LC_ALL=C grep -c '[^ -~]' ~/scom-denygroups.conf; echo "rc_non_ascii=$?  (0 affiché, 1 attendu)"
cat -A ~/scom-denygroups.conf; echo "rc_cat=$?"
sudo test ! -e /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_absent=$?  (0 attendu)"
sudo /usr/sbin/sshd -t; echo "rc_t_avant=$?  (0 attendu)"
sudo cp /etc/ssh/sshd_config /root/scom-sshd-temoin.conf; echo "rc_cp_temoin=$?  (0 attendu)"
printf '%s\n' 'ScomMotCleInconnu oui' | sudo tee -a /root/scom-sshd-temoin.conf > /dev/null; echo "rc_tee_temoin=${PIPESTATUS[1]}  (0 attendu)"
sudo /usr/sbin/sshd -t -f /root/scom-sshd-temoin.conf; RC_ST=$?; echo "rc_t_temoin=$RC_ST  (255 attendu, message nommant ScomMotCleInconnu)"
sudo rm /root/scom-sshd-temoin.conf; echo "rc_rm_temoin=$?  (0 attendu)"
[ "$(printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+')" = 3 ] && [ "$RC_ST" = 255 ] && [ "$(wc -l < ~/scom-denygroups.conf)" = 2 ] && [ "$(grep -cv '^#' ~/scom-denygroups.conf)" = 1 ] && [ "$(grep -cxF "DenyGroups ${GROUPE_Q:?vide} ${GROUPE_P:?vide}" ~/scom-denygroups.conf)" = 1 ] && ! LC_ALL=C grep -q '[^ -~]' ~/scom-denygroups.conf && sudo test ! -e /etc/ssh/sshd_config.d/10-scom-denygroups.conf && sudo /usr/sbin/sshd -t && G_A2=OK; echo "garde_a2=$G_A2  (OK attendu ; KO : rien ne sera posé)"
[ "$G_A2" = OK ] && sudo install -m 600 -o root -g root ~/scom-denygroups.conf /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_install=$?  (0 attendu)"
sudo restorecon -v /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_restorecon=$?"
sudo ls -lZ /etc/ssh/sshd_config.d/10-scom-denygroups.conf; echo "rc_ls=$?"
sudo /usr/sbin/sshd -t; echo "rc_t_apres=$?  (0 attendu)"
systemctl show -p ActiveEnterTimestamp sshd; echo "rc_show_avant=$?"
N_RELOAD=$(sudo awk 'END{print NR}' /var/log/secure); echo "rc_ref_reload=$? N_RELOAD=$N_RELOAD"
[ "$G_A2" = OK ] && sudo /usr/sbin/sshd -t && sudo systemctl reload sshd; echo "rc_reload=$?  (0 attendu)"
sleep 2; echo "rc_sleep=$?"
sudo tail -n "+$((N_RELOAD+1))" /var/log/secure | grep -E 'sshd(-session)?\[' | grep -c 'Received SIGHUP; restarting'; echo "rc_sighup=${PIPESTATUS[2]}  (1 affiché, 0 attendu)"
systemctl is-active sshd; echo "rc_active=$?  (0 attendu)"
systemctl show -p ActiveEnterTimestamp sshd; echo "rc_show_apres=$?  (inchangée : relevé)"
sudo /usr/sbin/sshd -T | grep -E '^denygroups '; echo "rc_T=${PIPESTATUS[0]}  (2 lignes, 0 attendu)"
sudo /usr/sbin/sshd -T -C "user=${COMPTE:?vide},host=localhost,addr=127.0.0.1" | grep -E '^denygroups '; echo "rc_T_C=${PIPESTATUS[0]}  (2 lignes, 0 attendu)"
echo "=== FIN EA2 ==="
```

**Attendu et conditions d'arrêt** — tout écart : **arrêt, puis R2** :
- **Garde `G_A2`** : `OK`, placée **avant** la pose et le rechargement. `KO` : rien n'est
  posé, **arrêt, sans R2**.
- **Témoin `sshd -t -f`** : `rc=255` **et** un message « Bad configuration option:
  ScomMotCleInconnu ». Un autre message est un **échec mal construit** (R-13) : arrêt.
- `ls -lZ` : `-rw-------`, `root root`, contexte `etc_t` attendu `@VERIF` : **relever**.
- **Le rechargement se prouve par le journal**, ligne « Received SIGHUP; restarting. »
  (`sshd.c`, § *Sources*). **Ni `sshd -T`, ni `ActiveEnterTimestamp` ne le prouvent** :
  `sshd -T` relit le **disque**, et l'horodatage ne change pas à un `reload` (mesuré sur une
  unité utilisateur du poste). Une date inchangée est donc **normale**.
- `sshd -T` : **exactement deux** lignes `denygroups`, une par groupe. Forme `-C` : les
  mêmes. **Si `rc_sighup` n'est pas `0`, le refus n'est pas en vigueur** : arrêt, R2.

---

## Étape V1 — Nouvelle session SSH de l'opérateur

**Depuis le poste de l'opérateur, sans fermer aucune session**, ouvrir une **nouvelle**
connexion SSH vers `host02`, puis :

```
id -un; echo "rc_id=$?  (0 attendu)"
```

**Elle doit aboutir.** **Sinon : R2 depuis la seconde session**, puis retester.

---

## Étape V2 — Refus du compte, témoin, sudo, certificat

**Le compte de service ne présente aucun mot de passe** (`PreferredAuthentications=none`).
**Les deux connexions échouent, avec le même code `255`** : c'est **le message de `sshd`**
qui les distingue (R-13). `StrictHostKeyChecking=no` et `UserKnownHostsFile=/dev/null` sont
**nécessaires** : sans eux, `BatchMode` échoue sur la clé d'hôte **avant toute demande
d'authentification**, et aucun refus n'est produit (mesuré). `-F none` écarte la
configuration personnelle de l'opérateur.

```
echo "=== DEBUT EV2 ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
date -u; echo "rc_date=$?  (repere UTC pour V3)"
N_COMPTE=$(sudo awk 'END{print NR}' /var/log/secure); echo "rc_ref_compte=$? N_COMPTE=$N_COMPTE"
ssh -F none -o BatchMode=yes -o PreferredAuthentications=none -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 -l "${COMPTE:?vide}" localhost true; echo "rc_ssh_compte=$?  (255 attendu)"
sleep 2; echo "rc_sleep=$?"
sudo tail -n "+$((N_COMPTE+1))" /var/log/secure | grep -E 'sshd(-session)?\['; echo "rc_lignes_compte=${PIPESTATUS[1]}  (0 attendu)"
sudo tail -n "+$((N_COMPTE+1))" /var/log/secure | grep -E 'sshd(-session)?\[' | grep -F "User ${COMPTE:?vide} from " | grep -c 'not allowed because a group is listed in DenyGroups'; echo "rc_refus_compte=${PIPESTATUS[3]}  (1 affiché, 0 attendu)"
N_TEMOIN=$(sudo awk 'END{print NR}' /var/log/secure); echo "rc_ref_temoin=$? N_TEMOIN=$N_TEMOIN"
ssh -F none -o BatchMode=yes -o PreferredAuthentications=none -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 -l "$(id -un)" localhost true; echo "rc_ssh_temoin=$?  (255 attendu)"
sleep 2; echo "rc_sleep=$?"
sudo tail -n "+$((N_TEMOIN+1))" /var/log/secure | grep -E 'sshd(-session)?\[' | grep -F "$(id -un)"; echo "rc_temoin_vu=${PIPESTATUS[2]}  (0 attendu : sshd a vu la connexion)"
sudo tail -n "+$((N_TEMOIN+1))" /var/log/secure | grep -E 'sshd(-session)?\[' | grep -c 'DenyGroups'; echo "rc_refus_temoin=${PIPESTATUS[2]}  (0 affiché, 1 attendu)"
sudo -l -U "${COMPTE:?vide}"; echo "rc_list=$?  (une seule regle)"
sudo openssl x509 -in /etc/opt/omi/ssl/omi.pem -noout -fingerprint -sha256 | diff ~/scom-kerberos-avant.fp -; echo "rc_openssl=${PIPESTATUS[0]} rc_diff=${PIPESTATUS[1]}  (0 0 attendus)"
diff ~/scom-kerberos-avant.fp /dev/null > /dev/null; echo "rc_diff_temoin=$?  (1 attendu)"
sudo grep -c 'EventId=20146' /var/opt/omi/log/omiserver.log; echo "rc_20146=$?  (affiché = ${N_20146:?vide})"
echo "=== FIN EV2 ==="
```

**Attendu et lecture :**
- **Compte** : `rc_ssh_compte=255` ; une ligne « User … from … not allowed because a group
  is listed in DenyGroups » ; `1`, `rc=0`. **Sans cette ligne, le refus n'est pas démontré** :
  arrêt, relever. **Si le message réel diffère de la source, le consigner tel quel.**
- **Témoin** : `rc_ssh_temoin=255`, **au moins une** ligne de `sshd` nommant l'opérateur
  (sur le poste : « Connection closed by authenticating user … [preauth] », forme non établie
  pour `host02`, `@VERIF`), et **aucune** ligne `DenyGroups` (`0`, `rc=1`). **Si `sshd` n'a
  rien journalisé, le témoin est mal construit** : il ne prouve rien, on ne l'interprète pas.
- `sudo -l -U` : **une seule règle**. Certificat : `0 0`, témoin `1`. **Écart : arrêt et
  alerte**, ne rien corriger.
- `EventId=20146` : égal à `N_20146`. **Inférieur** : le journal a pu tourner (F-259),
  **non concluant**. **Supérieur** : un nouveau refus, à dire en tête du rapport.

---

## Étape V3 — Après 15 à 30 minutes : collecte et état SCOM

**Ne pas fermer la session.** Attendre au moins 15 minutes après A1 : les connexions de SCOM
sont espacées d'environ 15 minutes (F-263, `[PILOTE-DÉCLARÉ]`).

```
echo "=== DEBUT EV3 ==="
printf '%s\n%s\n%s\n' "${COMPTE:?vide}" "${GROUPE_Q:?vide}" "${GROUPE_P:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (3 affiché, 0 attendu)"
date -u; echo "rc_date=$?"
sudo tail -n 8 /var/opt/omi/log/omiserver.log; echo "rc_tail=$?  (0 attendu)"
sudo grep -c 'EventId=20146' /var/opt/omi/log/omiserver.log; echo "rc_20146=$?  (affiché = ${N_20146:?vide})"
echo "=== FIN EV3 ==="
```

**Attendu :** au moins une ligne **postérieure au repère UTC de V2** (`omiserver.log` est en
UTC, F-254), de la forme relevée en F-263 ; `EventId=20146` inchangé. **Puis faire confirmer
par l'équipe SCOM** `[TIERS-DÉCLARÉ]` que l'hôte est toujours *Healthy*.

**Portée de cette preuve (F-281) :** elle montre que la **collecte périodique** se poursuit
sans la ligne retirée. Elle **ne couvre pas** le chemin privilégié, non observé (F-272,
PO-059), ni une tâche SCOM lancée à la demande qui appellerait `scxadmin` par `sudo`. Un
écart : **décision de l'opérateur**, R1 disponible.

---

## Étape N — Nettoyage, après validation

**Pas avant** la confirmation de V3 : la sauvegarde sert à R1. **Le fichier
`~/scom-kerberos-avant.fp` n'est pas supprimé ici** : il relève de l'étape N de
`kerberos-scom-host02.md` (D-023).

```
echo "=== DEBUT EN ==="
rm ~/scom-kerberos-d024.sudoers ~/scom-denygroups.conf; echo "rc_rm=$?  (0 attendu)"
sudo rm /root/scom-kerberos.avant-d024; echo "rc_rm_sauvegarde=$?  (0 attendu)"
ls -la ~/scom-kerberos-d024.sudoers ~/scom-denygroups.conf; echo "rc_ls=$?  (2 attendu)"
sudo ls -la /root/scom-kerberos.avant-d024; echo "rc_ls_sauvegarde=$?  (2 attendu)"
echo "=== FIN EN ==="
```

`rm` sans `-f` : un fichier déjà absent se signale.

---

## Ce que l'opérateur rapporte au pilote

Chaque bloc **entier**, marqueurs compris, en `[ÉCRAN-AAAA-MM-JJ]` (D-003), et en
particulier : les versions de 0a ; les lignes `sshd -T` ; `Include` et le contenu de
`sshd_config.d` ; l'état de `scom-kerberos` en 0b ; `id -u` de la seconde session ;
`garde_a1`, `garde_a2` ; le témoin `sshd -t -f` et son message ; `rc_sighup` ; les lignes de
refus et du témoin en V2 ; V3 ; la réponse de l'équipe SCOM, en `[TIERS-DÉCLARÉ]` ; toute
branche de R jouée.

**Aucune valeur réelle n'est à recopier dans le dépôt** — ni nom de compte, de groupe ou de
machine, ni adresse, ni empreinte (R-07).

---

## Gardes, branches et ce qui reste non démontré (R-04, R-12)

| Contrôle | Placé avant l'effet ? | Sens de l'échec |
|---|---|---|
| arrêt par `${…:?}` | oui, sur chaque ligne qui emploie un nom | témoin joué en S ; mesuré sur le poste |
| forme des noms, `rc_vars` | oui ; repris dans `G_A1`, `G_A2` | témoin `x,y` joué en S |
| version, `Include`, `Match`, directives existantes (0a) | oui, avant A2 | **écrit, non démontré** sur `host02` |
| état de `scom-kerberos` (0b) | oui, repris dans `G_A1` (348 octets, 5 lignes) | **écrit, non démontré** sur `host02` ; `KO` exercé en bac à sable |
| nom exact des groupes (0c) | oui, avant A2 | effet d'un nom à la mauvaise casse **mesuré** sur le poste : règle inerte |
| opérateur hors des groupes (0c) | oui, avant A2 | **écrit, non démontré** — le forcer exigerait de l'ajouter à un groupe |
| sauvegarde absente (SV) | oui, avant `cp` | exercé en bac à sable |
| `cmp` de la sauvegarde (SV) | oui, avant A1 | témoin `/dev/null`, joué en SV |
| contenu du sudoers, `G_A1` | oui, avant `install` | témoins de ligne voisine et `visudo` joués en A1 ; `KO` exercé en bac à sable |
| empreinte avant / après pose (A1) | **non** — constat | témoin `${H_AVANT}x`, joué en A1 |
| refus `sudo` de `scxadmin -status` (A1) | constat après la pose | **démontré par construction** sous les trois conditions de R-13 |
| contenu du fichier `sshd`, `G_A2` | oui, avant pose et rechargement | témoin de ligne voisine joué en A2 ; `KO` exercé en bac à sable |
| `sshd -t` (A2) | oui, avant `reload` | témoin à mot-clé inconnu, joué en A2 ; mesuré sur le poste (`rc=255`) |
| preuve du rechargement (A2) | **non** — constat | **écrit, non démontré** dans le sens de l'échec |
| nouvelle session de l'opérateur (V1) | avant la fermeture des sessions | R2 disponible |
| refus SSH du compte (V2) | **non** — constat | **démontré** par le message, contre le témoin de l'opérateur (R-13, R-14) |
| certificat, `EventId=20146` (V2, V3) | **non** — constats | témoin `/dev/null` pour le certificat ; compteur : **non forcé** |

**Branches non exercées par un jeu nominal :** R1, R2 et leurs variantes `root` ; le refus
d'un membre de `<GROUPE-SCOM-P>`, **non démontrable** s'il n'a pas de membre sur `host02` ;
tout cas d'arrêt de 0a à 0c.

**Limite connue de `DenyGroups` :** `sshd` lit les groupes de l'utilisateur **à la connexion**
par `getgrouplist` puis `getgrgid`, et **ignore** un groupe dont le nom ne se résout pas
(`groupaccess.c:65-85`, 10.2p1). Si SSSD ne résout plus le groupe, la règle ne s'applique
pas à ce membre : **elle n'est pas une protection en panne de résolution**.

---

## Sources lues pour cette procédure

Lues le **2026-11-23** sur le poste de rédaction, sans élévation ; téléchargements et essais
dans le répertoire temporaire de la session, hors dépôt. **Elles ne sont pas encore au
dossier** : à verser par une consignation ultérieure (R-15). Elles valent d'ici là comme
citations nommées `[SOURCE-EXTERNE]`, avec la réserve de transposition de F-184 et F-195.

- **`man sshd_config`** — `openssh-server-10.2p1-16.fc44`, page datée du 4 octobre 2025,
  `MANWIDTH=100` :
  - l. 8-9 : « Unless noted otherwise, for each keyword, the first obtained value will be
    used. » ;
  - l. 318-326, `DenyGroups` : « a list of group name patterns, separated by spaces » ;
    « Only group names are valid; a numerical group ID is not recognized » ; ordre
    « DenyGroups, AllowGroups » ; « **This keyword may appear multiple times in sshd_config
    with each instance appending to the list.** » ;
  - l. 335, ordre « DenyUsers, AllowUsers » ;
  - l. 547-551, `Include` : motifs développés et traités « in lexical order ».
- **`man sshd`** — même paquet, page datée du 4 octobre 2025 : `-C` (l. 30), `-f` (l. 59),
  `-T` (l. 95), `-t` (l. 100).
- **Code amont `openssh/openssh-portable`**, étiquettes **`V_10_2_P1`**, **`V_9_9_P1`**,
  **`V_8_7_P1`** (`http_code=200` pour chaque fichier). **Les correctifs de Fedora et de Red
  Hat n'ont pas été lus.**
  - `auth.c`, `allowed_user` : message « User %.100s from %.100s not allowed because a group
    is listed in DenyGroups » — l. 193 (10.2), 195 (9.9), 241 (8.7) ;
  - `servconf.c`, `parse_allowdenygroups` : `opt_array_append`, dans les trois versions ;
  - `groupaccess.c` : `match_pattern` sur le nom du groupe, l. 102 (10.2), 91 (9.9, 8.7) ;
  - `sshd.c` : « Received SIGHUP; restarting. » — l. 517 (10.2), 493 (9.9), 309 (8.7).
- **Mesures sur le poste**, `sshd` 10.2p1 **non privilégié**, écoute sur `127.0.0.1:22987`,
  configuration et clé d'hôte temporaires, processus limité à quelques secondes :
  - trois groupes répartis sur deux fichiers inclus, avec un `Match` dans un fichier
    antérieur : `sshd -T` et `sshd -T -C` affichent **les trois**, un par ligne ;
  - mot-clé inconnu : « Bad configuration option: ScomMotCleInconnu », `rc=255` ;
  - utilisateur dans un groupe refusé : `ssh` `rc=255`, journal « User … not allowed because
    a group is listed in DenyGroups » ;
  - témoin dans un autre groupe : `ssh` `rc=255`, journal « Connection closed by
    authenticating user … [preauth] », **aucune** ligne `DenyGroups` ;
  - sans les options de clé d'hôte : « Host key verification failed », **aucun** refus ;
  - nom du groupe en **majuscules** : **pas de refus** ;
  - unité `systemd --user` : `ActiveEnterTimestampMonotonic` **inchangé** après `reload`.
  *Limite :* ce `sshd` tournait avec `UsePAM no` (avertissement de ce paquet), `host02` avec
  `usepam yes` (F-197). Le refus lui-même y est donc à constater en V2.
- **Bac à sable sur le poste**, noms fictifs, chemins redirigés vers un répertoire
  temporaire ; `sudo`, `systemctl`, `install`, `restorecon`, `cp` et le client `ssh` simulés,
  `sshd -t`, `sshd -T` et `visudo -cf` réels :
  - enchaînement nominal SV, A1, A2, V2 : `garde_a1=OK`, `garde_a2=OK`, 232 octets posés ;
  - `G_A1` à `KO` sur une ligne altérée à taille égale, puis sur une taille différente, et
    `G_A2` à `KO` sur un fichier déjà présent : **rien n'est posé ni rechargé** ;
  - SV avec une sauvegarde déjà présente : **non écrasée** ;
  - R1 sans sauvegarde : **rien n'est restauré** ;
  - R1 et R2, avec et sans `sudo` : fichier d'origine rétabli, fichier `sshd` retiré.
  *Limites :* `visudo -c` **global** n'a pas pu s'y exercer (`rc=1`, `/etc/sudoers` non
  lisible sans privilège) ; ce bac à sable éprouve l'**enchaînement du shell**, pas `sudo`,
  SELinux, `systemd` ni le `sshd` en service de `host02`.

**Ce qui reste `@VERIF` :**
- le comportement de `DenyGroups`, `Include` et `reload` **sur la version de `host02`**,
  relevée en 0a et constaté en A2 et V2 ;
- la présence de `50-redhat.conf` et le contexte SELinux du fichier posé ;
- que les sessions établies survivent au `reload` sur `host02` ;
- la forme exacte de la ligne de `sshd` pour le témoin sur `host02` ;
- que la ligne « Received SIGHUP » arrive dans `/var/log/secure` sur `host02`.
