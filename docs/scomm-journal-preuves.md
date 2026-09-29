# Journal de preuves — projet certificat SCOM / ADCS

**Ce que ce fichier porte :** les sorties de commande relevées, numérotées `P-xx`, avec
leur code de retour. C'est la pièce justificative des faits `F-xxx` du dossier
d'autorité.

**Ce que ce fichier NE porte PAS :** aucun fait, aucune règle, aucune décision, aucun
point ouvert. Il ne s'interprète pas seul : un `P-xx` n'affirme rien, il montre.

**Ce fichier ne se relit pas à chaque session.** On y revient quand un `F-xxx` est
contesté, ou quand il faut rejouer une mesure. Le dossier d'autorité
(`scomm-certificat-facts.md`) et la description du brouillon
(`scomm-depot-actuel.md`) y renvoient par numéro.

Les preuves gardent leur numéro d'origine : elles ont été déplacées, pas renumérotées
(R-11).

**Avertissement de rejouabilité.** Les chemins personnels y ont été substitués par
`/home/<user>/` (PO-011, D-005) : les commandes ne sont plus copiables telles quelles.
C'est un coût assumé. **Là où la substitution a atteint un *motif de recherche* et non un
chemin, la commande est fausse et a été corrigée** — voir P-18 et P-26.

---

## Journal de preuves

Sorties relevées le **2026-09-18**, toutes exécutées sur le poste Fedora personnel,
toutes sans élévation. Extraits.

`[RÉVISÉE le 2026-09-21]` **ÉTAIT :** « Extraits ; les chemins personnels sont conservés
car le poste n'est pas un système cible. » Cette phrase a cessé d'être vraie le même jour
qu'elle a été écrite : la substitution du livrable 2 (PO-011, D-005) a remplacé ces
chemins par `/home/<user>/`. Elle n'a pas été mise à jour alors — c'est le même oubli que
celui corrigé en P-18, appliqué cette fois à une phrase et non à une commande.

**P-01** — `git -C … rev-parse --show-toplevel` / `status --porcelain -uall` / `rev-parse HEAD` / `log --oneline -5` / `remote -v`
```
/home/<user>/dev/scomm_rhel9                       rc=0
(status : aucune ligne)                              rc=0
5f0eca9a3d011e49c5888765bf112d800a64949e             rc=0
5f0eca9 Project init
b0c3426 Initial commit                               rc=0
origin	https://github.com/charMlessMan80/scomm_rhel9 (fetch)
origin	https://github.com/charMlessMan80/scomm_rhel9 (push)   rc=0
```

**P-02** — `git config --show-origin --get-all credential.helper` → `rc=1`, sortie vide.
`git config -l --show-origin | grep -i credential` → `rc=1`.

**P-03** — `git ls-files` → 15 lignes, `rc=0`.

**P-04** — `ls -la .gitignore` → `rc=2` ; `git ls-files --error-unmatch .gitignore` → `rc=1` ;
`ls -la CLAUDE.md` → `rc=2` ; `ls -la docs` → `rc=2`.

**P-05** — `cat -n` sur les 15 fichiers suivis, `rc=0` pour chacun. Base de tous les
`chemin:ligne` de la section 1.2.

**P-06** — `ansible-inventory --graph` :
```
@all:
  |--@ungrouped:
  |--@scomm_agents:
rc=0
```
et `find /home/<user> -maxdepth 6 -name ansible-lint -type f` →
`/home/<user>/.venvs/ansible-lint/bin/ansible-lint`, `rc=0` (idem `yamllint`).

**P-07** — `ansible --version` → `ansible [core 2.20.7]`,
`config file = /home/<user>/dev/scomm_rhel9/ansible.cfg`,
`python module location = /usr/lib/python3.14/site-packages/ansible`, `rc=0`.

**P-08** — `rpm -q certmonger` → `package certmonger is not installed`, `rc=1`.
`rpm -q cepces` → `package cepces is not installed`, `rc=1`.
`command -v getcert` → `rc=1`.

**P-09** — Contrôle de visibilité :
```
curl -s -o /tmp/gh.json -w 'http_code=%{http_code}' https://api.github.com/repos/charMlessMan80/scomm_rhel9
http_code=200   rc_curl=0
"private": false
GIT_TERMINAL_PROMPT=0 GIT_ASKPASS=/bin/true git ls-remote https://github.com/charMlessMan80/scomm_rhel9 HEAD
5f0eca9a3d011e49c5888765bf112d800a64949e	HEAD    rc=0
```

**P-10** — `curl … openSUSE/cepces/master/README.rst` → `http_code=200`, 10411 octets, `rc=0`.

**P-11** — `curl … openSUSE/cepces/master/doc/CERTMONGER.md` → `http_code=200`, 8971 octets, `rc=0`.

**P-12** — `dnf info cepces` → `Version: 0.5.0`, `Release: 2.fc44`, `Repository: updates`,
`URL: https://github.com/openSUSE/cepces`, `License: GPL-3.0-or-later`, `rc=0`.
`dnf repoquery --qf … certmonger` → `certmonger-0.79.21-4.fc44.x86_64 repo=Fedora 44 - x86_64`, `rc=0`.
`dnf repoquery --requires cepces` → `python3-cepces = 0.5.0-2.fc44`, `(cepces-selinux if selinux-policy-targeted)`, `rc=0`.

**P-12b** — **Échec déclaré et basculement d'outil.** `dnf repoquery -l cepces` a dépassé
le délai de 120 s et a été interrompu (`rc=143`, `Terminated`). Cause probable : le
téléchargement des métadonnées *filelists*. **Aucune reprise n'a été tentée** : la liste
des fichiers de la version Fedora n'aurait de toute façon pas valeur de preuve pour
RHEL 9 (PO-006). Les faits cepces reposent donc sur S-1/S-2, pas sur le paquet.

**P-13** — `getcert-request(1)` via mankier.com : descriptions littérales de
`-C`, `-B`, `-c`, `-k`, `-f`, `-N`, `-D`, `-K`, `-G`, `-g`, `-T`, `-U`, `-r`, `-R`.
`@VERIF : reconfirmer sur « man getcert-request » à la version certmonger réellement
installée sur RHEL 9 — mankier reproduit la page mais n'est pas le dépôt amont.`

