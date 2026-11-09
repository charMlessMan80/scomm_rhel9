# État du dépôt `scomm_rhel9` — description d'un **brouillon sans autorité**

**Ce que ce fichier porte :** la description, mesurée le 2026-09-18, de ce que le code de
ce dépôt fait *aujourd'hui*. Rien d'autre.

**Ce que ce fichier NE porte PAS :** aucune prescription, aucune décision, aucun point
ouvert, aucune règle. Rien ici ne fait autorité et rien ici ne doit servir de modèle.

**Pourquoi il est séparé.** Le code décrit ici a été produit avant l'établissement de la
méthode, sans source, et n'a jamais tourné en production : il est déclaré sans autorité
(F-044, `scomm-certificat-facts.md` § 2.2) et sa réécriture est actée (D-007). **Ce
fichier est donc périmable en bloc** : la réécriture le rendra faux, et c'est prévu. Le
séparer permet de savoir *sans relire* quelle part du dossier tombe ce jour-là.

Les faits gardent leur numéro d'origine (F-008 à F-021, F-038) : ils ont été déplacés,
pas renumérotés (R-11). Le dossier d'autorité renvoie ici ; il ne les recopie pas.

Faits mesurés le **2026-09-18** ; preuves dans `scomm-journal-preuves.md` (P-05, P-06).
`[NOTE du 2026-10-08]` Complété par **F-155 à F-161**, relevés le 2026-10-08 (relevé S2-a
et exécution du rôle sur `host02`), en fin de fichier. Le code n'a pas changé entre les
deux dates.
`[NOTE du 2026-10-20]` Complété par **F-188** (confrontation du gabarit sudoers à S-16),
en fin de fichier. Le code n'a pas changé.
`[NOTE du 2026-11-09]` **F-015 révisé** (filtre `default` de Jinja2, P-78). Le code n'a
pas changé.

---

### 1.2 Ce que `scomm_rhel9` fait aujourd'hui

- **F-008** — Point d'entrée unique : `integrate_scomm.yml:1-17`. Il cible le groupe
  `scomm_agents` (`integrate_scomm.yml:3`), `become: true` (`:4`), et n'importe qu'un rôle,
  `scomm_agent` (`:16-17`).
- **F-009** — Un garde-fou de distribution existe : `assert` sur
  `distribution in ['RedHat','CentOS','Rocky','AlmaLinux','OracleLinux']` et
  `distribution_major_version == '9'` (`integrate_scomm.yml:8-14`).
- **F-010** — **L'inventaire est vide.** `inventory/hosts.ini:6-8` ne contient que des
  hôtes commentés ; `ansible-inventory --graph` (rc=0) affiche
  `@scomm_agents:` sans aucun hôte (P-06). Le playbook ne cible donc **aucune machine**
  en l'état. Une variable commentée est une variable absente.
- **F-011** — Le paquet de l'agent vient d'un dépôt DNF déclaré par le rôle :
  `roles/scomm_agent/tasks/install.yml:9-17` (`yum_repository`), alimenté par
  `scomm_repo_baseurl`, dont la valeur par défaut est
  `https://packages.microsoft.com/rhel/9/prod/`
  (`roles/scomm_agent/defaults/main.yml:8`, redite en `group_vars/scomm_agents.yml:8`).
  Le `README.md:75-76` qualifie lui-même cette URL de *placeholder*.
  `[RÉVISÉE le 2026-10-08]` **Ce n'est pas un *placeholder* :** ce dépôt publie `omi` et
  `scx` (« Microsoft System Center Operations Manager for UNIX/Linux agent »), jusqu'à
  **1.9.4-0** (F-156) ; le rôle s'y est installé sur `host02` (F-149, dossier). Et le
  renvoi est **périmé** : depuis `59edd05`, le passage est à `README.md:148-149`.
  **ÉTAIT :** la phrase ci-dessus, sans réserve.
- **F-012** — Paquets installés : `omi` et `scx` (`defaults/main.yml:12-14`),
  via `ansible.builtin.dnf … state: present, update_cache: true`
  (`install.yml:25-30`). Aucune version n'est épinglée : `state: present`, pas de `version:`.
- **F-013** — Prérequis installés d'office : `openssl`, `ca-certificates`
  (`install.yml:2-7`).
- **F-014** — Configuration posée : service `omid` démarré et activé
  (`configure.yml:2-6`, nom depuis `scomm_service_name`, `defaults/main.yml:16`) et
  ouverture de `1270/tcp` dans firewalld (`configure.yml:8-14`), conditionnée par
  `scomm_manage_firewall` (défaut `true`, `defaults/main.yml:19`).
  `[NOTE du 2026-10-08]` Renvoi à compléter : le `notify` de cette tâche est à
  `configure.yml:15` (tâche entière : `:8-15`). Ouvrir le pare-feu n'ouvre pas le port de
  l'agent installé par RPM (PO-050, dossier).
