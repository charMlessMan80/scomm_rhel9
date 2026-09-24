# Procédure de diagnostic — préalables à un enrôlement CEP/CES

**Ce que ce fichier porte :** des blocs de commandes que **l'opérateur** joue lui-même
sur la machine de recette, dans l'ordre, et la lecture de chaque sortie.

## Ordre de jeu — le tableau à garder sous les yeux

| # | Étape | Machine | Identité | Un échec arrête-t-il la suite ? |
|---|---|---|---|---|
| 0 | Les outils sont-ils là | recette | session | **Oui** |
| 0b.1 | Paquets et versions | recette | session | **DÉJÀ FAITE** — F-087 |
| 0b.2 | Options sourcées sur la cible | recette | session | **Oui**, et elle précède tout le reste |
| 1 | Contexte : nom, résolution, heure | recette | session | **Non** pour l'écart **connu** ; **oui** pour tout autre |
| 2a | Accès au keytab | recette | session, `sudo` si besoin | **Oui** |
| 2b | Ticket initial de la machine | recette | **`<MACHINE>$`** | **Oui** |
| 3 | Ticket de service pour le VIP | recette | **`<MACHINE>$`** | **Oui** — la suite serait ininterprétable |
| 4 | Ce que le VIP présente en TLS | recette | aucune | **Oui** |
| 5 | Requête authentifiée sur CEP | recette | **`<MACHINE>$`** | dernière étape |
| 6 | Nettoyage du cache dédié | recette | session | **obligatoire, toujours jouée** |
| T | Témoin de contraste | **seed** | `<OP_AD>` | facultatif, seulement si 3 ou 5 échoue |
| K | Version de clé du compte machine | **seed** | `<OP_AD>` | séparateur, seulement si 2b échoue |

**Sous quelle identité :** celle de **la machine de recette**, depuis son keytab, dans un
cache dédié. Le ticket de l'opérateur n'y a aucun rôle, sauf en T et K. Matrice complète
au § 1 bis — **lisez-la avant de jouer**, faute de quoi vous tirerez d'un succès une
conclusion qui ne s'y trouve pas.

## Forme des commandes — une règle née d'une exécution cassée