**P-14** — `curl … SupportArticles-docs/main/support/system-center/scom/use-ca-certificate-on-scx-agent.md`
→ `http_code=200`, 10340 octets, 217 lignes, `rc=0`. Lu intégralement en texte brut.
Extrait décisif (lignes 186-194) :
```
openssl x509 -noout -in /etc/opt/microsoft/scx/ssl/scx.pem -subject -issuer -dates

subject= /DC=lab/DC=nfs/CN=RHEL7-02/CN=RHEL7-02.nfs.lab
issuer= /DC=lab/DC=nfs/CN=nfs-DC-CA
...
In typical scenarios, the `issuer` will be a management server/gateway in the
UNIX/Linux resource pool
```
(Les valeurs `nfs.lab` / `RHEL7-02` sont celles de l'exemple Microsoft, pas de cet
environnement.)

**P-15** — Récupération de `manage-security-administer-crossplat-agent?view=sc-om-2025`.
Extrait décisif, section `scxsslconfig` :
```
The generated certificate must be signed by Operations Manager management server in
order to be used in WS-Management communication. Overwriting a previously signed
certificate will require that the certificate be signed again.
```
et, dans l'aide de l'outil : `-f  - force certificate to be generated even if one exists`.

**P-17** — Contrôle syntaxique et collection manquante :
```
ansible-playbook --syntax-check integrate_scomm.yml                      rc=4
[WARNING]: provided hosts list is empty, only localhost is available.
[WARNING]: Error loading plugin 'ansible.posix.firewalld': No module named
           'ansible_collections.ansible.posix'
[ERROR]: couldn't resolve module/action 'ansible.posix.firewalld'.
Origin: …/roles/scomm_agent/tasks/configure.yml:8:3

ansible-galaxy collection list | grep -i posix                           rc=1
ls -la ~/.ansible/collections                                            rc=2 (absent)
ls -la /usr/share/ansible/collections                 → ansible_collections/ vide, rc=0
ansible-playbook --syntax-check … --skip-tags configure                  rc=4
```
Aucune installation n'a été tentée.

**P-18** — Contrôle de confidentialité sur les quatre fichiers produits :
```
grep -nE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' <4 fichiers>                  rc=1 (aucune IP)
grep -niE 'password[[:space:]]*[:=]…|BEGIN … PRIVATE KEY|api[_-]?key'    rc=1 (aucun secret)
grep -nE '\b[A-Z][A-Z0-9-]{2,}\.[A-Z][A-Z0-9.-]{1,}\b'                   rc=0
  → uniquement « EXAMPLE.COM » (valeur d'exemple)
grep -oE '<motif FQDN>' | grep -v '<domaines publics et d'exemple>'      rc=0
  → uniquement des faux positifs : ansible.posix.firewalld,
    ansible.builtin.dnf, cepces.conf.dist, hosts.local.ini
grep -c <IDENT-LOCAL> <4 fichiers>   → docs/…: 10 ; CLAUDE.md, README.md, .gitignore : 0
```
où `<IDENT-LOCAL>` **est le motif de recherche** : l'identifiant réel du compte local,
non reproduit ici (R-07). Ce n'est pas un chemin.

`[RÉVISÉE le 2026-09-21]` **ÉTAIT :** « `grep -c '<user>' <4 fichiers>   → docs/…: 10 ;
CLAUDE.md, README.md, .gitignore : 0` ».
**Pourquoi c'était faux :** la substitution du livrable 2 (D-005) a remplacé l'identifiant
par `<user>` **y compris à l'endroit où il était le motif de recherche**, pas un chemin.
La commande ainsi écrite n'a donc **jamais été exécutée** : elle cherche la chaîne
littérale `<user>`, qui n'existait pas dans les fichiers au moment de la mesure. Seul le
motif a été corrigé ; les chiffres et la date d'origine sont conservés. Rejeu mesuré des
deux formes : P-26.

`[NOTE le 2026-09-18]` Le « 10 » ci-dessus est la mesure **du livrable 1**, exacte à sa
date. L'ajout de P-17 et P-18 l'a ensuite porté à 13 lignes, avant substitution (P-19).
La ligne est conservée telle quelle : c'est une preuve datée, pas un compteur à jour.
**Ce que ce contrôle NE couvre PAS** — à ne pas confondre avec une garantie :
l'historique git (il n'inspecte que l'arbre de travail, pas les commits antérieurs ni
les objets déjà poussés) ; les fichiers déjà suivis avant ce livrable, qui n'ont pas été
expurgés (`inventory/hosts.ini:7-8` contient déjà des adresses d'exemple `10.0.0.11/12`
et `group_vars/scomm_agents.yml:8` une URL de dépôt — antérieurs, non introduits ici) ;
tout nom interne qui ne ressemblerait ni à une IP, ni à un FQDN, ni à un realm en
majuscules (un nom de VIP arbitraire, un identifiant de projet) ; et le contenu des
captures d'écran que l'opérateur produira ensuite.

**P-16** — Absence de `rhel_post_install`, trois recherches indépendantes :
```
find /home/<user> -maxdepth 8 \( -iname '*post_install*' -o -iname '*post-install*' \
  -o -iname '*postinstall*' \)              rc=0   lignes=0
find / -xdev \( -iname '*rhel_post*' -o -iname '*rhel-post*' \)   rc=1   lignes=0
find /home/<user> -maxdepth 8 -type d -name '.git'              rc=0
  → /home/<user>/dev/{workstation-config,system_auto-update,glass-hud,scomm_rhel9}/.git
    (+ sous-arbres de .cargo et .cache)
```

---

*Preuves ajoutées le **2026-09-18**, audit d'identifiant personnel (livrable 2).*

**P-19** — Substitution et contrôle **dans les deux sens**. Avant :
`grep -c <IDENT-LOCAL>` → `CLAUDE.md:0 (rc=1)`, `README.md:0 (rc=1)`, `.gitignore:0 (rc=1)`,
`docs/scomm-certificat-facts.md:13 lignes / 13 occurrences (rc=0)`.
Après substitution :
```
grep -n <identifiant> CLAUDE.md docs/…-facts.md README.md .gitignore        rc=1
  (négatif : plus aucune occurrence dans les quatre fichiers)
grep -c <identifiant> workstation-config/docs/machine-facts.md   → 7        rc=0
  (positif : la MÊME commande trouve toujours ailleurs — elle n'est pas cassée)
grep -n '<user>' docs/…-facts.md                                 → 13       rc=0
```
`[RÉVISÉE le 2026-09-21]` La première ligne du bloc « Avant » **ÉTAIT :** « `grep -c` →
`CLAUDE.md:0 (rc=1)` … » — sans motif du tout. `<IDENT-LOCAL>` y a été ajouté : une
commande qui ne dit pas ce qu'elle cherche ne se rejoue pas. Seul le motif a été rendu
explicite ; aucun chiffre n'a été touché.