- **F-015** — Un compte local de supervision est créé (`users.yml:5-19`) avec un
  fichier sudoers `/etc/sudoers.d/scom` validé par `visudo -cf %s`
  (`users.yml:21-28`, gabarit `templates/scom_sudoers.j2`). Le mot de passe est attendu
  **pré-haché** et par défaut **vide** (`defaults/main.yml:40` : `scomm_monitoring_user_password: ""`),
  donc `default(omit)` s'applique (`users.yml:18`) et aucun mot de passe n'est posé.
  `[NOTE du 2026-10-08]` Renvoi à compléter : la condition `when:` du sudoers est à
  `users.yml:29` (tâche entière : `:21-29`).
  `[RÉVISÉE le 2026-11-09 — FAUX sur le mot de passe]` **`default(omit)` ne s'applique
  pas à une chaîne vide.** Le filtre `default` de Jinja2 ne rend sa valeur par défaut que
  si la variable est **indéfinie**, ou **fausse avec `boolean=true`** — source à la
  version installée sur le poste (`python3-jinja2-3.1.6`, `filters.py:546-576`,
  `if isinstance(value, Undefined) or (boolean and not value)`), et mesure : `""` rendu
  tel quel, valeur par défaut rendue pour une variable indéfinie ou avec `true` (P-78).
  `scomm_monitoring_user_password` étant **définie** à `""` (`defaults/main.yml:40`),
  `users.yml:18` transmet **`password: ""`** au module `user`. Le commentaire
  `defaults/main.yml:39` (« Leave empty to skip setting a password ») dit la même chose
  que l'ancienne phrase, et il est faux pour la même raison. **Non établi :** l'effet de
  `password: ""` sur le compte ; la version de Jinja2 du seed. Risque : PO-065 (dossier).
  **ÉTAIT :** « … par défaut **vide** (`defaults/main.yml:40` :
  `scomm_monitoring_user_password: ""`), donc `default(omit)` s'applique
  (`users.yml:18`) et aucun mot de passe n'est posé. »
- **F-016** — **Ce que le dépôt fait aujourd'hui d'un certificat.** Tout est dans
  `roles/scomm_agent/tasks/register.yml` et se résume à trois choses :
  1. `stat` sur `/opt/microsoft/scx/bin/tools/scxsslconfig` (`register.yml:8-11`) ;
  2. exécution de `scxsslconfig -f -h <hostname> -d <domain>` (`register.yml:13-24`),
     c'est-à-dire la **(re)génération d'un certificat auto-signé local** ;
  3. copie facultative d'une CA du serveur d'administration vers
     `/etc/pki/ca-trust/source/anchors/scom-management-ca.crt` (`register.yml:26-34`),
     puis `update-ca-trust extract` via handler (`handlers/main.yml:11-13`).
- **F-017** — **Aucune brique ADCS, CEP, CES, certmonger, cepces ou F5 n'existe dans ce
  dépôt.** Vérifié par lecture intégrale des 15 fichiers suivis (P-05). Il n'y a rien à
  lire sur ces points : le projet est à écrire, pas à corriger.
  `[NOTE du 2026-10-08]` « 15 fichiers suivis » vaut au 2026-09-18. Le dépôt en suit 21
  aujourd'hui ; les 6 ajoutés sont de la documentation (`CLAUDE.md`, `.gitignore`,
  `docs/…`). Des 15 fichiers d'origine, seul `README.md` a changé (73 lignes ajoutées par
  `59edd05`, aucune retirée) : `git diff --stat 5f0eca9 HEAD`, rc=0. Le constat tient.
- **F-018** — Les deux tâches de `register.yml` qui touchent au certificat sont
  **conditionnelles et non prouvées** :
  `register.yml:21-23` exige `scomm_scxsslconfig.stat.exists` **et**
  `scomm_agent_domain | length > 0` — or `scomm_agent_domain` vaut
  `ansible_facts['domain'] | default('')` (`defaults/main.yml:23`), donc vide sur un hôte
  hors domaine ; `register.yml:33` exige `scomm_management_server_ca_src | length > 0`,
  or ce défaut est `""` (`defaults/main.yml:25`). **Aux valeurs par défaut du dépôt,
  ces deux tâches sont sautées.**
- **F-019** — Les variables attendues sont déclarées à deux endroits identiques :
  `roles/scomm_agent/defaults/main.yml:6-45` et `group_vars/scomm_agents.yml:6-49`.
  Le commentaire `defaults/main.yml:2-4` assume la duplication (« kept here so the role
  is self-contained »). Aucune variable ne provient d'un vault : il n'existe aucun
  fichier vaulté ni aucune tâche de récupération de secret dans le dépôt (F-006, F-017).
  Le `README.md:61-67` se borne à recommander `ansible-vault encrypt`.