**Une commande complète par ligne.** Pas de continuation `\`, pas de boucle, pas de tube
dont le code de retour importe sans `${PIPESTATUS[n]}` explicite. Chaque ligne se colle
seule et se suffit.

*Motif :* sur la recette, un bloc de l'étape 0b comportant des continuations et une boucle
a été collé dans un terminal. **Le collage l'a découpé** : `bash` a signalé une erreur de
syntaxe puis une redirection ambiguë, et le seul code de retour affiché fut celui de cet
échec. **Rien n'a été mesuré.** Deux autres collages multi-lignes ont ensuite produit des
fragments parasites. Une commande qui demande une saisie interactive est isolée et le dit.

**Ce que ce fichier NE porte PAS :** aucun fait, aucune décision, aucun point ouvert —
ils vivent dans `../scomm-certificat-facts.md`. Aucune réparation. Aucune tâche Ansible.

**Ce que cette procédure ne fait pas, et c'est une propriété, pas une précaution :**
elle **n'installe rien**, **ne configure rien**, **ne demande aucun certificat**, et
**ne modifie aucun magasin de confiance**. Le seul état qu'elle crée est un cache de
tickets Kerberos dédié, détruit à l'étape 6.

> **Aucune commande de ce fichier n'a été exécutée par l'agent qui l'a écrite.** Les
> machines cibles ne sont pas accessibles depuis le poste de rédaction (R-08). Ce qui a
> été exécuté, ce sont des lectures de pages de manuel et deux mesures locales de codes
> de retour : **F-051 à F-057**, preuves **P-29 à P-33**.

---

## 0. Avertissement de transposition — à lire avant de jouer quoi que ce soit

Les options employées ci-dessous sont sourcées sur un poste **Fedora 44** (F-051). **La
cible est RHEL 9 et les versions y diffèrent**, désormais mesurées : `krb5-workstation
1.21.1`, `openssl 3.5.5`, `curl 7.76.1`, `bind-utils 9.16.23` (F-087). Une option
présente ici peut être absente là-bas. Chaque bloc porte ses sources ; là où la source
manque, la commande porte le marqueur de vérification et dit où la confirmer.

**Les options de `klist`, `kinit`, `kvno` et `kdestroy` n'ont pas pu être sourcées** sur
le poste de rédaction (F-052) : elles portent toutes le marqueur de vérification, et
c'est le cœur des étapes 2 et 3. **L'étape 0b.2 les source sur la cible**, et lève cette
dette — pour ces quatre outils seulement. Pour `openssl`, `curl`, `dig`, `getent`,
`timedatectl` et `chronyc`, les options restent sourcées sur Fedora ; 0b.1 en a au moins
relevé les versions réelles (F-087).

---

## 1. Valeurs à substituer avant de jouer

Aucune valeur réelle ne figure dans ce dépôt, qui est public (R-07, F-024). Remplacez
les marqueurs ci-dessous **dans votre terminal**, pas dans ce fichier.

| Marqueur | Ce que c'est | Où le prendre |
|---|---|---|
| `<VIP_FQDN>` | Nom DNS pleinement qualifié du VIP F5 devant l'ADCS | Courriel de l'équipe SCOM/sécurité (F-045) |
| `<CEP_URL>` | URL du point d'entrée **CEP** (service de politique) | Courriel de l'équipe SCOM/sécurité (F-045) |
| `<CES_URL>` | URL du point d'entrée **CES** — *non employée par cette procédure*, listée pour que son absence soit visible | Courriel de l'équipe SCOM/sécurité (F-045) |
| `<REALM>` | Realm Kerberos, en majuscules | Configuration du domaine ; `realm list` sur la machine |
| `<GABARIT>` | Nom du gabarit ADCS — *non employé par cette procédure* | Courriel de l'équipe SCOM/sécurité (F-045) |
| `<OP_AD>` | Compte AD de l'opérateur — employé **uniquement** par le témoin de contraste (§ 8 bis) | La session ouverte sur le seed |
| `<MACHINE>` | Nom court de la machine de recette ; son principal est `<MACHINE>$@<REALM>` | `hostname` sur la recette (étape 1) |
| `<MACHINE-FQDN>` | Nom **pleinement qualifié attendu** de la recette. À composer à la main : la machine ne le rend pas (voir étape 1) | Le domaine AD, en minuscules |
| `<ADRESSE-INTERFACE>` | Adresse de l'interface, **relevée à l'étape 1** et retapée à l'étape 1b | Sortie de `getent hosts <MACHINE-FQDN>` |

`<CES_URL>`, `<GABARIT>` et `<REALM>` sont listés **et délibérément inutilisés par les
commandes** : la procédure s'arrête avant toute demande de certificat, et `kinit -k`
prend le realm par défaut dans `/etc/krb5.conf` plutôt que de se le faire dicter. Ils
figurent dans la table pour que leur absence soit **visible** : si l'un d'eux devenait
nécessaire ici, c'est que la procédure aurait changé de nature. `<REALM>` sert en
revanche à la lecture — c'est lui qu'on compare aux textes d'erreur de l'étape 2.

**Un cache de tickets dédié**, employé par les étapes 2, 3 et 5 :

```
CC=/tmp/diag-enrolement-$$.ccache
```

*Motif :* `kinit` écrit par défaut dans le cache du compte courant. L'y laisser écraser
le ticket de l'opérateur serait **une modification d'état**, dans une procédure qui n'en
admet aucune.

> **Garde-fou 1 — la destruction ne vise QUE `$CC`.** Un `kdestroy` sans `-c` détruit le
> cache **par défaut**. Sur le seed, celui-ci porte le ticket que SSSD a posé au login et
> que l'opérateur **n'a pas l'habitude de reconstituer** (F-058) : sa perte casserait des
> choses sans rapport visible avec ce diagnostic, et le lien ne serait pas fait. **Toute
> commande `kdestroy` de cette procédure porte `-c "FILE:$CC"`. Aucune exception.**
>
> **Garde-fou 2 — le ticket du cache dédié a une échéance.** Elle est relevée au moment
> de l'obtention (étape 2) et **revérifiée avant l'étape 5**. Une expiration en cours de
> diagnostic produit un échec qui **ressemble à un refus d'authentification** : un `401`
> sans cause réelle, cherché pendant des heures du mauvais côté.

---

## 1 bis. Sous quelle identité, sur quelle machine — et ce que chacune établit

**Trois identités, qui n'établissent pas la même chose.** Les confondre fait tirer d'un
succès une conclusion qui ne s'y trouve pas.

| Identité | Ce qu'elle est | Ce qu'elle peut prouver |
|---|---|---|
| **`<OP_AD>`** — compte AD de l'opérateur | Ticket obtenu **automatiquement au login** par SSSD sur le seed, validité 8 h, sans `kinit` manuel (F-058) | Que la **chaîne** répond à *un* principal du domaine : SPN existant, F5 traversé, IIS authentifiant. **Rien** sur le compte machine. |
| **`ansible_admin`** — compte **de la cible**, non du seed | `[RÉVISÉE le 2026-09-23]` **ÉTAIT :** « compte local PAM du seed — aucune identité dans l'annuaire, donc aucun ticket et aucun moyen d'en obtenir (F-059) ». **F-059 est infirmé** : sur le seed, l'opérateur est sous son compte AD, avec ticket (**F-074**, mesuré). `ansible_admin` est le compte sous lequel on agit **sur la cible** (`preflight.yml:24,29`). | Rien, en Kerberos : la tâche d'enrôlement n'a aucune identité à prendre, elle parle au démon `certmonger` local (PO-020, requalifié). |
| **`<MACHINE>$`** — compte machine de la recette | Secret dans `/etc/krb5.keytab`, créé par la jonction au domaine (F-046) | **Seul concerné par le groupe d'enrôlement, et seul qui enrôlera réellement.** C'est l'identité que cette procédure mesure. |

### Matrice, étape par étape

| Étape | Machine | Identité | Un résultat conforme établit, **pour cette identité** | Il n'établit pas, **pour les autres** |
|---|---|---|---|---|
| 0 — outillage | recette | `<OP_AD>` (session) | Les binaires sont présents **pour tout compte** | — *(l'outillage ne dépend d'aucune identité)* |
| 0b — sourcing | recette | `<OP_AD>` (session) | Ce que font les options **sur la cible** | Rien sur la chaîne d'enrôlement |
| 1 — contexte | recette | `<OP_AD>` (session) | Nom, résolution et heure de **la machine** | — *(aucune identité en jeu : pas d'authentification)* |
| 2 — keytab + ticket initial | recette | **`<MACHINE>$`**, via `kinit -k` | Le compte machine s'authentifie auprès du KDC | Ne dit rien de `<OP_AD>`, dont le ticket vient d'ailleurs et existe déjà |
| 3 — ticket de service | recette | **`<MACHINE>$`**, cache dédié | Le SPN existe **et** le compte machine peut en obtenir un ticket | Ne dit rien du droit d'enrôler — c'est une ACL, pas un ticket |
| 4 — TLS du VIP | recette | **aucune** | Ce que le VIP présente, et ce que **le magasin de la machine** valide | Ne dit rien de ce que valide le seed, ni `cepces` |
| 5 — CEP authentifié | recette | **`<MACHINE>$`**, cache dédié | IIS **authentifie le compte machine** à travers le F5 | Ne dit rien de l'autorisation (voir le `403`), ni de CES |
| 6 — nettoyage | recette | `<OP_AD>` (session) | Le cache dédié a disparu | — |
| **T1** — témoin, ticket de service | **seed** | **`<OP_AD>`** | Le SPN répond à *un* principal du domaine | **Ne valide rien** pour le compte machine |
| **T2** — témoin, CEP authentifié | **seed** | **`<OP_AD>`** | La chaîne authentifie *un* principal du domaine | **Ne valide rien** pour le compte machine |
| **K** — version de clé du compte machine | **seed** | **`<OP_AD>`** | Ce que l'**annuaire** porte comme version de clé : `adcli show-computer <MACHINE>` (sans `$`, F-084). Comparée à celle du keytab (2a), elle tranche « identité périmée » (F-085) | Ne dit rien du ticket ni des droits. **Se joue sur le seed** : `adcli` y est installé, la recette n'est pas supposée le porter (PO-026) |

**Aucune étape n'est sans identité déterminée.** Les trois cases « aucune » de la colonne
identité ne sont pas des trous : l'étape 4 n'authentifie personne, et c'est pour cela
qu'elle peut précéder l'étape 5 sans la contaminer.

**Limite que rien ici ne lève :** un ticket obtenu depuis le keytab **en session
interactive** n'est pas le contexte de `certmonger`. La procédure établit que la chaîne
répond **au compte machine** ; pas qu'un service y parviendra (PO-020).

---

## 2. Ce qui est interdit à qui joue cette procédure

| Interdit | Pourquoi |
|---|---|
| Installer un paquet, même « juste pour voir » | Un outil absent est **un fait à rapporter** (étape 0), pas un obstacle à lever. Le paquet qui le fournit est une information pour le livrable suivant. |
| Demander un certificat (`getcert request`, `certreq`) | Diagnostiquer et agir sont deux actes. Et chaque demande laisse un certificat émis dans la base de la CA, qu'un instantané ne défait pas (PO-017). |
| Modifier le magasin de confiance (`update-ca-trust`, ajout d'ancre) | L'étape 4 mesure **ce que la machine valide aujourd'hui**. Modifier le magasin remplace la mesure par son résultat souhaité. |
| Modifier `/etc/krb5.conf`, `/etc/hosts`, `/etc/resolv.conf` | Les étapes 1 et 3 mesurent la résolution **telle qu'elle est** — y compris son écart connu, que D-012 assume. |
| `curl -k` / `--insecure`, `openssl ... -verify 0` | **C'est l'interdit le plus important.** Le contournement détruit exactement l'information cherchée à l'étape 4 : il transforme un échec de validation, qui est le résultat, en succès apparent. |
| Réparer au milieu du diagnostic | Réparer détruit l'état qu'on mesurait. La procédure s'arrête (§ 9). |

---

## 3. Étape 0 — Les outils sont-ils là ?

**Dépend du droit d'enrôlement : non.** Aucune de ces commandes ne contacte l'annuaire.

**`klist`, `kinit`, `kvno` et `kdestroy` sont déjà établis présents dans `/usr/bin`**
(F-087, 0b.1) : ce bloc ne les revérifie pas.

```
echo "=== DEBUT E0 ==="
command -v openssl; echo "rc_openssl=$?"
command -v curl; echo "rc_curl=$?"
command -v getent; echo "rc_getent=$?"
command -v dig; echo "rc_dig=$?"
command -v timedatectl; echo "rc_timedatectl=$?"
openssl version; echo "rc_openssl_v=$?"
curl --version; echo "rc_curl_v=$?"
echo "=== FIN E0 ==="
```

**Sources.** `command -v` : intégrée POSIX. `curl --version` : `man curl`, `--negotiate`
— « This option requires a library built with GSS-API or SSPI support. Use --version to
see if your curl supports GSS-API/SSPI or SPNEGO. »
**Affichée en entier, non filtrée, délibérément :** un `grep -qi gssapi` échouerait aussi
bien si GSSAPI manquait que si RHEL 9 écrivait `GSS-API` — deux causes, un seul `rc=1`.
**Lisez `Features:` à l'œil** : `GSS-API` ou `SPNEGO` doit y figurer, sans quoi l'étape 5
est impossible.

**Il n'établit pas** que ces outils fonctionnent, ni que `curl` sait dialoguer avec le
KDC. **Si un outil manque : arrêtez ici** et rapportez le paquet qui le fournit —
`krb5-workstation`, `openssl`, `curl`, `bind-utils`, `glibc-common`, `systemd` (F-053,
F-087).

---

## 3 bis. Étape 0b — Sourcer sur la cible les options que le poste de rédaction ne peut pas sourcer

**Dépend du droit d'enrôlement : non.** Lectures de documentation, aucune authentification.

**Pourquoi cette étape existe.** Les options de `klist`, `kinit`, `kvno` et `kdestroy`
portent le marqueur de vérification : ces outils ne sont pas sur le poste de rédaction
(F-052). **Ils sont ici**, et c'est la cible qui fait foi.
**Les étapes 1 à 6 ne se jouent qu'APRÈS cette confirmation.** Si une option n'existe pas
ou ne fait pas ce que la procédure suppose, **arrêtez-vous** : la commande est à
réécrire, pas à lancer quand même.

### 0b.1 — Paquets et versions — **DÉJÀ JOUÉE, ne pas rejouer**

`[ÉCRAN ~2026-09-20]` Résultat inscrit en **F-087** : `krb5-workstation-1.21.1-10.el9_8`,
`openssl-3.5.5-6.el9_8`, `curl-7.76.1-40.el9_8.5`, `bind-utils-9.16.23-40.el9_8.8` ;
`klist`, `kinit`, `kvno`, `kdestroy` présents dans `/usr/bin`.

```
echo "=== DEBUT E0b1 ==="
rpm -q krb5-workstation; echo "rc_krb5=$?"
rpm -q openssl; echo "rc_openssl=$?"
rpm -q curl; echo "rc_curl=$?"
rpm -q bind-utils; echo "rc_bind=$?"
echo "=== FIN E0b1 ==="
```

> **Un `rpm -q` par ligne, et son propre code de retour.** Groupés, un seul `rc` couvre
> quatre paquets et ne dit pas lequel manque. Et le `rc` d'un tube est celui du dernier
> maillon : sur le poste de rédaction, un relevé de ce type a rendu `rc=0` pour tous les
> paquets — c'était le code de `head -1`, et `krb5-workstation` aurait été noté présent
> alors qu'il est absent (P-30).

### 0b.2 — Les options employées, et elles seules

Quatre extractions ciblées. **Pas la page entière** : elle ne tiendrait pas dans une
capture, et ce n'est pas elle qu'on vérifie.

```
echo "=== DEBUT E0b2 ==="
man klist 2>/dev/null | grep -A3 -E '^\s+-k( |$)|^\s+-c( |$)|^\s+-e( |$)'
echo "trouve_klist=$? (0 = le grep a trouve ; 1 = option absente OU motif inadapte)"
man kinit 2>/dev/null | grep -A3 -E '^\s+-k( |$)|^\s+-t( |$)'
echo "trouve_kinit=$?"
man kvno 2>/dev/null | grep -A3 -E '^\s+-c( |$)|^\s+service'
echo "trouve_kvno=$?"
man kdestroy 2>/dev/null | grep -A3 -E '^\s+-c( |$)|^\s+-A( |$)'
echo "trouve_kdestroy=$?"
man kerberos 2>/dev/null | grep -A4 -i 'KRB5CCNAME'
echo "trouve_ccname=$?"
echo "=== FIN E0b2 ==="
```

**Ce que vous cherchez à confirmer, littéralement :**

| Option | Ce que la procédure en suppose | Employée à |
|---|---|---|
| `klist -k <fichier>` | Lister les entrées d'un **keytab**, non d'un cache | 2a |
| `klist -c` / `KRB5CCNAME=FILE:<chemin>` | Lire **un cache désigné**, pas celui par défaut | 2b, 3, 5 |
| `klist -e` | Afficher les **types de chiffrement** | 3, table des indiscernables |
| `kinit -k <principal>` | S'authentifier depuis le keytab **pour un principal nommé** | 2b — **le principal y est désigné, pas déduit** |
| `kinit -c FILE:<chemin>` | Écrire le ticket dans **un cache désigné**, sans passer par la variable d'environnement | **2b branche B**, sous élévation : la variable ne traverse pas `sudo` |
| `kvno <service>/<hôte>` | Obtenir un **ticket de service** pour ce SPN | 3 |
| `kdestroy -c <cache>` | Détruire **ce cache-là**, et lui seul | 6 — **garde-fou 1** |

**Si `rc_ccname` ne trouve rien**, `KRB5CCNAME` peut être documenté ailleurs
(`man krb5.conf`, ou la page `kerberos(7)` d'une autre version). **Cherchez, ne devinez
pas** — et si vous ne trouvez pas, rapportez-le : la procédure emploie cette variable à
trois étapes, et le garde-fou 1 en dépend.

**Ce que ce sourcing règle :** ce que font les options **sur la cible**, dans la version
qui y est installée. Il retire la transposition Fedora → RHEL 9 du chemin critique.

**Ce qu'il NE règle PAS, et il faut le dire :** il établit que les commandes **font ce que
la procédure croit**, pas que la procédure **pose les bonnes questions**. Une commande
correctement sourcée peut mesurer la mauvaise chose. Le sourcing lève une dette de R-01 ;
il ne lève ni R-13 (l'échec doit discriminer) ni la lecture des sorties, qui reste
entière.

---

## 4. Étape 1 — Contexte de la machine : nom, résolution, heure

**Dépend du droit d'enrôlement : non.** Aucune de ces commandes ne s'authentifie.

```
echo "=== DEBUT E1 ==="
hostname -s; echo "rc_court=$?"
hostname -f; echo "rc_fqdn=$?"
getent hosts <MACHINE-FQDN>; echo "rc_getent_fqdn=$?"
getent hosts <VIP_FQDN>; echo "rc_getent_vip=$?"
timedatectl show -p NTPSynchronized --value; echo "rc_ntpsync=$?"
chronyc tracking; echo "rc_chrony=$?"
echo "=== FIN E1 ==="
```

**Étape 1b — résolution inverse, saisie manuelle :** relevez l'adresse rendue ci-dessus
par `getent hosts <MACHINE-FQDN>`, puis retapez-la. Isolée pour cela.

```
echo "=== DEBUT E1b ==="
dig +noall +answer -x <ADRESSE-INTERFACE>
echo "rc_dig_ignore=$? — sans valeur : dig rend 0 même sur NXDOMAIN (F-055). Lisez la sortie."
echo "=== FIN E1b ==="
```

**Sources.** `hostname -f` : `man hostname`, « Display the FQDN » — la page conseille
`--all-fqdns`, écarté parce qu'il en énumère plusieurs sans dire lequel compte, alors que
`-f` est **le** nom que le client Kerberos emploiera.
`getent hosts` : `man getent`, « pass each key to `gethostbyaddr(3)` or
`gethostbyname2(3)` », et sa section `EXIT STATUS` — `2` « One or more supplied key could
not be found in the database ». `timedatectl show` : « the same information as status,
but in machine readable form ». `chronyc tracking` : « parameters about the system's
clock performance », dont `System time : … seconds slow/fast of NTP time`.

**Pourquoi `getent` et non `dig` comme contrôle principal** (F-055) : `getent hosts`
emprunte la voie que les applications empruntent (`nsswitch`), celle que `kinit` et
`curl` suivront ; et son `rc=2` porte l'information, là où `dig` rend `0` même sur un nom
inexistant. `dig` ne sert donc ici qu'à **séparer des causes**, jamais à conclure.

**Pourquoi PAS `timedatectl timesync-status`** (F-056) : sur une machine sous `chronyd`
elle rend `rc=1` avec « Command requires systemd-timesyncd.service » — un échec sans
rapport avec l'heure, qui envoie chercher au mauvais endroit.

### Le résultat attendu, et l'écart qui est attendu lui aussi

**⚠ Sur cette machine, `hostname -f` rend le NOM COURT. C'est normal et connu — ce n'est
pas un motif d'arrêt.** L'entrée locale de résolution associe l'adresse de bouclage au
nom court **puis** au nom qualifié, et les fichiers sont consultés avant le DNS ; c'est
donc le nom court qui ressort (F-064, F-066 — origine datée, PO-021 ouvert).

| Observation | Verdict | Suite |
|---|---|---|
| `hostname -f` rend le **nom court** ; `getent hosts <MACHINE-FQDN>` rend l'adresse de l'interface (`rc=0`) ; 1b rend le **nom qualifié** | **Écart CONNU** — consignez-le, **continuez** | Étape 2, qui désigne le principal explicitement |
| `hostname -f` rend le **nom qualifié** | Conforme | Étape 2 |
| `rc_getent_fqdn` ≠ `0` — le nom qualifié ne se résout pas | **Écart DIFFÉRENT** | **ARRÊT** |
| 1b rend un autre nom, ou rien | **Écart DIFFÉRENT** — la résolution inverse ne concorde plus | **ARRÊT** |
| `rc_getent_vip` ≠ `0` | **ARRÊT** | — |

**La distinction tient en une phrase :** l'écart connu est **local** (le fichier des hôtes
masque le DNS) et le DNS reste juste dans les deux sens ; tout écart qui touche **le DNS
lui-même** est différent, et arrête.

**Décision inscrite au dossier (D-012) :** le diagnostic se joue **avant** la correction
de PO-021. Seule l'étape 2 dépendait de ce nom, et elle désigne désormais son principal
explicitement. **La correction reste nécessaire avant tout enrôlement** : si le sujet du
certificat est construit à partir du nom que la machine se donne, il sortirait avec le
nom court.

**Ce que l'étape n'établit PAS :**
- que l'horloge est proche de celle **du contrôleur de domaine**. `NTPSynchronized=yes`
  dit que le démon local suit *sa* source, qui peut n'être pas le DC. Kerberos compare à
  l'heure du KDC, tolérance **300 s** (`man krb5.conf`, `clockskew`). **Un crible, pas une
  preuve** ;
- que le nom résolu est celui que Kerberos emploiera : `dns_canonicalize_hostname` et
  `rdns` valent `true` par défaut (`man krb5.conf`) — le nom peut être réécrit en chemin ;
- que le VIP est joignable. Résoudre n'est pas atteindre.

**Échecs indiscernables.**

| Sortie | Causes possibles | Ce qui les sépare |
|---|---|---|
| `rc_getent_fqdn` ou `rc_getent_vip` = `2` | enregistrement absent ; résolveur injoignable ; domaine de recherche non appliqué ; `nsswitch.conf` qui n'interroge pas le DNS | Rejouer `dig <VIP_FQDN> A` **sans `+short`** : `status: NXDOMAIN` dit « le serveur a répondu, le nom n'existe pas » ; `connection timed out; no servers could be reached` dit « le résolveur est injoignable ». Deux causes, deux conduites opposées. |
| `NTPSynchronized=no` | démon arrêté ; source injoignable ; démarrage récent | `chronyc tracking` : `Reference ID` à `00000000` ⇒ aucune source ; une valeur renseignée avec un `System time` élevé ⇒ source atteinte, écart réel. |
| `rc_chrony` non nul | `chronyd` absent, arrêté, ou démon d'heure autre | `systemctl is-active chronyd systemd-timesyncd` — les deux, avant de conclure. |

---

## 5. Étape 2 — Identité Kerberos de la machine

**Identité : celle de la machine, `<MACHINE>$@<REALM>`.** Le ticket personnel de
l'opérateur n'intervient pas. C'est ici que la procédure quitte l'identité de la session
pour celle qui enrôlera.

**Dépend du droit d'enrôlement : non.** `kinit -k` n'emploie que les identifiants propres
du compte machine. Le droit d'*Enroll* sur un gabarit est une ACL d'Active Directory : il
gouverne ce que le compte peut **demander**, pas s'il peut **s'authentifier**.

### 5.1 — L'accès au keytab se MESURE, il ne se suppose pas

L'opérateur déclare pouvoir lire le keytab ; **il n'est pas établi que ce soit sans
élévation** (F-061). Ce bloc tranche, **et il choisit la branche de 2b**. Son résultat est
un **fait à inscrire au dossier** : si le keytab n'est lisible que par `root`, `certmonger`
devra s'exécuter avec ce privilège ou sous une identité à désigner (PO-020).

```
echo "=== DEBUT E2a ==="
id -un; echo "rc_id=$?"
ls -l /etc/krb5.keytab; echo "rc_ls=$?"
klist -k /etc/krb5.keytab; echo "rc_klist_sans_sudo=$?"
echo "--- elevation SEULEMENT si la ligne precedente a echoue ---"
sudo klist -k /etc/krb5.keytab; echo "rc_klist_avec_sudo=$?"
echo "=== FIN E2a ==="
```

**Les deux codes sont relevés, dans tous les cas** (R-06) — y compris quand le premier
réussit, parce qu'un `rc_klist_sans_sudo=0` est **le fait intéressant** : il signifie que
le keytab est lisible par un compte non privilégié, ce qui est un constat de sécurité en
soi et change les options de PO-020.

| `rc_sans_sudo` | `rc_avec_sudo` | Fait à inscrire |
|---|---|---|
| `0` | *(non joué)* | Keytab lisible **sans privilège**. Relevez le mode et le propriétaire de `ls -l`. |
| ≠ `0` | `0` | Keytab lisible **par `root` seul**. C'est le cas ordinaire. **Élévation à déclarer dans le rapport.** |
| ≠ `0` | ≠ `0` | Le keytab n'est pas lisible, ou n'existe pas. `rc_ls=2` sépare « absent » de « présent mais illisible ». |

### 5.2 — Ticket initial, **principal désigné explicitement**, dans le cache dédié

**Le principal n'est plus déduit du nom de la machine.** `kinit -k` sans argument choisit
d'après ce que l'hôte croit être son nom — ce que l'entrée locale fausse (PO-021). Il est
donc **nommé**, choisi **parmi les entrées que 2a vient d'afficher**.
**Forme retenue : `host/<MACHINE-FQDN>@<REALM>`**, la qualifiée : c'est celle que
l'enrôlement demandera (`PKI_enrolment.yml:105`, F-071) et **elle ne dépend pas du nom que
la machine se donne**. **Si 2a ne la montre pas**, repliez-vous sur `<MACHINE>$@<REALM>`
**et consignez l'absence** : un keytab sans SPN qualifié contredit F-086.

**Deux branches, selon ce que 2a a établi. Jouez UNE seule des deux.**

**Branche A — le keytab est lisible sans privilège** (`rc_klist_sans_sudo=0`) :

```
echo "=== DEBUT E2b-A ==="
CC=/tmp/diag-enrolement-$$.ccache
echo "$CC"
KRB5CCNAME="FILE:$CC" kinit -k "host/<MACHINE-FQDN>@<REALM>"
echo "rc_kinit=$?"
KRB5CCNAME="FILE:$CC" klist
echo "rc_klist_cc=$?"
echo "=== FIN E2b-A ==="
```

**Branche B — le keytab exige `root`** (`rc_klist_avec_sudo=0` seulement) :

```
echo "=== DEBUT E2b-B ==="
CC=/tmp/diag-enrolement-$$.ccache
echo "$CC"
sudo kinit -k -c "FILE:$CC" "host/<MACHINE-FQDN>@<REALM>"
echo "rc_kinit=$?"
sudo chown "$(id -un)" "$CC"
echo "rc_chown=$?"
ls -l "$CC"
echo "rc_ls=$?"
KRB5CCNAME="FILE:$CC" klist
echo "rc_klist_cc=$?"
echo "=== FIN E2b-B ==="
```

> **Pourquoi B n'est pas « A précédée de `sudo` ».** Trois raisons, chacune suffisante :
> l'affectation `CC=` ne se préfixe pas ; **une variable placée devant `sudo` ne traverse
> pas l'élévation**, et sa transmission après dépend de `sudoers`, que rien n'a sourcé ;
> et surtout **le cache créé sous `root` appartient à `root`** — les étapes 3 et 5, jouées
> sous la session, ne pourraient pas le lire, ni l'étape 6 le supprimer. D'où **l'option
> de cache de `kinit`** au lieu de la variable, puis le `chown` qui rend le fichier à la
> session ; `ls -l` le **prouve**. Étapes 3, 5 et 6 inchangées.
> `@VERIF : option -c de kinit — à confirmer à l'étape 0b.2, sur la cible.`
> **Ce que cela donne à l'opérateur :** un ticket de la machine pour la durée du
> diagnostic, sur une machine où il a **déjà `sudo`** (F-060), et détruit à l'étape 6 —
> rien qu'il n'ait déjà.