`[NOTE le 2026-09-21]` Les trois lignes du bloc « Après » sont **correctes en la forme** : les
deux premières notent explicitement `<identifiant>` comme motif, et la troisième cherche
bien la chaîne littérale `<user>`. C'est la seule forme à imiter. Le « 13 » est une
mesure datée du 2026-09-18 ; il vaut 19 au 2026-09-21 (P-26), l'écart venant des
paragraphes ajoutés depuis. Mesure périmée, pas mesure fausse.

**P-20** — Identités d'auteur, relevées par `git log --all --format='%an <%ae>'` :
```
workstation-config   53 <nom civil> <adresse personnelle>
                      6 <pseudonyme> <adresse noreply GitHub>        59 commits
system_auto-update   13 <pseudonyme> <adresse personnelle>           13 commits
scomm_rhel9           2 <pseudonyme> <adresse personnelle>            2 commits
```
Committers : identiques, plus `GitHub <noreply@github.com>` sur les commits créés via
l'interface web. rc=0 pour les trois dépôts.

**P-21** — Nom civil dans le **contenu suivi** de `workstation-config` :
`[RÉVISÉE le 2026-09-21]` **ÉTAIT :** « `git grep -n` → … », sans motif. Même défaut que
P-18 sous une autre forme : là où P-18 portait un *faux* motif, P-21 n'en portait
*aucun*. Motif rendu explicite, chiffres inchangés.
`git grep -n <NOM-CIVIL>` — où `<NOM-CIVIL>` est le motif de recherche, non reproduit
ici (R-07) — →
`roles/{android_env,android_jdk,android_sdk,bootstrap,completion,`
`copilot_cli,desktop,editor,gpu_cdi,gpu_mux,local_ai,recovery}/meta/main.yml:4`
(12 fichiers, champ `galaxy_info.author`) et `docs/ansible-chain.md:77`. rc=0.
Aucune occurrence dans `system_auto-update` ni `scomm_rhel9` (rc=1 pour les deux).

**P-22** — Publication effective, vérifiée **contre le serveur** et non sur des refs
locales potentiellement périmées :
```
git ls-remote origin HEAD refs/heads/main        (les 3 dépôts)        rc=0
  → SHA distant identique au HEAD local dans les trois cas
curl -s api.github.com/repos/…/<dépôt>  (NON authentifié)  http=200
  → private=False, forks_count=0, network_count=0   pour les trois
```
Tout l'historique des trois dépôts est donc publié, et aucun fork n'est recensé par
l'API **au moment de la mesure**.

**P-23** — Identité d'un **futur** commit dans ce dépôt :
`git var GIT_AUTHOR_IDENT` → `<pseudonyme> <adresse noreply GitHub>`, rc=0.
Le `user.email` local des trois dépôts est l'adresse `noreply`, alors que les commits
existants portent l'adresse personnelle : la configuration a été corrigée après coup.

**P-24** — Comptage par surface de l'identifiant du compte local :
```
DÉPÔT                S1 suivi   S2 hors .git   S3 historique   S4 messages
workstation-config   34 (rc=0)  34 (rc=0)      1167 lignes     0 (rc=1)
system_auto-update    0 (rc=1)   0 (rc=1)         0            0 (rc=1)
scomm_rhel9           0 (rc=1)  13 (rc=0)         0            0 (rc=1)
```
**P-25** — Branche SSH, exercée (elle ne l'avait pas été au livrable 1, le remote de ce
dépôt étant en `https://`). Les deux autres dépôts ont un remote SSH :
```
ssh -o BatchMode=yes -T git@github.com
Hi <pseudonyme>! You've successfully authenticated, but GitHub does not
provide shell access.                                                    rc=1
```
**`rc=1` ici ne signifie pas un échec** : GitHub authentifie puis refuse le shell, ce qui
produit ce code. L'information est dans la bannière, pas dans le code de retour. Le
contre-exemple vaut d'être noté — c'est exactement le piège que la règle R-03 vise.
Cela affine PO-007 sans le rouvrir : l'authentification du compte fonctionne ; le blocage
au push est propre au remote `https://` **de ce dépôt-ci**, dépourvu de
`credential.helper`. Aucun remote n'a été modifié.

Lecture des zéros, qui ne disent pas tous la même chose : `system_auto-update` rend zéro
**parce qu'il n'y a rien** (rc=1 sur toutes les surfaces). `scomm_rhel9` S1 rend zéro
`rc=1` **parce que les fichiers concernés n'étaient pas suivis** — `git grep` ne voit que
l'index ; c'est S2 qui les a trouvés. Un `rc=1` de `git grep` n'est donc pas une preuve
d'absence sur le disque.

---

*Preuves ajoutées le **2026-09-21** (livrable 3). Toutes relevées **au moment de leur
exécution**, sur le poste Fedora personnel, **toutes sans élévation** (R-15).*

**P-26** — **Rejeu de P-18, dans ses deux formes.** C'est la mesure qui établit que la
commande consignée en P-18 était fausse.
```
-- (a) commande TELLE QU'ECRITE dans le journal avant correction : motif litteral '<user>'
grep -c '<user>' CLAUDE.md docs/scomm-certificat-facts.md README.md .gitignore
CLAUDE.md:1
.gitignore:0
docs/scomm-certificat-facts.md:19
README.md:0                                                              rc=0

-- (b) commande d'ORIGINE : motif = identifiant reel du compte (U=$(id -un), non reproduit)
grep -c -- "$U" CLAUDE.md docs/scomm-certificat-facts.md README.md .gitignore
CLAUDE.md:0   README.md:0   .gitignore:0   docs/scomm-certificat-facts.md:0   rc=1

-- (c) controle positif : le meme motif reel trouve-t-il ailleurs ?
grep -c -- "$U" /home/<user>/dev/workstation-config/docs/machine-facts.md
7                                                                        rc=0
```
**Ce que ces trois lignes établissent, et il faut les lire ensemble :**
- (a) rejouée aujourd'hui **ne rend pas 10** : elle rend **19** pour le dossier et — ce
  que P-18 n'annonçait pas du tout — **1 pour `CLAUDE.md`**, où P-18 affirmait `0`.
  La commande ne mesure donc pas ce que sa ligne de résultat prétend ;