- **F-020** — Une seule collection est requise : `ansible.posix >= 1.5.0`
  (`requirements.yml:2-4`), employée par `ansible.posix.firewalld` (`configure.yml:9`).
- **F-021** — Structure du travail : un rôle unique, découpé en quatre imports tagués
  `[scomm, install|configure|users|register]` (`roles/scomm_agent/tasks/main.yml:2-17`).
  L'escalade est globale via `ansible.cfg:9-11` (`become = True`, `become_method = sudo`).


### Conflit avec la documentation de l'outil

- **F-038** — **Conflit direct entre S-5 et le code actuel du dépôt.** `register.yml:13-24`
  exécute `scxsslconfig -f`, dont `-f` signifie « force certificate to be generated even
  if one exists » (S-5, aide de l'outil). Appliqué après un enrôlement ADCS, il
  **écrase** le certificat et la clé installés. Cf. PO-003.


### Constats du 2026-10-08 (relevé S2-a, exécution sur `host02`)

Lecture du code à `31a536c`, identique à `5f0eca9` pour ces fichiers. Exécution relayée
`[ÉCRAN-2026-10-08, relayé]` ; dépôt de paquets lu sur le poste de rédaction (P-65).

- **F-155** — **Le rôle peut écraser un certificat posé à la main.** `register.yml:13-24`
  lance `scxsslconfig -f` si l'outil existe **et** si `scomm_agent_domain` n'est pas vide
  (`:21-23`). `templates/scom_sudoers.j2:11` donne en plus au compte de supervision le
  droit de lancer `scxsslconfig -*` par `sudo`. Contraire à R-10 ; cf. PO-051, D-015 (b).
- **F-156** — **Le dépôt par défaut est réel.** `packages.microsoft.com/rhel/9/prod`
  publie huit versions d'`omi` et de `scx`, de 1.7.0-0 à **1.9.4-0** (2026-08-21) ;
  résumé de `scx` : « Microsoft System Center Operations Manager for UNIX/Linux agent »
  (P-65). Révise F-011 ; `README.md:148-149` dit encore « placeholder ».
- **F-157** — **Deux réglages absents de cette description.** `ansible.cfg:4` :
  `host_key_checking = False`. `inventory/hosts.ini:11` : `ansible_user=ansible`, quand
  `rhel_post_install` désigne `ansible_admin` comme compte de la cible (F-072, dossier).
- **F-158** — **Message affiché même quand rien n'a été fait.** `register.yml:36-40`
  (`debug`, sans `when:`) annonce « Agent certificate generated at
  /etc/opt/omi/ssl/omi-host-… » même si la génération a été sautée (R-04).
- **F-159** — **Deux chemins d'installation.** Le rôle installe des RPM depuis un dépôt
  (`install.yml:25-30`) ; le sudoers autorise l'installation et la mise à jour par un
  *bundle* `scx-1.*.universal….sh` déposé dans `/tmp` (`scom_sudoers.j2:14-16`). Seul le
  *bundle* sait ouvrir le port 1270 (`--enable-opsmgr`, PO-050).
- **F-160** — **`--tags` saute la garde RHEL 9.** L'`assert` de `integrate_scomm.yml:8-14`
  n'a pas de tag ; `man ansible-playbook:124-125` (poste de rédaction, core 2.20.7) :
  « -t, --tags  only run plays and tasks tagged with these values ». Avec
  `--skip-tags`, elle s'exécute : l'exécution de `host02` l'a listée (F-149, dossier).
- **F-161** — **Dépendance cachée.** `ansible.cfg:6` : `stdout_callback = yaml`, fourni par
  `community.general` ; `requirements.yml:2-4` ne demande qu'`ansible.posix`. Sur
  `seed01`, le *callback* est introuvable (F-147, dossier).


### Constat du 2026-10-20 (confrontation à S-16)

Code relu à `4eaf1db`, inchangé depuis `31a536c` pour ce fichier ; S-16 lu sur le poste
de rédaction (P-69).

- **F-188** — **Le gabarit sudoers ne reproduit pas le modèle Microsoft qu'il revendique.**
  `templates/scom_sudoers.j2:3` : « Based on Microsoft's documented low-privileged
  UNIX/Linux account sudo config ». Or `scxadmin -status *` (`:9`), `scxsslconfig -*`
  (`:11`) et `/bin/sh /tmp/scx_operations_manager_*/*` (`:26`) n'apparaissent **dans
  aucun** des blocs de S-16, toutes versions et tous systèmes confondus (`grep -c` = 0 sur
  le texte de la page, P-69). Le gabarit donne en outre tout à **un seul** compte (F-015),
  quand S-16 sépare maintenance (`scomadm`) et supervision (`scomuser`). Cf. PO-065,
  D-017 (d), dossier.
