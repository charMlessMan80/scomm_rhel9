# Dossier d'autorité — certificat SCOM via ADCS / CEP-CES

Portée : projet « faire obtenir à des machines RHEL 9 un certificat destiné à SCOM,
délivré par une ADCS via CEP/CES derrière un reverse proxy F5, maintenu par certmonger ».

**Ce que ce fichier porte** — et c'est ce qui se relit à chaque session :
les **contraintes** (poste, dépôt, environnement), les faits établis par **documentation
nommée**, les faits **déclarés par l'opérateur**, les **points ouverts** et les
**décisions**, plus la **convention de marquage**.

**Ce que ce fichier NE porte PAS**, et où le trouver :

| Nature | Fichier | Pourquoi ailleurs |
|---|---|---|
| Règles de travail | `../CLAUDE.md` | Une règle vit à un seul endroit. |
| Ce que le code du dépôt fait aujourd'hui — **F-008 à F-021 et F-038** | `scomm-depot-actuel.md` | Brouillon sans autorité (F-044), périmable en bloc par la réécriture (D-007). **Toute citation de ces numéros dans ce fichier renvoie là-bas.** |
| Sorties de commande — **P-01 à P-55** | `scomm-journal-preuves.md` | Pièces justificatives, consultées au besoin, pas relues à chaque session. **Toute citation `P-xx` renvoie là-bas.** |
| Procédures jouées par l'opérateur sur les cibles | `procedures/` | Elles n'ont **aucune autorité propre** : tout ce qu'elles affirment vient d'ici, par renvoi `F-xxx` / `PO-xxx`. |

`[RÉVISÉE le 2026-09-29]` Deuxième ligne du tableau **ÉTAIT :** « Sorties de commande —
**P-01 à P-28** ». Déjà périmée avant ce livrable (P-47 existait) ; relevée par la
recherche de renvois du 2026-09-29.

Séparation appliquée le 2026-09-21 (D-006). **Aucun fait, aucune preuve, aucun point
ouvert n'a été renuméroté** : les objets ont été déplacés, pas réécrits (R-11). Les
numéros de section survivants ont eux aussi été conservés, pour que les renvois
antérieurs restent valides — d'où les trous en § 1.2 et § 5.

`[RÉVISÉE le 2026-09-21]` Cet en-tête **ÉTAIT :** « Ce fichier est le dépôt unique des
faits, des points ouverts et des décisions. `CLAUDE.md` porte les règles de travail et ne
duplique rien d'ici. » Le principe — un objet, un seul endroit — est inchangé ; c'est le
nombre d'endroits qui est passé de un à trois (D-006).

Dernière mise à jour : **2026-09-29** (consignation du 2026-09-25 au 2026-09-28).
`ÉTAIT : 2026-09-21 (livrables 3 puis 4).` — déjà périmée : des faits ont été ajoutés
jusqu'au 2026-09-24 sans que cette ligne suive.
`ÉTAIT : 2026-09-21 (livrable 3).`
`ÉTAIT : 2026-09-18.`

## Convention de marquage

| Marqueur | Sens |
|---|---|
| `@VERIF : <où le confirmer>` | Affirmation non établie par un artefact lu. Interdite telle quelle dans une prescription. |
| `[PILOTE-DÉCLARÉ]` | Affirmé par l'opérateur, non mesuré. |
| `[SOURCE-EXTERNE]` | Établi par une documentation externe nommée, pas par un artefact local. |
| `[TIERS-DÉCLARÉ]` | Affirmé par une autre équipe, à l'écrit ou à l'oral, **non mesuré** : plus faible qu'une mesure, plus fort qu'une supposition. |
| `[TIERS-MESURÉ]` | Sortie de commande produite par une autre équipe et transmise : c'est une mesure, mais ni rejouable ni vérifiable en contexte. |
| `[ÉCRAN-AAAA-MM-JJ]` | Mesure rapportée par capture d'écran depuis une machine professionnelle (cf. D-003). |
| `ÉTAIT` / `[RÉVISÉE le …]` | Marque d'historique. L'historique se marque, il ne s'efface pas. |

Le dépôt est **public** (cf. F-024). Aucune valeur réelle d'environnement ne figure
dans ce fichier : les exemples emploient `ca.example.com`, `EXAMPLE.COM`, `host01`.

---

## 1. Faits sourcés

### 1.1 État du poste et du dépôt (mesuré le 2026-09-18)

- **F-001** — La racine du dépôt est `/home/<user>/dev/scomm_rhel9`.
  Source : `git rev-parse --show-toplevel`, rc=0 (P-01).
- **F-002** — L'arbre de travail est propre au départ : `git status --porcelain -uall`
  ne produit aucune ligne, rc=0 (P-01).
- **F-003** — HEAD = `5f0eca9a3d011e49c5888765bf112d800a64949e`, historique de 2 commits
  (`5f0eca9 Project init`, `b0c3426 Initial commit`). Source : P-01.
- **F-004** — Le remote `origin` est en `https://` (`https://github.com/charMlessMan80/scomm_rhel9`),
  en fetch comme en push. Source : `git remote -v`, rc=0 (P-01).
- **F-005** — **Aucun `credential.helper` n'est configuré**, à aucune portée :
  `git config --show-origin --get-all credential.helper` → rc=1, sortie vide (P-02).
  Conséquence : un `git push` sur ce remote échouera faute d'identifiants. Cf. PO-007.
  Non corrigé volontairement : la configuration d'accès appartient à l'opérateur.
- **F-006** — Le dépôt suit **15 fichiers**. Source : `git ls-files`, rc=0 (P-03) :
  `LICENSE`, `README.md`, `ansible.cfg`, `group_vars/scomm_agents.yml`,
  `integrate_scomm.yml`, `inventory/hosts.ini`, `requirements.yml`,
  `roles/scomm_agent/{defaults,handlers}/main.yml`,
  `roles/scomm_agent/tasks/{main,install,configure,register,users}.yml`,
  `roles/scomm_agent/templates/scom_sudoers.j2`.
- **F-007** — Il n'existait **aucun** `.gitignore`, `CLAUDE.md` ni répertoire `docs/`
  avant ce livrable : `ls` → rc=2 sur les trois ; `git ls-files --error-unmatch .gitignore`
  → rc=1 (P-04). Confirme la prémisse opérateur « aucun des deux dépôts ne porte de `CLAUDE.md` »
  pour `scomm_rhel9` uniquement.


### 1.2 Ce que `scomm_rhel9` fait aujourd'hui — **DÉPLACÉ le 2026-09-21**

Les faits **F-008 à F-021** et **F-038** vivent désormais dans
`scomm-depot-actuel.md`, avec leurs numéros d'origine. Ils n'ont pas été modifiés.
*Motif :* ils décrivent un brouillon sans autorité (F-044) que la réécriture périmera en
bloc (D-007) ; les garder ici obligerait à relire à chaque session ce qui est déjà acté
comme jetable. Le numéro de section est conservé vide pour que les renvois « § 1.2 »
antérieurs continuent de pointer quelque part.

### 1.3 Outillage présent sur le poste (Fedora — **pas** la cible)

- **F-022** — `ansible [core 2.20.7]`, `/usr/bin/ansible`, modules Python sous
  `/usr/lib/python3.14/site-packages/ansible` (P-07). Ce n'est **pas** la version de la
  cible. @VERIF : version d'ansible-core réellement employée par le seed —
  `ansible --version` sur le nœud de contrôle professionnel, rapporté en `[ÉCRAN-…]`.
- **F-023** — `certmonger` et `cepces` ne sont **pas installés** sur ce poste :
  `rpm -q certmonger` → rc=1, `rpm -q cepces` → rc=1 ; `command -v getcert` → rc=1 (P-08).
  Aucune installation n'a été faite (règle du poste). Les faits certmonger/cepces
  ci-dessous sont donc `[SOURCE-EXTERNE]`, pas des mesures locales.
- **F-024** — **Le dépôt distant est public.** `curl` non authentifié sur
  `https://api.github.com/repos/charMlessMan80/scomm_rhel9` → `http_code=200` et
  `"private": false` ; `git ls-remote` sans identifiants (`GIT_TERMINAL_PROMPT=0`) → rc=0
  et renvoie le SHA de HEAD (P-09). Conséquence normative : cf. R-07 dans `CLAUDE.md`.
- **F-025** — Le venv de lint existe et est unique :
  `/home/<user>/.venvs/ansible-lint/bin/{ansible-lint,yamllint}` (P-06).
  Aucun second venv n'a été créé.
- **F-041** — **Le contrôle syntaxique du playbook échoue sur ce poste**, et ce n'est pas
  une faute du code : `ansible-playbook --syntax-check integrate_scomm.yml` → **`rc=4`**,
  avec `[ERROR]: couldn't resolve module/action 'ansible.posix.firewalld'` pointant
  `roles/scomm_agent/tasks/configure.yml:8:3` (P-17). Cause mesurée : la collection
  `ansible.posix`, requise par `requirements.yml:2-4`, n'est pas installée —
  `ansible-galaxy collection list | grep -i posix` → rc=1,
  `/home/<user>/.ansible/collections` → rc=2 (absent),
  `/usr/share/ansible/collections/ansible_collections` → vide (P-17).
  **Elle n'a pas été installée** : ce poste n'installe rien. `--skip-tags configure` ne
  contourne pas le problème (rc=4 également) : le contrôle syntaxique résout toutes les
  tâches indépendamment des tags. Cf. PO-010.

### 1.4 certmonger et cepces — `[SOURCE-EXTERNE]`

Sources nommées :
- **S-1** `[SOURCE-EXTERNE]` — `README.rst` du projet amont `openSUSE/cepces`,
  récupéré le 2026-09-18 (http_code=200, P-10).
- **S-2** `[SOURCE-EXTERNE]` — `doc/CERTMONGER.md` du même dépôt amont (http_code=200, P-11).
- **S-3** `[SOURCE-EXTERNE]` — page de manuel `getcert-request(1)` telle que publiée par
  mankier.com, consultée le 2026-09-18 (P-13).

- **F-026** — cepces fournit à certmonger un **helper de CA externe** :
  `/usr/libexec/certmonger/cepces-submit`. La CA apparaît dans `getcert list-cas` sous
  `ca-type: EXTERNAL`, `helper-location: /usr/libexec/certmonger/cepces-submit` (S-2, S-1).
- **F-027** — Déclaration de la CA (S-2) :
  `getcert add-ca -c cepces -e /usr/libexec/certmonger/cepces-submit`,
  les options pouvant être passées sur la ligne du helper :
  `getcert add-ca -c cepces -e '/usr/libexec/certmonger/cepces-submit --server=ca.example.com --keytab=/etc/krb5.keytab --principals=HOST/host01.example.com@EXAMPLE.COM'`.
- **F-028** — **Identité Kerberos d'enrôlement.** cepces liste quatre méthodes :
  Kerberos (GSSAPI), Username/Password, Certificate, Anonymous ; Kerberos « requires the
  client to be a Windows Domain Member with a valid Kerberos keytab » (S-1).
  L'identité est choisie par `--keytab` et `--principals` (F-027). Le README amont
  illustre un principal de compte machine (`MY-HOST$@EXAMPLE.COM`) comme un principal
  de service (`HOST/host01.example.com@EXAMPLE.COM`) selon l'exemple (S-1, S-2).
  @VERIF : lequel des deux est accepté par l'ADCS et le F5 de cet environnement —
  se tranche par PO-008.
- **F-029** — **Configuration** : `/etc/cepces/cepces.conf`, éventuellement livré en
  `cepces.conf.dist` à renommer. Deux réglages suffisent : `server` **ou** `endpoint`
  (le point d'entrée CEP), et `cas` — « a directory containing all CA certificates in
  your chain … or preferably a bundle file containing all CA certificates in the chain » (S-1).
  `cas` est le point exact où la terminaison TLS du F5 devient déterminante (PO-005).
  `[NOTE du 2026-09-29]` Complété par **F-120** : le `cepces.conf` livré par le paquet
  RHEL 9 **ne définit pas `cas`** ; la conséquence qu'en tire le pilote (confiance
  système) est consignée là-bas comme une déduction.
- **F-030** — **Renouvellement.** certmonger est « a service that monitors certificates,
  tracks their expiration, and automatically renews them before they expire » (S-2).
  Une requête suivie affiche `track: yes` / `auto-renew: yes` (S-2). L'option
  `getcert request -r` (« attempt to obtain a new certificate … when the expiration date
  nears ») est **le défaut** ; `-R` l'inhibe (S-3).
- **F-031** — **Point d'accroche après renouvellement** : `getcert request -C <commande>` —
  « When ever the certificate or the CA's certificates are saved to the specified
  locations, run the specified command as the client user **after** saving the
  certificates » ; `-B` fait de même **avant** (S-3). C'est l'accroche par laquelle un
  redémarrage d'`omid` peut être déclenché automatiquement (cf. F-036).
- **F-032** — Options de requête pertinentes pour ce projet (S-3) :
  `-c` CA, `-T` gabarit/template, `-k` fichier de clé, `-f` fichier de certificat
  (« do not use the same file specified with the -k option »), `-N` sujet
  (défaut `CN=hostname`), `-D` SAN DNS, `-K` SAN principal Kerberos, `-U` EKU,
  `-G` type de clé, `-g` taille de clé. Exemple amont (S-2) :
  `getcert request -c cepces -T Machine -I MachineCertificate -k <clé> -f <cert>`.
- **F-033** — Le helper expose les codes de retour `0 ISSUED`, `1 WAIT`, `2 REJECTED`,
  `3 CONNECTERROR`, `4 UNDERCONFIGURED`, `5 WAITMORE`, `6 UNSUPPORTED` (S-2). `3` et `4`
  sont les deux codes qu'un F5 mal traversé ou un `cas` incomplet produiront —
  ce sont eux qu'une procédure de diagnostic doit afficher.
  `[NOTE du 2026-09-29]` **Incomplet :** `rc=4` a aussi été obtenu sur une réponse
  `500` **à corps vide** du serveur, à travers le F5, Kerberos obtenu (F-123) ; le même
  appel rejoué par `curl` est authentifié (F-124).
  Un `4` ne désigne donc pas à lui seul le F5 ni `cas`.
- **F-034** — Version disponible **sur Fedora 44, à titre indicatif seulement** :
  `cepces 0.5.0-2.fc44` (source amont `https://github.com/openSUSE/cepces`, licence
  GPL-3.0-or-later) et `certmonger 0.79.21-4.fc44` (P-08, P-12).
  **Ces numéros ne valent pas pour RHEL 9** — cf. PO-006.

### 1.5 L'agent SCOM face à un certificat de PKI externe — `[SOURCE-EXTERNE]`

Sources nommées, lues intégralement en texte brut (pas par résumé) :
- **S-4** `[SOURCE-EXTERNE]` — Microsoft, « How to use a CA certificate on an SCX agent »
  (frontmatter `title: Convert self-signed SCX certificates to CA certificates`,
  `ms.date: 04/15/2024`), source markdown du dépôt `MicrosoftDocs/SupportArticles-docs`,
  fichier `support/system-center/scom/use-ca-certificate-on-scx-agent.md`,
  récupéré le 2026-09-18, http_code=200, 217 lignes (P-14).
- **S-5** `[SOURCE-EXTERNE]` — Microsoft Learn, « Administer and Configure the UNIX -
  Linux Agent », `manage-security-administer-crossplat-agent`, `ms.date: 2024-11-01`,
  moniker `sc-om-2025` (P-15).

- **F-035** — **Réponse à la question bloquante : oui, un certificat délivré par une PKI
  externe (ADCS) est accepté, et la procédure est documentée par Microsoft.** S-4 est
  entièrement consacré au remplacement du certificat auto-signé de l'agent SCX par un
  certificat émis depuis un gabarit ADCS. La preuve interne la plus nette est l'exemple
  de validation (S-4, lignes 186-192) :
  `openssl x509 -noout -in /etc/opt/microsoft/scx/ssl/scx.pem -subject -issuer -dates`
  → `issuer= /DC=lab/DC=nfs/CN=nfs-DC-CA`, soit une autorité d'entreprise ;
  le texte enchaîne (ligne 194) : « In typical scenarios, the `issuer` will be a
  management server/gateway in the UNIX/Linux resource pool », présentant donc la
  signature par le serveur d'administration comme le cas **habituel**, non comme le cas
  **obligatoire**.
- **F-036** — Détail de ce qu'attend l'agent (S-4) :
  - **Fichiers séparés** — clé privée `/etc/opt/omi/ssl/omikey.pem` (ligne 87),
    certificat `/etc/opt/omi/ssl/omi-host-$(hostname).pem` (ligne 95), plus un
    **lien symbolique** `/etc/opt/omi/ssl/omi.pem` → le certificat (lignes 101-103).
    `/etc/opt/microsoft/scx/ssl/scx.pem` est lui-même un lien vers
    `/etc/opt/omi/ssl/omi.pem` (ligne 205).
  - **Propriétaire et mode** (lignes 108-111) : clé `chmod 600` + `chown omi:omi` ;
    certificat et lien `chmod 640` + `chown root:omi`.
  - **Sujet** (lignes 63-68) : `CN=<FQDN>`, `CN=<nom court>`, puis les composants
    `DC=` du domaine, « preferably in the order » indiqué.
  - **EKU** (ligne 29) : **Server Authentication seul** — Client Authentication est
    explicitement retiré des Application Policies du gabarit.
  - **Clé exportable** (ligne 25) : « Allow private key to be exported » coché sur le
    gabarit, la procédure passant par un `.pfx`.
  - **Confiance côté Windows** (ligne 75) : la CA et les CA intermédiaires doivent être
    exportées dans le magasin *root* de **tous** les serveurs d'administration et
    passerelles du pool de ressources UNIX/Linux.
  - **Action après remplacement** : `scxadmin -restart` (ligne 117) ou
    `systemctl restart omid` (ligne 166).
  - **Non spécifiés par S-4** : le **SAN** et la **taille de clé**. Cf. PO-004.
- **F-037** — **La restriction citée en sens contraire ne porte pas sur le même objet.**
  S-5, section `scxsslconfig`, énonce : « The generated certificate must be signed by
  Operations Manager management server in order to be used in WS-Management
  communication. Overwriting a previously signed certificate will require that the
  certificate be signed again. » Le sujet de la phrase est *le certificat généré par
  `scxsslconfig`*, c'est-à-dire l'auto-signé. S-4 et S-5 ne se contredisent donc pas :
  un auto-signé doit être contresigné par le serveur d'administration ; un certificat
  ADCS suit la voie de S-4. Cette lecture reste une lecture, pas une mesure : cf. PO-001.
- **F-038** — *(déplacé le 2026-09-21 vers `scomm-depot-actuel.md`)* — conflit entre S-5
  et le code actuel du dépôt, `scxsslconfig -f`. Il décrit le brouillon, pas la cible :
  il vit donc avec le brouillon. Cf. PO-003 **requalifié** et R-10 **réécrite**.
- **F-039** — `scxsslconfig` accepte `-b bits` (« number of key bits ») et
  `-e days` (défaut 3650) (S-5). Ces options ne concernent que l'auto-signé et ne
  transposent pas au gabarit ADCS.
- **F-040** — Arborescence de l'agent (S-5) : OMI dans `/opt/omi`, agent dans
  `/opt/microsoft/scx/`, outils dans `/opt/microsoft/scx/bin/tools`, configuration et
  certificats dans `/etc/opt/microsoft/scx/`, configuration OMI dans `/etc/opt/omi`.
  Cohérent avec les chemins déjà employés par le dépôt (`register.yml:10,39`,
  `templates/scom_sudoers.j2:9-11`).

#### Relecture ciblée de S-4 au livrable 3 (2026-09-21)

Question posée, et elle seule : **dans la procédure décrite par S-4, où la clé privée
est-elle engendrée ?** S-4 a été re-téléchargé et relu (P-28), même URL, même taille
qu'au 2026-09-18 : `http_code=200`, 10340 octets, 217 lignes.

- **F-042** — **Dans S-4, la clé privée naît sur un hôte Windows, puis elle est
  transportée vers la machine Linux.** `[SOURCE-EXTERNE]` S-4 tranche, et quatre lignes
  suffisent :
  - ligne 52 : « On the **Windows Server** that has been given the permission to the
    template, open the **computer certificate store**. » — la requête est émise depuis le
    magasin de certificats d'un hôte Windows ;
  - ligne 74 : « Right-click the certificate and **export it with a private key**.
    Finally, there should be a **.pfx** file. » — la clé existe donc déjà côté Windows,
    et c'est elle qu'on exporte ;
  - ligne 83 : « **Copy the certificate to the Unix/Linux server** for which the
    certificate was issued. » — le transport est explicite ;
  - ligne 87 : `openssl pkcs12 -in <FileName>.pfx -nocerts -out
    /etc/opt/omi/ssl/omikey.pem -nodes` — côté Linux, on **extrait** la clé du `.pfx`,
    on ne l'engendre pas.

  La *Method 2* (lignes 128-179) ne change rien : son script `extract_scx_cert.sh` prend
  le `.pfx` en premier argument (ligne 135) et rejoue les deux mêmes `openssl pkcs12`
  (lignes 150, 153). **Les deux méthodes de S-4 partent d'un `.pfx`.** Aucune ligne de
  S-4 ne fait naître une clé sur l'hôte Linux, et aucune ne mentionne de CSR.

  **Conséquence, et c'est le point de l'exercice.** F-036 relève que le gabarit de S-4
  exige « Allow private key to be exported » (ligne 25). Cette exigence est maintenant
  **expliquée** : elle est la condition du `.pfx`, donc du transport. Elle n'a de sens
  que dans ce chemin-là. Or `certmonger` + `cepces` fait naître la clé **sur l'hôte qui
  l'utilise** et n'exporte jamais rien (F-026, F-032 : `-k` désigne un fichier de clé
  local). **S-4 établit donc que l'agent SCX accepte un certificat de PKI externe
  (F-035), mais ne documente pas le chemin visé par ce projet.** Cf. PO-015.