> **Garde-fou 2 — relevez l'échéance MAINTENANT**, colonne *Expires* de `klist` : elle
> sera revérifiée avant l'étape 5. Un ticket expiré en cours de diagnostic produit un
> `401` qui ressemble trait pour trait à un refus d'authentification, et n'en est pas un.

**Sources.** Aucune option de `klist`, `kinit` ou `kdestroy` n'a pu être sourcée sur le
poste de rédaction (F-052).
`@VERIF : -k de klist, -k de kinit, -c de kdestroy, et KRB5CCNAME avec sa syntaxe
« FILE: » — levés par l'étape 0b.2, sur la cible. Tant que cette capture n'est pas
produite, le marqueur tient ; trois étapes et le garde-fou 1 en dépendent.`
Ce qui **est** sourcé : `man krb5.conf` (`krb5-libs`, F-052), d'où viennent `clockskew`,
`dns_canonicalize_hostname` et `rdns`.

**Un résultat conforme établit :** le keytab existe, il porte des entrées, et la machine
obtient **un ticket initial** de son realm. L'identité Kerberos de la machine est
opérationnelle **à cet instant**.

**Il n'établit PAS :**
- que le compte machine a le moindre droit sur un gabarit — voir PO-008 ;
- que le ticket servira contre le VIP : c'est l'étape 3 ;
- que l'entrée employée est la bonne. Un keytab peut porter plusieurs `kvno` ; `kinit -k`
  en choisit une. Le succès ne dit pas laquelle.

