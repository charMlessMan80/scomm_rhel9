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
- **F-012** — Paquets installés : `omi` et `scx` (`defaults/main.yml:12-14`),
  via `ansible.builtin.dnf … state: present, update_cache: true`
  (`install.yml:25-30`). Aucune version n'est épinglée : `state: present`, pas de `version:`.
- **F-013** — Prérequis installés d'office : `openssl`, `ca-certificates`
  (`install.yml:2-7`).
- **F-014** — Configuration posée : service `omid` démarré et activé
  (`configure.yml:2-6`, nom depuis `scomm_service_name`, `defaults/main.yml:16`) et
  ouverture de `1270/tcp` dans firewalld (`configure.yml:8-14`), conditionnée par
  `scomm_manage_firewall` (défaut `true`, `defaults/main.yml:19`).
- **F-015** — Un compte local de supervision est créé (`users.yml:5-19`) avec un
  fichier sudoers `/etc/sudoers.d/scom` validé par `visudo -cf %s`
  (`users.yml:21-28`, gabarit `templates/scom_sudoers.j2`). Le mot de passe est attendu
  **pré-haché** et par défaut **vide** (`defaults/main.yml:40` : `scomm_monitoring_user_password: ""`),
  donc `default(omit)` s'applique (`users.yml:18`) et aucun mot de passe n'est posé.
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
