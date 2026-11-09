# Procédure — compte de supervision Kerberos sur `host02` : règle `sudo` et accès SSSD

**Ce que ce fichier porte :** des blocs de commandes que **l'opérateur** joue lui-même
sur `host02`, sous **son compte AD avec `sudo`**, dans l'ordre, et la lecture de chaque
sortie. Objet unique, sur `host02` seulement :
1. poser la règle `sudo` du **compte AD de supervision** `<COMPTE-SCOM-Q>` (D-022, D-023) ;
2. admettre son groupe `<GROUPE-SCOM-Q>` dans l'accès SSSD, par `simple_allow_groups`
   (D-022, PO-071).

**Ce que ce fichier NE porte PAS :** aucun fait, aucune décision, aucun point ouvert —
ils vivent dans `../scomm-certificat-facts.md`. Il n'a **aucune autorité propre** : tout
ce qu'il affirme renvoie au dossier (`F-xxx`, `PO-xxx`, `D-xxx`), ou aux sources lues
pour sa rédaction, citées au § *Sources*. Ce qui n'est établi ni par l'un ni par l'autre
porte `@VERIF`.

**Hors périmètre — aucune ligne de ce fichier n'y touche :**
- le **SPN et le keytab** : aucune écriture, aucun `adcli` (PO-070, F-222) ;
- la **suppression** des comptes locaux `scom_adm`, `scom_mon` et de
  `/etc/sudoers.d/scom-test` : plus tard, par une procédure distincte (D-023) ;
- la **configuration SSSD du parc** : elle relève de `rhel_post_install` (R-09, F-220) ;
  ce qui est fait ici vaut pour `host02` seulement ;
- la **découverte**, faite par l'équipe SCOM (D-020).

Elle ne touche **ni** au certificat, **ni** à `omiserver.conf`, **ni** à `sshd`, **ni**
au pare-feu, **ni** au rôle, et ne lance aucun outil qui fabrique un certificat (R-10).

> **Aucune commande de ce fichier n'a été exécutée par l'agent qui l'a écrite** (R-08).
> Ce qui l'a été, sur le poste de rédaction et avec un nom de compte fictif : les formes
> de contrôle d'A1 (`printf`, `wc`, `grep -cxF`, `grep -c '[^ -~]'`, `cat -A`,
> `visudo -cf` et son témoin, comparaison d'empreintes et son témoin), le contrôle de
> forme des noms et l'arrêt par `${…:?}` (§ *Sources*).
> **Réserve de transposition :** les pages et le code lus sont ceux de Fedora 44 ou de
> l'amont, non ceux de `host02` — même réserve qu'en F-184 et F-195. Les versions de
> `host02` sont relevées à l'étape 0c.

**R-07 :** aucun nom réel n'apparaît ici. Le nom du compte et celui du groupe sont
**saisis** en début de session (étape S) et ne figurent dans aucune commande écrite. Les
sorties affichées en contiennent : **rien n'en est recopié dans le dépôt.**

## Ordre de jeu

| # | Étape | Écrit sur `host02` ? | Un échec arrête-t-il la suite ? |
|---|---|---|---|
| S | Session : témoins, saisie des deux noms | non | **Oui** |
| 0a | Lecture seule : compte, groupe, SSSD | non | **Oui** |
| 0b | Lecture seule : sudoers, omi, PAM | non | **Oui** pour `sudoers.d` et la sauvegarde |
| 0c | Lecture seule : outils, empreinte « avant », état PAM du compte | `~/scom-kerberos-avant.fp` | **Oui** pour l'empreinte |
| R | **Retour arrière — à lire avant A, à jouer seulement si besoin** | — | — |
| SV | Sauvegarde de `sssd.conf` | `/root/sssd.conf.avant-scom-kerberos` | **Oui** |
| A1 | Règle `sudo` du compte | `~/scom-kerberos.sudoers`, `/etc/sudoers.d/scom-kerberos` | **Oui** → R |
| A2 | Admission du groupe dans SSSD | `/etc/sssd/sssd.conf`, redémarrage de `sssd` | **Oui** → R |
| V | Vérifications, sans rien exécuter | non | **Oui** → R, sauf ce qui est déclaré non concluant |
| N | Nettoyage — **après validation de Kerberos** (D-023) | suppressions | — |

**Toute la procédure se joue dans une seule session** : les deux noms y sont des
variables du shell, perdues à la déconnexion (PO-039). Une session perdue reprend à S.

## Forme des commandes

**Une commande par ligne**, suivie sur la même ligne de l'affichage de son code de
retour. Aucune continuation, aucun *heredoc*, aucune valeur à remplacer à la main :
chaque ligne se colle seule (`comptes-scom-host02.md`, « Forme des commandes »). Les
deux saisies de S sont interactives et isolées. Chaque bloc est encadré de marqueurs
`DEBUT` / `FIN` : une capture sans ses deux marqueurs est incomplète (R-08).

**Arrêt par variable vide.** Toute ligne qui emploie un des deux noms l'écrit
`"${COMPTE:?vide}"` ou `"${GROUPE:?vide}"`. Si la variable est vide ou absente, un shell
**interactif** n'exécute pas la commande et affiche `bash: COMPTE: vide` (`man bash`,
§ *Sources*). **Mesuré sur le poste :** c'est **toute la ligne** qui est abandonnée, y
compris le `echo "rc_…"` qui la suit — la ligne suivante, elle, s'exécute. **Une ligne
`rc_…` absente, avec ce message, vaut arrêt.** Ce comportement est démontré sur
`host02` même, à l'étape S.