**Échecs indiscernables — et c'est le point où PO-017 était trop étroit.**

`rc_kinit` non nul **ne prouve pas que la machine a perdu son identité.** Au moins cinq
causes produisent un échec, et **le code de retour ne les distingue pas** : l'information
est dans le **texte d'erreur**, qu'il faut donc lire et rapporter, pas résumer.

| Texte d'erreur (à citer tel quel) | Cause | Conduite |
|---|---|---|
| `Clock skew too great` | dérive d'horloge > 300 s vis-à-vis du KDC | Revenir à l'étape 1 : elle n'a pas suffi, parce qu'elle comparait à la mauvaise référence. |
| `Cannot find KDC for realm` | résolution des enregistrements `_kerberos._udp` en échec, ou realm mal orthographié | Problème de DNS ou de `krb5.conf`, **pas** d'identité. |
| `Cannot contact any KDC for realm` | KDC résolu mais injoignable — filtrage, port 88 | Problème de réseau, **pas** d'identité. |
| `Preauthentication failed` / `Password has expired` | **là** le secret du compte machine ne correspond plus à l'annuaire — le cas d'une restauration d'instantané antérieure à une rotation | **Étape K, depuis le seed** : comparer la version de clé du keytab à celle du compte dans l'annuaire (F-085). Égales ⇒ la cause est ailleurs ; différentes ⇒ identité périmée, prouvée. |
| `Key table entry not found` / `no suitable keys` | keytab présent mais sans entrée pour le principal demandé | Problème de contenu de keytab. |

