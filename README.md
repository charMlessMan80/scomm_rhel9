# SCOM (scomm) Agent Integration on RHEL 9

Ansible playbook to install, configure, and register the **System Center
Operations Manager (SCOM) Linux agent** on RHEL 9 hosts.

## À quoi sert ce dépôt

Objectif du projet : faire obtenir à des machines RHEL 9 un certificat destiné à
SCOM, **délivré par une ADCS via CEP/CES** derrière un reverse proxy F5, et
maintenu par `certmonger`.

L'état actuel du dépôt est en deçà de cet objectif, et il faut le savoir avant de
lire la suite :

- Ce qui existe aujourd'hui installe l'agent, le configure, et **régénère un
  certificat auto-signé** via `scxsslconfig`. C'est la section *Register*
  ci-dessous.
- **Aucune brique ADCS, CEP, CES, `certmonger`, `cepces` ou F5 n'est présente**
  dans ce dépôt. Elles sont à écrire.
- L'inventaire `scomm_agents` est **vide** : toutes les lignes d'hôtes de
  [inventory/hosts.ini](inventory/hosts.ini) sont commentées. En l'état, le
  playbook ne cible aucune machine.
- ⚠️ Le tag `register` lance `scxsslconfig -f`, qui **écrase** un certificat
  existant. Ne pas le jouer sur une machine déjà enrôlée par CEP/CES.

Documents d'autorité, à lire avant toute modification :

| Fichier | Contenu |
|---|---|
| [CLAUDE.md](CLAUDE.md) | Les règles de travail, chacune avec son motif |
| [docs/scomm-certificat-facts.md](docs/scomm-certificat-facts.md) | Faits sourcés, faits déclarés, points ouverts numérotés, décisions datées |
| [docs/scomm-depot-actuel.md](docs/scomm-depot-actuel.md) | Ce que le code de ce dépôt fait aujourd'hui — brouillon sans autorité |
| [docs/scomm-journal-preuves.md](docs/scomm-journal-preuves.md) | Journal de preuves : sorties de commande `P-xx` avec leur code de retour |
| [docs/procedures/](docs/procedures/) | Procédures jouées par l'opérateur sur les machines cibles |

⚠️ **Ce dépôt est public.** Aucun nom d'hôte réel, URL interne, realm Kerberos,
nom de VIP, adresse IP interne ni secret ne doit y être versionné. Employer des
valeurs d'exemple (`ca.example.com`, `EXAMPLE.COM`, `host01`).

## Ce qui est validable sur le poste de travail, et ce qui ne l'est pas

Le développement se fait sur un poste Fedora personnel. Aucune machine
professionnelle n'est accessible depuis ce poste.

**Validable localement :**

- la syntaxe et le lint YAML/Ansible (`ansible-lint`, `yamllint` — venv partagé
  et unique, hors dépôt ; ne pas en créer un second) ;
- la résolution de l'inventaire et des variables
  (`ansible-inventory --graph`, `--list`) — mesuré, `rc=0` ;
- la lecture du code et de la documentation amont.

**Mesuré comme NON validable en l'état**, alors qu'on l'attendrait :
`ansible-playbook --syntax-check integrate_scomm.yml` retourne **`rc=4`** sur ce
poste, parce que la collection `ansible.posix` — pourtant requise par
[requirements.yml](requirements.yml) et utilisée par
`roles/scomm_agent/tasks/configure.yml:8-14` — n'y est pas installée
(`ansible-galaxy collection list | grep posix` → rc=1 ;
`/usr/share/ansible/collections/ansible_collections` est vide). Elle n'a **pas**
été installée : ce poste n'installe rien. Le contrôle syntaxique redeviendra
possible dès que la collection sera disponible sur le nœud qui l'exécute. Voir
PO-010.

**Non validable localement — mesure obligatoirement déportée sur une cible :**

- l'installation réelle des paquets `omi` / `scx` (le poste est Fedora, pas
  RHEL 9 ; les versions de paquets observées ici ne valent pas pour la cible) ;
- le comportement de `certmonger` et `cepces`, **non installés** sur ce poste ;
- la traversée du F5, la terminaison TLS, et l'authentification Kerberos
  vers CEP/CES ;
- l'acceptation de l'agent par le serveur d'administration SCOM.

Ces mesures font l'objet de procédures courtes, délimitées par des marqueurs et
affichant leur code de retour, jouées par l'opérateur et rapportées par capture
d'écran sous l'étiquette `[ÉCRAN-AAAA-MM-JJ]`. Elles sont recensées dans les
points ouverts de
[docs/scomm-certificat-facts.md](docs/scomm-certificat-facts.md).

## What it does

1. **Install** — adds the SCOM agent DNF repository and installs the `omi`
   and `scx` packages.
2. **Configure** — enables and starts the `omid` service and opens the agent
   port (`1270/tcp`) in firewalld.
3. **Register** — regenerates the agent SSL certificate with the correct FQDN
   (via `scxsslconfig`) and trusts the SCOM management server CA so the
   management server can sign/discover the agent.

It also **creates the SCOM monitoring (Run As) account** and installs a
`/etc/sudoers.d/scom` file granting passwordless sudo for the scx maintenance
commands, so the management server can authenticate and elevate for privileged
workflows. The package-internal `omi` service user is created automatically by
the RPMs.

## Layout

```
ansible.cfg
integrate_scomm.yml            # main playbook
requirements.yml               # required collections
inventory/hosts.ini            # target hosts
group_vars/scomm_agents.yml    # environment variables (edit these)
roles/scomm_agent/             # install / configure / register logic
```

## Usage

1. Install the required collection:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. Edit [inventory/hosts.ini](inventory/hosts.ini) and add your RHEL 9 hosts.

3. Review and adjust [group_vars/scomm_agents.yml](group_vars/scomm_agents.yml)
   — set the real repo URL, and (optionally) the management server CA path.
   Encrypt any secrets with `ansible-vault`.

4. Run it:

   ```bash
   ansible-playbook integrate_scomm.yml
   ```

   Run a single phase with tags:

   ```bash
   ansible-playbook integrate_scomm.yml --tags install
   ansible-playbook integrate_scomm.yml --tags register
   ```
## Monitoring account

Set `scomm_monitoring_user_password` to a **pre-hashed** value and keep it in a
vaulted file:

```bash
openssl passwd -6 'YourStrongPassword'
ansible-vault encrypt group_vars/scomm_agents.yml
```

Use the same username/password when configuring the SCOM Run As accounts. Set
`scomm_manage_monitoring_user: false` to skip account creation, or
`scomm_monitoring_user_sudo: false` to skip the sudoers file.

## Notes

- `scomm_repo_baseurl` defaults to a placeholder Microsoft packages URL —
  point it at whatever repo mirrors the `omi`/`scx` RPMs in your environment.
- The package-internal `omi` user/`omiusers` group are created automatically by
  the RPMs — the monitoring account above is the separate SCOM Run As identity.
- Final agent approval/signing happens on the **SCOM management server**
  (Discovery Wizard or `Approve` in pending management). This playbook prepares
  the agent so that step succeeds.