- **F-043** — **Le gabarit décrit par S-4 est réglé « Supply in the request ».**
  `[SOURCE-EXTERNE]` S-4, ligne 33 : « On the **Subject Name** tab, select **Supply in
  the request**. Accept the prompt. There's no risk. » C'est ce réglage qui rend
  atteignable la forme de sujet à deux `CN` de F-036 : le sujet vient de la requête, pas
  de l'annuaire. **Ce que cela n'établit pas :** le réglage du gabarit **de cet
  environnement**, qui n'a pas été fourni par S-4 et qu'aucune documentation ne peut
  établir. Cf. PO-016.

### 1.6 Outillage sourçable pour la procédure de diagnostic (mesuré le 2026-09-21)

Ces faits sont nés de la rédaction de `procedures/diagnostic-enrolement.md`. **Toutes ces
mesures ont été prises sur le poste Fedora de rédaction**, aucune sur une cible (R-08).

Preuves : **P-29 à P-33** du journal. `[RÉVISÉE le 2026-09-21]` Ces sept faits portaient
leur commande et leur code de retour en ligne, faute de pouvoir écrire au journal dans le
périmètre du livrable 4 ; l'entorse à D-006 est soldée, les sorties sont au journal.

- **F-051** — **Versions des outils sur le poste de rédaction**, et rien d'autre.
  `openssl 3.5.7-2.fc44`, `curl 8.18.0-8.fc44`, `bind-utils 9.18.50-1.fc44`,
  `chrony 4.8-5.fc44`, `iproute 6.17.0-2.fc44`, `systemd 259.8-1.fc44`,
  `krb5-libs 1.22.2-4.fc44`. `curl` annonce `mit-krb5/1.22.2` parmi ses bibliothèques,
  ce qui rend `--negotiate` utilisable ici. Source : P-29.
  **⚠ TRANSPOSITION : ces numéros sont ceux de Fedora 44 et ne valent PAS pour RHEL 9.**
  Une option sourcée sur ces pages peut être absente ou différente sur la cible.
  @VERIF : rejouer `rpm -q` et `curl --version` sur la machine de recette, en
  `[ÉCRAN-…]`, avant de tenir une seule option de la procédure pour acquise.

- **F-052** — **Les outils Kerberos ne sont pas sourçables depuis ce poste.**
  `krb5-workstation` absent ; `command -v` et `man -w` en échec pour `klist`, `kinit`,
  `kvno`, `kdestroy`. Source : P-30.
  **Conséquence directe :** toutes les options de `klist`, `kinit`, `kvno` et `kdestroy`
  employées dans la procédure portent le marqueur de vérification. Ce n'est pas une
  précaution de forme — ce sont les étapes 2 et 3, c'est-à-dire le cœur.
  `[RÉVISÉE le 2026-09-21]` **Levée partielle :** ces options sont désormais sourcées
  **sur la cible**, par l'étape 0b de la procédure (§ 2 de ce livrable). Le marqueur reste
  tant que l'opérateur n'a pas produit cette capture.
  **Ce qui reste lisible ici :** `man krb5.conf` et `man kerberos`, fournis
  par `krb5-libs` (P-30). C'est la seule source
  Kerberos de ce livrable : `clockskew` « The default value is 300 seconds, or five
  minutes », `dns_canonicalize_hostname` « The default value is true », `rdns` « reverse
  name lookup will be used in addition to forward name lookup … The default value is
  true ».
  **Option écartée, et c'est une décision, pas un oubli :** la page de manuel pourrait
  être extraite du paquet `krb5-workstation` sans l'installer (`dnf download` +
  `rpm2cpio`). **Non fait d'initiative** — cela reviendrait à rapatrier un paquet sur un
  poste personnel, ce qui appartient à l'opérateur (R-06), et la page obtenue serait de
  toute façon celle de Fedora 44, donc soumise au même avertissement que F-051.

- **F-053** — **Le paquet qui fournit les outils Kerberos est `krb5-workstation`** :
  `/usr/bin/{klist,kinit,kvno,kdestroy}` et leurs quatre pages de manuel. Version
  disponible `1.22.2-4.fc44` (**Fedora 44, transposition à vérifier** — @VERIF :
  `dnf info krb5-workstation` sur une RHEL 9 abonnée). Source : P-31, qui déclare aussi
  que l'échec de P-12b ne s'est pas reproduit.

- **F-054** — **`openssl s_client` rend `rc=0` alors que la validation de chaîne a
  échoué** ; `rc=1` avec `-verify_return_error`. Mesuré sur un serveur TLS local
  (P-32), pas déduit. Cohérent avec `man openssl-s_client`, qui écrit à propos de `-verify` : « As a side
  effect **the connection will never fail due to a server certificate verify failure** »,
  et à propos de `-verify_return_error` : « returns verification errors instead of
  continuing ».
  **Ce fait gouverne l'étape 4 de la procédure** : sans `-verify_return_error`, le code
  de retour ne porte pas le résultat ; l'information est dans la ligne `Verify return
  code`. **Troisième membre de la famille F-041 / P-25** — un code de retour qui ne dit
  pas ce qu'on croit. Réserve de méthode (un processus serveur bref est un état créé) :
  consignée en P-32.

- **F-055** — **`dig` ne renseigne pas son code de retour ; `getent` si.**
  `dig +short` rend `rc=0` sur un nom inexistant, et `man dig` **ne comporte aucune
  section `EXIT STATUS`** sur ce poste. `getent hosts` rend `rc=2`, documenté
  littéralement : « 2 : One or more supplied key could not be found in the database ».
  Source : P-33.
  **Conséquence pour la procédure :** le contrôle de résolution s'appuie sur `getent`, et
  `dig` ne sert qu'à **séparer** « le serveur a répondu NXDOMAIN » de « aucun serveur n'a
  répondu » — ce que `+short` masque, d'où son absence dans ce rôle.

- **F-056** — **`timedatectl timesync-status` échoue sur une machine sous `chronyd`**
  (`rc=1`, « Command requires systemd-timesyncd.service, but it is not available »), pour
  une raison étrangère à l'heure, alors que `chronyc tracking` et
  `timedatectl show -p NTPSynchronized` répondent sans élévation. Source : P-33.
  **Conséquence :** la procédure emploie `timedatectl show` et `chronyc tracking`, jamais
  `timesync-status`, qui enverrait chercher un problème d'horloge là où il n'y en a pas.

- **F-057** — **`curl` ne traite pas un code HTTP d'erreur comme un échec.**
  `man curl`, à `-f, --fail` : « **By default, curl does not consider HTTP response codes
  to indicate failure.** » Mesuré sur le serveur local de F-054 : `rc=60` /
  `http_code=000` sans chaîne de confiance, `rc=0` / `http_code=200` avec (P-32).
  `man curl`, codes de sortie, littéralement : `6` « Could not resolve host », `7`
  « Failed to connect to host », `22` « HTTP page not retrieved », `28` « Operation
  timeout », `35` « SSL connect error », `60` « Peer certificate cannot be authenticated
  with known CA certificates », `67` « The username, password, or similar was not
  accepted ».
  **Conséquence pour l'étape 5 :** un `401` laisse `rc=0`. Le résultat de
  l'authentification est dans `%{http_code}`, pas dans le code de retour. Et `-f` est
  **écarté** parce qu'il rendrait `22` indistinctement pour `401`, `403`, `404` et `405`
  — c'est-à-dire pour les quatre cas qu'il faut justement séparer.

**Renvoi :** la procédure qui emploie ces sept faits est
[`procedures/diagnostic-enrolement.md`](procedures/diagnostic-enrolement.md). Elle
n'a **aucune autorité propre** : tout ce qu'elle affirme vient d'ici.


### 1.7 La protection étendue de l'authentification — `[SOURCE-EXTERNE]`

Source nommée, lue intégralement en texte brut (P-34) :
- **S-6** `[SOURCE-EXTERNE]` — Microsoft, « Windows Extended Protection
  `<extendedProtection>` », dépôt `MicrosoftDocs/iis-docs`, fichier
  `iis/configuration/system.webServer/security/authentication/windowsAuthentication/extendedProtection/index.md`,
  `ms.date: 09/26/2016`, 213 lignes, http_code=200, récupéré le 2026-09-21.

- **F-062** — **La protection étendue repose sur *deux* mécanismes, pas un, et c'est
  le second qui gouverne le cas d'un intermédiaire.** S-6, lignes 20-21 :
  « Channel-binding information that is specified through a Channel Binding Token (CBT),
  **which is primarily used for SSL connections** » et « Service-binding information that
  is specified through a Service Principle Name (SPN), which is primarily used for
  connections that do not use SSL, **or when a connection is established through a
  scenario that provides SSL-offloading, such as a proxy server or load-balancer** ».
  Deux réglages les commandent (S-6, lignes 170-171), **tous deux à `None` par défaut** :
  `tokenChecking` ∈ {`None`, `Allow`, `Require`} — `None` « emulates the behavior that
  existed before extended protection » — et `flags`, dont `Proxy` « specifies that part
  of the communication path will be through a proxy ».
  Le tableau de scénarios (S-6, lignes 51-57) tranche notre cas, littéralement :
  *« Client connects to proxy server using SSL and proxy server connects to the
  destination server using HTTP (SSL off-loading) »* → flags `Proxy` → **« SPN checking
  will be used and channel-binding token checking will not be used. »** La ligne
  précédente dit la même chose du SSL de bout en bout à travers un proxy.
  Enfin, S-6 ligne 23 : « if a client is connecting to a destination server through a
  proxy server, **the SPN collection on the destination server would need to contain the
  SPN for the proxy server** », chaque entrée préfixée `HTTP/`.


### 1.8 `rhel_post_install`, inventorié — `[DÉPÔT-PUBLIC-2026-09-22 @ a1012bf]`

> **Réserve de provenance, qui s'applique à TOUS les faits F-063 à F-072.** Ce qui suit
> décrit **l'état du dépôt public au 2026-09-22**, commit `a1012bf`. Ce n'est **pas**
> l'état de ce qui a été joué sur les machines du parc. Aucune formulation du type « ce
> que fait la jonction aujourd'hui » n'est recevable ici — et l'étape 1 de ce livrable
> établit que les deux **divergent** effectivement. Le dépôt reste en **lecture seule**
> (R-09) ; non-modification prouvée en P-35.

- **F-063** — Le dépôt est cloné en lecture seule à `/home/<user>/dev/rhel_post_install`,
  **hors de l'arbre de `scomm_rhel9`**, depuis `git@github.com:` (l'URL `https://` aurait
  échoué, F-005). `HEAD = a1012bf`, branche `main`, 42 commits, dernier commit
  2026-09-15. Source : P-35.

#### Origine de l'entrée de bouclage (étape 1)

- **F-064** — **Mesure `[ÉCRAN-2026-09-22]`**, sur deux machines distinctes du parc : le
  fichier des hôtes porte **en première ligne** une entrée associant l'adresse de
  bouclage au **nom court puis au nom pleinement qualifié**. Sur l'une d'elles, le DNS
  résout le nom qualifié vers l'adresse de l'interface **et** la résolution inverse rend
  le même nom qualifié : la résolution concorde **dans les deux sens**. L'ordre des
  sources consulte les fichiers avant le DNS. **L'entrée locale masque donc une
  concordance existante ; elle ne pallie pas une absence.**
  *Valeurs d'exemple employées ici (R-07) :* `host01`, `host01.example.com`. Ni nom réel,
  ni domaine, ni adresse, ni serveur de noms ne figurent dans ce dépôt public.

- **F-065** — **Le dépôt écrit bien une entrée de bouclage, mais pas dans l'ordre
  mesuré.** `tasks/set_hostname.yml:6-11` emploie `lineinfile` avec
  `regexp: '^\s*127\.0\.0\.1'` et
  `line: "127.0.0.1  {{ inventory_hostname }}.{{ ad_domain }} {{ inventory_hostname }}"`,
  soit **FQDN puis nom court** — l'inverse de F-064. Historique de cette ligne : nom
  court **seul** depuis l'init (`5992c5c`, 2026-07-07), FQDN ajouté le **2026-07-13**
  (`71698af`). **Aucune version de ce fichier n'a jamais écrit « court puis qualifié ».**
  Source : P-36.

- **F-066** — **L'ordre mesuré vient d'un script supprimé.** `shell/ad_pki.sh:91`,
  retiré le **2026-07-08** par `e34b04c` (« Removed shell script. Will not be maintained
  anymore. »), écrivait
  `sed -i -E "s|^\s*127\.0\.0\.1.*|127.0.0.1 ${INVENTORY_HOSTNAME} ${FQDN}|" /etc/hosts`
  et, en repli, `echo "127.0.0.1  ${INVENTORY_HOSTNAME} ${FQDN}" >> /etc/hosts` — soit
  **nom court puis FQDN**, exactement l'ordre de F-064. Source : P-36.

- **F-067** — **Aucun autre écrivain du fichier des hôtes dans ce dépôt**, ni à HEAD ni
  dans l'historique : une seule occurrence de `path: /etc/hosts` parmi 28 chemins écrits,
  aucun modèle ne contient d'adresse de bouclage, aucun `sed` à HEAD, et les deux seuls
  fichiers jamais supprimés sont `shell/ad_pki.sh` (F-066) et `tmp.sh` (qui n'y touche
  pas). **Chaque motif a été exercé sur un cas positif réel avant de conclure** — dont un
  contrôle qui a dû être repris parce qu'il ne trouvait que des sous-chaînes. Source :
  P-36.

- **F-068** — **Motif documenté de cette écriture : il n'y en a pas, au-delà du sujet du
  commit.** Le corps de `71698af` est **vide** ; `tasks/set_hostname.yml` ne porte aucun
  commentaire sur ce point ; `README.md:12` dit seulement « Set FQDN and update
  `/etc/hosts` ». Le seul énoncé disponible est le sujet du commit : « **Added FQDN to
  resolve loopback** » — qui dit *ce qui* a été fait, pas *pourquoi*. Source : P-36.

#### Inventaire (étape 2)

- **F-069** — **Jonction au domaine.** `tasks/AD_join.yml`. Identifiant de jonction
  **obtenu de HPAM** (`:5-17`), jamais fourni d'avance. Outil : `realm join
  --automatic-id-mapping=no --user=… [--computer-ou=…]` via `shell`, `no_log: true`
  (`:54-61`), conditionné par `ad_domain not in realm_list_check.stdout` (`:61`).
  Paquets installés (`:30-42`) : `realmd`, `oddjob`, `oddjob-mkhomedir`, `sssd`,
  `sssd-tools`, `adcli`, **`krb5-workstation`**, `samba-common-tools`, `NetworkManager`.
  SSSD configuré par `blockinfile` depuis `templates/sssd_ad.conf.j2` (`:98-103`).
  **Sur le keytab : il n'y a rien à lire.** Le dépôt ne lit, n'écrit ni n'inspecte
  `/etc/krb5.keytab` nulle part ; ses entrées sont celles que `realm join` produit, et
  ce dépôt n'en dit rien.

- **F-070** — **HPAM / vault.** `tasks/hpam-read.yml`, écrit pour être appelé **comme une
  fonction** : `include_tasks` + `vars: hpam_read_secret_title`, retour dans le fait
  `hpam_read_result.{username,password,secret_id}` (`:5-18`, `:168-174`).
  API BeyondTrust Password Safe, module `uri`, **`use_gssapi: true`** (`:65`) — donc
  Kerberos Negotiate. **S'exécute sur le nœud de contrôle** : l'appelant pose
  `apply: {delegate_to: localhost, become: false}` (`AD_join.yml:8-10`).
  Préconditions écrites dans le fichier (`:27-29`) : « python3-gssapi » et « a valid
  Kerberos ticket (kinit) **for the running user** ». L'identité est
  `hpam_operator: "{{ lookup('pipe', 'id -un') }}"`, **résolue sur le nœud de contrôle**
  (`group_vars/all/00-defaults.yml:18-22`). Session ouverte puis fermée dans
  `block`/`always` (`:90`, `:176-192`), `no_log: true` sur tout ce qui porte un secret.
  **Point à relever : `validate_certs: false` sur les cinq appels `uri`** (`:66`, `:99`,
  `:121`, `:152`, `:187`) — la validation TLS est désactivée face au vault. Cf. PO-022.
  **Ce que cela commande pour ce projet :** les deux URL et le nom du gabarit (F-045)
  peuvent parvenir au rôle **par un titre de secret**, sans jamais figurer dans un dépôt
  public (R-07) — c'est la forme que `ad_join_user_title` emploie déjà.

- **F-071** — **`cepces` et `certmonger` sont déjà installés et configurés par ce dépôt**,
  dans `tasks/PKI_enrolment.yml`, conditionné par
  `adcs_server | default('', true) | length > 0` (`main.yml:41`) — donc **sauté par
  défaut**, `adcs_server` étant commenté (`group_vars/rhel9_hosts/00-defaults.yml:62`).
  - Installation : `dnf name: [certmonger, cepces, ca-certificates] state: present`
    (`:16-22`). **Aucune version épinglée, aucun dépôt déclaré** : le paquet est supposé
    disponible dans les dépôts activés de la cible. **Cela n'instruit donc pas PO-006** —
    le dépôt suppose ce que PO-006 demande de mesurer, il ne le mesure pas.
  - `cepces.conf` écrit par `community.general.ini_file` (`:48-76`) :
    `global.endpoint = https://{{ adcs_server }}/ADPolicyProvider_CEP_Kerberos/service.svc/CEP`,
    `global.auth = Kerberos`, `kerberos.realm = {{ ad_domain | upper }}`.
    **Aucune option `cas` n'est posée** — or c'est elle que F-029 désigne comme le point
    exact où la terminaison TLS du F5 devient déterminante (PO-005).
  - `getcert add-ca` n'est **jamais** appelé ; le dépôt se borne à vérifier que
    `getcert list-cas` mentionne `cepces` et échoue sinon (`:78-88`). Il suppose donc que
    le paquet enregistre lui-même la CA.
    `[NOTE du 2026-09-29]` **Le paquet RHEL 9 échoue à le faire** (F-121). Ce contrôle
    échouerait donc aujourd'hui — et c'est ce qu'on attend d'une garde. Cf. PO-041.
  - Demande : `getcert request -c cepces -k /etc/pki/tls/private/<fqdn>.key
    -f /etc/pki/tls/certs/<fqdn>.crt -g 2048 -N "CN=<fqdn>" -D <fqdn>
    -K host/<fqdn>@<REALM> -T {{ adcs_cert_template | default('Machine') }}
    -C "systemctl reload httpd || true"` (`:96-108`).
  **Trois écarts avec ce qu'attend l'agent SCX (F-036), à constater sans conclure :**
  le sujet n'a **qu'un seul `CN`** et aucun composant `DC=` ; les chemins ne sont pas
  ceux de l'agent (`/etc/opt/omi/ssl/…`) ; l'accroche `-C` vise `httpd`, pas `omid`.

- **F-072** — **Identité d'exécution, et elle contredit une déclaration.**
  `ansible.cfg:4-6` : `become = true`, `become_method = sudo` ; `main.yml:4` :
  `become: true`. Les tâches s'exécutent donc **en `root` sur la cible**.
  `tasks/preflight.yml:24` et `:29` nomment **`ansible_admin` comme le compte de la
  CIBLE** (« mapped to the staff_u SELinux login on this **TARGET's** image »,
  « `semanage login -a -s unconfined_u ansible_admin` »).
  Et `tasks/preflight.yml:135-160` **exige un ticket Kerberos sur le NŒUD DE CONTRÔLE** :
  `klist -s` en `delegate_to: localhost, become: false`, avec l'avertissement « No valid
  Kerberos ticket found on the control node … Run: `kinit` ». Même forme pour
  `python3-gssapi` (`:96-133`).
  *Cf. PO-020, que ce fait instruit largement, et la contradiction avec F-059 qu'il
  soulève.*

- **F-073** — **Forme sous laquelle le dépôt accepte du travail.**
  **Aucun rôle, aucune collection locale, aucun greffon** : `roles/`, `collections/` et
  `plugins/` sont absents (rc=2 pour les trois). Un playbook unique `main.yml` qui
  enchaîne **18 `ansible.builtin.include_tasks`** vers `tasks/<Sujet>.yml`, chacun
  conditionné par un `when:` sur une variable de groupe. Handlers centralisés dans
  `main.yml:71-102`. Variables dans `group_vars/all/00-defaults.yml` (HPAM, nœud de
  contrôle) et `group_vars/rhel9_hosts/00-defaults.yml` (cible), **commentées par défaut**.
  Nommage des fichiers de tâches : mixte — `AD_join.yml`, `PKI_enrolment.yml`,
  `DNS_config.yml` en capitales, `preflight.yml`, `local_admin.yml` en minuscules.
  **Une seule étiquette existe : `preflight`** (`main.yml:10`, et 8 fois dans
  `preflight.yml`). Aucune autre tâche n'est étiquetée.
  Lint : `.yamllint` étend `default`, `line-length: 190`, `document-start: enable`,
  `truthy: ["true","false"]`.
  Collections épinglées dans `requirements.yml:8-12` : `ansible.posix <2.0.0`,
  `community.general <9.0.0`, **au motif écrit (`:5-7`) que les versions récentes ont
  abandonné `ansible-core` 2.14, « the ansible-core shipped with RHEL 9 »**.
  *C'est la première source lue qui nomme la version d'`ansible-core` de la cible.*
  **Constat seulement : aucune forme n'est proposée ici** (D-001 reste hors sujet).

---

## 2. Faits `[PILOTE-DÉCLARÉ]`, non mesurés

### 2.1 Prémisses du livrable 1 (2026-09-18)