**Sans le texte, ces cinq cas sont un seul `rc` non nul.** Rapportez la sortie complète,
pas le code seul.

---

## 6. Étape 3 — Ticket de service pour le nom du VIP

**Dépend du droit d'enrôlement : non.** Obtenir un ticket de service exige que le SPN
existe dans l'annuaire et soit rattaché à un compte ; cela ne dit rien des ACL du gabarit.

**Si cette étape échoue, les étapes 4 et 5 sont ininterprétables** — pas « à retenter ».
Et la cause est dans l'annuaire, pas sur la machine.

```
echo "=== DEBUT E3 ==="
CC=/tmp/diag-enrolement-$$.ccache
KRB5CCNAME="FILE:$CC" kvno HTTP/<VIP_FQDN>;  echo "rc_kvno=$?"
KRB5CCNAME="FILE:$CC" klist;                 echo "rc_klist=$?"
echo "=== FIN E3 ==="
```

**Sources.** Aucune sur le poste de rédaction : `man kvno` y rend `rc=16` (F-052).
`@VERIF : syntaxe « kvno <service>/<hôte> » — levé par l'étape 0b.2, sur la cible.`

**Un résultat conforme établit :** le SPN `HTTP/<VIP_FQDN>` existe, il est rattaché à un
compte de l'annuaire, et **cette machine** peut en obtenir un ticket de service.

**Il n'établit PAS** que le service au bout du VIP acceptera ce ticket — obtenir un ticket
est une transaction avec le KDC seul. Ni que le nom employé est celui que vous croyez :
`dns_canonicalize_hostname` vaut `true` par défaut, **la sortie de `klist` porte le nom
réellement demandé** — comparez-la à ce que vous avez tapé.

**Échecs indiscernables.**

