# SCOM (scomm) Agent Integration on RHEL 9

Ansible playbook to install, configure, and register the **System Center
Operations Manager (SCOM) Linux agent** on RHEL 9 hosts.

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