`[RÉVISÉE le 2026-09-21]` Cette table portait auparavant le titre de la section entière.
Elle est devenue **§ 2.1** pour faire place aux déclarations du livrable 3 (§ 2.2). Son
contenu n'a pas changé, hormis deux renvois ajoutés en colonne de statut.

Aucun de ces points n'a été établi par un artefact, sauf mention contraire.

| Réf | Prémisse déclarée | Statut | Mesure qui la trancherait |
|---|---|---|---|
| D-P-01 | `scomm_rhel9` est cloné sur ce poste | **VÉRIFIÉE** — F-001, F-002 | — |
| D-P-02 | Aucun des deux dépôts ne porte de `CLAUDE.md` | **VÉRIFIÉE pour `scomm_rhel9`** (F-007) ; **non vérifiable** pour `rhel_post_install` (PO-002) | `ls CLAUDE.md` à la racine de `rhel_post_install` |
| D-P-03 | `rhel_post_install` joint les machines au domaine aujourd'hui | **NON VÉRIFIABLE** — dépôt absent du poste (PO-002). `[RÉVISÉE le 2026-09-21]` Repris et **restreint à la recette** par F-046 ; toujours non mesuré | Lecture du dépôt ; `realm list` sur un hôte provisionné `[ÉCRAN-…]` |
| D-P-04 | Le rôle est joué actuellement par le seed, en ansible-core | **NON VÉRIFIABLE** depuis ce poste. `[RÉVISÉE le 2026-09-21]` Redéclaré par F-050 ; la mesure reste due | `ansible --version` sur le nœud de contrôle `[ÉCRAN-…]` |
| D-P-05 | `cepces` est installable via `rhel_post_install` | `[RÉVISÉE le 2026-09-29]` **Moitié « paquet disponible » établie** : `cepces` est dans AppStream de RHEL 9 (F-119, `[ÉCRAN-2026-09-28, relayé]`). « **Via `rhel_post_install`** » reste non exercé (F-071 : tâche sautée par défaut). **ÉTAIT :** « **NON VÉRIFIABLE** — dépôt absent ; et `cepces` n'est pas dans les dépôts RHEL 9 de base à ma connaissance mesurée (F-023 ne mesure que Fedora) » — **infirmé** sur le second point | `dnf info cepces` sur une cible RHEL 9 `[ÉCRAN-…]` ; PO-006 |
| D-P-06 | CEP et CES sont configurés en Kerberos | **NON VÉRIFIABLE** depuis ce poste | PO-008 |
| D-P-07 | Le F5 est en reverse proxy devant l'ADCS ; terminaison TLS ou passthrough inconnue | **NON VÉRIFIABLE** ; l'opérateur déclare lui-même l'inconnue | PO-005 |
| D-P-08 | Les valeurs sensibles vivent dans un vault atteint par des tâches HPAM dans `rhel_post_install` | **NON VÉRIFIABLE** — dépôt absent. Constat contraire **dans `scomm_rhel9`** : aucun vault, aucune tâche HPAM (F-019) | Lecture de `rhel_post_install` |
| D-P-09 | Aucun certificat n'a jamais été délivré par cette chaîne F5/CEP/CES | **NON VÉRIFIABLE** ; cohérent avec F-017 (aucune brique n'existe côté code) | PO-001 |

**Prémisse infirmée :** l'énoncé du livrable suppose que `rhel_post_install` est
consultable sur ce poste (étape 2). **Il ne s'y trouve pas.** Trois recherches
indépendantes le confirment (P-16) : `find /home/<user> -maxdepth 8` sur
`*post_install*|*post-install*|*postinstall*` → 0 ligne, rc=0 ; `find / -xdev` sur
`*rhel_post*|*rhel-post*` → 0 ligne ; l'énumération de tous les `.git` sous le home ne
donne que `workstation-config`, `system_auto-update`, `glass-hud`, `scomm_rhel9` (plus
des sous-modules de caches `.cargo`/`.cache`). Le dépôt n'a **pas** été cloné : décision
réservée à l'opérateur. **L'étape 2 du livrable n'a donc pas pu être instruite** —
elle n'est pas escamotée, elle est bloquée. Cf. PO-002.


### 2.2 Faits déclarés au livrable 3 (2026-09-21) — `[PILOTE-DÉCLARÉ]`

**Aucun de ces sept faits n'est mesuré.** Ils sont déclarés par l'opérateur et inscrits
comme tels. Ils ne valent pas source au sens de R-01 : ils ne peuvent pas fonder une
prescription tant qu'ils ne sont pas vérifiés. La colonne de droite dit **par quoi** —
une case « aucune » est un aveu, pas un oubli.

- **F-044** — **Le code actuel de ce dépôt n'a aucune autorité.** `[PILOTE-DÉCLARÉ]`
  Il a été produit avant l'établissement de la méthode, sur une demande unique, sans
  source, à partir de documentation et de renseignements pris sur internet. **Il n'a
  jamais tourné en production.** Il peut être modifié ou entièrement réécrit.
  *Ce que cela change :* les faits F-008 à F-021 et F-038 cessent d'être des contraintes
  et deviennent la description d'un point de départ jetable (`scomm-depot-actuel.md`).
  *Vérifiable ?* **Non, et c'est définitif.** « N'a jamais tourné en production » est une
  affirmation négative sur le passé : aucune mesure présente ne peut l'établir. Un
  historique d'exécution côté seed pourrait au mieux la contredire. Elle restera
  `[PILOTE-DÉCLARÉ]`.

- **F-045** — **L'exigence d'un certificat ADCS est une décision de l'équipe
  SCOM/sécurité.** `[PILOTE-DÉCLARÉ]` Reçue par courriel, elle fournit **les deux URL**
  (point d'entrée CEP et point d'entrée CES) et **le nom du gabarit**. Ce n'est donc pas
  une hypothèse technique à valider : c'est une contrainte, avec un auteur et une date.
  Les valeurs ne sont **pas** reproduites dans ce dépôt (R-07) ; elles vivent dans le
  vault (cf. D-P-08).
  *Ce que cela change :* la question « faut-il vraiment ADCS ? » n'est plus ouverte. Ce
  qui reste ouvert est la **faisabilité** (PO-001, PO-005, PO-008), pas l'opportunité.
  *Vérifiable, et par quoi :* le courriel lui-même — auteur, date, objet — cité sans ses
  valeurs. @VERIF : identifier l'auteur et la date du courriel, et confirmer que le nom
  du gabarit qu'il porte est bien celui employé par `getcert request -T` (F-032).

- **F-046** — **Les machines de recette sont déjà jointes au domaine**, par le playbook
  `rhel_post_install`. `[PILOTE-DÉCLARÉ]`
  *Ce que cela change :* le prérequis Kerberos de cepces (F-028, « requires the client to
  be a Windows Domain Member with a valid Kerberos keytab ») est déclaré satisfait. La
  chaîne à écrire n'a donc **pas** à joindre le domaine, et ne doit pas y toucher (R-09).
  *Vérifiable, et par quoi :* @VERIF — `realm list` et `klist -k /etc/krb5.keytab` sur une
  machine de recette, rapportés en `[ÉCRAN-…]`. C'est la même mesure que PO-008.
  Reprend et précise D-P-03, qui portait sur le parc entier.

- **F-047** — **Le VIP F5 est joignable depuis la zone de recette.** `[PILOTE-DÉCLARÉ]`
  *Ce que cela change :* un `3 CONNECTERROR` du helper (F-033) ne pourra plus être imputé
  au filtrage réseau sans mesure. C'est ce qui rend la procédure de PO-005 jouable.
  *Vérifiable, et par quoi :* @VERIF — la procédure de PO-005 elle-même. Si le
  `openssl s_client` aboutit, la joignabilité est prouvée par surcroît ; s'il échoue, il
  faut distinguer « pas de route » de « TLS refusé », donc relever le code de retour
  **et** le message.

- **F-048** — **Le gabarit et les droits d'enrôlement couvrent les comptes machine de
  recette.** `[PILOTE-DÉCLARÉ]`
  *Ce que cela change :* la moitié « ACL » de PO-008 est déclarée résolue — S-4 (ligne 39)
  demande d'ajouter « the computer object of the server where the certificate will be
  enrolled » en *Read*, *Write*, *Enroll* (ligne 43), et l'opérateur déclare que c'est
  fait. **Ce qui reste entier**, c'est l'autre moitié : quel principal le client
  présentera réellement (compte machine `HOST01$@EXAMPLE.COM` ou principal de service
  `HOST/host01.example.com@EXAMPLE.COM`, F-028), et si le F5 le laisse passer.
  *Vérifiable, et par quoi :* @VERIF — côté AD, lecture des ACL du gabarit par l'équipe
  PKI ; côté hôte, `klist -k`. Les deux sont nécessaires : l'une sans l'autre ne prouve
  pas la correspondance.

- **F-049** — **Les essais se font sur une machine virtuelle unique**, avec instantané
  préalable et restauration. `[PILOTE-DÉCLARÉ]`
  *Ce que cela change :* le rayon d'action d'un essai raté est borné à une machine, ce qui
  autorise des essais destructifs — **mais pas tous** : l'instantané ne défait rien de ce
  qui est sorti de la machine. Cf. PO-017, qui énonce exactement ce qu'il ne défait pas.
  *Vérifiable, et par quoi :* @VERIF — identifiant de la VM et horodatage de l'instantané,
  rapportés en `[ÉCRAN-…]` **avant** le premier essai. Un instantané déclaré après coup ne
  prouve rien.

- **F-050** — **Le nœud de contrôle est le seed, en `ansible-core`. Ce n'est pas ce
  poste.** `[PILOTE-DÉCLARÉ]`
  *Ce que cela change :* tout contrôle qui suppose un nœud de contrôle
  (`--syntax-check`, `ansible-lint`, résolution de collections) se joue **là-bas** et
  revient en `[ÉCRAN-…]`. Ce poste reste un poste de rédaction. Requalifie PO-010.
  *Vérifiable, et par quoi :* @VERIF — `ansible --version` sur le seed, en `[ÉCRAN-…]`.
  C'est la mesure déjà demandée par F-022 et par D-P-04 ; elle n'a toujours pas été
  produite.

### 2.3 Les trois identités en présence (2026-09-21) — `[PILOTE-DÉCLARÉ]`

**Aucun de ces quatre faits n'est mesuré.** Ils commandent pourtant toute la lecture de la
procédure de diagnostic : une même commande n'établit pas la même chose selon l'identité
sous laquelle elle est jouée.

- **F-058** — **Sur le seed, l'opérateur obtient son ticket Kerberos automatiquement au
  login**, par SSSD, en session interactive, d'une validité de **huit heures**. Il ne fait
  **jamais** de `kinit` manuel. `[PILOTE-DÉCLARÉ]`
  *Conséquence :* sur le seed, un `klist` qui répond ne prouve **rien** sur la chaîne
  d'enrôlement — il prouve que la session est ouverte. Et y détruire un cache est coûteux :
  l'opérateur n'a pas l'habitude de reconstituer ce ticket.
  @VERIF : `klist` sur le seed, en `[ÉCRAN-…]`, principal et échéance relevés.

- **F-059** — **Les playbooks sont joués depuis le seed sous `ansible_admin`, compte local
  PAM**, donc **sans identité dans l'annuaire**. `[PILOTE-DÉCLARÉ]`
  `[RÉVISÉE le 2026-09-23 — INFIRMÉE par mesure]` **ÉTAIT** les deux blocs ci-dessus.
  **Mesuré `[ÉCRAN-2026-09-23]` sur `seed01`** : `id -un` rend le compte AD, `klist -s`
  rend `0` (F-074, P-38). Les playbooks sont joués **sous l'identité AD de l'opérateur**,
  qui porte un ticket — pas sous `ansible_admin`. L'objection tirée du code au livrable 6
  était fondée, mais c'est la **mesure** qui tranche. Reste non mesuré : PO-025.
  *Conséquence :* ce compte n'a aucun ticket Kerberos et ne peut pas en obtenir. C'est la
  question qu'ouvre PO-020.
  @VERIF : `getent passwd ansible_admin` et `klist` sous ce compte sur le seed, en
  `[ÉCRAN-…]` — un `klist` en échec y est le résultat **attendu**, pas une anomalie.

- **F-060** — **L'opérateur dispose d'un accès interactif à la machine de recette**, jointe
  au domaine, et d'un **`sudo` complet** sur celle-ci via son compte AD. `[PILOTE-DÉCLARÉ]`
  *Conséquence :* la procédure est jouable telle qu'elle est écrite, son unique élévation
  comprise. Mais l'identité de l'opérateur n'est **pas** celle qui enrôlera.

- **F-061** — **L'opérateur déclare pouvoir lire le keytab de la machine de recette ; il
  n'est pas établi que ce soit sans élévation.** `[PILOTE-DÉCLARÉ]`
  *Pourquoi ce n'est pas un détail :* si le keytab n'est lisible que par `root`, alors
  `certmonger` devra s'exécuter avec ce privilège, ou sous une identité qu'il faudra
  désigner. Le mode et le propriétaire du keytab **conditionnent la conception du rôle**,
  pas seulement le confort de la procédure.
  **Mesure prévue :** étape 2 de `procedures/diagnostic-enrolement.md`, qui tente
  **d'abord sans privilège** et n'élève qu'en cas d'échec, les deux relevés avec leur code
  de retour (R-06). **Le résultat est un fait à inscrire ici**, pas à laisser dans une
  capture.

**Ce que ces identités n'établissent pas l'une pour l'autre** : la matrice — quelle étape,
sous quelle identité, sur quelle machine, et ce qu'elle établit pour cette identité
seulement — vit dans la procédure, § 1 bis. Elle n'est pas recopiée ici.

### 2.4 Mesures et déclarations relayées le 2026-09-23 (livrable 7)

**Aucune n'a été produite sur ce poste.** Sorties, et **réserve de transmission qu'il faut lire** : P-38 à P-42. Libellés réduits (R-07) : cf. P-38.

- **F-074** — `[ÉCRAN-2026-09-22]` `[RÉVISÉE le 2026-09-24 — ÉTAIT 2026-09-23]`
  Sur `seed01`, en session interactive, `id -un` rend le
  **compte AD de l'opérateur** et `klist -s` rend **0**. **Infirme F-059.** P-38.
- **F-075** — `[TIERS-DÉCLARÉ]`, par écrit, équipe ADCS : le gabarit `<GABARIT>` est réglé
  **« Supply in the request »**. Clôt PO-016.
- **F-076** — `[TIERS-MESURÉ]` Sortie `setspn` de l'équipe ADCS : `HTTP/<VIP-FQDN>` est
  enregistré **dans l'annuaire, sur le compte de service du pool d'applications CES**.
  **Ce que cela établit :** le KDC peut émettre un ticket de service pour ce SPN — c'est
  ce que l'**étape 3** de la procédure met à l'épreuve.
  **Ce que cela n'établit PAS :** quoi que ce soit du réglage d'IIS. **Non vérifié :**
  l'absence de doublon sur un autre compte, qui empêcherait l'émission de tickets ; à
  contrôler **seulement si** l'étape 3 échoue. P-39.
  `[RÉVISÉE le 2026-09-24]` **ÉTAIT :** « … — **la condition que S-6 pose pour l'accès par
  proxy** (F-062) ». **Transposition entre deux objets distincts**, relue sur S-6 :
  l'enregistrement d'annuaire mesuré par `setspn` n'est pas la collection `<spn>` de S-6,
  qui est un **réglage d'IIS** — ligne 23, « The `<extendedProtection>` element **may
  contain a collection of `<spn>` elements** », et lignes 173-179, où `spn` figure parmi
  les *Child Elements* de cet élément de configuration.
- **F-077** — `[TIERS-DÉCLARÉ]`, non mesuré, formulé au conditionnel (« I think ») :
  réglages de protection étendue supposés **aux valeurs par défaut**, ce que S-6 donne
  comme `None` (F-062).
  `[RÉVISÉE le 2026-09-29 — INFIRMÉE]` L'administrateur ADCS déclare que `tokenChecking`
  **valait `Require`** avant le 2026-09-25 (F-112), et la configuration relevée donne
  aujourd'hui `Allow` (F-111). **La déclaration a bien été faite** : elle reste consignée
  comme telle ; c'est son **contenu** qui est infirmé. **ÉTAIT :** le paragraphe
  ci-dessus, sans réserve.
- **F-078** — `[TIERS-DÉCLARÉ]`, oral, en réunion : le serveur d'administration SCOM **de
  QA** fait confiance à la chaîne ADCS. **Non rejouable** : première piste à rouvrir si
  l'agent est refusé.
- **F-079** — `[TIERS-DÉCLARÉ]` **Personne ne sait dire ce que le serveur d'administration
  exige du certificat.** Aucun agent Linux n'a encore été intégré par ce chemin : **ce
  projet est le premier.** Cf. PO-023.
- **F-080** — `[TIERS-DÉCLARÉ]` + capture de l'équipe : `<GROUPE-T>` est membre de
  `<GROUPE-ALL>`, lui-même membre de `<GROUPE-PKI>`. L'équipe **n'a pas la main** sur
  `<GROUPE-PKI>` et a créé ses propres groupes.
- **F-081** — `[ÉCRAN-2026-09-23]` Lectures d'annuaire depuis `seed01`, **contre un seul
  contrôleur de domaine** (ce qui écarte la réplication) : témoin positif concluant,
  `host01` sans appartenance directe **avant**, membre direct de `<GROUPE-T>` **après**
  ajout de `host01$`, et une règle suivant les imbrications renvoie `host01` pour
  `<GROUPE-PKI>` — ce qui établit la chaîne de F-080 **et** que la règle parcourt bien les
  imbrications, la machine n'y étant pas membre directe. P-40.
- **F-082** — `[ÉCRAN-2026-09-23]` **`<COMPTE-JONCTION>` a le droit d'écrire les membres
  de `<GROUPE-T>`** — mesuré par l'écriture elle-même (F-081). Fait utile à
  l'automatisation ; cf. D-010 et PO-024.
- **F-083** — `[ÉCRAN-2026-09-23]` `host01` a été **redémarrée** après l'ajout, pour que son
  ticket porte l'appartenance. **Non vérifié sur le ticket** : aucune lecture du contenu du
  ticket n'a été produite.
- **F-084** — `[ÉCRAN-2026-09-23]` `adcli` : `add-member` attend le compte machine **avec**
  `$`, `show-computer` le nom court **sans**. `show-computer` affiche une **liste fixe**
  d'attributs, vides compris, **sans les appartenances de groupe** — il ne peut donc **pas**
  relire un ajout. Le **retrait** d'un compte machine d'un groupe **n'est pas documenté** :
  le synopsis ne mentionne que des utilisateurs. P-41.
- **F-085** — `[ÉCRAN-2026-09-23]` `adcli show-computer` affiche la **version de clé** du
  compte machine dans l'annuaire : comparée à celle du keytab, **elle dit directement si
  une machine restaurée porte une identité périmée**, avant qu'un `kinit` n'échoue pour une
  raison indiscernable. **Séparateur pour PO-017.** P-41.
- **F-087** — `[ÉCRAN ~2026-09-20]` **Étape 0b.1 : jouée.** `[NOTE le 2026-09-24]` Date
  **non corrigée** : la reprise du 2026-09-24 ne la couvre pas. Elle reste approximative.
  Sur `host01` :
  `krb5-workstation-1.21.1-10.el9_8`, `openssl-3.5.5-6.el9_8`, `curl-7.76.1-40.el9_8.5`,
  `bind-utils-9.16.23-40.el9_8.8` ; `klist`, `kinit`, `kvno`, `kdestroy` présents dans
  `/usr/bin`. **Date approximative**, dérivée de l'horodatage du fichier de capture : elle
  ne vient pas d'un relevé horodaté à l'exécution (R-15), et c'est une faiblesse de cette
  preuve, pas un détail. **0b.2 — la lecture des options sur la cible — reste à faire** et
  précède toujours les étapes suivantes.
- **F-088** — `[ÉCRAN-2026-09-21/22]` Sur `host01`, `hostname -f` rend le **nom court**.
  `[RÉVISÉE le 2026-09-24 — ÉTAIT « ~2026-09-20 », posée sans source]`
  Cohérent avec F-064 et F-066 : l'entrée locale masque le DNS, qui reste juste dans les
  deux sens. Rend l'étape 1 de la procédure **interprétable sans arrêt** (D-012).
- **F-086** — `[ÉCRAN-2026-09-23]` Les SPN du compte machine incluent la forme
  `host/<FQDN>`, celle que `PKI_enrolment.yml:105` demande comme principal (F-071).
  Élément pour PO-008, qui **ne le clôt pas**. P-41.

### 2.5 Le diagnostic joué le 2026-09-24 — `[ÉCRAN-2026-09-24, relayé]`

**Aucune de ces mesures n'a été vue par l'agent.** Provenance et réserve : **P-43 à P-47,
à lire avant cette section**. Libellés réduits (R-07) : cf. P-43.

**A — Sources des options, sur la cible.**
- **F-089** — **Étape 0b.2 jouée** ; les pages de manuel sont présentes sur `host01`.
  **Confirmées par la page :** `klist` lit un keytab et les types de chiffrement ;
  `kinit` depuis le keytab **et avec option de cache** ; option de cache de `kvno` et de
  `kdestroy` ; `KRB5CCNAME` en `FILE:chemin`, la page précisant que **ce type n'est pas
  une collection**. **Non capturés par la page, établis à l'usage :** la syntaxe du
  service de `kvno`, la désignation du principal par `kinit`. **Défaut de la recherche :**
  chaque motif couvrait plusieurs options sous **un seul `rc`**, qu'une trouvaille
  suffisait à rendre nul (PO-034). P-43.