| Texte d'erreur | Causes possibles | Ce qui les sépare |
|---|---|---|
| `Server not found in Kerberos database` | **(a) SPN en DOUBLE — à regarder en premier.** F-076 établit que `HTTP/<VIP_FQDN>` **est** enregistré sur le compte du pool d'applications CES ; « non trouvé » ne peut donc plus vouloir dire « absent ». Un même SPN sur deux comptes empêche l'émission de tickets ; (b) canonicalisation : le SPN existe sous le nom du nœud F5, pas sous celui du VIP | **(a) : `setspn -X` ou `setspn -Q HTTP/<VIP_FQDN>`, joué par l'équipe de l'annuaire** — ce n'est pas une mesure de la recette. (b) : relire le nom dans `klist`, et rejouer `kvno` avec le nom du nœud si l'équipe F5 le fournit. |
| `KDC has no support for encryption type` | désaccord de types de chiffrement entre le compte machine et le compte porteur du SPN | Lire `klist -k -e /etc/krb5.keytab` — `@VERIF : option -e de klist — levé par l'étape 0b.2` — et comparer avec ce que l'équipe AD déclare sur le compte du SPN. |
| `Ticket expired` alors que l'étape 2 venait de réussir | horloge qui dérive **pendant** la procédure | Rejouer l'étape 1 juste après : un écart qui a changé en quelques minutes est un problème d'horloge actif, pas un incident. |
| `No credentials cache found` / `Permission denied` sur `$CC` | **cache illisible par la session** : 2b branche B jouée sans le `chown`, ou `chown` en échec | `ls -l "$CC"` : propriétaire `root` ⇒ c'est cela, et rien d'autre. Distinct d'un ticket absent (`klist` sur un cache **lisible** mais vide) et d'un ticket expiré (`Expires` dépassé). Rejouez le `chown` de 2b-B ; ne rejouez pas `kinit`. |

---

## 7. Étape 4 — Ce que le VIP présente en TLS, et si la machine le valide

**Dépend du droit d'enrôlement : non.** Aucune authentification ici : on observe une
poignée de main TLS.

C'est l'étape qui instruit **PO-005** (terminaison TLS au F5, ou passthrough).

### 7.1 — Ce qui est présenté

```
echo "=== DEBUT E4a ==="
openssl s_client -connect <VIP_FQDN>:443 -servername <VIP_FQDN> -showcerts </dev/null 2>&1 | grep -E '^ *[0-9]+ s:|^ *i:|Verify return code|Protocol *:'
echo "rc_ignore=$? — code du grep, pas d'openssl. Sans valeur ici : on LIT la sortie."
echo "=== FIN E4a ==="
```

### 7.2 — Le certificat de tête, en clair

```
echo "=== DEBUT E4b ==="
openssl s_client -connect <VIP_FQDN>:443 -servername <VIP_FQDN> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
echo "rc_x509=${PIPESTATUS[1]} — code d'openssl x509, pas du tube"
echo "=== FIN E4b ==="
```

### 7.3 — Le magasin de la machine valide-t-il la chaîne ? *(le contrôle qui compte)*

```
echo "=== DEBUT E4c ==="
openssl s_client -connect <VIP_FQDN>:443 -servername <VIP_FQDN> -verify_return_error -brief </dev/null 2>&1 | tail -12
echo "rc_verify=${PIPESTATUS[0]} — code d'openssl, PAS de tail"
curl -sS -o /dev/null -w 'http_code=%{http_code} ssl_verify=%{ssl_verify_result} ip=%{remote_ip}\n' -m 20 https://<VIP_FQDN>/
echo "rc_curl=$?"
echo "=== FIN E4c ==="
```

**Sources** (`man openssl-s_client`, `man openssl-x509`, `man curl`, poste de rédaction) :
`-connect` « the host and optional port to connect to » ; `-servername` « Set the TLS SNI
… extension » — **indispensable devant un F5**, qui peut servir plusieurs certificats sur
une adresse ; `-showcerts` « the server certificate list **as sent by the server** …
**It is not a verified chain** » — c'est ce qu'on veut en 7.1, avant tout jugement ;
`-verify_return_error` « **returns verification errors instead of continuing** » ;
`-brief` « a brief summary of connection parameters » ; `-dates`, `-ext
subjectAltName` ; `ssl_verify_result` « **0 means the verification was successful** » ;
`-m` « the maximum time … that you allow each transfer to take ».

**Piège de code de retour, mesuré (F-054, F-057)**, face à un certificat non validable :

```
openssl s_client ... (sans -verify_return_error)   rc=0   « Verify return code: 18 »
openssl s_client ... -verify_return_error          rc=1   « Verify return code: 18 »
curl (sans --cacert adapté)                        rc=60  http_code=000
curl (avec --cacert adapté)                        rc=0   http_code=200
```

**`rc=0` alors que la validation a échoué** — c'est pourquoi 7.1 et 7.2 ne concluent rien
sur la confiance, et pourquoi 7.3 emploie `-verify_return_error`.

**`rc_curl=60`** signifie, d'après `man curl` : « Peer certificate cannot be
authenticated with known CA certificates ». C'est **un résultat**, pas une panne.

**Un résultat conforme établit :**
- `rc_verify=0` et `ssl_verify=0` : le magasin de confiance **de cette machine, tel
  qu'il est aujourd'hui**, valide la chaîne présentée par le VIP ;
- l'`issuer` lu en 7.2 répond à PO-005 : une autorité d'entreprise oriente vers
  passthrough ou ré-émission ; une autorité propre au F5 signe une terminaison TLS.

**Il n'établit PAS :**
- que le service derrière le VIP est CEP. Un `http_code=200` sur `/` peut venir d'une
  page d'accueil IIS ;
- que le bundle `cas` de `cepces.conf` sera le bon (F-029). Le magasin système et ce
  bundle sont deux choses distinctes : `cepces` ne lit pas le magasin système
  nécessairement. `@VERIF : confirmer si cepces emploie le magasin système ou uniquement
  le bundle « cas », par « man cepces.conf » ou la configuration livrée, sur la cible.`
- que le nom du VIP figure dans le certificat. **Lisez `subjectAltName` en 7.2** : c'est
  lui qui compte, pas le `CN`.

**Échecs indiscernables.**

| Sortie | Causes possibles | Ce qui les sépare |
|---|---|---|
| `rc_curl=60`, `ssl_verify` non nul | (a) émetteur absent du magasin de la machine ; (b) **chaîne incomplète** : le F5 n'envoie que le certificat de tête, sans intermédiaire ; (c) certificat expiré | **7.1 les sépare toutes les trois.** Compter les blocs `s:`/`i:` : un seul bloc → (b). Comparer le dernier `i:` à ce que le magasin contient → (a). Lire `-dates` en 7.2 → (c). C'est pour cela que 7.1 précède 7.3. |
| `rc_curl=7` | VIP résolu mais port 443 fermé ou filtré | `rc_getent_vip=0` à l'étape 1 a déjà exclu la résolution. Reste réseau ou service arrêté : question à l'équipe F5. |
| `rc_curl=35` (« SSL connect error ») | désaccord de version TLS ou de suites, **avant** toute validation de certificat | Distinct de 60 : en 35 on n'a jamais vu de certificat. La sortie `Protocol :` de 7.1 est alors vide. |
| `rc_curl=6` | le nom ne se résout pas | Contredit l'étape 1 : si `getent` avait réussi et que `curl` échoue, les deux n'empruntent pas la même voie — à rapporter tel quel, c'est anormal. |
| `http_code=000` | aucun échange HTTP n'a eu lieu | **Toujours** lire `rc_curl` avec : `000` dit seulement « la couche HTTP n'a pas été atteinte », il ne dit pas pourquoi. |

---

## 8. Étape 5 — Requête authentifiée contre le service de politique (CEP)