**Chaque bloc commence par la ligne `rc_vars`**, qui vérifie que les deux noms sont non
vides **et** de la forme `[a-z0-9_-]+` : noms courts en minuscules (F-219, F-221), sans
aucun des caractères que `sudoers` impose d'échapper dans un nom (`!`, `=`, `:`, `,`,
`(`, `)`, `\`), ni des caractères que `realm` refuse dans une liste (`,`, `$`)
(§ *Sources*). **Seule exception : R**, qui n'emploie aucun des deux noms, pour rester
jouable depuis la seconde session ouverte avant A1, où ils ne sont pas définis.

**Élévations.** Toutes les lignes `sudo` sont des élévations. Là où une voie sans
privilège est plausible, elle est tentée d'abord (R-06) : `realm list`, `getent`, `id`,
`/etc/pam.d/*`, `omiserver.conf`, `/etc/sudoers.d/`.

---

## Étape S — Session : témoins, puis saisie des deux noms

**But :** montrer **sur `host02`** que l'arrêt par variable vide et le contrôle de forme
savent échouer (R-12), puis saisir les deux noms.

```
echo "=== DEBUT ES ==="
unset SCOM_TEMOIN_VIDE; echo "rc_unset=$?"
echo "${SCOM_TEMOIN_VIDE:?vide}"; echo "rc_temoin_vide=$?  (cette ligne NE DOIT PAS s'afficher)"
printf '%s\n' 'x,y' | grep -cxE '[a-z0-9_-]+'; echo "rc_forme_temoin=${PIPESTATUS[1]}  (0 affiché, 1 attendu)"
echo "=== FIN ES ==="
```

Puis, **chacune seule, interactive** — taper le nom court, en minuscules :

```
read -r -p 'Compte de supervision (nom court, minuscules) : ' COMPTE
```

```
read -r -p 'Groupe de supervision (nom court, minuscules) : ' GROUPE
```

Puis :

```
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
```

**Attendu et conditions d'arrêt :**
- témoin vide : le message `bash: SCOM_TEMOIN_VIDE: vide` et **aucune** ligne
  `rc_temoin_vide`. **Arrêt** si la ligne `rc_temoin_vide` s'affiche : ce shell exécute
  la commande malgré la variable vide, et aucune ligne de cette procédure n'est protégée.
  Version de `bash` de `host02` : relevée en 0c ; le comportement a été mesuré sur
  `bash` 5.3.9 (§ *Sources*).
- témoin de forme : `0`, et `rc_forme_temoin=1`. **Arrêt sinon.**
- `rc_vars` : `2` affiché et `rc_vars=0`. **Arrêt** si la ligne est absente, si le
  compte n'est pas `2`, ou si le code n'est pas `0` : reprendre les deux `read`.

---

## Étape 0a — Lecture seule : compte, groupe, SSSD

```
echo "=== DEBUT E0A ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
getent passwd "${COMPTE:?vide}"; echo "rc_getent_compte=$?  (0 attendu)"
id "${COMPTE:?vide}"; echo "rc_id_compte=$?  (0 attendu)"
getent group "${GROUPE:?vide}"; echo "rc_getent_groupe=$?  (0 attendu)"
realm list; echo "rc_realm=$?  (0 attendu)"
sudo grep -nE '^[[:space:]]*(access_provider|simple_allow_groups|use_fully_qualified_names)[[:space:]]*=' /etc/sssd/sssd.conf; echo "rc_grep_sssd=$?  (0 attendu)"
sudo stat -c '%a %U %G %C' /etc/sssd/sssd.conf; echo "rc_stat_sssd=$?  (0 attendu)"
systemctl show -p ActiveEnterTimestamp sssd; echo "rc_show_sssd=$?  (0 attendu)"
systemctl is-active sssd; echo "rc_active_sssd=$?  (0 attendu)"
echo "=== FIN E0A ==="
```

**Attendu et conditions d'arrêt :**

| Ligne | Attendu | Sinon |
|---|---|---|
| `rc_vars` | `2`, `0` | **Arrêt** (voir S). |
| `getent passwd` | une ligne : nom court, uid = gid, `/home/…`, `/bin/bash` (F-221) ; `rc=0` | **Arrêt** si `rc=2` (« key could not be found », F-055) : compte non résolu, rien de ce qui suit n'aurait d'objet. |
| `id` | `rc=0` ; **dire si `<GROUPE-SCOM-Q>` figure parmi les groupes** | Groupe absent : **signaler, sans arrêter** — c'est l'état connu (F-221). A2 reste jouable ; le contrôle d'accès de V **ne pourra pas conclure**. |
| `getent group` | une ligne, `rc=0` ; dire si des membres sont listés | **Arrêt** si `rc=2` : A2 admettrait un nom que SSSD ne résout pas. |
| `realm list` | `<REALM>`, `kerberos-member`, client `sssd`, `permitted-groups` = `<GROUPES-ADMIN-LINUX>` (F-213) | **Arrêt** si `rc` n'est pas `0` : `realm permit` d'A2 passe par le même service. Un écart de contenu avec F-213 : **relever et arrêter** — le dossier serait périmé. |
| lignes de `sssd.conf` | `access_provider = simple` ; une ligne `simple_allow_groups` ; **deux** lignes `use_fully_qualified_names` (F-219) | **Arrêt** si `access_provider` n'est pas `simple`, ou si `simple_allow_groups` manque : A2 ne ferait pas ce que décrit cette procédure. **Recopier ces lignes** au rapport, numéros compris : A2 s'y compare. |
| `stat` | **relevé** — mode, propriétaire, groupe, contexte SELinux. Le brief de rédaction annonce `600 root root` ; **non relevé au dossier** | — ; **recopier la ligne** : SV, A2 et R s'y comparent. |
| `ActiveEnterTimestamp`, `is-active` | une date ; `active`, `rc=0` | **Arrêt** si `sssd` n'est pas actif. **Recopier la date** : A2 s'y compare. |

---

## Étape 0b — Lecture seule : sudoers, omi, PAM

```
echo "=== DEBUT E0B ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
ls -l /etc/sudoers.d/; echo "rc_ls_sudoersd=$?"
sudo ls -l /root/sssd.conf.avant-scom-kerberos; echo "rc_ls_sauvegarde=$?  (2 attendu)"
ls -l /etc/opt/omi/conf/omiserver.conf; echo "rc_ls_omiconf=$?"
grep -nE '^[[:space:]]*(pemfile|keyfile|httpsport)[[:space:]]*=' /etc/opt/omi/conf/omiserver.conf; echo "rc_grep_omiconf=$?"
cat /etc/pam.d/omi; echo "rc_cat_pam_omi=$?  (0 attendu)"
cat /etc/pam.d/sshd; echo "rc_cat_pam_sshd=$?  (0 attendu)"
echo "=== FIN E0B ==="
```

Si `rc_ls_sudoersd` n'est pas `0` (répertoire non lisible sans privilège), rejouer
seulement :

```
sudo ls -l /etc/sudoers.d/; echo "rc_sudo_ls_sudoersd=$?"
```

**Le chemin `/etc/opt/omi/conf/omiserver.conf` n'est pas au dossier** — @VERIF : le
dossier ne connaît que `omiserver.conf` (F-150, F-151) et le répertoire `/etc/opt/omi`
(F-040). Si `rc_ls_omiconf` n'est pas `0`, le localiser, en lecture seule :

```
sudo find /etc/opt/omi -name omiserver.conf; echo "rc_find=$?"
```

puis rejouer la ligne `grep` sur le chemin trouvé, **tapé à la main**, et le dire au
rapport.

**Attendu et conditions d'arrêt :**

| Ligne | Attendu | Sinon |
|---|---|---|
| `/etc/sudoers.d/` | `scom-test` présent (D-023) ; **pas** de `scom-kerberos` | **Arrêt** si `scom-kerberos` existe déjà : on ne pose pas par-dessus un fichier qu'on n'a pas lu. |
| sauvegarde | absente, `rc=2` | **Arrêt** si elle existe : SV l'écraserait, et R restaurerait un état inconnu. |
| `omiserver.conf` | **relevé, pas critère** : les lignes `pemfile`, `keyfile`, `httpsport` | — ; lève le `@VERIF` de F-216. `httpsport` doit porter `1270` (F-151) : un écart se dit en tête du rapport, sans arrêt — cette procédure n'y touche pas. |
| `/etc/pam.d/omi` | **relevé** ; comparer à F-180 | — ; un écart se dit en tête du rapport (PO-071). |
| `/etc/pam.d/sshd` | **relevé** | — ; lève le `@VERIF` de F-194 et F-197. |

---

## Étape 0c — Outils, empreinte « avant », état PAM du compte

```
echo "=== DEBUT E0C ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
rpm -q bash sudo realmd sssd-tools; echo "rc_rpm=$?"
grep -rn 'sssd-restart-service' /usr/lib/realmd/; echo "rc_grep_realmd=$?"
sssctl user-checks --help; echo "rc_help=$?"
sudo openssl x509 -in /etc/opt/omi/ssl/omi.pem -noout -fingerprint -sha256 | tee ~/scom-kerberos-avant.fp; echo "rc_openssl=${PIPESTATUS[0]} rc_tee=${PIPESTATUS[1]}  (0 0 attendus)"
sudo env LC_ALL=C sssctl user-checks -a acct -s omi "$(id -un)"; echo "rc_uc_temoin=$?  (ne discrimine pas)"
sudo env LC_ALL=C sssctl user-checks -a acct -s omi "${COMPTE:?vide}"; echo "rc_uc_compte=$?  (ne discrimine pas)"
echo "=== FIN E0C ==="
```

**Pourquoi `env LC_ALL=C` :** le verdict de `user-checks` est un message de
`pam_strerror`, traduisible selon la langue de la session. Il est forcé en anglais pour
être comparé aux libellés ci-dessous (§ *Sources*). `env` est passé comme **commande**
à `sudo`, et non comme affectation devant la commande.

**Le code de retour de `sssctl user-checks` ne dit rien** : le code amont se termine
par `ret = EOK` quel que soit le résultat PAM (§ *Sources*). **Le verdict est la ligne
`pam_acct_mgmt: …`**, et elle seule. C'est la famille de F-054 : un `rc=0` qui ne porte
pas le résultat.

**Attendu et conditions d'arrêt :**
- `rpm -q` : quatre versions — **relevé** ; elles fixent la portée de la réserve de
  transposition. Si `sssd-tools` manque : **les lignes `user-checks` de 0c et de V ne
  sont pas jouées** (branche déclarée au rapport) ; rien n'est installé ici. Le reste
  de la procédure reste jouable.
- `grep` dans `/usr/lib/realmd/` : une ligne `sssd-restart-service = … systemctl restart
  sssd.service`, comme sur le poste. **Relevé, pas critère** : si elle manque ou
  diffère, ce que fait `realm permit` après l'écriture n'est pas établi pour `host02` ;
  A2 le **mesure** par l'horodatage de `sssd`.
- `--help` : il doit lister `-a` / `--action` et `-s` / `--service`. **Sinon, les lignes
  `user-checks` de 0c et de V ne sont pas jouées** : leur syntaxe vient du code amont
  (§ *Sources*), et c'est cette aide qui la confirme **sur la cible**.
- Empreinte : une ligne `sha256 Fingerprint=…`, `0 0`. **Arrêt** si `rc_openssl` n'est
  pas `0` : V ne pourrait rien comparer. L'empreinte se lit **par
  `openssl x509 -fingerprint` et par rien d'autre** (F-209). Elle **n'est recopiée ni
  ici, ni au dossier** (R-07, F-174).
- `user-checks` du **témoin** — le compte de l'opérateur, qui ouvre des sessions sur
  `host02` (F-060) et relève donc de `<GROUPES-ADMIN-LINUX>` : `pam_acct_mgmt: Success`.
  **Si ce n'est pas `Success`, le contrôle ne discrimine pas** : les lignes
  `user-checks` du compte, ici et en V, sont **non concluantes** — le dire, sans arrêter
  A1 ni A2.
- `user-checks` du **compte** : **refus attendu** (PO-071), `pam_acct_mgmt: Permission
  denied`. Un `Success` ici est **à dire en tête du rapport** : le compte serait déjà
  admis, par un chemin que le dossier ne connaît pas. Une ligne `InfoPipe User lookup
  … failed` peut précéder le verdict : elle ne le concerne pas (§ *Sources*).

---

## Étape R — Retour arrière, écrit avant la première écriture

**À lire maintenant, à ne jouer qu'en cas de besoin.** Il défait A1 et A2. Il ne touche
**ni** à `scom-test`, **ni** aux comptes locaux, **ni** au keytab.

```
echo "=== DEBUT ER ==="
sudo rm -f /etc/sudoers.d/scom-kerberos; echo "rc_rm=$?  (0 attendu, présent ou non)"
sudo ls -l /etc/sudoers.d/scom-kerberos; echo "rc_ls=$?  (2 attendu : absent)"
sudo visudo -c; echo "rc_visudo=$?  (0 attendu)"
sudo cp -a /root/sssd.conf.avant-scom-kerberos /etc/sssd/sssd.conf; echo "rc_cp=$?  (0 attendu)"
sudo chmod 600 /etc/sssd/sssd.conf; echo "rc_chmod=$?"
sudo restorecon -v /etc/sssd/sssd.conf; echo "rc_restorecon=$?"
sudo stat -c '%a %U %G %C' /etc/sssd/sssd.conf; echo "rc_stat=$?  (ligne identique à E0A)"
sudo systemctl restart sssd; echo "rc_restart=$?  (0 attendu)"
systemctl is-active sssd; echo "rc_active=$?  (0 attendu)"
sudo diff /root/sssd.conf.avant-scom-kerberos /etc/sssd/sssd.conf; echo "rc_diff=$?  (0 attendu, rien d'affiché)"
getent passwd "${SUDO_USER:-$(id -un)}"; echo "rc_getent_operateur=$?  (0 attendu)"
echo "=== FIN ER ==="
```

**`rm -f`, et pourquoi il est suivi d'un `ls` :** `-f` rend R jouable à tout moment,
que le fichier ait été posé ou non ; il rend aussi l'absence **muette**. La ligne `ls`
qui suit rend le bruit : `rc=2` dit que le fichier n'est plus là, quoi qu'ait fait `rm`
(`comptes-scom-host02.md`, étape R, emploie `rm` sans `-f` pour la même raison).

**Si R est jouée avant SV**, la sauvegarde n'existe pas : `cp` échoue bruyamment
(`rc_cp` non nul) — **ne pas jouer** les lignes suivantes sur `sssd.conf`, qui n'a pas
été modifié.

**Si `sudo` ne fonctionne plus** (fichier sudoers cassé malgré A1), jouer ce bloc
**depuis la seconde session ouverte avant A1**, où l'on est déjà `root`, **en retirant le
préfixe `sudo `** de chaque ligne : `sudo` relit le fichier cassé et refuserait même à
`root`. La dernière ligne y désigne l'opérateur par `SUDO_USER`, que `sudo -i` pose.

**Attendu :** `rc_ls=2`, `rc_visudo=0`, `rc_cp=0`, `stat` identique à 0a,
`rc_active=0`, `rc_diff=0` sans sortie, `rc_getent_operateur=0`. **Tout écart : arrêt et
alerte**, rien d'autre n'est tenté.

---

## Étape SV — Sauvegarde de `sssd.conf`, avant toute écriture

```
echo "=== DEBUT ESV ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
sudo cp -a /etc/sssd/sssd.conf /root/sssd.conf.avant-scom-kerberos; echo "rc_cp=$?  (0 attendu)"
sudo sha256sum /etc/sssd/sssd.conf /root/sssd.conf.avant-scom-kerberos; echo "rc_sha=$?  (0 attendu)"
sudo cmp /etc/sssd/sssd.conf /root/sssd.conf.avant-scom-kerberos; echo "rc_cmp=$?  (0 attendu)"
sudo cmp /root/sssd.conf.avant-scom-kerberos /dev/null; echo "rc_cmp_temoin=$?  (1 attendu)"
sudo stat -c '%a %U %G %C' /root/sssd.conf.avant-scom-kerberos; echo "rc_stat=$?"
echo "=== FIN ESV ==="
```

**Attendu :** les deux empreintes **égales à l'œil** et `rc_cmp=0` — c'est `cmp` qui
tranche, l'affichage n'est là que pour la capture ; le témoin rend `1` (« EOF » ou
« differ ») et montre que `cmp` **sait** échouer. Mode et propriétaire de la copie
identiques à 0a (`cp -a` : « same as -dR --preserve=all », § *Sources*). **Arrêt** si
`rc_cp` ou `rc_cmp` n'est pas `0`, ou si le témoin ne rend pas `1` : R ne pourrait pas
restaurer. Ces empreintes ne sont recopiées nulle part.

---

## Étape A1 — Règle `sudo` du compte (D-022)

**But :** poser `/etc/sudoers.d/scom-kerberos`, qui donne au **compte**, et à lui seul
(D-022 : pas au groupe), la même commande que la ligne de `scom_mon` dans `scom-test`
(D-017 d) : `/opt/microsoft/scx/bin/scxlogfilereader -p`. Rien d'autre.

**Avant ce bloc, ouvrir une seconde session SSH sur `host02` et y lancer `sudo -i`.**
Interactive, isolée, laissée ouverte jusqu'à la fin de V. *Motif :* si le fichier posé
rendait `sudo` inutilisable, cette session est le seul chemin vers R.

**Aucune empreinte de ce fichier ne peut figurer au dépôt** : il contient le nom réel du
compte (R-07). Sa garde est donc une **garde de contenu**, faite sur `host02` même.

```
echo "=== DEBUT EA1 ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
printf '%s\n' '# scom-kerberos : regle sudo du compte AD de supervision SCOM (D-022, PO-071).' '# Pose a la main, host02 seulement ; procedures/kerberos-scom-host02.md.' "${COMPTE:?vide} ALL=(root) NOPASSWD: /opt/microsoft/scx/bin/scxlogfilereader -p" > ~/scom-kerberos.sudoers; echo "rc_printf=$?  (0 attendu)"
wc -l < ~/scom-kerberos.sudoers; echo "rc_wc=$?  (3 affiché)"
grep -cv '^#' ~/scom-kerberos.sudoers; echo "rc_actives=$?  (1 affiché)"
grep -cxF "${COMPTE:?vide} ALL=(root) NOPASSWD: /opt/microsoft/scx/bin/scxlogfilereader -p" ~/scom-kerberos.sudoers; echo "rc_ligne=$?  (1 affiché, 0 attendu)"
grep -cxF "${COMPTE:?vide} ALL=(root) NOPASSWD: /bin/true" ~/scom-kerberos.sudoers; echo "rc_ligne_temoin=$?  (0 affiché, 1 attendu)"
LC_ALL=C grep -c '[^ -~]' ~/scom-kerberos.sudoers; echo "rc_non_ascii=$?  (0 affiché, 1 attendu)"
cat -A ~/scom-kerberos.sudoers; echo "rc_cat=$?"
printf '%s\n' "${COMPTE:?vide} ALL=(root NOPASSWD: /bin/true" > ~/scom-temoin.sudoers; echo "rc_printf_temoin=$?"
sudo visudo -cf ~/scom-temoin.sudoers; echo "rc_visudo_temoin=$?  (1 attendu)"
rm ~/scom-temoin.sudoers; echo "rc_rm_temoin=$?"
sudo visudo -cf ~/scom-kerberos.sudoers; echo "rc_visudo_fichier=$?  (0 attendu)"
H_AVANT=$(sha256sum < ~/scom-kerberos.sudoers); echo "rc_sha_avant=$? $H_AVANT"
sudo install -m 440 -o root -g root ~/scom-kerberos.sudoers /etc/sudoers.d/scom-kerberos; echo "rc_install=$?  (0 attendu)"
H_APRES=$(sudo cat /etc/sudoers.d/scom-kerberos | sha256sum); echo "rc_sha_apres=$? $H_APRES"
[ "$H_AVANT" = "$H_APRES" ]; echo "rc_egal=$?  (0 attendu)"
[ "$H_AVANT" = "${H_AVANT}x" ]; echo "rc_egal_temoin=$?  (1 attendu)"
sudo restorecon -v /etc/sudoers.d/scom-kerberos; echo "rc_restorecon=$?"
sudo ls -lZ /etc/sudoers.d/scom-kerberos; echo "rc_ls=$?"
sudo visudo -c; echo "rc_visudo_global=$?  (0 attendu)"
echo "=== FIN EA1 ==="
```

**Attendu et conditions d'arrêt** — tout écart : **arrêt, puis R** :
- **Contenu** (la garde, placée **avant** l'`install`, R-13 antériorité) : `3` lignes ;
  `1` ligne non commentée ; la ligne active trouvée **exactement une fois**, en ligne
  entière (`grep -x`) ; le témoin d'une ligne voisine rend `0` et `rc=1` — il montre que
  la comparaison **sait** échouer ; **aucun** caractère hors ASCII imprimable
  (`rc_non_ascii=1`) — les commentaires sont écrits sans accents à dessein. `cat -A` :
  trois lignes, chacune terminée par `$` seul — ni `^M`, ni `M-`, ni `^I`.
- **Témoin `visudo`** : la parenthèse non fermée, et elle seule — un message
  « syntax error » et `rc_visudo_temoin=1` (F-189, F-202, F-205). **Ne pas
  « simplifier » le témoin** : une ligne `NOPASSWD` sans « : » **passe** `visudo`
  (`rc=0`) et ne démontre rien (F-189, F-202). **Arrêt** si le témoin passe.
- `visudo -cf` du fichier : `parsed OK`, `rc=0`. **Arrêt sinon, avant l'`install`.**
- **Empreinte avant et après la pose** (leçon de F-201) : relevée **immédiatement
  avant** l'`install`, comparée **immédiatement après** sur le fichier posé ;
  `rc_egal=0`, et le témoin `rc_egal_temoin=1`. Ce contrôle **constate** après la pose ;
  il ne la prévient pas — c'est la garde de contenu qui la prévient. **Si `rc_egal`
  n'est pas `0` : R immédiatement.** Ces empreintes ne sont recopiées nulle part.
- **Nom du fichier posé** : `scom-kerberos`, **sans point** — `sudo` ignore les fichiers
  de `/etc/sudoers.d` dont le nom contient « . » ou finit par « ~ » (F-185). Un
  `scom-kerberos.sudoers` posé tel quel serait ignoré **en silence**.
- `ls -lZ` : `-r--r-----`, `root root`, contexte `etc_t` — l'état relevé pour
  `scom-test` (F-205). Un autre contexte : **relever**, pas d'arrêt.
- `visudo -c` global : `rc=0`. **Sinon : R immédiatement**, depuis la seconde session
  si `sudo` refuse.

---

## Étape A2 — Admission du groupe dans SSSD (D-022, PO-071)

**But :** ajouter `<GROUPE-SCOM-Q>` à `simple_allow_groups` par `realm permit -g`, et
**rien d'autre**.

**Ce que fait `realm permit -g`, selon le code de realmd 0.17.1** (§ *Sources*) — à
transposer à la version de `host02`, relevée en 0c :
- il réaffirme `access_provider = simple` et **ajoute** le nom à la fin de
  `simple_allow_groups`, liste réécrite avec le séparateur `", "` ; un nom déjà présent,
  à la casse près, n'est pas ajouté ;
- le fichier est réécrit **ligne à ligne** : toute ligne non modifiée est recopiée
  **octet pour octet**, commentaires et lignes en double compris. Les **deux** lignes
  `use_fully_qualified_names` (F-219) doivent donc ressortir **intactes** ; c'est la
  **dernière** que realmd tiendrait pour la valeur de la clé ;
- l'écriture remplace le fichier (fichier temporaire puis renommage, sous un masque
  `077`), puis **`sssd` est redémarré** par la commande `sssd-restart-service` —
  `systemctl restart sssd.service` dans la configuration de realmd du poste.
**Aucune de ces trois propriétés n'est tenue pour acquise sur `host02`** : le bloc
ci-dessous les **mesure** — `diff`, `stat`, horodatage de `sssd`.

```
echo "=== DEBUT EA2 ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
sudo cmp /root/sssd.conf.avant-scom-kerberos /etc/sssd/sssd.conf; echo "rc_cmp_avant=$?  (0 attendu)"
sudo realm permit -g "${GROUPE:?vide}"; echo "rc_permit=$?  (0 attendu)"
sudo diff /root/sssd.conf.avant-scom-kerberos /etc/sssd/sssd.conf; echo "rc_diff=$?  (1 attendu)"
sudo diff /root/sssd.conf.avant-scom-kerberos /etc/sssd/sssd.conf | grep -c '^[<>]'; echo "rc_grep_lignes=${PIPESTATUS[1]}  (2 affiché)"
sudo diff /root/sssd.conf.avant-scom-kerberos /etc/sssd/sssd.conf | grep '^>' | grep -c -- "simple_allow_groups.*${GROUPE:?vide}"; echo "rc_grep_groupe=${PIPESTATUS[2]}  (1 affiché, 0 attendu)"
sudo grep -c '^[[:space:]]*use_fully_qualified_names[[:space:]]*=' /etc/sssd/sssd.conf; echo "rc_grep_fqn=$?  (2 affiché)"
sudo stat -c '%a %U %G %C' /etc/sssd/sssd.conf; echo "rc_stat=$?  (ligne identique à E0A)"
systemctl show -p ActiveEnterTimestamp sssd; echo "rc_show_sssd=$?  (date postérieure à E0A)"
systemctl is-active sssd; echo "rc_active_sssd=$?  (0 attendu)"
getent passwd "$(id -un)"; echo "rc_getent_operateur=$?  (0 attendu)"
echo "=== FIN EA2 ==="
```

**Attendu et conditions d'arrêt** — tout écart : **arrêt, puis R** :
- `rc_cmp_avant=0` : le fichier n'a pas bougé depuis SV. **Arrêt sinon, avant
  `realm permit`** : le `diff` qui suit ne mesurerait plus le seul effet de `realm`.
- `rc_permit=0`. Sinon : relever le message, **arrêt, puis R**.
- **La garde de A2 — le `diff`.** `rc_diff=1` (une différence ; `man diff` du poste :
  « Exit status is 0 if inputs are the same, 1 if different, 2 if trouble. ») ; **exactement deux** lignes marquées
  (`2` affiché) : une ligne `<` et une ligne `>`, toutes deux `simple_allow_groups` ; la
  ligne `>` reprend **tous** les groupes de la ligne `<`, dans le même ordre, et
  **ajoute** `<GROUPE-SCOM-Q>` à la fin (`1` affiché, `rc_grep_groupe=0`). **Tout autre
  écart** — une ligne `access_provider` réécrite, même à valeur égale ; une ligne
  `use_fully_qualified_names` touchée ; un groupe disparu ou réordonné ; une ligne
  ajoutée — **arrête**, puis R. *Sens de l'échec :* le `cmp` de SV et le `cmp_avant`
  ont montré le même contrôle à l'égalité ; ici il doit montrer **une** différence, et
  celle-là seulement (R-12, R-14 : deux états qu'on attend différents).
- `use_fully_qualified_names` : toujours **deux** lignes (F-219). Une seule : realmd a
  touché au doublon — **arrêt, puis R**.
- `stat` : **identique** à 0a (mode, propriétaire, groupe, contexte). Le fichier a été
  **remplacé**, non modifié en place : son contexte SELinux vient de la règle du
  répertoire, et il n'est pas établi qu'il soit le même. **Tout écart : arrêt, puis
  R** (dont `restorecon`).
- `ActiveEnterTimestamp` : une date **postérieure** à celle de 0a — `sssd` a été
  redémarré, comme le dit le code. **Si la date n'a pas changé** : le redémarrage n'a
  pas eu lieu, la nouvelle liste **n'est pas en vigueur**, et le contrôle d'accès de V
  **ne mesurerait que l'ancienne** ; le dire en tête du rapport, **sans redémarrer
  d'ici** — c'est une décision. `is-active` : `active`.
- `getent passwd` de l'opérateur : `rc=0` — SSSD répond encore pour un compte AD.
  *Limite du témoin :* SSSD peut répondre **depuis son cache** ; ce `rc=0` ne prouve pas
  que l'annuaire a été joint.

---

## Étape V — Vérifications, sans rien exécuter

**But :** établir que la politique `sudo` posée en A1 accorde la seule commande prévue
et refuse le reste (R-12, R-13), et relever l'effet de A2 sur la phase `account` du
service `omi` (PO-071).

**Deux formes, qui n'éprouvent pas la même chose** (F-185, F-203 ; `man sudoers`,
`pam_acct_mgmt`, § *Sources*) :
- `sudo -l -U <compte> <commande>`, lancé par l'opérateur : **la politique seule**. La
  validation de compte PAM porte sur l'**opérateur**, qui invoque.
- `sudo -u <compte> sudo -n -l <commande>` : le `sudo` intérieur est **invoqué par le
  compte**, et `sudo` fait la validation de compte PAM **de celui qui l'invoque** ;
  « account validation will be performed regardless of whether or not a password is
  required ». **Tant que le compte n'est pas
  admis par SSSD**, cette forme peut échouer **pour une cause SSSD**, sur la ligne permise
  comme sur les refusées — et l'échec n'est alors pas imputable à la règle (R-13).

```
echo "=== DEBUT EV ==="
printf '%s\n%s\n' "${COMPTE:?vide}" "${GROUPE:?vide}" | grep -cxE '[a-z0-9_-]+'; echo "rc_vars=${PIPESTATUS[1]}  (2 affiché, 0 attendu)"
sudo -l -U "${COMPTE:?vide}"; echo "rc_list=$?"
sudo -l -U "${COMPTE:?vide}" /opt/microsoft/scx/bin/scxlogfilereader -p; echo "rc_U_permis=$?  (0 attendu : PERMIS)"
sudo -l -U "${COMPTE:?vide}" /bin/cat /etc/shadow; echo "rc_U_refus=$?  (1 attendu : REFUSE)"
sudo -u "${COMPTE:?vide}" sudo -n -l /opt/microsoft/scx/bin/scxlogfilereader -p; echo "rc_u_permis=$?  (0 attendu si le compte est admis par SSSD)"
sudo -u "${COMPTE:?vide}" sudo -n -l /bin/cat /etc/shadow; echo "rc_u_refus=$?  (1 attendu, sans message)"
sudo env LC_ALL=C sssctl user-checks -a acct -s omi "${COMPTE:?vide}"; echo "rc_uc_compte=$?  (ne discrimine pas)"
sudo openssl x509 -in /etc/opt/omi/ssl/omi.pem -noout -fingerprint -sha256 | diff ~/scom-kerberos-avant.fp -; echo "rc_openssl=${PIPESTATUS[0]} rc_diff=${PIPESTATUS[1]}  (0 0 attendus)"
diff ~/scom-kerberos-avant.fp /dev/null; echo "rc_diff_temoin=$?  (1 attendu)"
echo "=== FIN EV ==="
```

**Attendu :**
- `sudo -l -U` : parmi les commandes permises, **une seule** règle, celle de
  `/opt/microsoft/scx/bin/scxlogfilereader -p`. La mise en forme exacte de la liste
  n'est pas sourcée ici : **relever la sortie entière**. **Toute autre règle** — venue
  d'un autre fichier, d'un groupe, d'une règle `ALL` — **arrête** la procédure, puis R :
  le compte aurait plus que D-022.
- Forme `-U` : permis → la commande réaffichée, chemin complet, `rc=0` ; refusé →
  `rc=1`, **aucun message**.
- Forme `-u … sudo -n -l` : **si** 0a a montré le compte dans le groupe **et** si A2 a
  redémarré `sssd`, mêmes attendus que la forme `-U`. **Sinon**, un message de validation
  de compte (du type « account validation failure ») sur les **deux** lignes est
  l'effet attendu de SSSD, pas de `sudoers` : **non concluant**, à consigner tel quel —
  pas d'arrêt, pas de R. Ce résultat **instruit PO-071** pour le service `sudo`.
- `user-checks` du compte : **même lecture que 0c, par la ligne `pam_acct_mgmt: …`**.
  Compte dans le groupe, `sssd` redémarré, témoin de 0c à `Success` : `Success` attendu
  — c'est la réponse à PO-071 pour le service `omi`. **Sinon : non concluant**, à
  consigner tel quel (F-221) ; un `Permission denied` est alors l'état attendu, pas un
  échec de A2.
- Empreinte : `rc_openssl=0`, `rc_diff=0`, rien d'affiché — le certificat n'a pas changé.
  Le témoin affiche une différence et rend `1` : `diff` **sait** échouer. **Si
  `rc_diff` n'est pas `0` : arrêt et alerte** — rien ici ne touche le certificat ; ne
  rien corriger, rapporter.

**Discrimination du refus (R-13)** — un `rc=1` ne nomme pas sa cause : `sudo` rend aussi
`1` pour un échec d'authentification ou de configuration (F-185). Le refus n'est
imputable **à la règle absente** que si les trois conditions sont réunies, **pour chaque
forme** :
1. la ligne **permise** de la même forme a rendu `0` **juste avant** — ce n'est donc ni
   `-n`, ni le `sudo` extérieur, ni la validation de compte, ni la forme de la commande ;
2. **aucun message** n'accompagne le `rc=1` — un message de mot de passe requis, de
   validation de compte, ou « not allowed to execute … » venu du `sudo` extérieur,
   désigne une autre cause : **l'échec forcé est mal construit**, on ne l'interprète pas ;
3. la ligne refusée **n'apparaît pas** dans la liste de `sudo -l -U`.

**La forme `-U` suffit à démontrer la règle `sudo`** ; la forme `-u` ajoute la preuve que
le compte **lui-même** peut interroger `sudo` — ce qui dépend d'A2 et de l'appartenance
au groupe. Dire, au rapport, laquelle a rempli les trois conditions.

---

## Étape N — Nettoyage, après validation de Kerberos (D-023)

**Pas avant :** la sauvegarde sert à R tant que la configuration n'est pas validée, et
l'empreinte « avant » sert à la comparaison après découverte.

```
echo "=== DEBUT EN ==="
rm ~/scom-kerberos.sudoers ~/scom-kerberos-avant.fp; echo "rc_rm=$?"
sudo rm /root/sssd.conf.avant-scom-kerberos; echo "rc_rm_sauvegarde=$?"
ls -la ~/scom-kerberos.sudoers ~/scom-kerberos-avant.fp; echo "rc_ls=$?  (2 attendu)"
sudo ls -la /root/sssd.conf.avant-scom-kerberos; echo "rc_ls_sauvegarde=$?  (2 attendu)"
echo "=== FIN EN ==="
```

`rm` sans `-f` : un fichier déjà absent se signale. N n'emploie aucun des deux noms.

---

## Ce que l'opérateur rapporte au pilote

Chaque bloc **entier**, marqueurs compris, en `[ÉCRAN-AAAA-MM-JJ]` (D-003), et en
particulier :
- de S : l'absence de la ligne `rc_temoin_vide`, et `rc_forme_temoin` ;
- de 0a : les lignes de `sssd.conf`, la ligne `stat`, l'horodatage de `sssd`, et si le
  groupe figure dans `id` ;
- de 0b : `pemfile`, `keyfile` et `httpsport` d'`omiserver.conf`, **avec le chemin
  effectivement lu** ; `/etc/pam.d/omi` ; `/etc/pam.d/sshd` ;
- de 0c : les versions ; la ligne `sssd-restart-service` ; l'aide de `user-checks` ; les
  deux lignes `pam_acct_mgmt: …` ;
- le **`diff` entier** d'A2, et les lignes `stat` et `ActiveEnterTimestamp` d'A2 ;
- de V : la sortie de `sudo -l -U`, les quatre codes des deux formes et leurs messages,
  la ligne `pam_acct_mgmt: …`, `rc_diff` ;
- toute élévation hors de ce fichier.

**Aucune valeur réelle n'est à recopier dans le dépôt** — ni nom de compte, de groupe ou
de machine, ni empreinte (R-07). Une capture qui les montre se transmet au pilote ; elle
ne se versionne pas.

---

## Gardes, branches et ce qui reste non démontré (R-04, R-12)

| Contrôle | Placé avant l'effet ? | Sens de l'échec |
|---|---|---|
| arrêt par `${…:?}` (toutes étapes sauf R, N) | oui, sur chaque ligne qui emploie un nom | témoin `SCOM_TEMOIN_VIDE`, joué en S ; mesuré sur le poste (bash 5.3.9) |
| forme des noms, `rc_vars` | oui, en tête de chaque bloc | témoin `x,y`, joué en S |
| `getent passwd` du compte (0a) | oui, avant A1 | **écrit, non démontré** sur `host02` : un `rc=2` de `getent` est documenté (F-055), non forcé ici |
| `scom-kerberos` absent, sauvegarde absente (0b) | oui, avant SV et A1 | **écrit, non démontré** — le forcer demanderait de créer ces fichiers |
| empreinte « avant » (0c) | oui, avant A1 | constat de lecture ; son usage est en V |
| `cmp` de la sauvegarde (SV) | oui, avant A1 | témoin `/dev/null`, joué en SV |
| contenu du fichier sudoers (A1) | oui, avant `install` | témoin de ligne voisine et témoin non ASCII, joués en A1 ; mesurés sur le poste |
| `visudo -cf` (A1) | oui, avant `install` | témoin à parenthèse non fermée, joué en A1 (F-189, F-205) |
| empreinte avant / après pose (A1) | **non** — constat juste après la pose | témoin `${H_AVANT}x`, joué en A1 |
| `cmp` avant `realm permit` (A2) | oui, avant `realm` | exercé à l'égalité en SV |
| `diff` de `sssd.conf` (A2) | **non** — constat après `realm`, rattrapé par R | exercé dans les **deux** sens : égalité en SV et `cmp_avant`, différence attendue ici |
| `stat` et horodatage de `sssd` (A2) | **non** — constat | **écrit, non démontré** dans le sens de l'échec |
| refus `sudo` (V) | oui, avant toute découverte | **démontré par construction** sous les trois conditions de R-13, pour la forme qui les remplit |
| `user-checks` (0c, V) | — vérification, pas une garde | témoin de l'opérateur en 0c ; **non concluant** tant que le compte n'est pas dans le groupe (F-221) |
| empreinte du certificat (V) | **non** — constat après coup | témoin `/dev/null`, joué en V |

**Branches :** la seconde forme de V est **non concluante** tant que le compte n'est pas
dans le groupe ; les lignes `user-checks` ne sont pas jouées si `sssd-tools` manque ou
si son aide ne liste pas `-a` et `-s` ; R se joue depuis la seconde session, sans
`sudo`, si `sudo` est cassé ; le `find` de 0b ne sert que si le chemin d'`omiserver.conf`
n'est pas le bon. Le rapport dit lesquelles ont été jouées.

---

## Sources lues pour cette procédure

Lues le **2026-11-09** sur le poste de rédaction, sans élévation ; téléchargements dans le
répertoire temporaire de la session, hors dépôt. **Elles ne sont pas encore au dossier**
— ni `F-xxx`, ni `P-xx` : le périmètre d'écriture de cette rédaction n'incluait que ce
fichier et une note sous D-022. **À verser au dossier et au journal** par une
consignation ultérieure (R-15) ; d'ici là, elles valent comme citations nommées
`[SOURCE-EXTERNE]`, avec la réserve de transposition de F-184 et F-195.

- **`man realm`** — `realmd-0.17.1-19.fc44`, page datée 01/17/2026, section *PERMIT* :
  `--groups, -g` « Treat the specified names as groups rather than user login names.
  Permit login by users in the specified groups. » **La page ne dit ni si SSSD est
  redémarré, ni quelles lignes de `sssd.conf` sont touchées.**
- **Code de realmd, étiquette `0.17.1`** (`gitlab.freedesktop.org/realmd/realmd`,
  `http_code=200` ×4, 09:05:55 UTC) — la version du poste, **pas** établie comme celle de
  `host02` :
  - `service/realm-sssd.c:74-116` (`realm_sssd_set_login_policy`) : `access_provider`
    posé, puis `realm_ini_config_set_list_diff (…, "simple_allow_groups", ",", …)` ;
    `:212-214` : après l'écriture, `realm_service_restart ("sssd", …)` ;
    `:120-123` : `,` et `$` refusés dans un nom ;
  - `service/realm-ini-config.c:485-487` : une clé en double est indexée sur sa
    **dernière** occurrence ; `:579-601` (`write_to_string`) : réécriture ligne à ligne,
    octets d'origine conservés ; `:665-708` : `g_file_set_contents`, masque `077` pour un
    fichier privé ; `:1000-1050` : liste rejointe par `", "`, nom déjà présent ignoré à
    la casse près ;
  - `service/realm-sssd-config.c:34` : `sssd.conf` ouvert avec `REALM_INI_PRIVATE` ;
  - `service/realm-service.c:98-106` : la commande est `sssd-restart-service`.
  - `/usr/lib/realmd/realmd-distro.conf:41` du poste : `sssd-restart-service =
    /usr/bin/systemctl restart sssd.service`.
- **`sssctl`** — **`man sssctl` est absent du poste** (`sssd-tools` non installé, `rpm -q`
  → `rc=1`, `man -w` → `rc=16`) ; rien n'a été installé (R-06). **Basculement d'outil
  déclaré (R-15)** : source amont de la page, `SSSD/sssd`, `src/man/sssctl.8.xml` sur
  `master` (`http_code=200`, 2 331 octets, dernier commit du fichier le 2024-02-11) —
  **elle ne documente pas `user-checks`** et renvoie à `sssctl COMMAND --help`. **Second
  basculement** : code amont, `src/tools/sssctl/sssctl_user_checks.c` sur `master`
  (`http_code=200`, 9 504 octets, dernier commit du fichier le 2024-09-02) :
  `:234-238` options `-a`/`--action` (« PAM action [auth|acct|setc|chau|open|clos],
  default: acct ») et `-s`/`--service` ; un nom d'utilisateur en argument ; `:285-288`
  `pam_acct_mgmt` puis `ERROR("pam_acct_mgmt: %s\n\n", pam_strerror (…))` ; `:254-261`
  messages de recherche du nom et InfoPipe ; **`:317-318` : `ret = EOK` quel que soit le résultat PAM.** Version
  de `host02` : relevée en 0c, syntaxe confirmée par `--help` sur la cible.
- **`pam_strerror`** de `pam-1.7.2-2.fc44`, appelé sur le poste : `0` « Success », `6`
  « Permission denied », `10` « User not known to the underlying authentication
  module ».
- **`man sudoers`** — `sudo-1.9.17-10.p2.fc44` : grammaire `User` (l. 327-336) ; noms
  entre guillemets ou échappés (l. 346-363) ; caractères à échapper dans un nom :
  « ‘!’, ‘=’, ‘:’, ‘,’, ‘(’, ‘)’, ‘\’ » (l. 1034-1035) ; `pam_acct_mgmt` (l. 1476-1482) :
  « sudo will perform PAM account validation for the invoking user by default […]
  regardless of whether or not a password is required. This flag is on by default. »
- **`man bash`** — `bash-5.3.9-3.fc44`, `${parameter:?word}` (l. 1601-1606) : « An
  interactive shell does not exit, but does not execute the command associated with the
  expansion. » **Mesuré** (`bash --norc -i`, 2026-11-09) : la ligne entière est
  abandonnée, `echo "rc_…"` compris, avec `bash: COMPTE: vide` ; la ligne suivante
  s'exécute.
- **`man cp`** — `coreutils-9.10-5.fc44` : `-a` « same as -dR --preserve=all ».
- **Formes de contrôle d'A1, mesurées sur le poste** avec un nom fictif (09:10:09 UTC) :
  `wc -l` → `3` ; `grep -cv '^#'` → `1` ; `grep -cxF` ligne exacte → `1`, `rc=0`, ligne
  voisine → `0`, `rc=1` ; `LC_ALL=C grep -c '[^ -~]'` → `0`, `rc=1`, et `1`, `rc=0` sur
  un `é` ; `visudo -cf` → `parsed OK`, `rc=0` ; témoin à parenthèse → « syntax error »
  à 1:20, `rc=1` ; comparaison `H_AVANT` / `H_APRES` → `rc=0`, témoin → `rc=1` ; contrôle
  de forme → `2`, `rc=0`, et `0`, `rc=1` sur `x,y`.

**Ce qui reste `@VERIF` :**
- le chemin `/etc/opt/omi/conf/omiserver.conf` (0b) ;
- le comportement de `realm permit -g` **sur `host02`** — redémarrage de `sssd`, lignes
  touchées, contexte SELinux du fichier remplacé : établi par le code **du poste**,
  **mesuré** en A2 ;
- la syntaxe de `sssctl user-checks` **sur `host02`** : établie par le code amont,
  confirmée en 0c par `--help` ;
- le comportement de `${…:?}` dans le `bash` de `host02` : démontré en S.

`man diff` du poste (`diffutils-3.12-5.fc44`, l. 220) : « Exit status is 0 if inputs are
the same, 1 if different, 2 if trouble. »