**B — Contexte de la machine.**
- **F-090** — Sur `host01` : `hostname -f` rend le **nom court** (confirme F-088).
  `getent hosts <MACHINE-FQDN>` rend une **adresse IPv6 de lien local** ; `getent ahosts`
  rend l'**adresse de bouclage**, avec le **nom court** pour nom canonique. Le DNS, lui,
  rend l'adresse de l'interface, et la résolution inverse rend `<MACHINE-FQDN>` :
  **le DNS est juste dans les deux sens, ce sont les voies locales qui divergent.**
  Horloge synchronisée, décalage de l'ordre de la **microseconde**. P-44.
- **F-091** — **Deux mécanismes locaux faussent la résolution du nom, pas un.** L'entrée
  du fichier local pour IPv4 (F-064, F-066), et — **hypothèse, non mesurée** — la source
  de résolution `myhostname` pour IPv6. **Conséquence pour PO-021 : corriger le seul
  fichier pourrait démasquer le second**, et non régler la question.

**C — Identité Kerberos de la machine.**
- **F-092** — Keytab `root:root`, mode `600` : **lecture refusée sans privilège, accordée
  avec** ; élévation déclarée. **Répond à F-061** — `certmonger` devra ce privilège. Il
  porte, en **versions de clé 2 et 3**, les principaux du compte machine, `host/` et
  `RestrictedKrbHost/`, chacun sous **les deux formes de nom**. P-44.
- **F-093** — **PO-017 appliqué, sans échec.** Version de clé **3** au keytab comme dans
  l'annuaire ; la date de dernier changement de mot de passe, convertie, **tombe à la
  minute** de la dernière modification du keytab. **Identité non périmée** — le séparateur
  de F-085 a servi à l'établir, non à constater une panne.
- **F-094** — **`host/<MACHINE-FQDN>` est refusé comme client** — « Client … not found in
  Kerberos database » — **alors que sa clé est au keytab et que le SPN existe dans
  l'annuaire** ; **`<MACHINE>$@<REALM>` est accepté.** Répond à la part de PO-008 « quel
  principal la machine présente » : **son nom de compte**. Et corrige une confusion de la
  procédure entre le principal inscrit **dans le certificat** (`-K`, F-071) et l'identité
  **d'authentification** — un SPN est un service, pas un client (PO-031).
- **F-095** — Piège de citation : entre **guillemets doubles**, `$@` est remplacé par le
  shell. Le nom du compte machine se passe **entre apostrophes**.

**D — Ticket de service.**
- **F-096** — Ticket obtenu pour `HTTP/<VIP_FQDN>`, **sans canonisation du nom**.
  **L'hypothèse d'un SPN en double tombe**, et `[TIERS-MESURÉ]` l'équipe l'a confirmé par
  une recherche sur **tout le domaine**. La réserve posée en F-076 est levée. P-45.

**E — Chaîne de confiance : un blocage nouveau, et sa résolution.**
- **F-097** — Le VIP envoie une **chaîne complète** : certificat ← `<CA-N2>` ←
  `<ROOT-CA>` ; TLS 1.3, dates valides, **nom du VIP dans les noms alternatifs**. Le cas
  (b) de l'étape 4 — chaîne incomplète — est écarté. P-45.
- **F-098** — **La machine ne fait pas confiance à `<ROOT-CA>`** : erreur **19**, `curl`
  **60** — le cas (a) prévu par l'étape 4. Le contrôle a fonctionné.
- **F-099** — `[PILOTE-DÉCLARÉ]` **Aucun mécanisme ne distribue les autorités de
  l'entreprise aux machines RHEL.** Satellite n'est pas encore déployé et ses essais
  viendront **après** ce projet. Cf. PO-028.
- **F-100** — **Racine établie par deux sources indépendantes qui concordent.** L'annuaire
  publie **une seule** racine ; son empreinte SHA-256, lue par un canal **authentifié par
  Kerberos**, est **identique** à celle de la racine envoyée par le VIP. L'annuaire publie
  **trois autorités émettrices**, dont `<CA-N3>`, dont la **signature a été vérifiée contre
  `<ROOT-CA>`**. **Une seule ancre couvre le VIP et l'autorité d'enrôlement.** P-46.
  *C'est l'exercice de R-14 :* deux sources indépendantes, comparées **sur l'empreinte**.
- **F-101** — La page de manuel de `ldapsearch` **ne documente pas** l'option de repli des
  longues lignes ; `-t` a été établi **à l'usage**. @VERIF : reconfirmer sur la page de la
  version employée, ou accepter l'usage comme seule source.

**F — Authentification contre les services : la cause.**
- **F-102** — Sans identifiants : **`401`, avec proposition `Negotiate`**. La garde de
  l'étape 5 est satisfaite — l'accès anonyme n'est pas permis, l'écart est interprétable.
- **F-103** — Avec le ticket : **`401`, sans jeton en retour**, **dans toutes les
  combinaisons** : machine **et** opérateur, CEP **et** CES, URL réécrites **et**
  officielles, par le F5 **et en direct au serveur**. **Écartés :** le F5, l'identité,
  l'appartenance aux groupes.
  `[NOTE du 2026-09-29]` **Vrai au 2026-09-24, dépassé depuis** : à partir du 2026-09-25,
  le ticket de la machine est accepté (F-115 à F-117). La mesure reste vraie à sa date ;
  ce qui a changé est la configuration du serveur (F-112).
- **F-104** — **Le jeton est bien envoyé** : sortie détaillée, **envoi anticipé dès la
  première requête**, et **même refus** s'il est envoyé **après** le défi du serveur.
- **F-105** — `[TIERS-MESURÉ]` Équipe ADCS : SPN **unique** ; les **deux pools** tournent
  sous `<COMPTE-SVC-CES>` ; `useAppPoolCredentials` **et** `useKernelMode` **activés** sur
  les deux sites. **L'hypothèse d'un déchiffrement avec la clé du compte machine du
  serveur est réfutée.**
- **F-106** — `[TIERS-MESURÉ]` Journal IIS : `401`, **sous-code 1**, statuts Windows
  `0x8009030E` et `0xC000035B`.
- **F-107** — `[TIERS-DÉCLARÉ]` Lecture de l'équipe : « le client n'envoie pas
  d'identifiants ». **Réfutée par la mesure** (F-104) — le dire évite de la voir revenir.
- **F-108** — `[SOURCE-EXTERNE]` **S-7** — projet `curl`, *issue* **22466** : le statut
  `0xC000035B` y est attribué à un **échec de liaison de canal sous protection étendue**,
  et **la désactiver a fait passer `curl`**. *Réserve :* un fil de discussion n'est pas
  une documentation d'éditeur — il établit un précédent concordant, non le mécanisme.
- **F-109** — **Mesure décisive.** **En HTTP, en direct au serveur, le même ticket est
  ACCEPTÉ** : réponse SPNEGO décodée par `openssl asn1parse` — **état `00`**, mécanisme
  **Kerberos**, **réponse d'authentification mutuelle** — puis **`403`** parce que le site
  exige HTTPS. **La seule différence avec l'échec est le canal TLS.** P-47.
- **F-110** — Courriels envoyés à l'équipe ADCS ; **son administrateur revient le
  2026-09-28** : la confirmation côté serveur est datée, non indéfinie.
  `[NOTE du 2026-09-29]` Échéance tenue : configuration relevée le 2026-09-28 (F-111 à
  F-114). Nouvelle échéance : F-127.

### 2.6 Du 2026-09-25 au 2026-09-28 — tests, configuration du serveur, premier essai de `cepces`

**Aucune mesure de la recette ni du serveur n'a été vue par l'agent.** Provenance et
réserve de transmission : **P-48 à P-54, à lire avant cette section** ; la réserve de P-43
à P-47 s'y applique à l'identique. Libellés réduits (R-07) : cf. P-48. **Seule exception :**
les lectures des sources S-8 à S-10, faites sur le poste de rédaction (P-55).

**Machine.** Les faits du 2026-09-25 au 2026-09-28 portent sur **`host02`**, une autre
machine de recette que celle du 2026-09-24 (`host01`, § 2.5) `[PILOTE-DÉCLARÉ, opérateur,
2026-09-29]`. **Seule exception :** la requête CES de 13:22 le 2026-09-25, partie d'une
**troisième** machine de recette (F-115).

**Heures.** Toutes en **UTC**. Les en-têtes `Date` HTTP et le journal IIS sont en UTC ;
l'observateur d'événements de l'administrateur est en heure locale **+02:00** — aucun
de ses horodatages n'est repris ici.

**Classes de source.** `[TIERS-MESURÉ]` : sorties brutes transmises par l'administrateur
ADCS (`appcmd`, lignes du journal IIS, capture des pools). `[TIERS-DÉCLARÉ]` : ses
affirmations sans sortie. `[ÉCRAN-AAAA-MM-JJ, relayé]` : écrans de l'opérateur, lus par le
pilote. `[PILOTE-DÉCLARÉ]` : interprétations et corrections du pilote.

**A — Configuration du serveur, relevée le 2026-09-28.**
- **F-111** — `[TIERS-MESURÉ]` (`appcmd` et captures, P-54) Les deux applications — CEP,
  `ADPolicyProvider_CEP_Kerberos`, et CES, `<CA-N3>_CES_Kerberos` — portent
  `windowsAuthentication` avec `enabled="true"`, `authPersistNonNTLM="true"`,
  `useKernelMode="true"`, `useAppPoolCredentials="true"`, **le seul fournisseur
  `Negotiate`**, et `extendedProtection tokenChecking="Allow"`. **Ni `flags` ni collection
  `<spn>` ne sont affichés.** Concorde avec F-105 sur `useKernelMode` et
  `useAppPoolCredentials`.
- **F-112** — `[TIERS-DÉCLARÉ]` `tokenChecking` **valait `Require`** avant le 2026-09-25 ;
  l'administrateur l'a passé à `Allow`, puis a redémarré IIS **après** le test de 13:22
  UTC. **Ni l'heure du changement ni celle du redémarrage ne sont communiquées.**
  Infirme F-077. C'est le cas que PO-018 décrivait comme « une erreur de configuration,
  pas le cas général ».
- **F-113** — `[TIERS-MESURÉ]` (capture des pools, P-54) Pools `WSEnrollmentPolicyServer`
  (CEP) et `WSEnrollmentServer` (CES) : **démarrés**, sous **la même identité**,
  `<COMPTE-SVC-CES>` — le compte qui porte le SPN `HTTP/<VIP_FQDN>` (F-076, F-105).
- **F-114** — `[TIERS-DÉCLARÉ]` « Rien de pertinent » dans les journaux d'événements
  consultés. Formulation de l'administrateur, **sans liste des journaux consultés** : ne
  vaut pas absence d'erreur dans le journal Application (cf. F-125).

**B — Tests de l'opérateur.**
- **F-115** — **2026-09-25, vers 13:22 UTC** (P-48).
  `[TIERS-MESURÉ]` Journal IIS : CEP **sans** ticket → `401 2 5` ; CEP **avec** le ticket de
  `host02$` → **`500 0 0`**, `cs-username` = `<DOMAINE>\host02$` ; CES → `401 1`,
  `sc-win32-status` `3221226331` = `0xC000035B` ; événement **4625** sur `<SERVEUR-ADCS>`,
  paquet Kerberos, même statut.
  `[ÉCRAN-2026-09-25, relayé]` + `[PILOTE-DÉCLARÉ, opérateur]` La requête CES est partie
  d'**une autre machine de recette** que `host02` : déclaration de l'opérateur,
  corroborée par l'historique shell de cette machine, qui contient les commandes CES.
  **Le journal IIS ne le prouve pas** : le F5 traduit les adresses (F-118).
  `[PILOTE-DÉCLARÉ]` Identité de la requête CES **non établie** — probablement le ticket
  utilisateur de l'opérateur.
- **F-116** — **2026-09-25 à 15:33:15 UTC**, depuis `host02`, sous `host02$` (P-49).
  `[TIERS-MESURÉ]` Journal IIS : CES **sans** ticket → `401 2 5` ; CES **avec** ticket →
  **`500 0 0`**, `cs-username` = compte machine ; CEP **avec** ticket → `401 1`,
  `sc-win32-status` `2148074254` = `0x8009030E` = `SEC_E_NO_CREDENTIALS` (S-8).
  **Même statut que F-106**, relevé le 2026-09-24 sur `host01`.
- **F-117** — **2026-09-28, de 13:39:05 à 13:39:11 UTC**, depuis `host02`, **un seul
  ticket**, dans l'ordre CEP, CES, CEP, CES (P-50). `[ÉCRAN-2026-09-28, relayé]` Les
  quatre → **`500`**, avec `Persistent-Auth: true` ; sans ticket → `401`.
  **`SEC_E_NO_CREDENTIALS` non reproduit** : l'incident de F-116 reste **non expliqué**
  (PO-045).
- **F-118** — **Réseau.**
  - **Traduction d'adresse source au F5.** `[ÉCRAN-2026-09-25, relayé]` Les deux machines
    de F-115 ont des adresses **différentes** ; `[TIERS-MESURÉ]` IIS voit pour elles
    **la même** adresse client. Aucune adresse n'est écrite ici (R-07).
    **Conséquence :** l'adresse client du journal IIS et la source d'un événement 4625
    **ne désignent pas l'hôte d'origine**.
  - **Réécriture des chemins.** `[PILOTE-DÉCLARÉ]`, appuyé sur le `cs-uri-stem` du journal
    IIS `[TIERS-MESURÉ]` : le F5 publie **un chemin court par service**
    (`<CHEMIN-CEP-PUBLIÉ>`, `<CHEMIN-CES-PUBLIÉ>`) et le réécrit vers le chemin interne
    de l'application.
  - **Chaîne du VIP.** `[ÉCRAN-2026-09-28, relayé]` certificat ← `<CA-N2>` ← `<ROOT-CA>` ;
    le certificat serveur est émis par `<CA-N2>`, **pas** par `<CA-N3>`. Concorde avec
    F-097.
- **S-8** `[SOURCE-EXTERNE]` — page « Windows error 0x8009030E, -2146893042:
  SEC_E_NO_CREDENTIALS », `windows-hexerror.linestarve.com/0x8009030E`, lue le
  **2026-09-29** (`http_code=200`, P-55). Lignes lues : « SEC_E_NO_CREDENTIALS »,
  « No credentials are available in the security package », « Declared in winerror.h ».
  *Réserve, celle de S-7 :* ce n'est pas une documentation d'éditeur ; elle nomme le code,
  elle n'en donne pas la cause dans ce contexte.

**C — `cepces` sur `host02`, 2026-09-28.** `[ÉCRAN-2026-09-28, relayé]` sauf mention.
- **F-119** — **RHEL 9.8.** `cepces`, `cepces-certmonger`, `cepces-selinux`
  **0.3.17-1.el9**, `python3-gssapi` et `python3-requests-gssapi` sont dans le dépôt
  **AppStream** de RHEL : **aucun dépôt tiers n'est nécessaire**. `certmonger`
  **0.79.21-1.el9** est lui aussi dans AppStream. **EPEL n'est pas joignable** depuis la
  recette (connexion réinitialisée). **Clôt PO-006** ; infirme D-P-05 sur la disponibilité
  du paquet. P-51.
- **F-120** — **`cepces.conf` livré par le paquet**, lu **avant installation** par
  `rpm2cpio` (P-51) : `type=Policy`, `auth=Kerberos`, `endpoint` par défaut
  `https://${server}/ADPolicyProvider_CEP_${auth}/service.svc/CEP`, **`cas` non défini** ;
  section `[kerberos]` : keytab système, `principals` = `${shortname}$ ${SHORTNAME}$
  host/${SHORTNAME} host/${fqdn}`, **`delegate=True`** — commentaire du paquet : nécessaire
  si CES et l'autorité ne sont pas sur la même machine —, `enctypes` incluant des **types
  faibles**. `[PILOTE-DÉCLARÉ]` Déduction du pilote, **non lue dans le fichier** : `cas` non
  défini ⇒ confiance du **magasin système**. Complète F-029.
- **F-121** — **DÉFAUT DU PAQUET `cepces-certmonger`.** Son script `%post` exécute
  `getcert add-ca -c cepces -e /usr/libexec/certmonger/cepces-submit --install >/dev/null || :`.
  **Observé :** après installation, `getcert list-cas -c cepces` est **vide**, et
  `certmonger` est actif. `cepces-submit --help` : « --install  Installation mode: handle
  authentication errors gracefully ». P-51.
  `[PILOTE-DÉCLARÉ]` Lecture du mécanisme : `--install` n'étant pas dans les guillemets de
  `-e`, il est lu comme une option de `getcert` ; l'enregistrement échoue, et
  `>/dev/null || :` en efface la trace. **Échec silencieux** — exactement la forme que
  R-03 décrit.
- **F-122** — **DÉRIVE sur `host02`** : cinq gestes faits à la main, dans cet ordre (P-52,
  sur le modèle de P-42). Autorisés par D-014.
  1. **`/etc/hosts`** — sauvegarde `/etc/hosts.avant-scomm` ; la ligne « bouclage, nom
     court, FQDN » remplacée par la ligne `localhost` standard, et la ligne `::1`
     standard ajoutée. `hostname -f` rend désormais le **FQDN** (il rendait le nom court).
     `nsswitch` : `files dns myhostname`.
  2. **Ancre** — `<ROOT-CA>` prise **sur le VIP après contrôle de l'empreinte**, posée sous
     `/etc/pki/ca-trust/source/anchors/`, puis `update-ca-trust`. `curl` **sans**
     `--cacert` → `ssl_verify=0`. Nom du fichier non écrit (R-07).
  3. **Paquets** — transaction `dnf` **5** : `certmonger`, puis
     `systemctl enable --now certmonger` ; transaction **6** : `cepces`,
     `cepces-certmonger`, `cepces-selinux` et leurs dépendances, **12 paquets modifiés**.
     La transaction précédente est la **4** (jonction au domaine).
  4. **`cepces.conf`** — sauvegarde `cepces.conf.avant-scomm` ; `server = <VIP_FQDN>`,
     `endpoint = https://<VIP_FQDN>/<CHEMIN-CEP-PUBLIÉ>/CEP`.
  5. **Autorité** — `getcert add-ca -c cepces -e '/usr/libexec/certmonger/cepces-submit --install'`.
     **`--install` a été inclus PAR ERREUR du pilote** `[PILOTE-DÉCLARÉ]` : le helper
     enregistré **masque les erreurs d'authentification**. **À corriger avant toute
     demande de certificat** (PO-042).
  **Retours arrière disponibles.** Ciblés : les sauvegardes `.avant-scomm`, le retrait de
  l'ancre, `dnf history undo 6` puis `undo 5`. **Complet :** un instantané Hyper-V de
  `host02` pris **avant** le 2026-09-28, date exacte non communiquée
  `[PILOTE-DÉCLARÉ, opérateur]` (cf. F-049).
- **F-123** — **Premier appel de `cepces-submit`** :
  `CERTMONGER_OPERATION=GET-SUPPORTED-TEMPLATES cepces-submit` (P-53). **Kerberos obtenu
  sous `host02$`** — après l'échec de la forme en **minuscules**, que le keytab ne porte
  pas (il ne porte que la forme en majuscules). Le `POST GetPolicies` vers CEP rend un
  **corps vide** → `ParseError`, **`rc=4`**. **Aucun refus SELinux.**
- **F-124** — **Rejeu du même `POST` par `curl`**, **2026-09-28 à 14:29:01 et 14:29:03
  UTC**, avec deux valeurs de l'en-tête WS-Addressing `To` : chemin publié, puis chemin
  interne (P-53). Les deux → `HTTP/1.1 500 System.ServiceModel.ServiceActivationException`,
  `Content-Length: 0`, `Persistent-Auth: true`, jeton `Negotiate` envoyé. **L'en-tête
  `To` n'est pas en cause.**

**D — Interprétations. Aucune n'est mesurée.**
- **F-125** — **Hypothèse** `[PILOTE-DÉCLARÉ]`, appuyée sur `[SOURCE-EXTERNE]` S-9 et S-10 :
  depuis le passage à `Allow`, le réglage de protection étendue **de WCF**, dans le
  `web.config` des services, ne correspond plus à celui **d'IIS**, et WCF refuse
  d'activer le service. **Non confirmée.** La preuve serait le journal **Application** du
  serveur, source `System.ServiceModel`, **demandé à l'administrateur le 2026-09-28**.
  **Ce que les sources ne disent pas :** la correspondance entre les valeurs d'IIS
  (`None`, `Allow`, `Require`) et celles de WCF (`Never`, `WhenSupported`, `Always`)
  **n'est énoncée par aucune source lue**.
- **S-9** `[SOURCE-EXTERNE]` — `support.gfi.com/article/114306-error-the-extendedprotectionpolicy-policyenforcement-values-do-not-match-iis-has-a-value-of-whensupported-while-the-wcf-transport-has-a-value-of-never`.
  **Citée par le pilote, NON LUE** : `http_code=404` le 2026-09-29 (P-55). Seule l'URL
  est connue ; son libellé **n'est pas** un contenu lu.
- **S-10** `[SOURCE-EXTERNE]` — billet « WCF Error The ExtendedProtectionPolicy.PolicyEnforcement
  values do not match », `mydevtalks.blogspot.com/2016/07/wcf-error-extendedprotectionpolicypolic.html`,
  daté du 2016-07-20, lu le **2026-09-29** (`http_code=200`, P-55). Message cité : « The
  extended protection settings configured on IIS do not match the settings configured on
  the transport. The ExtendedProtectionPolicy.PolicyEnforcement values do not match. IIS
  has a value of Never while the WCF Transport has a value of Always. » Liaison citée :
  `<extendedProtectionPolicy policyEnforcement="Always"/>`. Solution donnée : régler la
  protection étendue d'IIS à « Required ».
  *Réserve :* un billet de blog, pas une documentation d'éditeur. Son cas est **l'inverse
  du nôtre** (IIS plus faible que WCF) : il établit que le **désaccord** produit ce
  message, pas le sens du désaccord ici.