**Dépend du droit d'enrôlement : partiellement, et il faut distinguer.**
Cette étape teste **l'authentification**, qui ne dépend pas de l'appartenance au groupe
d'enrôlement : un `401` est un refus d'authentifier, antérieur à toute autorisation.
Un `403` en revanche **serait** un signal d'autorisation, et celui-là peut dépendre du
groupe. La distinction se lit dans le code HTTP, pas ailleurs.

**Cette requête ne demande aucun certificat.** C'est un `GET` sans corps SOAP : elle
n'émet ni `GetPolicies`, ni `RequestSecurityToken`. Rien n'est demandé, rien n'est émis.

```
echo "=== DEBUT E5 ==="
CC=/tmp/diag-enrolement-$$.ccache
echo "--- garde-fou 2 : le ticket de E2b est-il TOUJOURS valide ? ---"
KRB5CCNAME="FILE:$CC" klist
echo "rc_echeance=$? — comparez la colonne Expires a ce que vous avez note en E2b"
echo "--- TEMOIN SANS IDENTIFIANTS : doit rendre 401 ---"
curl -sS -o /dev/null -m 30 -w 'http_code_anonyme=%{http_code}\n' <CEP_URL>
echo "rc_anonyme=$? — lisez http_code_anonyme, pas ce code"
KRB5CCNAME="FILE:$CC" curl -sS --negotiate -u : -o /dev/null -D - -m 30 -w 'http_code=%{http_code} ssl_verify=%{ssl_verify_result}\n' <CEP_URL>
echo "rc_curl=$? — sans valeur sur un 401 : curl rend 0 (F-057). Lisez http_code."
KRB5CCNAME="FILE:$CC" klist
echo "rc_klist=$?"
echo "=== FIN E5 ==="
```

**Sources** (`man curl`) : `--negotiate` « Enable Negotiate (SPNEGO) authentication. …
**you must also provide a fake --user option** … **Sending a '-u :' is enough** » — c'est
la source du `-u :`, qui sinon paraît être une faute de frappe ; `-D -` « Specify "-" …
to have it written to stdout » ; `-s` « Do not show progress meter or error messages » et
`-S` « makes curl show an error message if it fails » ; `-w`, `-m` : voir § 7. `curl`
compilé avec GSSAPI : vérifié à l'étape 0 (F-051, **non transposable** à RHEL 9).

**Piège de code de retour, sourcé.** `man curl`, `-f, --fail` : « **By default, curl does
not consider HTTP response codes to indicate failure.** » **`curl` rend donc `0` sur un
`401`** ; c'est `%{http_code}` qui porte le résultat. Et `-f` est **écarté délibérément** :
il rendrait `22` indistinctement pour `401`, `403`, `404` et `405` — les quatre qu'il faut
justement séparer.

### La garde : le témoin sans identifiants

**Ce qui prouve qu'une authentification a eu lieu n'est pas le code obtenu, c'est l'écart
entre deux codes.** Le témoin anonyme est le **sens de l'échec** de cette étape : sans
lui, aucun code ne se distingue d'un accès qui n'aurait rien demandé.