- (b) rend **0 partout, `rc=1`** : la substitution du livrable 2 est complète, et la
  clôture de PO-011 tient ;
- (c) rend **7, `rc=0`** sur un fichier hors périmètre : **la commande n'est pas cassée**.
  Sans (c), le `rc=1` de (b) ne prouverait rien — un motif mal écrit rend exactement le
  même `rc=1`. C'est le contrôle dans les deux sens exigé par R-03 et, désormais, R-12.

**Les deux `0` de ce bloc ne disent pas la même chose** : celui de (b) dit « le motif est
bon et ne trouve rien » ; celui de (a) pour `README.md`/`.gitignore` dit « ces fichiers
n'ont jamais contenu le placeholder », ce qui n'était pas la question posée.

**P-27** — **État de départ du livrable 3**, relevé **avant toute écriture** :
```
git status --porcelain -uall                                             rc=0
 M README.md
?? .gitignore
?? CLAUDE.md
?? docs/scomm-certificat-facts.md
git rev-parse HEAD  → 5f0eca9a3d011e49c5888765bf112d800a64949e           rc=0
wc -l CLAUDE.md docs/scomm-certificat-facts.md → 193 / 749               rc=0
```
**Arbre sale, signalé et non touché** (contrôle d'entrée de `CLAUDE.md`) : `README.md`
est modifié et non commité ; il appartient à l'opérateur et n'a pas été lu au-delà de
ses deux renvois, ni modifié. **Le `-uall` fait son office** : la ligne `?? docs/` du
`git status` ordinaire se déplie ici en `?? docs/scomm-certificat-facts.md`.
**Prémisse de l'énoncé corrigée :** le livrable annonçait « le dossier dépasse 520
lignes » ; il en comptait **749**.

**P-28** — **Relecture ciblée de S-4** (étape 1 du livrable 3). Même URL qu'en P-14,
re-téléchargée pour être relue et non citée de mémoire :
```
curl -s -o s4.md -w 'http_code=%{http_code} size=%{size_download}'
  https://raw.githubusercontent.com/MicrosoftDocs/SupportArticles-docs/main/
  support/system-center/scom/use-ca-certificate-on-scx-agent.md
http_code=200 size=10340   rc_curl=0
wc -l s4.md → 217                                                        rc=0
```
Taille et nombre de lignes **identiques à P-14** (2026-09-18) : la source n'a pas bougé,
les numéros de ligne de F-036 restent valides. Lignes décisives, citées littéralement :
```
52: On the **Windows Server** that has been given the permission to the template,
    open the computer certificate store.
74: Right-click the certificate and export it with a private key. Finally, there
    should be a .pfx file.
83: Copy the certificate to the Unix/Linux server for which the certificate was issued.
87: openssl pkcs12 -in <FileName>.pfx -nocerts -out /etc/opt/omi/ssl/omikey.pem -nodes …
33: On the Subject Name tab, select **Supply in the request**.
135: # Usage: sudo ./extract_scx_cert.sh /path/to/certificate.pfx <pfx_password>
150: openssl pkcs12 -in "$PFX_FILE" -nocerts -out "$KEY_FILE" -nodes …
```
Les deux méthodes de S-4 (lignes 81-126 et 128-179) partent d'un `.pfx`. **Aucune ligne
du fichier ne fait naître une clé sur l'hôte Linux ; aucune ne mentionne de CSR.**
Fonde F-042 et F-043. Fichier de travail conservé hors du dépôt (répertoire temporaire de
session), non versionné.

---

*Preuves ajoutées le **2026-09-21** (livrable 4, reportées au livrable 5). Toutes relevées
au moment de leur exécution sur le poste Fedora de rédaction, **toutes sans élévation**.*

**P-29** — Versions des outils sourçables. `rpm -q <paquet>`, rc=0 pour chacun :
```
openssl-3.5.7-2.fc44   curl-8.18.0-8.fc44   bind-utils-9.18.50-1.fc44
chrony-4.8-5.fc44      iproute-6.17.0-2.fc44   systemd-259.8-1.fc44
krb5-libs-1.22.2-4.fc44
curl --version | head -1  → « … mit-krb5/1.22.2 … »                      rc=0
```
Fonde F-051. **Fedora 44 : ne vaut pas pour RHEL 9.**

**P-30** — Absence des outils Kerberos, et ce qui reste lisible :
```
rpm -q krb5-workstation   → package krb5-workstation is not installed    rc=1
command -v klist|kinit|kvno|kdestroy                                     rc=1 (les 4)
man -w  klist|kinit|kvno|kdestroy                                        rc=16 (les 4)
man -w  krb5.conf                                                        rc=0
man -w  kerberos                                                         rc=0
rpm -ql krb5-libs | grep /man/  → krb5.conf.5.gz, kerberos.7.gz          rc=0
```
**Le premier relevé de ce bloc était faux et a été repris.** Il affichait `rc=0` pour
tous les paquets : c'était le code de `head -1` en fin de tube, pas celui de `rpm -q`.
Repris en `out=$(rpm -q "$p" 2>&1); rc=$?`. Sans cette reprise, `krb5-workstation` aurait
été noté présent. C'est le piège que R-03 vise, rencontré pendant la rédaction même de la
procédure qui l'enseigne. Fonde F-052.

**P-31** — Paquet fournisseur des outils Kerberos :
```
dnf repoquery --qf '%{name}-%{version}-%{release} repo=%{reponame}' krb5-workstation
  → krb5-workstation-1.22.2-2.fc44 (Fedora 44) / -4.fc44 (Updates)       rc=0
dnf repoquery -l krb5-workstation | grep -E '/(klist|kinit|kvno|kdestroy)$|man[0-9]/…'
  → /usr/bin/{kdestroy,kinit,klist,kvno}
    /usr/share/man/man1/{kdestroy,kinit,klist,kvno}.1.gz                 rc=0
```
**Basculement d'outil : aucun, et c'est le fait notable.** `dnf repoquery -l` est l'outil
qui avait dépassé le délai et été interrompu en P-12b (`rc=143`). Retenté ici sous
`timeout 90`, **il a abouti, rc=0**. L'échec de P-12b ne s'est pas reproduit ; noté parce
que l'inverse l'avait été. Fonde F-053.

**P-32** — **Piège de code de retour mesuré : `openssl s_client` et `curl` face à une
chaîne non validable.** Serveur TLS monté localement pour la mesure, sur un port non
privilégié, puis arrêté :
```
openssl req -x509 -newkey rsa:2048 -nodes -keyout k.pem -out c.pem \
  -subj '/CN=localhost' -days 1                                         rc=0
openssl s_server -accept 44330 -cert c.pem -key k.pem -www &   (arriere-plan)
ss -ltn | grep :44330                                                    rc=0

openssl s_client -connect localhost:44330 -servername localhost          rc=0   ← 0 !
  → verify error:num=18:self-signed certificate
  → Verify return code: 18 (self-signed certificate)
openssl s_client … -verify_return_error                                  rc=1
  → mêmes deux lignes

curl -s -o /dev/null -w 'http_code=%{http_code}' https://localhost:44330/
  → http_code=000                                                        rc=60
curl … --cacert c.pem --resolve localhost:44330:127.0.0.1 …
  → http_code=200                                                        rc=0
kill <s_server>                                                          rc=0
```
**`rc=0` alors que la validation a échoué.** Troisième membre de la famille F-041 / P-25.
Fonde F-054 et F-057.
*Réserve de méthode :* un processus serveur, même bref, est un état créé sur le poste.
Aucun privilège, aucun paquet installé, aucun fichier hors du répertoire temporaire de
session.

**P-33** — **Pièges de code de retour mesurés : résolution de noms et heure.**
```
dig +short <nom inexistant>.invalid A        rc=0   sortie vide      ← 0 sur NXDOMAIN
dig +short localhost A                       rc=0
getent hosts <nom inexistant>.invalid        rc=2
getent hosts localhost                       rc=0
man dig | sed -n '/^EXIT/,/^FILES/p'         → aucune ligne : pas de section EXIT STATUS
man getent | sed -n '/EXIT STATUS/,…'        → « 2 : One or more supplied key could not
                                                 be found in the database »

timedatectl timesync-status                  rc=1
  → « Command requires systemd-timesyncd.service, but it is not available: … »
systemctl is-active chronyd                  rc=0   → active
chronyc tracking                             rc=0   (sans élévation)
timedatectl show -p NTPSynchronized --value  rc=0   → yes
```
Les deux zéros de `dig` ne disent pas la même chose que ceux de `getent` : l'un est un
défaut de conception du code de retour, l'autre une absence documentée. Le `rc=1` de
`timesync-status` n'a rien à voir avec l'heure. Fonde F-055 et F-056.

**P-34** — *(livrable 5)* Récupération de la source **S-6** sur la protection étendue de
l'authentification, lue intégralement en texte brut :
```
curl -s -o epa-iis.md -w 'http_code=%{http_code} size=%{size_download}' \
  https://raw.githubusercontent.com/MicrosoftDocs/iis-docs/main/iis/configuration/\
  system.webServer/security/authentication/windowsAuthentication/extendedProtection/index.md
http_code=200 size=17011                                                 rc=0
wc -l → 213                                                              rc=0
```
Extraits décisifs, cités en F-062 et dans PO-018. Fichier de travail conservé hors du
dépôt (répertoire temporaire de session), non versionné.

---

*Preuves ajoutées le **2026-09-22** (livrable 6). Poste Fedora de rédaction, **toutes sans
élévation**. Aucune commande jouée sur une cible (R-08).*

**P-35** — **Clone de `rhel_post_install`, en lecture seule.** Emplacement choisi :
`/home/<user>/dev/rhel_post_install`, **hors de l'arbre de `scomm_rhel9`** et voisin des
autres dépôts. L'emplacement était libre avant (`ls` → rc=2).
```
GIT_TERMINAL_PROMPT=0 git clone git@github.com:charMlessMan80/rhel_post_install.git   rc=0
git -C <clone> remote -v   → git@github.com:… (fetch et push)                         rc=0
git -C <clone> rev-parse HEAD            → a1012bf6a9962ba480eb9ea0f75b7871f8caeb4d   rc=0
git -C <clone> rev-parse --abbrev-ref HEAD → main                                     rc=0
git -C <clone> rev-list --count HEAD     → 42                                         rc=0
git -C <clone> log -1 --date=iso         → 2026-09-15 14:37:39 +0200
                                            « Update host entry in hosts.ini »        rc=0
```
**URL en `git@github.com:` et non `https://`** : le remote `https://` de `scomm_rhel9` est
dépourvu de `credential.helper` (F-005, PO-007), alors que l'authentification SSH du
compte fonctionne (P-25). Le clone a abouti du premier coup, sans invite.

**Preuve de non-modification, relevée après toutes les lectures :**
```
git -C <clone> status --porcelain -uall   → AUCUNE ligne                              rc=0
git -C <clone> rev-parse HEAD             → a1012bf6a9962ba480eb9ea0f75b7871f8caeb4d  rc=0
git -C <clone> stash list                 → AUCUNE ligne                              rc=0
git -C <clone> branch -a                  → main, remotes/origin/{HEAD,main}          rc=0
git -C <clone> reflog                     → UNE SEULE entree :                        rc=0
  a1012bf HEAD@{2026-09-22 …}: clone: from github.com:charMlessMan80/rhel_post_install.git
```
Identiques à l'état du clone. **Aucune écriture, aucun commit, aucune branche, aucun
fichier créé** dans ce dépôt (R-09). **Le `reflog` ne porte qu'une entrée, le clone
lui-même** : aucune autre opération git n'a jamais écrit dans ce dépôt. C'est la preuve
la plus directe, parce qu'elle ne dépend pas de l'état de l'arbre de travail mais de
l'historique des mouvements de `HEAD`.

**Étiquette de provenance, à reporter sur toute affirmation qui en découle :**
`[DÉPÔT-PUBLIC-2026-09-22 @ a1012bf]`. C'est l'état du **dépôt public au 2026-09-22**, et
non l'état de ce qui a été joué sur les machines. Les deux peuvent diverger — et
l'étape 1 de ce livrable montre qu'ils divergent effectivement.

**P-36** — **Recherche de l'origine de l'entrée de bouclage, avec contrôle positif de
chaque motif.** Aucune absence n'est affirmée sans que le motif ait d'abord été vu
trouver quelque chose de réel (R-03).

| Motif | Résultat | Contrôle positif du **même** motif |
|---|---|---|
| `git grep -- '/etc/hosts'` (HEAD) | 4 lignes : `README.md:12`, `set_hostname.yml:6`, `:8`, `audit.rules.j2:62` — rc=0 | `git grep -c -- '/etc/'` → 8 fichiers, rc=0 |
| `git log --all -S'/etc/hosts'` | 3 commits : `e34b04c`, `ed1176b`, `5992c5c` — rc=0 | `-S'ad_domain' -- tasks/set_hostname.yml` → `71698af`, rc=0 |
| `path:`/`dest:` de tous les modules d'écriture | **une seule** occurrence de `/etc/hosts`, dans `set_hostname.yml` — rc=0 | la même commande liste 27 autres chemins réels (`/etc/sssd/sssd.conf`, `/etc/cepces/cepces.conf`, …) |
| `127\.0\.0\.1` dans `templates/*` | **aucune** — rc=1 | le même motif sur tout l'arbre → `set_hostname.yml:10`, rc=0 |
| `\bsed\b` à HEAD | **aucun** — rc=1 | le même motif sur `e34b04c^:shell/ad_pki.sh` → 5 lignes, rc=0 |
| fichiers jamais supprimés | `tmp.sh` (`c70d5d1`) et `shell/ad_pki.sh` (`e34b04c`) — rc=0 | `tmp.sh` inspecté : 60 lignes, aucune mention de `/etc/hosts` ni de `127.0.0.1` (rc=1 sur le motif, rc=0 sur le contenu) |

**Le premier contrôle positif de `sed ` a échoué et a été repris.** Le motif `sed `
(avec espace) trouvait `used `, `caused `, `instead ` — il « trouvait », mais sur des
sous-chaînes : **il n'établissait rien.** Repris en `\bsed\b`, qui rend `rc=1` à HEAD et
`rc=0` sur le script retiré. Sans cette reprise, l'absence de `sed` à HEAD aurait été
affirmée sur un contrôle qui ne contrôlait pas.

**Lignes décisives, citées littéralement.**
`tasks/set_hostname.yml:6-11`, à HEAD :
```yaml
- name: Ensure hostname is in /etc/hosts
  ansible.builtin.lineinfile:
    path: /etc/hosts
    regexp: '^\s*127\.0\.0\.1'
    line: "127.0.0.1  {{ inventory_hostname }}.{{ ad_domain }} {{ inventory_hostname }}"
    state: present
```
Historique de cette seule ligne (`git log --follow`, rc=0) :
```
5992c5c 2026-07-07 Project init                 line: "127.0.0.1  {{ inventory_hostname }}"
71698af 2026-07-13 Added FQDN to resolve loopback
                                                line: "127.0.0.1  {{ …}}.{{ ad_domain }} {{ … }}"
1e2e561 2026-08-20 Lint for tasks usage         (inchangée)
f0c355a 2026-08-20 Use FQCN module names        (inchangée)
```
`shell/ad_pki.sh:91`, **supprimé** par `e34b04c` le 2026-07-08 :
```bash
sed -i -E "s|^\s*127\.0\.0\.1.*|127.0.0.1 ${INVENTORY_HOSTNAME} ${FQDN}|" /etc/hosts
...
echo "127.0.0.1  ${INVENTORY_HOSTNAME} ${FQDN}" >> /etc/hosts
```

**P-37** — Sémantique de `lineinfile`, lue sur ce poste (`ansible-core 2.20.7`, F-022) :
```
ansible-doc ansible.builtin.lineinfile | sed -n '/^   regexp/,/^   [a-z]/p'          rc=0
  → « For `state=present', the pattern to replace if found.
      Only the last line found will be replaced. »
```
**⚠ Transposition :** la cible emploie `ansible-core 2.14` (F-072). @VERIF : rejouer
`ansible-doc` sur le seed avant de tenir cette sémantique pour acquise sur la cible.

---

*Preuves ajoutées le **2026-09-23** (livrable 7). **Aucune n'a été produite sur ce
poste** : toutes viennent de machines auxquelles l'agent n'a pas accès (R-08).*

> **Réserve qui s'applique à P-38 → P-42, et qu'il faut lire avant elles.**
> Ce qui m'a été transmis est une **description** de ces mesures, pas leur sortie
> littérale. Je n'ai reçu ni capture d'écran, ni texte brut de `stdout`, ni code de
> retour relevé caractère par caractère — sauf là où l'énoncé en cite un explicitement.
> **Aucune sortie n'est donc reproduite ici**, parce que la reproduire supposerait de
> l'inventer (R-01). Ces entrées consignent **ce qui a été affirmé de la mesure**, avec
> le nom de la commande employée. Elles sont plus faibles qu'un `[ÉCRAN-…]` complet et
> devront être remplacées par la capture si l'une d'elles est un jour contestée.

**P-38** — `[ÉCRAN-2026-09-23]` **Identité effective sur le seed**, en session
interactive, sur `seed01`. Commandes : `id -un` et `klist -s; echo $?`.
Résultat relayé : `id -un` rend **le compte AD de l'opérateur** ; `klist -s` rend **0**,
donc un ticket valide est présent.
**Ce que cela établit :** l'identité sous laquelle une session interactive s'exécute sur
le seed. **Ce que cela n'établit pas :** sous quel compte `ansible-playbook` se connecte
à la cible — `ansible_user` n'a pas été relevé (PO-025). Fonde F-074, infirme F-059.

**P-39** — `[TIERS-MESURÉ]` **SPN du VIP**, sortie `setspn` produite par l'équipe ADCS et
transmise. Résultat relayé : `HTTP/<VIP-FQDN>` est enregistré **sur le compte de service
qui exécute le pool d'applications CES**.
**Statut de la preuve :** c'est une mesure, mais **ni rejouable ni vérifiable en
contexte** par l'agent — ni la commande exacte, ni le contrôleur interrogé, ni l'horodatage
ne sont connus. **Ce que la mesure ne couvre pas, et qui n'a pas été demandé :** que ce
même SPN ne soit **pas** enregistré sur un autre compte. Un doublon empêcherait l'émission
de tickets de service. Fonde F-076.

**P-40** — `[ÉCRAN-2026-09-23]` **Appartenance de `host01$` au groupe d'enrôlement**,
lectures directes de l'annuaire depuis `seed01`, **contre un seul et même contrôleur de
domaine** — ce qui écarte la réplication comme explication d'un écart avant/après.
Séquence relayée, dans l'ordre :
1. **témoin positif** — la requête affiche bien l'appartenance d'un compte qui en a ;
2. **avant** — `host01` n'a **aucune** appartenance directe ;
3. **écriture** — ajout de `host01$` à `<GROUPE-T>`, sous `<COMPTE-JONCTION>`, dans un
   **cache Kerberos dédié** ; le ticket de session de l'opérateur a été **vérifié intact
   avant et après** ;
4. **après** — `host01` est membre **direct** de `<GROUPE-T>` ;
5. **imbrication** — une règle de recherche qui suit les imbrications renvoie `host01`
   pour `<GROUPE-PKI>`, dont la machine **n'est pas** membre directe.
**Ce que l'étape 5 établit deux fois :** la chaîne `<GROUPE-T>` → `<GROUPE-ALL>` →
`<GROUPE-PKI>`, **et** le fait que la règle employée parcourt réellement les
imbrications — sans quoi le résultat aurait été vide. Le témoin (1) et l'état avant (2)
séparent « la machine est membre » de « la requête ne sait pas répondre ».
**Le garde-fou 1 du livrable 5 a été appliqué** : cache dédié, cache par défaut vérifié
intact des deux côtés de l'écriture.
Fonde F-080 à F-082.

**P-41** — `[ÉCRAN-2026-09-23]` **Pages de manuel d'`adcli` et sortie de
`adcli show-computer`**, lues sur `seed01`. Points relayés :
- `adcli add-member` attend le nom du compte machine **avec** `$` ; `adcli show-computer`
  attend le nom court **sans** `$` (exemples des pages de manuel) ;
- `adcli show-computer` affiche une **liste fixe** d'attributs, y compris vides ;
  **l'appartenance aux groupes n'en fait pas partie** ;
- le **retrait** d'un compte machine d'un groupe **n'est pas documenté** : le synopsis du
  retrait ne mentionne que des utilisateurs ;
- `adcli show-computer` affiche la **version de clé** du compte machine dans l'annuaire ;
- les SPN du compte machine incluent la forme `host/<FQDN>`.
Fonde F-084 à F-086.

**P-42** — `[ÉCRAN-2026-09-23]` **Dérive constatée sur le seed.** `openldap-clients` a été
installé **à la main** sur `seed01`, depuis le dépôt de base de RHEL 9, pour les lectures
de P-40. **Cette installation n'est décrite dans aucun code** — ni dans
`rhel_post_install`, ni ailleurs. Fonde PO-026.

---

*Preuves ajoutées le **2026-09-24** (livrable 11). **Aucune n'a été produite sur ce
poste** (R-08).*

> **Réserve de transmission — elle gouverne P-43 à P-47 et ne doit pas être oubliée.**
> Ces mesures ont été **jouées par l'opérateur**, **lues sur capture par le pilote**, et
> **transmises ici décrites**. Je n'ai vu aucune sortie : ni texte brut, ni capture, ni
> code de retour relevé caractère par caractère. **Aucune sortie n'est donc reproduite**,
> parce que la reproduire supposerait de l'inventer (R-01). Ces entrées consignent ce qui
> a été **affirmé** de la mesure, et le nom de ce qui a été mesuré. Étiquette :
> `[ÉCRAN-2026-09-24, relayé]` — plus faible qu'une capture, laquelle ne pourrait entrer
> dans ce dépôt public qu'expurgée (R-07).

**P-43** — `[ÉCRAN-2026-09-24, relayé]` **Étape 0b.2 sur `host01`** : pages de manuel des
quatre outils Kerberos, lues sur la cible. Fonde F-089.

**P-44** — `[ÉCRAN-2026-09-24, relayé]` **Étapes 1, 1b et 2a/2b sur `host01`** : nom,
résolution par quatre voies distinctes, horloge ; puis keytab (propriétaire, mode,
contenu, versions de clé) et obtention du ticket initial, **avec une élévation déclarée**.
Fonde F-090 à F-096.

**P-45** — `[ÉCRAN-2026-09-24, relayé]` **Étapes 3 et 4 sur `host01`** : ticket de service
pour `HTTP/<VIP_FQDN>`, puis chaîne TLS présentée par le VIP et verdict du magasin de
confiance de la machine. Fonde F-097 à F-099.

**P-46** — `[ÉCRAN-2026-09-24, relayé]` **Établissement de la racine depuis `seed01`**,
par interrogation de l'annuaire sur un canal authentifié par Kerberos, puis comparaison
d'empreintes et vérification de signature. Fonde F-100 à F-102.

**P-47** — `[ÉCRAN-2026-09-24, relayé]` **Étape 5 et ses variantes**, sur `host01` et sur
`seed01` : requêtes anonymes et authentifiées, contre CEP et CES, par le F5 et en direct,
en HTTPS puis en HTTP, avec décodage du jeton SPNEGO. Fonde F-103 à F-110.
S'y ajoutent, de provenance distincte : `[TIERS-MESURÉ]` la configuration et le journal
d'IIS relevés par l'équipe ADCS, `[TIERS-DÉCLARÉ]` sa lecture, et `[SOURCE-EXTERNE]` S-7.

---

*Preuves ajoutées le **2026-09-29**. **P-48 à P-54 n'ont pas été produites sur ce poste**
(R-08) ; **P-55 l'a été**.*

> **Réserve de transmission — elle gouverne P-48 à P-54, à l'identique de P-43 à P-47.**
> Mesures **jouées par l'opérateur** sur la recette, **lues sur capture par le pilote**,
> et **transmises ici décrites** ; ou sorties **transmises par l'administrateur ADCS**. Je
> n'ai vu aucune sortie : **aucune n'est reproduite**, la reproduire supposerait de
> l'inventer (R-01). Étiquettes : `[ÉCRAN-AAAA-MM-JJ, relayé]` pour les écrans de
> l'opérateur, `[TIERS-MESURÉ]` pour les sorties de l'administrateur.
> **Heures en UTC.** Les en-têtes `Date` HTTP et le journal IIS sont en UTC ;
> l'observateur d'événements de l'administrateur est en heure locale **+02:00**.

**P-48** — `[ÉCRAN-2026-09-25, relayé]` + `[TIERS-MESURÉ]` **Test du 2026-09-25 vers 13:22
UTC.** Requêtes CEP depuis `host02` sous `host02$` ; requête CES depuis **une autre
machine de recette** ; lignes du journal IIS et événement 4625 sur `<SERVEUR-ADCS>`,
transmis par l'administrateur. Relevé de l'opérateur : l'historique shell de l'autre
machine contient les commandes CES, et les deux machines ont des adresses différentes.
Fonde F-115 et la part « traduction d'adresse » de F-118.
**Libellés réduits (R-07), introduits ici :** `host02` (machine de recette des
2026-09-25 à 28, **distincte de `host01`**), `<DOMAINE>` (préfixe de domaine du champ
`cs-username`), `<SERVEUR-ADCS>`, `<CHEMIN-CEP-PUBLIÉ>`, `<CHEMIN-CES-PUBLIÉ>` (chemins
publiés par le F5). Déjà en usage : `<CA-N3>`, `<CA-N2>`, `<ROOT-CA>`, `<COMPTE-SVC-CES>`,
`<VIP_FQDN>`. **Aucune adresse IP n'est écrite.**

**P-49** — `[ÉCRAN-2026-09-25, relayé]` + `[TIERS-MESURÉ]` **Test du 2026-09-25 à
15:33:15 UTC**, depuis `host02`, sous `host02$`, `curl -v --stderr <fichier>` : CES sans
ticket, CES avec ticket, CEP avec ticket ; lignes du journal IIS correspondantes.
Compteur `grep -c '^> Authorization: Negotiate'` : **0** sur la requête sans ticket,
**1** sur chacune des deux requêtes avec. Fonde F-116 et la révision de PO-040.

**P-50** — `[ÉCRAN-2026-09-28, relayé]` **Test du 2026-09-28, de 13:39:05 à 13:39:11
UTC**, depuis `host02`, un seul ticket, CEP, CES, CEP, CES, plus un témoin sans ticket.
Compteur : **0** sur le témoin, **1** sur chacune des quatre requêtes avec ticket. Fonde
F-117 et la révision de PO-040.

**P-51** — `[ÉCRAN-2026-09-28, relayé]` **Paquets sur `host02`.** Version de RHEL ;
disponibilité et version de `cepces`, `cepces-certmonger`, `cepces-selinux`,
`python3-gssapi`, `python3-requests-gssapi` et `certmonger` dans AppStream ; échec de
connexion à EPEL ; contenu de `cepces.conf` lu **avant installation** par `rpm2cpio` ;
texte du `%post` de `cepces-certmonger` ; `getcert list-cas -c cepces` vide après
installation ; aide de `cepces-submit`. Fonde F-119 à F-121.

**P-52** — `[ÉCRAN-2026-09-28, relayé]` **Dérive constatée sur `host02`**, sur le modèle de
P-42 : les cinq gestes faits à la main — `/etc/hosts` et sa sauvegarde, ancre
`<ROOT-CA>` et `update-ca-trust`, transactions `dnf` 5 et 6, `cepces.conf` et sa
sauvegarde, `getcert add-ca`. Relevés associés : `hostname -f` après correction ;
`curl` sans `--cacert` → `ssl_verify=0` ; historique `dnf` (transaction précédente : 4).
**Aucun de ces gestes n'est décrit dans un code.** Fonde F-122 ; autorisés par D-014.

**P-53** — `[ÉCRAN-2026-09-28, relayé]` **Premier appel de `cepces-submit`**
(`CERTMONGER_OPERATION=GET-SUPPORTED-TEMPLATES`) : échec du principal en minuscules,
réussite en majuscules, `ParseError`, `rc=4`, aucun refus SELinux. Puis **rejeu du `POST`
par `curl`**, 2026-09-28 à 14:29:01 et 14:29:03 UTC, deux valeurs de l'en-tête `To` :
`HTTP/1.1 500 System.ServiceModel.ServiceActivationException`, `Content-Length: 0`,
`Persistent-Auth: true`. Compteur : **1** sur chacune des deux requêtes. Fonde F-123,
F-124 et la révision de PO-040.

**P-54** — `[TIERS-MESURÉ]` **Configuration IIS relevée par l'administrateur ADCS le
2026-09-28** : sorties `appcmd` des deux applications et capture des deux pools. S'y
ajoutent, sans sortie, `[TIERS-DÉCLARÉ]` : l'ancienne valeur `Require`, le redémarrage
d'IIS après le test de 13:22, et « rien de pertinent » dans les journaux d'événements.
**Statut de la preuve :** une mesure, mais ni rejouable ni vérifiable en contexte par
l'agent. Fonde F-111 à F-114.

**P-55** — **Lecture des sources S-8 à S-10**, le **2026-09-29 à 07:14:37 UTC**, sur le
poste de rédaction, sans élévation, dans le répertoire temporaire de la session (hors
dépôt). Trois commandes de la forme
`curl -sS -L -o <fichier> -m 30 -w '… http_code=%{http_code} size=%{size_download}\n' <URL>` :
```
s8 http_code=200 size=27963    rc_curl_s8=0
s9 http_code=404 size=14876    rc_curl_s9=0
s10 http_code=200 size=74186   rc_curl_s10=0
```
Conversion en texte par `sed`, puis recherche par `grep -n` (`rc=0` pour S-8 et S-10).
Lignes citées : S-8, lignes 1, 8, 11 et 12-13 du texte extrait ; S-10, lignes 644, 657
et 663-670. **S-9 non lue** : `404`, titre de la page `GFI Support`. Fonde S-8 à S-10
(F-116, F-125).