- **F-126** — **CORRECTION D'INTERPRÉTATION** `[PILOTE-DÉCLARÉ]`. Les `500` sur `GET` du
  2026-09-25 et du 2026-09-28 avaient été lus comme l'effet probable d'un `GET` envoyé à
  un service SOAP. **Cette lecture est RETIRÉE** : depuis `tokenChecking=Allow`, ces `500`
  sont vraisemblablement **la même erreur d'activation** que F-124.
  **« Authentifié » reste établi** (`cs-username` au journal IIS, `Persistent-Auth: true`) ;
  **« service fonctionnel » ne l'est pas.**
  *Sans antécédent écrit retrouvé :* au commit `be1a70b`, le motif `500` n'apparaît dans
  **aucun des 21 fichiers suivis** (`git grep`, rc=1 ; témoin `401` trouvé, rc=0) ni dans
  **aucun commit de l'historique** (`git log -G'500'`, sortie vide ; témoin `F-106`
  trouvé) ; « erreur interne », « internal server », « 5xx » et « ServiceActivation » en
  sont absents aussi (rc=1). **Ces recherches n'excluent pas** une formulation qui
  n'emploierait aucun de ces termes. Aucun texte retrouvé, donc pas d'`ÉTAIT` à poser.
  Le texte voisin le plus proche est l'hypothèse du `405` dans la procédure (PO-047).
- **F-127** — **État.** Blocage **côté serveur**, hors des droits de l'équipe.
  L'administrateur a été sollicité le 2026-09-28 ; **retour annoncé le 2026-10-01**
  `[TIERS-DÉCLARÉ]`.

---

## 3. Points ouverts


### 3.0 Requalification au 2026-09-21 — une ligne par point ouvert

Chaque point ouvert de PO-001 à PO-014 est ici **maintenu**, **requalifié** ou **clos**.
Les fiches détaillées qui suivent portent la marque `[RÉVISÉE le 2026-09-21]` là où le
statut a changé ; rien n'a été effacé (R-11).

| Réf | Verdict | En quoi / par quoi |
|---|---|---|
| PO-001 | **Maintenu**, et **élargi** | F-042 montre que S-4 ne documente pas le chemin certmonger ; ce n'est plus seulement « cette installation l'accepte-t-elle ? » mais « l'accepte-t-elle **sans `.pfx`** ? ». Scindé : PO-015 porte la part nouvelle. |
| PO-002 | **Maintenu** | `rhel_post_install` reste absent de ce poste ; F-046 le cite mais ne le rend pas lisible. Inchangé. |
| PO-003 | **Requalifié** en défaut de brouillon | F-044 : le code n'a aucune autorité et sera réécrit. Ce n'est pas un risque d'exploitation mais une chose **à ne pas reconduire**. R-10 réécrite en conséquence — voir la contestation partielle en fiche. |
| PO-004 | **Maintenu** | SAN et taille de clé restent non spécifiés par S-4 (F-036) ; F-042 n'y change rien. |
| PO-005 | **Requalifié**, rétréci | Reste entier sur la terminaison TLS ; F-047 retire la joignabilité de l'inconnue. Devient le cœur du livrable 4 avec PO-008. |
| PO-006 | **Maintenu** | Versions RHEL 9 de `certmonger`/`cepces` toujours non mesurées ; F-050 désigne où mesurer, pas quoi. `[RÉVISÉE le 2026-09-29]` Depuis : **CLOS** par F-119 — voir la fiche. |
| PO-007 | **Maintenu** | Aucun `credential.helper` ; rien n'a été modifié. Affiné mais non rouvert par P-25. |
| PO-008 | **Requalifié**, scindé | F-048 déclare la moitié « ACL du gabarit » résolue ; la moitié « quel principal est réellement présenté, et le F5 le laisse-t-il passer » reste entière. C'est elle qui va au livrable 4. |
| PO-009 | **CLOS le 2026-09-21** | Un inventaire vide est une propriété normale d'un brouillon (F-044). Rien à trancher. La mesure F-010 reste au dossier du brouillon et continue d'illustrer R-03. |
| PO-010 | **Requalifié** — change d'objet | F-050 : le nœud de contrôle est le seed, pas ce poste. La mesure se joue là-bas. F-041 reste vrai **de ce poste**, mais cesse d'être un obstacle au projet. |
| PO-011 | **Reste clos**, mais sa clôture était incomplète | La substitution avait atteint un **motif de recherche** en P-18 ; corrigé le 2026-09-21. La clôture tient, le contrôle qui l'établissait était faux. |
| PO-012 | **Maintenu**, et devenu bloquant pour une règle | Non arbitré. L'extension de R-07 aux identifiants personnels en dépend : voir D-008. |
| PO-013 | **Maintenu** | Irréversible, non touché. Aucune réécriture d'historique n'a été faite ni proposée. |
| PO-014 | **Maintenu** | `glass-hud` toujours non audité ; hors périmètre de ce livrable aussi. |

**Contestation de l'avis de l'opérateur, là où je ne le suis pas entièrement** (R-02) :

1. **PO-003 / R-10 — d'accord sur le fond, pas sur la forme du retrait.** Requalifier en
   « à ne pas reconduire » est juste : F-010 (inventaire vide) fait qu'aucune machine
   n'est atteignable depuis ce dépôt, et F-044 acte la réécriture. **Mais supprimer R-10
   supprimerait aussi la seule trace du piège.** Le danger n'est pas le fichier
   `register.yml` : c'est le geste « régénérer un certificat local » placé après un
   enrôlement. Ce geste peut renaître sous un autre nom dans la réécriture. R-10 est donc
   **réécrite** pour viser le comportement et non le fichier, et elle porte désormais sa
   propre date de révision. Retirer le garde et garder le motif aurait perdu le motif.

2. **PO-005 et PO-008 ne restent pas « entiers ».** F-047 et F-048 en retirent chacun une
   moitié — par déclaration, pas par mesure, donc la moitié retirée reste `[PILOTE-DÉCLARÉ]`
   et peut revenir. Les nommer « entiers » ferait chercher au livrable 4 ce qui est déjà
   déclaré acquis, et masquerait que ce qui reste est **plus étroit et plus dur** : non
   pas « le F5 est-il joignable » mais « que présente-t-il, et qu'accepte-t-il ».

3. **PO-010 : d'accord, avec une conséquence que l'avis ne tire pas.** Déplacer le
   contrôle syntaxique sur le seed le rend **médiat et lent** : chaque vérification passe
   par l'opérateur et revient en capture d'écran. Le livrable 4 ne doit donc pas fonder
   sa conception sur un aller-retour de contrôle rapide. C'est un coût, pas seulement un
   changement d'adresse.

4. **PO-009 : clos, sans réserve.** L'avis est juste et je n'ai rien à y opposer.

### PO-001 — Le serveur d'administration SCOM acceptera-t-il, *dans cet environnement*, un agent dont le certificat est émis par l'ADCS ?
`[RÉVISÉE le 2026-09-23]` — **maintenu ; une piste s'ajoute, une inconnue se nomme.**
F-078 `[TIERS-DÉCLARÉ]`, oral : le serveur d'administration **de QA** fait confiance à la
chaîne ADCS — non rejouable, **première piste à rouvrir si l'agent est refusé.** F-079 :
**personne ne sait ce que le serveur d'administration exige du certificat**, et **aucun
agent Linux n'a été intégré par ce chemin — ce projet est le premier.** F-036 cesse donc
d'être une contrainte à respecter pour devenir une **variable à découvrir** : PO-023.

`[RÉVISÉE le 2026-09-21]` — **maintenu et élargi.** La relecture de S-4 (F-042) montre
que la documentation qui fonde la faisabilité décrit un chemin avec `.pfx`, que ce projet
n'emprunte pas. La part nouvelle — « l'accepte-t-il **sans** `.pfx`, la clé étant née sur
l'hôte ? » — est portée par **PO-015**. La fiche ci-dessous est inchangée par ailleurs.

**Statut :** la faisabilité générale est établie par documentation (F-035, F-036) et la
contradiction apparente est levée (F-037). Ce qui reste non mesuré, c'est **cette**
installation.
**Pourquoi ça compte :** c'est la question qui peut annuler l'intérêt du projet. Une
réponse négative rend inutiles CEP/CES, le F5 et certmonger pour l'usage SCOM.
**Mesure qui tranche :** sur **un** hôte pilote, poser un certificat ADCS conforme à
F-036, redémarrer `omid`, puis lancer une découverte depuis la console SCOM et constater
l'état *Healthy*. Contrôle agent, court et délimité :
```
echo "=== DEBUT PO-001 ==="
openssl x509 -noout -in /etc/opt/microsoft/scx/ssl/scx.pem -subject -issuer -dates; echo "rc=$?"
systemctl is-active omid; echo "rc=$?"
ss -ltnp 2>/dev/null | grep -w 1270; echo "rc_grep=$?"
echo "=== FIN PO-001 ==="
```
**Qui peut la produire :** l'opérateur, avec l'administrateur SCOM pour la découverte.
Rapport en `[ÉCRAN-AAAA-MM-JJ]`.

### PO-002 — `rhel_post_install` est introuvable sur ce poste — **CLOS le 2026-09-22**
`[RÉVISÉE le 2026-09-22]` — **clos par l'accès, non par la découverte.** Le dépôt a été
cloné en lecture seule sur autorisation explicite de l'opérateur (F-063, P-35) ; il est
inventorié en § 1.8. Ce qui bloquait l'étape 2 du livrable 1 est levé.
**Ce que la clôture n'emporte pas :** le clone est l'état du **dépôt public** au
2026-09-22, pas celui des machines (réserve `[DÉPÔT-PUBLIC-…]` en tête du § 1.8). F-065
et F-066 montrent que les deux divergent. Un dépôt lisible ne rend pas le parc mesurable.

**Statut :** infirme une prémisse du livrable. Bloque intégralement l'étape 2
(jonction au domaine, tâches HPAM/vault, installation de `cepces`, structure du dépôt).
**Mesure qui tranche :** l'opérateur indique le chemin s'il existe ailleurs, ou autorise
explicitement un clone en lecture seule. **Aucun clone n'a été fait d'initiative.**
**Qui peut la produire :** l'opérateur.

### PO-003 — `register.yml` détruirait un certificat ADCS
`[RÉVISÉE le 2026-09-21]` — **requalifié : défaut de brouillon, non risque d'exploitation.**
F-044 : ce code n'a aucune autorité et n'a jamais tourné ; F-010 : l'inventaire est vide,
donc aucune machine n'est atteignable depuis ce dépôt. Ce n'est donc pas une menace
courante à surveiller, mais **une chose à ne pas reconduire dans la réécriture** (D-007).
*Ce qui subsiste, et c'est délibéré :* **R-10 n'est pas supprimée, elle est réécrite**
pour viser le **geste** — régénérer un certificat local sur un hôte enrôlé — et non le
fichier `register.yml`, qui va disparaître. Un garde attaché à un nom de fichier meurt
avec le fichier ; le piège, lui, peut renaître sous un autre nom.
*Ce qui cesse :* la fiche n'appelle plus de mesure. Le « avant/après `--tags register` »
ci-dessous ne sera pas joué : on ne mesure pas un brouillon qu'on jette.

**Statut :** conflit lu, pas supposé. `register.yml:13-24` lance `scxsslconfig -f`, et
`-f` force la régénération même si un certificat existe (F-038). Toute exécution du tag
`register` après un enrôlement écrase clé et certificat.
**Mesure qui tranche :** sur un hôte pilote, relever l'`issuer` avant/après un
`ansible-playbook … --tags register`. À ne tenter qu'après PO-001.
**Qui peut la produire :** l'opérateur. **Conséquence immédiate, sans attendre :** ne pas
exécuter le tag `register` sur une machine enrôlée par CEP/CES.

### PO-004 — SAN et taille de clé attendus par l'agent SCX
**Statut :** S-4 ne les spécifie pas (F-036). certmonger sait poser un SAN DNS (`-D`) et
une taille (`-g`) (F-032) — reste à savoir ce que l'agent et le gabarit exigent.
`@VERIF : politique du gabarit ADCS + inspection d'un certificat SCX en production.`
**Mesure qui tranche :** `openssl x509 -noout -text -in /etc/opt/microsoft/scx/ssl/scx.pem`
sur un agent sain, section *X509v3 Subject Alternative Name* et *Public-Key*.
**Qui peut la produire :** l'opérateur `[ÉCRAN-…]`.

### PO-005 — Le F5 termine-t-il le TLS, ou est-il en passthrough ?
`[RÉVISÉE le 2026-09-29]` — **un élément nouveau, non une clôture.** Le F5 réécrit les
chemins publiés vers les chemins internes (F-118). **Raisonné, non mesuré :** réécrire un
chemin HTTP suppose de lire la requête, donc de **déchiffrer le TLS** au F5 — terminaison,
éventuellement suivie d'un rechiffrement vers le serveur. Ce raisonnement **ne tranche pas**
entre terminaison simple et ré-chiffrement ; la configuration du F5 seule le ferait.

`[PILOTE-DÉCLARÉ]` **Indice plus fort, toujours non mesuré :** sous `tokenChecking=Require`
(avant le 2026-09-25, F-112), les tickets de la machine et de l'opérateur ont été refusés
(F-103), avec `0xC000035B` au journal (F-106), statut attribué à un échec de liaison de
canal (F-108, S-7, source non éditeur) ; sous `Allow`, le ticket de la machine est accepté
(F-116, F-117). Réserve : machines différentes (`host01` le 24, `host02` ensuite). C'est
l'effet attendu si le canal TLS vu par IIS n'est pas celui du client — terminaison au F5,
avec ou sans rechiffrement. **Ne tranche pas** entre ces deux cas, et ne vaut pas mesure de
la configuration du F5. F-115 n'est pas retenu : l'heure du passage à `Allow` n'est pas
connue (F-112), et la requête CES de 13:22 ne portait pas le ticket de la machine.

`[RÉVISÉE le 2026-09-21]` — **requalifié, rétréci ; cœur du livrable 4.** F-047 déclare le
VIP joignable depuis la zone de recette : la joignabilité sort de l'inconnue — **par
déclaration, non par mesure**, et peut donc y revenir. Ce qui reste est plus étroit et
plus dur : non pas « peut-on l'atteindre » mais **« que présente-t-il, et qu'accepte-t-il »**.

**Statut :** déclaré inconnu par l'opérateur (D-P-07). Détermine deux choses :
(a) le contenu du bundle `cas` de `cepces.conf` (F-029) — chaîne de l'ADCS si passthrough,
chaîne du F5 si terminaison ; (b) le SPN que le client Kerberos demandera, donc si
l'authentification GSSAPI peut aboutir (F-028).
**Mesure qui tranche, depuis une cible :**
```
echo "=== DEBUT PO-005 ==="
openssl s_client -connect ca.example.com:443 -servername ca.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer; echo "rc=$?"
echo "=== FIN PO-005 ==="
```
L'`issuer` désigne qui a émis le certificat présenté : l'ADCS/PKI interne (passthrough ou
ré-émission) ou l'autorité du F5 (terminaison). Rapport en `[ÉCRAN-…]`.
**Qui peut la produire :** l'opérateur, ou l'équipe F5 sur simple question.

### PO-006 — Versions de `certmonger` et `cepces` disponibles sur RHEL 9 — **CLOS le 2026-09-29**
`[RÉVISÉE le 2026-09-29]` — **CLOS** par **F-119** `[ÉCRAN-2026-09-28, relayé]` : sur RHEL
9.8, `cepces` **0.3.17-1.el9** et `certmonger` **0.79.21-1.el9** sont dans **AppStream** ;
aucun dépôt tiers n'est nécessaire, et EPEL n'est pas joignable depuis la recette. La
supposition « EPEL ou autre » du `@VERIF` ci-dessous est donc tranchée : **ni l'un ni
l'autre**. Titre **ÉTAIT :** « PO-006 — Versions de `certmonger` et `cepces` disponibles
sur RHEL 9 ».