| `http_code_anonyme` | Puis `http_code` avec le ticket | Lecture |
|---|---|---|
| **`401`** | un code **différent** (`200`, `405`, `403`…) | **L'authentification a eu lieu.** C'est la seule combinaison qui l'établit. |
| **`401`** | `401` | L'authentification a été **tentée et refusée** — table des indiscernables ci-dessous. |
| tout autre code | *(quel qu'il soit)* | **L'accès anonyme est permis** : le code obtenu avec le ticket **ne prouve rien**. **Rapportez les deux codes tels quels, sans conclure**, et arrêtez l'interprétation ici. |

`[HYPOTHÈSE — non sourcée]` Un `405` serait le comportement d'un point d'entrée SOAP
attendant un `POST` et refusant un `GET` après avoir authentifié. **Rien ne l'établit
ici** : c'est le témoin qui tranche. Conservée parce qu'elle oriente, pas comme preuve.

**Il n'établit PAS** que l'enrôlement aboutira : l'autorisation sur le gabarit, la
construction du sujet (PO-016) et le comportement de CES restent entiers.

**Échecs indiscernables — c'est ici que la protection étendue se révèle.**

| `http_code` | Causes possibles | Ce qui les sépare |
|---|---|---|
| `401` | Dans **cet ordre** : (c) cache vide ou **ticket expiré depuis E2b** ; (d) le serveur n'offre pas `Negotiate` ; (b) SPN non accepté ; **(a) protection étendue — EN DERNIER**, par élimination | **(c)** : comparez *Expires* au relevé d'E2b, et cherchez `HTTP/<VIP_FQDN>` dans le `klist` final — absent ⇒ `curl` n'en a jamais obtenu. **(d)** : `WWW-Authenticate` annonçant `NTLM` seul. **(b)** : exclu si l'étape 3 a réussi — c'est la raison de son ordre. **(a)** n'est atteinte que si les trois autres sont écartées, et ce qui l'écarte est **F-077 seul** — déclaration au conditionnel, non mesurée (PO-018). F-076 n'y contribue pas : il mesure l'annuaire, pas la configuration d'IIS. |
| `403` | Authentifié, **non autorisé** | **La chaîne de groupes n'est plus une explication** : `<GROUPE-T>` → `<GROUPE-ALL>` → `<GROUPE-PKI>` est prouvée au niveau de l'annuaire (F-080 à F-082). Restent deux pistes : un **ticket qui ne porte pas l'appartenance**, ou une **particularité des droits sur le gabarit**. À porter à l'équipe PKI. *Note, non affirmée :* un ticket initial obtenu **après** l'ajout au groupe devrait porter l'appartenance, redémarrage ou non — **à confirmer par la mesure**, pas à poser comme acquis. |
| `404` | URL erronée, ou réécriture au F5 | Comparer l'URL substituée au courriel de l'équipe, caractère par caractère. |
| `000` avec `rc_curl` 60 ou 35 | On n'a jamais atteint HTTP | L'étape 4 aurait dû le montrer. Si 4 était conforme et 5 ne l'est pas, les deux n'ont pas visé le même hôte — comparer `remote_ip`. |
| `401` **et** `rc_echeance` non nul | **cache illisible par la session**, non « authentification refusée » : `curl` n'a jamais eu de ticket à présenter | `ls -l "$CC"` — propriétaire `root` ⇒ 2b branche B sans `chown`. **Le `401` ne dit alors rien du serveur.** Distinct de (c) : là, `klist` lit le cache et le trouve expiré ; ici, `klist` ne le lit pas du tout. |

> **La cause (a), et deux objets à ne pas confondre.** S-6 (`<extendedProtection>`, F-062)
> établit que dans le scénario « SSL off-loading » **c'est le SPN qui est vérifié**, non le
> jeton de liaison. Mais ce SPN est celui de la **collection `<spn>` d'IIS** — un réglage
> de configuration (ligne 23 ; *Child Elements*, lignes 173-179). **Ce n'est PAS
> l'enregistrement d'annuaire que `setspn` mesure** (F-076), lequel conditionne l'émission
> d'un ticket par le KDC et s'éprouve à l'**étape 3**, pas ici.
> **Conséquence pour ce `401` :** `tokenChecking` et `flags` ont déjà été demandés et
> **fournis au conditionnel** (F-077, « I think »). Reste à demander **le contenu de la
> collection `<spn>` d'IIS**, et seulement si les autres causes sont écartées (PO-018).

---

## 8 bis. Témoin de contraste — les mêmes questions, depuis le seed, sous `<OP_AD>`

**À ne jouer qu'après les étapes 3 et 5, jamais à leur place.**

**Ce que c'est :** un **séparateur de causes**, pas une validation. Un succès ici ne
valide rien pour le compte machine — les deux identités sont distinctes, et seule la
seconde enrôlera (§ 1 bis).

**Motif.** Les étapes 3 et 5 échouent pour deux familles de raisons qu'elles ne
distinguent pas : ce qui est **commun au domaine** (SPN absent ou en double, F5, IIS) et
ce qui est **propre au compte machine** (principal, appartenance, secret périmé). Les
rejouer sous une identité **connue pour fonctionner** sépare les deux en une manipulation.
**Identité : `<OP_AD>`**, ticket déjà présent (F-058). Aucun `kinit`, **aucun cache créé
ni détruit** — mais **pas « en lecture seule »** : obtenir un ticket de service l'**ajoute**
au cache de session, effet sans danger qui se périme seul. La règle qui tient est
qu'**aucun cache n'est détruit ici**.

```
echo "=== DEBUT T ==="
hostname -s;                              echo "rc_hote=$?"   # doit etre le SEED
klist; echo "rc_klist=$? — ticket de <OP_AD>"
kvno HTTP/<VIP_FQDN>;                     echo "rc_T1=$?"
curl -sS --negotiate -u : -o /dev/null -D - -m 30 -w 'http_code=%{http_code}\n' <CEP_URL>
echo "rc_T2=$? — lisez http_code, pas ce code"
echo "=== FIN T ==="
```

> **Aucun `kdestroy` dans ce bloc, et ce n'est pas un oubli.** Détruire ici viserait le
> cache **par défaut** du seed, c'est-à-dire le ticket que SSSD a posé au login et que
> l'opérateur ne reconstitue pas à la main (garde-fou 1).

**Lecture croisée — c'est tout l'intérêt du témoin :**

| Étape 3/5 (machine) | Témoin T1/T2 (`<OP_AD>`) | Ce que cela oriente |
|---|---|---|
| échec | **échec** | Cause **commune** : le SPN n'existe pas, ou l'intermédiaire (F5, IIS) rompt quelque chose pour tout le monde. **Question à l'équipe AD ou F5**, pas à la machine. |
| échec | **succès** | Cause **propre au compte machine** : principal, keytab, ou appartenance au groupe d'enrôlement. **C'est PO-008**, et c'est la machine qu'il faut regarder. |
| succès | *(non nécessaire)* | Le témoin n'apporte rien : ne le jouez pas. |
| succès | échec | Inattendu — à rapporter tel quel, sans interpréter. |

**Ce que le témoin n'établit dans aucun cas :** que le compte machine a le droit
d'enrôler. Il ne sait dire que « commun » ou « propre », jamais « autorisé ».

---

## 9. Étape 6 — Nettoyage, obligatoire

**À jouer sur la machine de recette uniquement.** Ce bloc ne concerne **pas** le seed :
rien n'y a été créé (§ 8 bis).

```
echo "=== DEBUT E6 ==="
CC=/tmp/diag-enrolement-$$.ccache
kdestroy -c "FILE:$CC";   echo "rc_kdestroy=$?"
ls -l "$CC" 2>&1;         echo "rc_ls=$?  (2 attendu : le cache doit avoir disparu)"
klist >/dev/null 2>&1;    echo "rc_cache_defaut=$?  (0 attendu : il doit SURVIVRE)"
echo "=== FIN E6 ==="
```

> **Garde-fou 1, contrôlé et non seulement énoncé.** `kdestroy` porte `-c "FILE:$CC"` :
> **jamais** `kdestroy` nu, qui viserait le cache par défaut. La dernière ligne le
> vérifie dans l'autre sens : le cache de la session doit **survivre**. Un
> `rc_cache_defaut` non nul signifie qu'on a détruit le mauvais cache — signalez-le, ce
> n'est pas rattrapable en silence.

`@VERIF : option -c de kdestroy — levé par l'étape 0b.2. Le garde-fou 1 en dépend :
sans `-c`, ce bloc détruirait le cache par défaut.`

**Le contrôle est dans les deux sens** : `rc_kdestroy=0` ne suffit pas, parce qu'un
`kdestroy` sur un cache déjà absent peut rendre `0`. C'est `rc_ls=2` qui établit la
disparition. Un `rc_ls=0` signifie que le fichier est encore là : **supprimez-le à la
main et signalez-le**, la procédure a laissé un état derrière elle.

---

## 10. Point d'arrêt

**La procédure s'arrête à la première étape en échec, et le dit.** Une étape en échec ne
rend pas les suivantes « à retenter » : elle les rend **ininterprétables**. Jouer l'étape 5
après un échec de l'étape 3 produit un `401` dont on ne saura jamais s'il vient de la
protection étendue ou de l'absence de ticket. *Exception, la seule :* l'écart **connu** de
l'étape 1 n'arrête pas (voir sa table de verdict).

**Rapport attendu**, en `[ÉCRAN-AAAA-MM-JJ]` (D-003) : la capture de l'**étape 0b.2**, qui
conditionne toutes les autres ; le numéro de la dernière étape **conforme** ; la capture
de l'étape en échec **entière**, marqueurs compris ; les **textes d'erreur cités**, pas
résumés — ils portent ce que les codes de retour ne portent pas ; **les deux codes de
l'étape 2a dans tous les cas** (F-061, c'est un fait à inscrire, pas un détail) et toute
autre élévation ; et tout outil absent.

**Aucune correction n'est proposée par cette procédure, et c'est délibéré.** Diagnostiquer
et réparer sont deux actes. Réparer au milieu d'un diagnostic détruit l'état qu'on
mesurait, et la mesure suivante portera sur autre chose que ce qu'on croit.

---

## 11. Ce que cette procédure ne peut pas instruire

| Point ouvert | Pourquoi elle ne l'atteint pas |
|---|---|
| **PO-001** — l'agent SCOM acceptera-t-il le certificat | Demande un certificat posé et une découverte console. Hors d'une procédure en lecture seule. |
| **PO-004** — SAN et taille de clé attendus | Demande l'inspection d'un certificat SCX **sain**, que la recette n'a pas. |
| **PO-006** — versions RHEL 9 de certmonger/cepces | `dnf info` ne fait pas partie de cette procédure, qui ne touche pas aux paquets. À joindre au rapport si l'opérateur le souhaite. |
| **PO-015** — chemin sans `.pfx` ; **PO-017** — certificats déjà émis | L'un demande d'engendrer une clé ; l'autre ne se voit que depuis la console de la CA. |
| **PO-020** — identité d'une tâche Ansible | Session interactive, et `certmonger` n'est pas installé ici. |
| **PO-023** — forme du sujet acceptée par l'agent | Se découvre par essais d'enrôlement, que cette procédure s'interdit. |

| Point ouvert | Étape qui l'instruit |
|---|---|
| **PO-005** — terminaison TLS au F5 | **Étape 4**, par l'`issuer` du certificat présenté. |
| **PO-008** — identité Kerberos acceptée | **Étapes 2 et 3**, pour la part « quel principal est présenté et le SPN existe-t-il ». La part ACL reste à l'équipe PKI. |
| **PO-017** — identité Kerberos après restauration | **Étape 2**, et sa table d'échecs indiscernables — qui est la correction de la fiche. |
| **PO-018** — protection étendue : mode et collection `<spn>` | **Étape 5**, par élimination seulement — la configuration ne se lit que côté IIS. |
| **PO-008** — cause commune au domaine, ou propre au compte machine | **§ 8 bis**, le témoin de contraste, qui sépare les deux familles sans trancher l'ACL. |