`[RÉVISÉE le 2026-09-22]` — **maintenu ; l'inventaire ne l'instruit pas, il en déplace
l'enjeu.** `PKI_enrolment.yml:16-22` installe `certmonger`, `cepces` et `ca-certificates
` en `state: present`, **sans version épinglée et sans déclarer de dépôt** : le paquet est
**supposé** disponible dans les dépôts activés de la cible. Le dépôt suppose donc
exactement ce que PO-006 demande de mesurer (F-071). Ce que cela apprend malgré tout :
quelqu'un a jugé l'installation possible — mais un `dnf` écrit n'est pas un `dnf` qui a
réussi, et rien ici ne dit qu'il a tourné. **Mesure inchangée.**

**Statut :** non mesuré. Les numéros F-034 sont ceux de Fedora 44 et ne valent pas pour
la cible. `cepces` n'est pas un paquet RHEL de base à ma connaissance **non mesurée** —
`@VERIF : dnf info cepces sur une RHEL 9 abonnée, et identification du dépôt qui le
fournit (EPEL ou autre).`
**Mesure qui tranche :**
```
echo "=== DEBUT PO-006 ==="
dnf info certmonger cepces 2>&1 | grep -E '^(Name|Version|Release|Repo)'; echo "rc=$?"
echo "=== FIN PO-006 ==="
```
**Qui peut la produire :** l'opérateur sur une cible RHEL 9 `[ÉCRAN-…]`.

### PO-007 — Aucun `credential.helper` : le push échouera
**Statut :** mesuré (F-004, F-005). Non corrigé : configuration d'accès réservée à
l'opérateur, et le push est le seul acte irréversible du dépôt.
**Mesure qui tranche :** l'opérateur configure son accès, puis `git push`.
**Qui peut la produire :** l'opérateur seul.

### PO-008 — Quelle identité Kerberos l'ADCS accepte-t-elle pour l'enrôlement ?
`[RÉVISÉE le 2026-09-29]` — **un élément s'ajoute côté client.** Les `principals` par
défaut du `cepces.conf` livré sont `${shortname}$ ${SHORTNAME}$ host/${SHORTNAME}
host/${fqdn}` (F-120) ; sur `host02`, la forme en **minuscules** échoue d'abord (absente du
keytab), la forme en **majuscules** réussit (F-123). Le ticket de `host02$` est **accepté**
par IIS (F-115 à F-117). **Reste ouverte, inchangée :** la part ACL — l'autorisation
d'enrôler ne se voit qu'à une demande de certificat.

`[RÉVISÉE le 2026-09-24]` — **la part « quel principal la machine présente » est répondue
par mesure : son nom de compte, `<MACHINE>$@<REALM>`** ; `host/<MACHINE-FQDN>` est refusé
comme **client** bien que sa clé soit au keytab (F-094). Reste ouverte la part
« l'ADCS l'autorise-t-il à enrôler » : une ACL, qui ne se lit pas depuis la machine.

`[RÉVISÉE le 2026-09-22]` — **l'inventaire apporte une réponse partielle, et une seule.**
`PKI_enrolment.yml:105` pose `-K host/{{ ansible_fqdn }}@{{ ad_domain | upper }}` : le
principal inscrit en SAN Kerberos est donc un **principal de service `host/`**, et non le
compte machine `MACHINE$@REALM` — c'est l'une des deux formes que F-028 laissait ouvertes.
**Ce que cela n'apporte pas :** `-K` désigne ce qui est *demandé dans le certificat*, pas
l'identité sous laquelle le client *s'authentifie*. Celle-ci reste celle que `cepces`
choisit, et `cepces.conf` n'y pose ni `--keytab` ni `--principals` (F-071) : il s'en
remet à ses défauts. **La moitié « quel principal est réellement présenté » reste
entière**, et c'est elle que mesure l'étape 3 de la procédure. Par ailleurs,
`AD_join.yml:39` installe `krb5-workstation` sur les cibles — l'étape 0b de la procédure
y trouvera donc `klist`, `kinit`, `kvno` et `kdestroy`.

`[RÉVISÉE le 2026-09-21]` — **requalifié, scindé ; cœur du livrable 4.** F-048 déclare
résolue la moitié « ACL du gabarit » (le compte machine de recette y a *Read*, *Write*,
*Enroll*). Reste entière la moitié qui se mesure sur l'hôte : **quel principal le client
présente réellement** (compte machine ou principal de service, F-028) et si le F5 le
laisse passer. F-046 déclare par ailleurs les machines de recette déjà jointes, donc
pourvues d'un keytab — déclaration, pas mesure.

**Statut :** cepces accepte keytab + principal (F-028), mais le droit d'*Enroll* sur le
gabarit se donne à un objet Active Directory précis — S-4 (ligne 39) indique d'ajouter
« the computer object of the server where the certificate will be enrolled » avec
*Read*, *Write*, *Enroll* (ligne 43). Reste à confirmer que le compte machine RHEL créé
par la jonction au domaine est bien celui-là, et qu'il figure dans le keytab.
**Mesure qui tranche, depuis une cible jointe :**
```
echo "=== DEBUT PO-008 ==="
klist -k /etc/krb5.keytab; echo "rc=$?"
echo "=== FIN PO-008 ==="
```
puis, côté ADCS, vérification des ACL du gabarit.
**Qui peut la produire :** l'opérateur pour le keytab ; l'équipe PKI/AD pour les ACL.

### PO-009 — Le dépôt ne cible aucune machine — **CLOS le 2026-09-21**
`[RÉVISÉE le 2026-09-21]` — **clos, sans réserve.** Un inventaire vide est une propriété
normale d'un brouillon (F-044), pas une anomalie à instruire. Il n'y a rien à trancher :
le livrable 4 renseignera un inventaire côté seed (F-050), ou n'en renseignera pas.
*Ce qui ne disparaît pas :* la mesure F-010 reste au dossier du brouillon
(`scomm-depot-actuel.md`) et continue de servir de motif à **R-03** — une variable
commentée est une variable absente. Clore le point ouvert n'efface pas la leçon.

**Statut :** mesuré (F-010). L'inventaire `scomm_agents` est vide. Rien de ce que décrit
le `README.md` n'a pu s'exécuter depuis ce dépôt en l'état.
**Mesure qui tranche :** `ansible-inventory --graph` après renseignement de l'inventaire.
**Qui peut la produire :** l'opérateur.

### PO-010 — Le contrôle syntaxique n'est pas exécutable sur ce poste
`[RÉVISÉE le 2026-09-21]` — **requalifié : le point change d'objet.** F-050 : le nœud de
contrôle est le **seed**, pas ce poste. F-041 reste exact — `ansible.posix` manque ici —
mais cesse d'être un obstacle au projet : ce poste n'a jamais eu à exécuter le playbook.
La mesure se joue sur le seed, et revient en `[ÉCRAN-…]`.
*Conséquence que la requalification ne doit pas masquer :* le contrôle devient **médiat
et lent**. Chaque vérification passe par l'opérateur et une capture d'écran. Le livrable
4 ne doit donc **pas** être conçu en supposant une boucle de contrôle rapide — et c'est
une des raisons d'être de R-12 et R-13 dans `CLAUDE.md` : si la démonstration coûte cher,
elle doit être décidée d'avance, pas improvisée.

**Statut :** mesuré (F-041). `ansible.posix` manque et ne sera pas installée ici.
**Pourquoi ça compte :** c'est le seul garde-fou local avant de proposer du code Ansible.
Sans lui, une faute de module ne se verra qu'à l'exécution, sur une cible.
**Mesure qui tranche :** sur le nœud de contrôle professionnel, où la collection est
présumée présente :
```
echo "=== DEBUT PO-010 ==="
ansible-galaxy collection list 2>/dev/null | grep -i posix; echo "rc=$?"
ansible-playbook --syntax-check integrate_scomm.yml; echo "rc=$?"
echo "=== FIN PO-010 ==="
```
**Qui peut la produire :** l'opérateur `[ÉCRAN-…]`. Alternative, décision de l'opérateur
seul : autoriser `ansible-galaxy collection install -r requirements.yml` sur ce poste —
non fait d'initiative, car c'est une installation.

### PO-011 — Nom d'utilisateur local dans ce fichier — **CLOS le 2026-09-18** par substitution
`[RÉVISÉE le 2026-09-21]` — **reste clos, mais le contrôle qui l'établissait était faux.**
La substitution a atteint un **motif de recherche** et non un chemin, en P-18 : la
commande qui y est consignée n'a jamais été exécutée sous cette forme et ne rendrait pas
10 aujourd'hui. Corrigé dans le journal, mesuré en P-26. **La clôture tient** — le rejeu
avec le motif réel donne 0 occurrence dans les quatre fichiers, `rc=1` (P-26). Ce qui
était faux, c'était la preuve, pas la conclusion. Deux autres énoncés périmés par la même
substitution ont été relevés et marqués (en-tête du journal, P-19, P-21).

**Statut : CLOS pour ce dépôt.** L'identifiant a été remplacé par `<user>` dans les
quatre fichiers non encore publiés. Contrôle après substitution, dans les deux sens
(P-19) : le motif ne trouve plus rien ici (`rc=1`) et trouve toujours 7 lignes dans
`workstation-config/docs/machine-facts.md` (`rc=0`) — la commande n'est donc pas cassée.

`[RÉVISÉE le 2026-09-18]` **ÉTAIT :** « **Statut :** mesuré. `grep -c` → **10
occurrences** … **Arbitrage laissé à l'opérateur** … Deux options : conserver (les
chemins rendent les preuves rejouables) ou substituer `/home/<user>/` partout. »
Le chiffre de 10 était exact au moment de sa mesure (livrable 1) ; l'ajout ultérieur des
preuves P-17 et P-18 l'a porté à **13 lignes / 13 occurrences** avant substitution.
Deux mesures différentes d'un objet qui a changé entre-temps, pas une correction d'erreur.

**Coût assumé, consigné plutôt qu'effacé :** le journal de preuves **perd en
rejouabilité**. Les chemins du journal de preuves (§ 5, déplacé le 2026-09-21 vers
`scomm-journal-preuves.md`) ne sont plus copiables tels quels ; il faut
y substituer le compte réel du poste. C'est un coût accepté sciemment, pas un oubli.
Portée exacte : `/home/<user>/dev/scomm_rhel9`, `/home/<user>/.venvs/ansible-lint/`,
`/home/<user>/.ansible/collections`.

**Ce que cette clôture ne règle PAS** — voir PO-012, PO-013 et PO-014 : elle ne porte que
sur des fichiers **non encore publiés** de ce dépôt. Rien de ce qui est déjà poussé n'est
touché, et rien ne pouvait l'être.

### PO-012 — L'identité nominative réelle est publiée dans `workstation-config`
`[RÉVISÉE le 2026-09-21]` — **maintenu, et devenu bloquant pour une règle.** Tant que ce
point n'est pas arbitré, la **justification par la confidentialité** de l'extension de
R-07 aux identifiants personnels reste **provisoire** : elle applique un côté d'une
tension que ce point déclare non tranchée. Le motif de R-07 a été refondé sur la
**portabilité des chemins**, argument qui ne dépend d'aucun arbitrage, de sorte que
PO-012 puisse se trancher dans un sens ou dans l'autre sans que la règle tombe. Voir
**D-008**. Ce livrable **n'a pas tranché** PO-012 et n'a rien proposé à son sujet.

**Statut :** mesuré, **irréversible**, hors du périmètre d'écriture de ce livrable.
`git log --all --format='%an <%ae>' | sort | uniq -c` → **53 commits sur 59** sont
authentifiés par le **nom civil** de l'opérateur et son **adresse personnelle**. Les 6
autres portent le pseudonyme et l'adresse `noreply` de GitHub (P-20).
S'y ajoute, dans le contenu suivi, le champ `galaxy_info.author` renseigné au nom civil
dans **12** fichiers `roles/*/meta/main.yml`, plus `docs/ansible-chain.md:77` (P-21).
*Les valeurs elles-mêmes ne sont pas reproduites ici : ce fichier est destiné au même
dépôt public, et les recopier reviendrait à publier une fois de plus ce que l'on
recense. Elles se relisent dans `workstation-config` avec les commandes de P-20/P-21.*
Le dépôt est public et `HEAD` local == `HEAD` distant (P-22) : tout est publié.

**Tension à arbitrer par l'opérateur, pas par l'agent.** Le présent livrable pose que les
dépôts sont « publiés sous un pseudonyme délibéré ». Or `workstation-config` porte une
décision contraire, écrite et datée : **D4 (2026-08-04, amendée)**,
`workstation-config/docs/machine-facts.md:782`, qui accepte explicitement « hostname et
nom d'utilisateur de ce poste personnel, ainsi que **l'identité de l'auteur dans
l'historique git** », au motif que « les expurger dégraderait la traçabilité sans rien
protéger, puisque l'identité de l'auteur figure déjà dans chaque commit ».
Ces deux positions ne peuvent pas être vraies ensemble. **Aucune n'a été tranchée ici.**
**Qui peut la produire :** l'opérateur seul — c'est son identité et sa décision antérieure.

### PO-013 — L'adresse personnelle est publiée dans les trois dépôts
**Statut :** mesuré, **irréversible**. Au-delà du nom civil (PO-012), **l'adresse de
courriel personnelle** de l'opérateur (valeur non reproduite ici, même motif qu'en
PO-012) est l'adresse d'auteur de **la totalité** des commits de
`system_auto-update` (13/13) et de `scomm_rhel9` (2/2), et de 53/59 de
`workstation-config` (P-20). Le `user.email` configuré localement dans les trois dépôts
est pourtant l'adresse `noreply` de GitHub : la configuration a donc été corrigée
**après** ces commits, qui en gardent la trace.
**Effet sur la suite :** un **nouveau** commit ici serait authentifié
par le **pseudonyme** et l'adresse `noreply` fournie par GitHub — vérifié par
`git var GIT_AUTHOR_IDENT` (P-23). Le futur est propre ; le passé ne l'est pas.
**Qui peut la produire :** l'opérateur.

### PO-014 — Un quatrième dépôt git local n'a pas été audité
**Statut :** non audité, hors périmètre explicite de ce livrable.
`/home/<user>/dev/glass-hud` est un dépôt git (constaté en P-16, livrable 1) mais ne
figure pas dans les trois dépôts que ce livrable désigne. Il n'a **pas** été inspecté :
ni son contenu, ni son historique, ni ses identités d'auteur, ni sa visibilité.
Deux répertoires voisins, `glass-hud-reports` et `workstation-config-reports`, ne sont
pas des dépôts git (P-16) et n'ont pas été examinés non plus.
**Mesure qui tranche :** rejouer le protocole des surfaces S1 à S5 sur ce dépôt.
**Qui peut la produire :** l'opérateur, en étendant le périmètre.

### PO-015 — Aucune source lue ne documente le chemin visé : clé engendrée **sur l'hôte Linux**, jamais exportée *(ouvert le 2026-09-21)*
**Statut :** conséquence directe de F-042, qui est une lecture, pas une supposition.
S-4 décrit une clé née dans le magasin d'un hôte Windows, exportée en `.pfx`, transportée
et extraite côté Linux. `certmonger` + `cepces` fait l'inverse : la clé naît sur l'hôte
qui l'utilise (F-032, `-k`) et rien n'est jamais exporté. Les deux chemins aboutissent
au même **état de fichiers** (F-036 : `omikey.pem` en 600 `omi:omi`, certificat et lien
en 640 `root:omi`), mais par des routes différentes.
**Pourquoi ça compte :** c'est le point exact où « Microsoft documente que ça marche »
cesse de couvrir ce projet. Ce qui est documenté, c'est l'**acceptation** d'un certificat
de PKI externe par l'agent (F-035) — pas la **manière** de l'obtenir ici.
**Ce que ce point n'est PAS :** ce n'est pas une raison de douter que le chemin
certmonger fonctionne. C'est la constatation qu'aucune source **lue** ne l'établit.
**Mesure qui tranche :** produire, par certmonger, l'état de fichiers exact de F-036 sur
la VM de recette (F-049), redémarrer `omid`, puis rejouer la validation de PO-001.
L'égalité de l'état final est ce qui se mesure ; le chemin ne se mesure pas.
**Qui peut la produire :** l'opérateur. **Aucune source supplémentaire n'a été cherchée
dans ce livrable** — c'était une consigne explicite, pas un manque de curiosité.

### PO-016 — Le gabarit ADCS construit-il le sujet depuis l'annuaire, ou le prend-il dans la requête ? — **CLOS le 2026-09-23**
`[RÉVISÉE le 2026-09-23]` **Clos par F-075** `[TIERS-DÉCLARÉ]`, réponse écrite de l'équipe
ADCS : le gabarit est réglé **« Supply in the request »**. Tranché dans le sens le plus
exigeant : **la charge de produire la forme du sujet revient à la chaîne Linux** — et ce
qui la rend difficile n'est plus le gabarit mais PO-023. *Clôture par déclaration, non par
mesure :* le contrôle a posteriori ci-dessous reste utile si un sujet émis surprenait.

**Statut :** non instruit, et **non instruisable localement**.
F-036 exige un sujet à deux `CN` — FQDN puis nom court — suivi des composants `DC=`.
F-043 établit que le gabarit **de S-4** est réglé « Supply in the request » (S-4,
ligne 33). Rien n'établit le réglage du gabarit **de cet environnement**, dont seul le
nom est connu (F-045).
**Pourquoi ça compte, et c'est une bifurcation, pas un détail :**
- si le gabarit **construit le sujet depuis l'annuaire** — réglage courant pour les
  gabarits de compte machine — alors le sujet demandé par `getcert request -N` (F-032)
  est **ignoré et remplacé**. La forme de F-036 ne sera **jamais** obtenue, quoi que la
  chaîne Linux fasse. Le projet échoue sur une cause qui n'est pas dans le code ;
- s'il le **prend dans la requête**, alors la charge revient à la chaîne : il faut
  produire cette forme exacte, deux `CN` compris, et vérifier qu'elle ressort intacte.
**Ce que ce point interdit :** écrire du code qui suppose l'une des deux branches. Tant
qu'il est ouvert, toute tâche qui compose un sujet est à écrire **sans** être prescrite.
**Mesure qui tranche — c'est une question, pas une commande.** À poser à **l'équipe
ADCS**, pas à jouer sur une cible : *sur le gabarit `<nom fourni par courriel>`, l'onglet
« Subject Name » est-il réglé sur « Supply in the request » ou sur « Build from this
Active Directory information » ?* Réponse attendue sous forme de capture de l'onglet.
**Contrôle a posteriori, une fois un certificat obtenu**, qui ne remplace pas la
question mais la confirme :
```
echo "=== DEBUT PO-016 ==="
openssl x509 -noout -subject -in /etc/opt/microsoft/scx/ssl/scx.pem; echo "rc=$?"
echo "=== FIN PO-016 ==="
```
Un sujet à un seul `CN` sans composants `DC=` signe une construction par l'annuaire.
**Qui peut la produire :** l'équipe ADCS. L'opérateur pour le contrôle a posteriori,
en `[ÉCRAN-…]`.

### PO-017 — Ce qu'une restauration d'instantané ne défait pas *(ouvert le 2026-09-21)*
`[RÉVISÉE le 2026-09-24]` **Contrôle appliqué sur `host01`, sans échec** : version de clé
**3** des deux côtés, dates concordant à la minute (F-093). Le point reste ouvert pour les
essais à venir ; il a servi une fois et a tenu.
**Statut :** conséquence de F-049, à énoncer **avant** le premier essai, pas après.
L'instantané borne les effets **dans** la machine. Deux effets lui échappent :
1. **Chaque essai laisse un certificat émis dans la base de la CA.** Restaurer la VM
   n'annule pas une émission : le certificat existe, il est valide, et il reste
   attribué à ce compte machine jusqu'à révocation. **La révocation appartient à
   l'équipe PKI** — ce n'est ni une étape de la procédure, ni quelque chose que
   l'opérateur peut faire seul. Une série d'essais laisse donc une série de certificats
   vivants.
2. **Une restauration antérieure à une rotation du mot de passe du compte machine
   invalide l'identité Kerberos de la machine restaurée.** La VM revient avec un secret
   périmé, l'annuaire en a un autre : le keytab ne s'authentifie plus. Un enrôlement
   échouerait alors **pour une raison étrangère à l'objet testé**, et produirait très
   probablement un `3 CONNECTERROR` ou un `4 UNDERCONFIGURED` du helper (F-033) —
   c'est-à-dire exactement les codes qu'on attribuerait au F5 ou à `cepces.conf`.
   **C'est un faux négatif qui ressemble à un vrai.**
**Conséquence normative à appliquer dès le livrable 4 :** *toute procédure d'essai
postérieure à une restauration commence par vérifier l'identité Kerberos*, avant toute
tentative d'enrôlement :
Le contrôle lui-même n'est plus reproduit ici : il est devenu l'**étape 2** de
`procedures/diagnostic-enrolement.md`, avec sa table d'échecs indiscernables.

`[RÉVISÉE le 2026-09-21 — deux défauts corrigés]`

**ÉTAIT**, conclusion : « `rc_kinit` non nul **avant** tout essai : la machine n'a plus
son identité, il faut la rejoindre au domaine — et l'essai qui suivrait ne prouverait
rien. »

**Pourquoi c'était faux :** cette phrase tirait **la conclusion la plus étroite** d'une
observation qui en admet au moins cinq. Un `kinit -k` en échec ne prouve **pas** que la
machine a perdu son identité. Au moins quatre autres causes produisent le même code de
retour non nul :

| Cause | Ce que c'est réellement |
|---|---|
| Dérive d'horloge > `clockskew` (300 s par défaut, `man krb5.conf`) | Problème d'heure. L'identité est intacte. |
| KDC non résolu (`Cannot find KDC for realm`) | Problème de DNS ou de `krb5.conf`. |
| KDC injoignable (`Cannot contact any KDC`) | Problème de réseau. |
| Keytab sans entrée utilisable (`Key table entry not found`) | Problème de contenu de keytab. |
| **`Preauthentication failed`** | **Là seulement** : le secret ne correspond plus à l'annuaire. |

**Ce que la correction change en pratique :** le code de retour ne discrimine pas ; c'est
le **texte d'erreur** qui porte l'information. Conclure « identité perdue » et rejoindre
la machine au domaine sur la foi d'un `rc` non nul, c'est réparer ce qui n'est pas cassé
et détruire l'état qu'on mesurait.

**ÉTAIT**, second défaut, dans le bloc de commandes supprimé : `kinit -k` y était appelé
**sans cache dédié**, donc écrivait dans le cache du compte courant et écrasait le ticket
de l'opérateur. Une fiche qui prescrit un contrôle « avant tout essai » ne doit pas
modifier l'état de la session qui le joue. Corrigé à l'étape 2 de la procédure par
`KRB5CCNAME="FILE:$CC"` et une destruction explicite du cache.
**Ce que ce contrôle ne couvre pas :** il ne dit rien des certificats déjà émis (point 1),
qui ne se voient que depuis la console de la CA.
**Qui peut la produire :** l'opérateur pour le contrôle Kerberos ; l'équipe PKI pour
l'état et la révocation des certificats émis.

### PO-018 — Sous quel mode la protection étendue est-elle configurée sur CEP/CES, et le SPN du VIP y figure-t-il ? *(ouvert le 2026-09-21 — **sourcé et REQUALIFIÉ le 2026-09-21**)*

`[RÉVISÉE le 2026-09-29]` — **requalifié, pas clos.** Les trois questions ci-dessous
(« `tokenChecking` vaut-il … ? », « `flags` porte-t-il `Proxy` ? », « la collection
`<spn>` … ? ») ont reçu leur réponse : **`Require` jusqu'au 2026-09-25, `Allow` depuis**
(F-112, F-111), **ni `flags` ni `<spn>` affichés** (F-111). Le cas décrit plus bas comme
« une erreur de configuration, pas le cas général » — `Require` sans `Proxy` — **était
bien celui-ci**. La phrase « Ce qui manque : la confirmation de l'équipe ADCS, attendue le
2026-09-28 » est satisfaite.
**Depuis `Allow`, le ticket de la machine est accepté** (F-115 à F-117) : le blocage
d'authentification est levé. **Pourquoi pas clos :** rien ne dit que `Allow` soit le
réglage **durable** — aucune déclaration de l'équipe ADCS n'en fait état ; le passage à
`Allow` est soupçonné d'avoir fait naître un désaccord avec WCF (F-125, hypothèse,
PO-046) ; et l'autre voie que S-6 décrit pour un intermédiaire — `flags` `Proxy` et
collection `<spn>` (F-062) — n'a pas été examinée. Le choix appartient à l'équipe ADCS.

`[RÉVISÉE le 2026-09-21]` **ÉTAIT :** « La protection étendue de l'authentification
est-elle active sur CEP/CES, et que devient-elle derrière un F5 ? — **Statut : affirmé
par l'énoncé du livrable 4, non sourcé, non mesuré.** … si la protection étendue est
active côté IIS et que le F5 **termine** le TLS, le jeton de liaison présenté par le
client ne correspond plus au canal vu par le serveur ; l'authentification échoue **alors
que tout le reste est correct**. »

**Ce que S-6 établit, et en quoi cela contredit l'énoncé du livrable 4 (F-062).**
La formulation antérieure était **partiellement fausse et matériellement incomplète** :

1. **Elle nommait le mauvais mécanisme pour le cas décrit.** Dans le scénario « SSL
   off-loading » — le nôtre si le F5 termine le TLS — S-6 énonce que **le contrôle de
   liaison de canal n'est pas employé** et que c'est le **SPN** qui est vérifié. La panne
   n'est donc pas « le jeton de liaison ne correspond plus » ; c'est « le SPN présenté
   n'est pas dans la collection du serveur ».
2. **Elle supposait la protection active.** S-6 donne `tokenChecking = None` et
   `flags = None` **par défaut**, `None` signifiant explicitement le comportement
   antérieur à la protection étendue. Le cas par défaut est donc *désactivé*.
3. **Ce qui subsiste de l'affirmation antérieure**, et c'est un cas réel : si
   l'administrateur a réglé `tokenChecking = Require` **sans** poser le drapeau `Proxy`,
   alors la liaison de canal est bien exigée, le F5 la rompt, et l'authentification
   échoue comme décrit. **C'est une erreur de configuration, pas le cas général** — et
   c'est cette distinction que l'énoncé de mémoire avait perdue.

**Ce qui reste ouvert, et qui est plus précis qu'avant :**
- `tokenChecking` vaut-il `None`, `Allow` ou `Require` sur les sites CEP et CES ?
- `flags` porte-t-il `Proxy` ?
- si oui, **la collection `<spn>` d'IIS contient-elle `HTTP/<VIP_FQDN>`** ? Question
  **distincte** de l'enregistrement d'annuaire de F-076 (S-6 ligne 23).

**Couplage avec PO-005, inchangé :** la question n'a de portée que si le F5 termine le
TLS. Si PO-005 conclut au passthrough, le scénario applicable devient « SSL de bout en
bout à travers un proxy », qui selon S-6 emploie **lui aussi** le contrôle de SPN.
Autrement dit, **dans les deux branches de PO-005 c'est le SPN qui compte**, ce que le
livrable 4 ne disait pas.

**Mesure qui tranche — ce n'est pas une commande.** Question à l'équipe ADCS/IIS :
*sur les sites CEP et CES, quelles sont les valeurs de `tokenChecking` et de `flags` de
l'élément `<extendedProtection>`, et quelles entrées contient la collection `<spn>` ?*
Réponse attendue sous forme de capture de la configuration. Ce que la procédure peut
faire de son côté reste l'élimination décrite à son étape 5 — qui **désigne**, mais ne
**prouve** pas.

`[RÉVISÉE le 2026-09-24, second amendement]` — **ce n'est plus une cause hors chemin
critique : c'est LA cause désignée**, en attente de confirmation côté serveur.
L'authentification échoue en HTTPS et **réussit en HTTP avec le même ticket** (F-109) :
SPNEGO `00`, Kerberos, authentification mutuelle, puis `403` parce que le site exige
HTTPS — **la seule différence est le canal TLS**. S'y ajoutent `0xC000035B` au journal IIS
(F-106) et le précédent concordant de S-7 (F-108).
**Raisonné, non mesuré :** derrière un F5 qui termine TLS, **aucun client ne peut
satisfaire une liaison de canal exigée**, le canal qu'il voit n'étant pas celui du serveur.
**Ce qui manque :** la confirmation de l'équipe ADCS, attendue le 2026-09-28 (F-110).

`[RÉVISÉE le 2026-09-24]` — **sortait du chemin critique sur la seule foi de F-077, et c'est
mince** : une déclaration au conditionnel (« I think »), non mesurée, donnant les réglages
aux valeurs par défaut. **Pas clos**, et pour une raison qui n'est pas celle inscrite la
veille. **ÉTAIT** (`[RÉVISÉE le 2026-09-23]`) : « … F-076 … **soit la condition que S-6
pose pour l'accès par proxy** (F-062) ; F-077 … ». **F-076 ne portait pas ce poids** : il
mesure l'annuaire, non IIS (voir F-076 révisé) ; il sort **l'étape 3** de l'inconnue, pas
PO-018.
**Ce que S-6 ne permet pas de conclure**, et qu'il ne faut donc pas écrire : que la
collection `<spn>` soit ignorée quand `tokenChecking` vaut `None`. Les deux attributs sont
décrits séparément (lignes 27-42) et `flags = None` porte « SPN checking is enabled »
(ligne 38) ; seule « emulates the behavior that existed before extended protection »
(ligne 31) suggère une désactivation d'ensemble. **La source ne tranche pas.**

**Qui peut la produire :** l'équipe ADCS/IIS. Aucune commande jouée sur la machine de
recette ne l'établit.

### PO-019 — R-12 range `when:` parmi les gardes démontrables ; un `when:` n'a pas de mode d'échec — **CLOS le 2026-09-23**
`[RÉVISÉE le 2026-09-24]` **Clos par D-011** (décision du 2026-09-23) : voie **(a)**
retenue, `when:` retiré de R-12 et renvoyé à R-04. La fiche ci-dessous est conservée
telle quelle, comme énoncé du problème résolu.

**Statut :** **défaut relevé dans une règle de ce dépôt**, pas dans le code. Signalé
ici ; **la règle n'est pas réécrite dans ce livrable.**
**Ce que R-12 énonçait alors :** « Toute garde — `assert`, `when:`, `failed_when:`, `stat`
préalable, contrôle de préconditions — est démontrée par **deux exécutions relevées** :
le cas nominal, et l'échec forcé, où l'on relève le code de retour **et** le message
effectivement affiché. »
**Pourquoi `when:` n'y a pas sa place.** Les autres gardes de la liste s'**arrêtent** :
un `assert` faux échoue la tâche et affiche son message, un `failed_when:` vrai marque la
tâche en échec. **Un `when:` faux ne casse pas : il saute, en silence, et le playbook
continue comme si de rien n'était.** Il n'a donc pas de mode d'échec, donc pas de message
à relever, donc **pas d'échec forcé possible**. Exiger de le démontrer dans les deux sens
est une exigence qu'on ne peut pas satisfaire — et une règle inapplicable finit par être
ignorée **avec les autres**, ce que l'avertissement de `CLAUDE.md` § *Ce que ces règles
n'incluent PAS* énonce déjà.
**Ce n'est pas une objection théorique.** Ce dépôt en porte la démonstration :
`register.yml:21-23` et `:33` sautent leurs deux tâches aux valeurs par défaut, sans un
mot, et personne ne l'a vu avant qu'on lise la condition (F-018,
`scomm-depot-actuel.md`). Un `when:` faux ne produit aucune sortie à capturer. Il ne
s'est pas trompé : il n'a rien fait.
**La distinction qui manque à R-12 :** un `when:` est un **aiguillage**, pas une garde.
Ce qui **protège** doit s'arrêter bruyamment ; ce qui **oriente** peut sauter. Les deux
demandent des preuves différentes — R-04 (énumérer les branches exercées et non
exercées) convient à l'aiguillage ; R-12 convient à ce qui s'arrête.
**Ce qui tranche :** une décision de l'opérateur sur la forme à donner à R-12. Trois
voies possibles, non arbitrées ici :
(a) retirer `when:` de la liste de R-12 et le renvoyer à R-04 ;
(b) exiger qu'une précondition **critique** soit portée par un `assert`, jamais par un
    `when:` seul — auquel cas R-12 s'applique sans changement ;
(c) exiger qu'un `when:` protecteur soit **doublé** d'une tâche qui échoue quand la
    condition attendue n'est pas réunie.
**Qui peut la produire :** l'opérateur. C'est une règle, et une règle ne se modifie pas
au fil d'un livrable qui l'applique.


### PO-020 — Sous `ansible_admin`, compte local sans identité dans l'annuaire, comment une tâche prendra-t-elle l'identité de la machine ? *(ouvert le 2026-09-21)*
`[RÉVISÉE le 2026-09-29]` — **en partie instruit.** La condition « **mesurable seulement
une fois `cepces` configuré sur la recette** » est remplie sur `host02` (F-122). F-123
établit que `cepces-submit`, **lancé à la main**, obtient Kerberos sous `host02$` avec les
`principals` par défaut (F-120). **Ce qui n'est toujours pas mesuré :** l'identité et le
contexte du **démon `certmonger`** quand c'est lui qui appelle le helper — la mesure
`systemctl show` / `ps` ci-dessous n'a pas été produite.
`[RÉVISÉE le 2026-09-23]` — **requalifié : il n'y a pas de problème d'identité côté
Ansible.** F-074 infirme la prémisse (F-059) : l'opérateur joue sous son compte AD, avec
ticket. Et la tâche d'enrôlement n'a de toute façon **aucune identité Kerberos à prendre** :
elle s'exécute avec élévation et s'adresse au **démon `certmonger` local**, qui
s'authentifie ensuite seul depuis le keytab de la machine.
**Ce qui reste, et c'est tout ce qui reste :** sous quelle identité ce démon s'authentifie
**réellement** — `cepces.conf` ne pose ni `--keytab` ni `--principals` (F-071) et s'en
remet à ses défauts. **Mesurable seulement une fois `cepces` configuré sur la recette**,
donc pas par la procédure de diagnostic actuelle, qui n'installe rien.
**Cause d'échec à connaître d'avance — un piège, pas un point ouvert.** Le ticket de
l'opérateur sur le seed est obtenu **au login** et vaut **huit heures** (F-058). Passé ce
délai un playbook **échoue à la lecture du vault** (`hpam-read.yml`, `use_gssapi`) avec un
message HTTP ou d'API **qui ne parle pas de Kerberos** — « secret introuvable », « authentification refusée » — alors que rien n'a changé côté vault. `klist -s` d'abord.

`[RÉVISÉE le 2026-09-22]` — **largement instruit par l'inventaire, et une prémisse en
ressort contredite.**

**Ce que le dépôt montre :** la question était mal posée. Dans `PKI_enrolment.yml`, **la
tâche ne prend aucune identité Kerberos.** Elle s'exécute en `root` sur la cible
(`ansible.cfg:4-6`, `main.yml:4`) et appelle `getcert request` (`:96-108`), c'est-à-dire
qu'elle **parle au démon `certmonger` local**, lequel s'authentifie ensuite seul, via
`cepces` et le keytab de la machine. C'est la piste (b) ouverte au livrable 5 — mais sans
`--keytab` ni `--principals` : le dépôt s'en remet aux défauts de `cepces` (F-071).

**Prémisse contredite (R-02).** **F-059** déclare que « les playbooks sont joués depuis le
seed sous `ansible_admin`, compte local PAM, **donc sans identité dans l'annuaire** ».
Le dépôt dit autre chose, à deux endroits :
- `preflight.yml:24` et `:29` nomment `ansible_admin` comme le compte **de la CIBLE**
  (« mapped to the staff_u SELinux login on this **TARGET's** image ») ;
- `preflight.yml:135-160` **exige un ticket Kerberos sur le NŒUD DE CONTRÔLE**
  (`klist -s`, `delegate_to: localhost`, « Run: `kinit` »), et `hpam_operator` y vaut
  `id -un` **du nœud de contrôle** (`group_vars/all/00-defaults.yml:18-22`).

Si le playbook était réellement lancé sous `ansible_admin` **sur le seed**, `klist -s`
échouerait et **toutes les lectures HPAM échoueraient avec lui** — dont celle qui fournit
l'identifiant de jonction au domaine (`AD_join.yml:5-17`). Or la jonction fonctionne
(F-046). **Les deux ne peuvent pas être vraies ensemble.**
La lecture la plus économique — **à confirmer, pas à retenir** — est que `ansible_admin`
est le compte **distant sur la cible**, et que `ansible-playbook` est lancé sur le seed
sous le compte AD de l'opérateur, qui porte le ticket SSSD de F-058.

**Mesure qui tranche, et elle est courte :** sur le seed, pendant qu'un playbook tourne
ou juste avant : `id -un`, `klist -s; echo $?`, et `grep -rn 'ansible_user' <inventaire>`.
Le premier dit qui lance, le deuxième s'il a un ticket, le troisième sous quel compte on
se connecte à la cible. **Qui peut la produire :** l'opérateur, en `[ÉCRAN-…]`. F-059 sera
révisé ou confirmé par cette mesure, pas avant.

**Ce que l'inventaire n'apporte pas :** que ce mécanisme **fonctionne**. `PKI_enrolment.yml`
est sauté par défaut (`main.yml:41`, `adcs_server` commenté) — c'est du code écrit, pas
du code exercé (R-04).

**Statut :** conséquence directe de F-059, non instruite. **Elle conditionne la conception
du rôle**, pas seulement son exécution : ce n'est pas une question à régler au moment des
essais.
**Le problème en une phrase :** `cepces` s'authentifie par keytab et principal (F-027,
F-028) ; le compte qui joue le playbook n'a aucun ticket et ne peut pas en obtenir
(F-059) ; or c'est l'identité **de la machine** qui doit enrôler, pas celle du compte qui
lance la tâche.
**Ce qui n'est pas la réponse :** faire faire un `kinit` par le playbook sous
`ansible_admin`. Ce compte n'existe pas dans l'annuaire — il n'y a pas de secret à
présenter. La question n'est pas « comment l'authentifier » mais « comment **ne pas**
l'authentifier et passer quand même par le compte machine ».
**Pistes à instruire, non tranchées et non hiérarchisées ici :**
(a) `become: true` et exécution sous `root` sur la cible, `certmonger` lisant lui-même
    `/etc/krb5.keytab` — dépend de F-061 ;
(b) `getcert add-ca` recevant `--keytab` et `--principals` sur la ligne du helper (F-027),
    l'authentification étant alors faite par le service `certmonger`, jamais par la tâche ;
(c) une identité de service dédiée, à créer dans l'annuaire — ce qui déborde du périmètre
    technique et devient une demande à l'équipe AD.
**Mesure qui tranche :** sur la machine de recette, sous quelle identité le processus
`certmonger` s'exécute et quel keytab il peut lire —
`systemctl show -p User -p Group certmonger` et `ps -o user= -C certmonger`, complétés par
le résultat de F-061. À jouer **après** installation de `certmonger`, donc hors de la
procédure de diagnostic actuelle, qui n'installe rien.
**Qui peut la produire :** l'opérateur pour les deux commandes ; l'équipe AD pour (c).


### PO-021 — L'entrée de bouclage masque une résolution DNS qui fonctionne : est-ce voulu, et que faire des machines déjà déployées ? *(ouvert le 2026-09-22)*
`[RÉVISÉE le 2026-09-29]` — **corrigé À LA MAIN sur `host02` seulement** (F-122, geste 1 ;
D-014). **Rien n'est porté dans `rhel_post_install`** : la question du parc — motif,
machines déjà déployées, correction — reste **entière**, et la phrase « Aucune correction
n'est écrite ici » reste vraie **du dossier**. Non mesuré après la correction : le rôle de
la source `myhostname` (F-091), que `nsswitch` sur `host02` porte (F-122, geste 1 ;
relevé non daté par rapport à la correction).
`[RÉVISÉE le 2026-09-24]` **Élargi : deux mécanismes, pas un** (F-091). Corriger le seul
fichier local pourrait **démasquer** la source `myhostname` en IPv6 — hypothèse non
mesurée. **Non tranché ici.**
**Statut :** origine **établie**, motif **absent**, correction **non arbitrée**.
**Ce qui est établi (F-064 à F-068) :** l'ordre mesuré — bouclage, nom court, puis FQDN —
est celui qu'écrivait `shell/ad_pki.sh:91`, **supprimé le 2026-07-08**. Le chemin Ansible
actuel écrit l'ordre **inverse** (FQDN puis court) depuis le 2026-07-13, et écrivait le
nom court seul avant. Aucun autre écrivain n'existe dans le dépôt.
**Ce que cela suggère, et ce n'est pas établi :** les deux machines mesurées auraient été
provisionnées par le script, avant son retrait — et `tasks/set_hostname.yml` n'aurait pas
rejoué dessus depuis, sans quoi `lineinfile` aurait remplacé la ligne (P-37 : « Only the
last line found will be replaced »). **Ce raisonnement suppose que la tâche a la même
sémantique en `ansible-core` 2.14 ; c'est à confirmer.**
**Trois lectures restent ouvertes**, et ce livrable n'en retient aucune : (a) les machines
datent d'avant le 2026-07-08 ; (b) le playbook n'a pas rejoué `set_hostname` dessus ;
(c) une troisième source — modèle de machine virtuelle, installateur, service
d'installation — écrit cette ligne, et le dépôt n'y est pour rien.
**Mesure qui tranche, et elle est simple :** sur les **mêmes** deux machines, relever la
date de création du fichier des hôtes et la date de la dernière exécution du playbook
(`stat /etc/hosts` ; journal d'exécution côté seed), puis comparer au 2026-07-08 et au
2026-07-13. Si une machine postérieure au 2026-07-13 porte encore l'ordre « court puis
qualifié », la lecture (c) devient la seule tenable. **Qui peut la produire :**
l'opérateur, en `[ÉCRAN-…]`.
**Aucune correction n'est écrite ici, et c'est délibéré.** Le geste peut être volontaire —
faire résoudre à un hôte son propre nom sans dépendre du DNS au démarrage est une
pratique défendable. **Mais le dépôt n'en donne aucun motif** (F-068) : corps de commit
vide, aucun commentaire, `README.md:12` purement descriptif. On ne corrige pas une
intention qu'on n'a pas lue.
**Ce qui rend le point non trivial**, à verser au dossier sans le trancher : le `regexp`
`'^\s*127\.0\.0\.1'` ne vise pas une ligne dédiée mais **la dernière ligne qui
commence par l'adresse de bouclage** — laquelle, sur un fichier RHEL 9 d'origine, est la
ligne `localhost`. Et F-064 relève que l'ordre des sources place les fichiers avant le
DNS, et que la résolution DNS concorde déjà dans les deux sens. Conséquences pour ce
projet : le nom rendu par la résolution inverse du bouclage dépend de l'ordre des deux
noms, et c'est ce nom que la canonicalisation Kerberos peut employer (F-052 :
`dns_canonicalize_hostname` et `rdns` valent `true` par défaut). **Lien à instruire avec
PO-016** (construction du sujet) et avec l'étape 1 de la procédure de diagnostic.

### PO-022 — `validate_certs: false` sur tous les appels au vault *(ouvert le 2026-09-22, une ligne)*
`tasks/hpam-read.yml:66,99,121,152,187` désactivent la validation TLS face à HPAM ; à
signaler à l'équipe qui maintient ce dépôt, hors périmètre de ce projet.


### PO-023 — La forme du sujet est une variable à découvrir par essai, pas une contrainte connue *(ouvert le 2026-09-23)*
**Statut :** conséquence de F-079 et F-075. Le gabarit prend le sujet dans la requête, donc
la chaîne doit le produire — mais **personne ne sait quelle forme l'agent acceptera**, et
F-036 est une documentation générique, pas l'exigence de cette installation.
**Trois conséquences, à tenir pour acquises dès la conception :** (1) le rôle devra
**paramétrer** la forme du sujet, pas la composer en dur, sans quoi chaque essai demandera
de modifier du code ; (2) **la série d'essais sera plus longue que prévu** — on cherche une
valeur, on ne vérifie pas une valeur connue ; (3) **chaque essai laisse un certificat émis** que l'instantané ne défait pas et dont la révocation appartient à l'équipe PKI (PO-017, point 1) : le coût de la recherche est porté par une équipe tierce.
**À consigner impérativement :** le sujet exact d'un certificat **accepté**, au caractère
près — ordre des `CN`, composants `DC=`, espaces. **Ce sera la seule référence** ; aucune
documentation ne la donnera. **Qui peut la produire :** l'opérateur et l'administrateur
SCOM, par essais successifs.

### PO-024 — Automatiser l'ajout du compte machine à `<GROUPE-T>` *(ouvert le 2026-09-23)*
`<COMPTE-JONCTION>` en a le droit (F-082), l'équipe ADCS le suggère ; à instruire après
D-010, dans `rhel_post_install` (R-09), pas ici.

### PO-025 — Sous quel compte Ansible se connecte-t-il à la cible ? *(ouvert le 2026-09-23)*
Non mesuré : F-074 établit l'identité de la **session** sur `seed01`, pas de la connexion.
Mesure : `grep -rn 'ansible_user' <inventaire>` sur le seed.

### PO-026 — `openldap-clients` installé à la main sur le seed *(ouvert le 2026-09-23)*
Installé depuis le dépôt de base RHEL 9 pour F-081, **décrit dans aucun code** (P-42) ; à
reprendre par la chaîne qui configure le seed, ou à retirer.


### PO-028 — Distribuer `<ROOT-CA>` comme ancre de confiance aux machines RHEL *(ouvert le 2026-09-24)*
`[RÉVISÉE le 2026-09-29]` — **posée À LA MAIN sur `host02` seulement** (F-122, geste 2 ;
D-014), prise sur le VIP après contrôle de l'empreinte. **La distribution au parc reste
entière**, ainsi que la source non publiée au déploiement.
**Statut :** blocage **mesuré** (F-098) qui **n'appartient pas à ce projet**. Aucun
mécanisme ne distribue les autorités de l'entreprise aux machines RHEL (F-099) : c'est une
**propriété de tout le parc**, donc de `rhel_post_install` (R-09 : lecture seule ici).
**Contrainte qui commande la conception :** le certificat porte le nom de l'entreprise —
**il ne peut pas être versionné dans un dépôt public** (R-07) ; il doit venir d'une
**source non publiée au déploiement** (vault F-070, annuaire, artefact interne).
**Acquis :** la racine est identifiée sans ambiguïté et **une seule ancre couvre le VIP et
l'autorité d'enrôlement** (F-100, D-013). **Qui :** l'opérateur pour la conception,
l'équipe de `rhel_post_install` pour la mise en œuvre.

### PO-029 à PO-040 — Douze défauts de la procédure, relevés en la jouant *(ouverts le 2026-09-24)*
**Constat seul : aucun n'est corrigé ici** — le périmètre exclut la procédure.

| Réf | Défaut relevé en jouant |
|---|---|
| PO-029 | L'étape 1 attend de `getent` l'adresse de l'interface ; il ne passe pas que par le DNS (F-090). |
| PO-030 | L'étape 1 ignore la source de résolution `myhostname` (F-091). |
| PO-031 | L'étape 2b désigne un **principal de service** comme identité client ; son repli ne peut pas se déclencher (F-094). |
| PO-032 | L'étape 2a se contredit sur l'élévation : le texte relève les deux codes, la commande n'élève qu'en cas d'échec. |
| PO-033 | Le nettoyage vérifie la survie du cache par défaut **sans avoir relevé son état préalable**. |
| PO-034 | L'étape 0b.2 couvre plusieurs options par **un seul code de retour**, et ne cherche pas l'option de cache de `kinit` qu'elle déclare (F-089). |
| PO-035 | La requête anonyme de l'étape 5 doit recevoir la racine, sinon elle échoue **avant** HTTP (F-098). |
| PO-036 | L'étape 4 prétend répondre à PO-005 par l'émetteur ; un F5 qui termine TLS présente souvent un certificat d'entreprise. |
| PO-037 | Dans IIS, un refus d'**autorisation** produit aussi un `401` ; seul le **sous-code** distingue (F-106). |
| PO-038 | Aucune étape ne prévoit l'**absence de la racine** du magasin de confiance. |
| PO-039 | Le chemin du cache dépend du processus du shell : **toute la procédure doit se jouer dans une seule session**. |
| PO-040 | `[RÉVISÉE le 2026-09-29 — INFIRMÉE dans un périmètre délimité]` **Sur `host02`, sous la forme `curl -v --stderr <fichier>`, les préfixes `>` sont présents** : le compteur `grep -c '^> Authorization: Negotiate'` rend **0 sur chaque requête sans ticket et 1 sur chaque requête avec** — 2026-09-25 à 15:33 (1 sans, 2 avec), 2026-09-28 à 13:39 (1 sans, 4 avec), 2026-09-28 à 14:29 (2 avec) `[ÉCRAN, relayé]`, P-49, P-50, P-53. Il discrimine donc dans les deux sens. **Limite de la révision :** PO-040 **ne consignait pas sa forme d'appel** ; il portait sur `host01` le 2026-09-24, dont le `curl` est `7.76.1-40.el9_8.5` (F-087). La version de `curl` sur `host02` **n'est pas au dossier**. Pour une autre forme d'appel, ou sur `host01`, PO-040 **n'est pas révisé**. **ÉTAIT :** « Sur cette version de `curl`, la sortie détaillée n'a pas les préfixes `>` et `<`. » |


### PO-041 — Contourner le défaut du `%post` de `cepces-certmonger` dans `rhel_post_install` *(ouvert le 2026-09-29)*
Le paquet n'enregistre pas l'autorité, et en silence (F-121). **À porter dans
`rhel_post_install`** (R-09 : rien n'y est écrit d'ici) : un `getcert add-ca` **explicite**,
**sans `--install`**, suivi d'une **garde** sur `getcert list-cas` — ce que
`PKI_enrolment.yml:78-88` fait déjà en partie (F-071, note du 2026-09-29). La garde devra
être démontrée dans les deux sens (R-12). Le moment relève de D-001 et de la logique de
D-010.

### PO-042 — Corriger l'autorité `cepces` enregistrée sur `host02` *(ouvert le 2026-09-29)*
L'autorité a été enregistrée **avec `--install`**, par erreur (F-122, geste 5) : les
erreurs d'authentification sont masquées. **À corriger avant toute demande de
certificat** — retrait de `--install` de la ligne du helper, puis relevé de
`getcert list-cas -c cepces` montrant le helper corrigé.

### PO-043 — Quelle adresse de CES la politique CEP annonce-t-elle ? *(ouvert le 2026-09-29)*
`[PILOTE-DÉCLARÉ]` Avec `type=Policy` (F-120), `cepces` obtient l'adresse de CES **de la
réponse de CEP**. Le F5 réécrit les chemins (F-118) : la politique annoncera-t-elle
`<CHEMIN-CES-PUBLIÉ>` ou le chemin interne, et ce dernier est-il joignable par le VIP ?
**À observer au premier `GetPolicies` réussi** — impossible tant que PO-046 bloque.
Repli possible, selon le pilote : `type=Enrollment`. **Aucune source lue ici ne décrit
ces deux modes** ; S-1 et S-2 sont à relire sur ce point avant d'en dépendre.

### PO-044 — La délégation Kerberos (`delegate=True`) est-elle nécessaire ? *(ouvert le 2026-09-29)*
Le paquet l'active, avec ce motif en commentaire : nécessaire si CES et l'autorité ne
sont pas sur la même machine (F-120). **Non établi** pour cet environnement : où tourne
l'autorité par rapport à CES, et le compte `<COMPTE-SVC-CES>` est-il autorisé à déléguer ?
Question à l'équipe ADCS.

### PO-045 — `SEC_E_NO_CREDENTIALS` sur CEP le 2026-09-25 à 15:33 : non expliqué *(ouvert le 2026-09-29)*
`401 1`, `0x8009030E` (F-116, S-8) ; **non reproduit** le 2026-09-28 (F-117). Le **même
statut** figurait déjà au journal le 2026-09-24 (F-106), sous l'ancien réglage. **À
surveiller** : s'il revient, relever l'heure exacte, pour que l'administrateur la retrouve
au journal IIS.

### PO-046 — Erreur d'activation WCF sur CEP et CES *(ouvert le 2026-09-29)*
`500 System.ServiceModel.ServiceActivationException`, corps vide (F-124), et
vraisemblablement les `500` sur `GET` (F-126). Hypothèse : désaccord de protection
étendue entre WCF et IIS (F-125). **Blocage côté serveur, hors des droits de l'équipe**
(F-127). En attente de l'administrateur, **retour annoncé le 2026-10-01**. Mesure qui
tranche : le journal **Application** du serveur, source `System.ServiceModel`.

### PO-047 — Passages de `procedures/diagnostic-enrolement.md` que les faits du 2026-09-25 au 2026-09-28 rendent faux ou incomplets *(ouvert le 2026-09-29)*
**Constat seul : la procédure n'est pas modifiée par ce livrable.** Numéros de ligne
relevés sur le fichier au commit `be1a70b`.

| Ligne(s) | Passage | Ce qui le rend faux ou incomplet |
|---|---|---|
| `:43-46` | « n'installe rien … ne modifie aucun magasin » | Vrai de la procédure, mais `host02` ne représente plus le parc (F-122) : E1, E4 et E5 y rendent désormais d'autres résultats qu'une machine du parc. |
| `:79-81`, `:89-94` | `<CES_URL>` « non employée » | CES est testé (F-115 à F-117) ; la table ne distingue pas chemin publié et chemin interne (F-118). |
| `:140` | Matrice, étape 5 : « ne dit rien … ni de CES » | CES testé hors de la procédure. |
| `:151-152` | Ticket de session ≠ contexte de `certmonger` | Toujours vrai ; F-123 a lancé le helper à la main, pas `certmonger` (PO-020). |
| `:328-335` | « ⚠ Sur cette machine, `hostname -f` rend le NOM COURT. C'est normal et connu » | **Faux sur `host02`** depuis F-122, geste 1. |
| `:345-349` | Paragraphe D-012 | La correction a été faite à la main sur `host02` (D-014). |
| `:411` | L'entrée locale « fausse » le nom (PO-021) | Plus vrai sur `host02`. |
| `:546`, `:605`, `:820` | L'étape 4 instruirait PO-005 par l'émetteur | F-118 l'instruit autrement (réécriture des chemins) ; PO-036 déjà ouvert. |
| `:611-614` | `@VERIF` : `cepces` emploie-t-il le magasin système ou le bundle `cas` ? | En partie instruit par la configuration livrée (F-120 : `cas` non défini). |
| `:638-639` | « un `GET` sans corps SOAP » | Vrai, mais un `GET` ne teste pas le service : le `POST` aboutit au même `500` (F-124). |
| `:676-680` | Table du témoin : « code différent (200, 405, 403…) ⇒ authentification » | Il manque `500` ; et « authentifié » ≠ « fonctionnel » (F-126). |
| `:682-684` | Hypothèse du `405` | Jamais observée ; c'est un `500` qui l'a été. |
| `:691-697` | Table des échecs | Pas de ligne pour `500` ; ligne `401`, cause (a) : « ce qui l'écarte est F-077 seul » — **F-077 est infirmée** ; `401 1` avec `0x8009030E` vu de façon intermittente (F-116). |
| `:699-707` | « `tokenChecking` et `flags` … fournis au conditionnel (F-077) ; reste à demander `<spn>` » | Répondu par F-111 et F-112. |
| `:648`, `:650` | Requêtes sans `--cacert` (PO-035) | Sans effet sur `host02` depuis F-122, geste 2 ; toujours vrai pour le parc. |
| `:77-87` | Table des valeurs à substituer | N'a pas les libellés `<DOMAINE>`, `<SERVEUR-ADCS>`, `<CHEMIN-CEP-PUBLIÉ>`, `<CHEMIN-CES-PUBLIÉ>`, `<CA-N3>`, `<CA-N2>`, `<ROOT-CA>`, `<COMPTE-SVC-CES>` (P-48). |
| `:813` | § 11, PO-006 : « `dnf info` ne fait pas partie » | PO-006 est clos (F-119). |
| `:815` | § 11, PO-020 : « `certmonger` n'est pas installé ici » | **Faux sur `host02`** depuis F-122, geste 3. |
| `:823` | § 11, PO-018 : « par élimination seulement — la configuration ne se lit que côté IIS » | Lue par l'administrateur (F-111). |

---

## 4. Décisions

Chaque décision est datée, numérotée, révisable. Une révision se marque, elle n'efface
pas le texte antérieur.

### D-001 — Destination finale du code : **NON TRANCHÉE** (2026-09-18)
Deux options restent ouvertes : dépôt `scomm_rhel9` autonome, ou intégration dans
`rhel_post_install`. **À décider seulement après que la chaîne ait produit un certificat
au moins une fois.**
*Motif :* `rhel_post_install` provisionne le parc ; y poser un mécanisme jamais exercé
met un non-prouvé sur le chemin de tous les hôtes.
*Ce qui la débloquera :* PO-001, puis PO-002.

### D-002 — Les documents d'autorité vivent dans `scomm_rhel9`, à titre temporaire (2026-09-18)
`CLAUDE.md`, ce fichier, `docs/scomm-depot-actuel.md` et `docs/scomm-journal-preuves.md`
résident ici. Ils **migreront** si D-001 conclut à l'intégration.
`[RÉVISÉE le 2026-09-21]` **ÉTAIT :** « `CLAUDE.md` et ce fichier résident ici. » La
séparation D-006 a porté les documents de deux à quatre ; ils migreraient ensemble.
*Motif :* il faut un lieu unique et versionné dès maintenant ; le choisir définitivement
supposerait D-001 tranchée, ce qu'elle n'est pas.

### D-003 — Étiquetage des mesures rapportées par écran (2026-09-18)
Toute mesure provenant d'une machine professionnelle porte `[ÉCRAN-AAAA-MM-JJ]`.
*Motif :* ces mesures sont authentiques mais **non rejouables par l'agent**, **périmables**,
et **exposées à une erreur de transcription**. L'étiquette porte la date pour que leur
péremption soit visible, et les distingue des sorties que l'agent a lui-même produites.

### D-004 — `rhel_post_install` n'a pas été cloné (2026-09-18) — **AMENDÉE le 2026-09-22**
`[RÉVISÉE le 2026-09-22]` **Le dépôt a été cloné**, en lecture seule, à
`/home/<user>/dev/rhel_post_install`, **sur autorisation explicite de l'opérateur**
(livrable 6). F-063, P-35. **R-09 est inchangée et s'applique intégralement** : aucune
écriture, aucun commit, aucune branche, aucun fichier créé — non-modification prouvée
en P-35.
*Ce qui change et ce qui ne change pas :* le dépôt est désormais **lisible**, il n'est
pas devenu **modifiable**. Et le clone persiste sur ce poste personnel : c'est un ajout
durable, signalé plutôt que tu.

**ÉTAIT :** « Le dépôt est absent du poste (PO-002). Il n'a pas été cloné.
*Motif :* règle explicite du livrable ; cloner un dépôt professionnel sur un poste
personnel est une décision de l'opérateur, pas de l'agent. »
*Le motif n'est pas infirmé : c'est bien l'opérateur qui a décidé, et le clone n'a eu
lieu qu'après.*

### D-005 — L'identifiant personnel est expurgé du non-publié, le publié n'est pas réécrit (2026-09-18)
Dans les quatre fichiers non encore publiés de ce dépôt, l'identifiant du compte local est
remplacé par `<user>` (PO-011 clos). **Aucune réécriture d'historique n'a été faite ni
proposée comme action**, dans aucun des trois dépôts.
*Motif du retrait :* ces fichiers ne sont pas encore poussés ; les expurger coûte
seulement de la rejouabilité, et ce coût est réversible par l'opérateur qui connaît son
propre chemin `$HOME`.
*Motif du non-retrait de l'historique :* ce qui est publié l'est définitivement. Forks,
caches, vues web et clones tiers ne se réécrivent pas. Une réécriture donnerait
**l'illusion** du retrait tout en cassant tous les SHA et en invalidant les références
croisées des documents. Cette décision appartient à l'opérateur, et à lui seul (PO-012,
PO-013).
`[COMPLÉTÉE le 2026-09-21]` D-005 restait le motif affiché de l'extension de R-07. Ce
motif a été refondé sur la portabilité par **D-008**, précisément parce qu'il s'appuyait
sur la tension ci-dessous. D-005 elle-même **n'est pas révisée** : ce qui a été expurgé
le reste, et rien n'a été réécrit.

*Tension non tranchée :* D-005 ne s'aligne pas avec **D4** de `workstation-config`
(`docs/machine-facts.md:782`), qui accepte l'identifiant et l'identité d'auteur. Les deux
dépôts suivent donc aujourd'hui deux politiques différentes. Signalé, pas arbitré.

### D-006 — Le dossier est séparé en trois fichiers, par **nature de fait** (2026-09-21)
`docs/scomm-certificat-facts.md` conserve ce qui fait autorité et se relit à chaque
session : contraintes, faits sourcés, faits déclarés, points ouverts, décisions,
convention de marquage. `docs/scomm-depot-actuel.md` reçoit la description du brouillon
(F-008 à F-021, F-038). `docs/scomm-journal-preuves.md` reçoit les `P-xx`.
*Motif :* le critère n'est **pas** le volume — 749 lignes se lisent. C'est que la
réécriture décidée en D-007 va périmer une partie du contenu **et pas le reste**, et
qu'il faut pouvoir dire laquelle sans relire. Une fois séparé, cette question a une
réponse d'une ligne : tout `scomm-depot-actuel.md`.
*Ce qui a été refusé :* renuméroter les `F-xxx` et les `P-xx` pour les rendre contigus
dans leur nouveau fichier. Cela aurait cassé tous les renvois existants et effacé
l'ordre de découverte, que R-11 protège. Les trous de numérotation sont le prix, et
c'est le bon prix.
*Ce qui a été refusé aussi :* renuméroter les sections survivantes. § 1.2 et § 5 sont
laissés **vides avec un renvoi**, pour qu'aucun « § 1.2 » ou « § 5 » écrit ailleurs ne
devienne muet.

### D-007 — Le code de ce dépôt est un brouillon sans autorité ; sa réécriture est autorisée (2026-09-21)
Sur déclaration de l'opérateur (F-044). Le contenu de `roles/scomm_agent/` et
`integrate_scomm.yml` peut être modifié ou entièrement réécrit à partir du livrable 4.
*Motif :* ce code a été produit sans source et n'a jamais tourné ; le traiter comme un
existant à préserver ferait porter à la suite du projet les choix d'un brouillon que
personne ne défend. **Ce que cette décision ne change pas :** R-09 — `rhel_post_install`
reste en lecture seule, et D-001 (destination finale du code) reste **non tranchée**.
Autoriser la réécriture **ici** ne dit rien de l'endroit où le résultat vivra.
*Ce qui la rendrait fausse :* que F-044 soit contredit, par exemple si ce code s'avérait
déployé quelque part. Aucune mesure ne peut l'établir (voir F-044) ; seul l'opérateur
peut le corriger.

### D-008 — Le motif de l'extension de R-07 est refondé sur la **portabilité** ; son argument de confidentialité reste suspendu à PO-012 (2026-09-21)
L'extension de R-07 aux identifiants personnels (2026-09-18) est **conservée**, mais son
motif est réécrit sur un argument qui ne dépend d'aucun arbitrage : un chemin absolu
contenant un compte local n'est **pas portable** — ni vers le seed (F-050), ni vers une
cible RHEL 9, ni vers la machine d'un autre opérateur. `/home/<user>/…` se transpose ;
`/home/<compte réel>/…` se copie-colle et échoue.
*Pourquoi ce changement de motif, et pas un arbitrage :* le motif d'origine était un
motif de **confidentialité**, et il appliquait un côté d'une tension que **PO-012**
déclare expressément non arbitrée — ce dépôt-ci expurge, `workstation-config` assume
(D4, `machine-facts.md:782`). Une règle ne peut pas tirer sa force d'une question ouverte.
*Ce qui est donc signalé, et non tranché :* **la justification par la confidentialité de
l'extension de R-07 reste provisoire tant que PO-012 est ouvert.** Si l'opérateur
arbitre en faveur de la position de `workstation-config`, c'est cette justification-là
qui tombe — **pas la règle**, qui tient alors sur la seule portabilité. C'est
précisément pourquoi le motif a été refondé : pour que l'arbitrage de PO-012 change le
*pourquoi* sans rouvrir le *quoi*.
*Ce que cette décision ne fait pas :* elle ne retire rien, ne réécrit aucun historique,
et ne préjuge pas de PO-012 ni de PO-013.

### D-009 — La relecture de S-4 est close ; aucune source supplémentaire n'a été cherchée (2026-09-21)
La question de l'étape 1 du livrable 3 — où naît la clé privée dans S-4 — est **tranchée
par la source** (F-042). Aucune recherche complémentaire n'a été engagée.
*Motif :* consigne explicite du livrable. La conséquence est inscrite comme point ouvert
(PO-015) plutôt que comblée par une source non demandée. Un trou déclaré vaut mieux
qu'un comblement hors mandat.

### D-010 — L'ajout au groupe d'enrôlement est fait à la main pour la recette (2026-09-23)
`host01$` a été ajouté manuellement à `<GROUPE-T>` (F-081). **L'automatisation dans
`rhel_post_install`, suggérée par l'équipe ADCS, ne viendra qu'après qu'un enrôlement ait
réussi au moins une fois** (PO-024).
*Motif :* même logique que D-001 — on ne met pas sur le chemin du parc un mécanisme que
rien n'a exercé. Le droit d'écriture existe (F-082) : c'est le moment qui est choisi, pas
la possibilité.


### D-011 — Un `when:` est un aiguillage, pas une garde : R-12 réécrite (2026-09-23)
**Décision de l'opérateur** sur PO-019, prise le 2026-09-23, appliquée à `CLAUDE.md` le
2026-09-24. Voie **(a)** : `when:` retiré de R-12 et renvoyé à R-04. La définition
retenue de « garde » vit dans R-12 et n'est pas recopiée ici.
*Motif :* un `when:` faux n'a pas de mode d'échec — il saute en silence. Exiger de le
démontrer « dans les deux sens » était insatisfiable, et une règle inapplicable finit par être ignorée **avec les autres**.
*Ce que la décision ne relâche pas :* R-12 porte la contrepartie — **un aiguillage dont le
saut pourrait passer pour un succès doit être doublé d'une garde qui vérifie l'état
attendu.** Pointeur inverse posé dans R-04, seule modification qui lui soit faite, pour
que la délimitation soit lisible des deux côtés.

**ÉTAIT** dans R-12 (R-11) — les trois seuls fragments révisés : **(1)** ouverture :
« Toute garde — `assert`, **`when:`**, `failed_when:`, `stat` préalable, contrôle de
préconditions — est démontrée par **deux exécutions relevées** » ; **(2)** fin du motif :
« C'est aussi ce qui distingue cette règle de R-04 : R-04 demande d'**énumérer** les
branches exercées, R-12 demande de **provoquer** celles qui ne le sont pas. » ; **(3)**
réserve du 2026-09-21, **retirée** : « *Réserve ouverte :* le cas de `when:` dans cette
liste est contesté — voir PO-019. »

### PO-027 — `PKI_enrolment.yml` saute en silence quand la variable ADCS est absente *(ouvert le 2026-09-24)*
Aiguillage légitime aujourd'hui (`main.yml:41`, `adcs_server` commenté, F-071) ; le jour
où ce fichier sera activé, il faudra **une garde vérifiant que la variable est renseignée
là où l'enrôlement est attendu** — sinon une machine non enrôlée passera pour une machine enrôlée. Premier cas d'application de R-12 réécrite (D-011). **Constat seulement : `rhel_post_install` reste en lecture seule (R-09).**


### D-012 — Le diagnostic se joue AVANT la correction de PO-021 (2026-09-24)
**Décision de l'opérateur.** L'entrée locale de résolution fausse le nom que la machine se
donne (F-064, F-088) ; la correction attend.
*Motifs :* la **seule** étape du diagnostic qui en dépendait est l'obtention du ticket de
la machine — elle désigne désormais son principal **explicitement** et n'est plus
affectée ; le ticket de service pour le VIP n'en a jamais dépendu.
*Ce que la décision ne relâche pas :* **la correction reste nécessaire avant tout
enrôlement.** Si le sujet du certificat est construit à partir du nom que la machine se
donne, il sortirait avec le **nom court** — et PO-023 dit que la forme du sujet est
précisément ce qu'on cherche. Diagnostiquer sur une machine faussée est sans risque ;
enrôler ne l'est pas.
`[NOTE du 2026-09-29]` Sur `host02`, la correction a été faite **avant** l'essai de
`cepces` (F-122, geste 1), par décision de l'opérateur : D-014.


### D-013 — La racine est établie par nos propres droits, non demandée à l'équipe PKI (2026-09-24)
**Décision de l'opérateur.** `<ROOT-CA>` a été identifiée par lecture de l'annuaire sur un
canal authentifié, confrontée à ce que le VIP envoie (F-100).
*Motif :* deux sources indépendantes qui concordent valent mieux qu'une affirmation de
tiers, et n'attendent personne. *Ce qu'elle ne fait pas :* autoriser à **poser** l'ancre —
cela reste PO-028, et relève de `rhel_post_install`.


### D-014 — Sur `host02`, la recette est préparée à la main (2026-09-28)
**Décisions de l'opérateur**, prises le 2026-09-28 `[PILOTE-DÉCLARÉ]` et inscrites le
2026-09-29 :
1. les paquets `certmonger` et `cepces*` sont pris dans **AppStream** (F-119) ;
2. `<ROOT-CA>` est posée en **ancre système** (F-122, geste 2) ;
3. `/etc/hosts` est corrigé **avant** l'essai (F-122, geste 1) ;
4. le **suivi `certmonger` est conservé** après un enrôlement réussi.
**Portée : la recette `host02` seulement.** Rien de cela ne vaut pour le parc ni pour
`rhel_post_install` (R-09), sur le modèle de D-010.
*Tension, constatée et non résolue ici :* D-013 dit de l'identification de la racine
qu'elle n'autorise **pas** à poser l'ancre (« cela reste PO-028 ») ; PO-021 dit
qu'« aucune correction n'est écrite ici ». Ni D-013 ni PO-021 ne sont réécrits : **D-014
les dépasse pour `host02` seulement**, par une décision explicite de l'opérateur ; pour le
parc, ils restent vrais.
*Retour arrière :* voir F-122 — retours ciblés et instantané complet.


---

## 5. Journal de preuves — **DÉPLACÉ le 2026-09-21**

Les preuves **P-01 à P-28** vivent désormais dans `scomm-journal-preuves.md`, avec leurs
numéros d'origine. Le numéro de section est conservé vide pour que les renvois « § 5 »
antérieurs continuent de pointer quelque part.
*Motif :* un journal se consulte, il ne se relit pas. Le garder ici faisait relire à
chaque session 222 lignes de sorties de commande pour atteindre les quatre sections qui
prescrivent quelque chose.
