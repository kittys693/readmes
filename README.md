- [Reconfigure AWX](#reconfigure-awx)
- [Fragmental Reconfiguration](#fragmental-reconfiguration)
    + [stages](#stages)
    + [templates](#templates)
- [Dynamic Inventory Scripts](#dynamic-inventory-scripts)
- [Discrete Permissions Model](#discrete-permissions-model)
- [Credentials](#credentials)
- [Sequence of Tasks](#sequence-of-tasks)

# Reconfigure AWX
This role will read inputs from defined variables and vault files and create an Ansible AWX configuration based upon those inputs.

Refer to [CAWX Best Practices](https://gitlab.healthpartners.com/Emtech/reference-docs/blob/master/automation-technology/cawx-best-practices.md) for more information on how this role is intended for use.

# Fragmental Reconfiguration
Reconfiguration of the entire instance can be filtered by including two optional vars to the process.  

### stages  
A *list* which limits which stages of the reconfiguration process run.  **Omitting this var will run all stages.**

Example:  
```yaml
stages:
  - python-packages
  - ansible-configuration
  - awx-settings
  - clean-drift
```

Valid stages include:  

`python-packages`  
- Updates python packages.
- Uses the following config items:  
  - `pip_install`

`ansible-configuration`  
- Updates Ansible configuration
- Updates `ansible.cfg` on the awx_task container
- Uses the following config items:  
  - `callback_plugins`

`awx-settings`  
- Updates AWX settings for ldap and job settings
- Uses the following config items:  
  - `ldap`
  - `settings_jobs`

`clean-drift`  
- Removes items not found in source
- Uses the following config items:  
  - `local_users`
  - `job_templates`
  - `projects`
  - `survey_spec`

`post-src`  
- Adds items in source not in application
- Uses the following config items:  
  - `local_users`
  - `projects`
  - `inventories`
  - `inventory_sources`

`patch-existing`  
- Updates items in app with source configuration  
- Uses the following config items:  
  - `local_users`
  - `projects`
  - `inventories`
  - `inventory_scripts`
  - `inventory_sources`
 

`job-template`
- Posts and Updates items in app with source configuration related to templates 
- Uses the following config items:  
  - `job_templates`  
  - `template_vault_credentials`
  - `survey_spec`

`resync-inventories`  
- Resyncs all inventories
- Uses the following config items:  
  - `inventory_sources`

### templates
A *list* of template names to target specifically.  

Example:  
```yaml
templates:
  - build-kong-sandbox
  - build-kong-dev
  - generic-mf
```

# Dynamic Inventory Scripts
Specify an `inventory_scripts` variable that points at a URL(raw code) of a script used to dynamically generate an inventory.  

```yaml
inventory_scripts:

  - name:  myScript
    description: "script to generate a dynamic inventory thing"
    url: "https://gitlab.healthpartners.com/holden/dynamicinventory/raw/master/inventory.py"
```

Assign that script to a inventory using the following format:

```yaml
inventory_sources:
  - name: scripty_mcscripterson
  - source_script: "myScript"
  - inventory: "inventory_mcinventoryerson"
```

# Discrete Permissions Model

**update 10-30-19**
added the ability to add already existing users to teams.  example update below.  These need to be users that exist already or the operation will fail.  

CAWX is now able to be used to assign granular permissions at a team level to job templates.  

This is done through the `teams` variable, which is likely supplied by your `group_vars/all/teams` in an AWX configuration repository. See an example of a team configuration below:  


```yaml
teams:

  - name: Example_Team_One
    description: "This is the first example team."
    ldap_binds:
      - 'CN=ocp-cluster-admins,OU=HPGroups,OU=Accounts,DC=HealthPartners,DC=int'
    remove: true
    users:
      - local_user_one
      - local_user_two
    roles:
      admin:
        - "hello_world"
        - "hlo_example"
      read:
        - "minnesota_wild"
      execute:
        - "leonard_bernstein"
```

Here is a key of what these elements are:


| paramater | description | example | required | default
| -- | --- | --- | - | --- |
`name` | The name of the team | DevOps Cloud Computing Engineers | **Y** | N/A
`description` | Description of the team | Team that enables DevOps Scrum Culture using Kubernetes | **N** | Description not provided.
`ldap_binds` | **list** of all DNs for team, users in these groups will be added to appropriate AWX groups. | `CN=Slack_Users,OU=HPGroups,OU=Accounts,DC=HealthPartners,DC=int` | **N** | `CN=Is&t EmTech,OU=DLGroups,OU=Accounts,DC=HealthPartners,DC=int`
`remove` | when `True`, a user who is not a member of the given groups will be removed from the team | True/False | **N** | `False`
`roles/admin` | **list** of job templates team can modify, execute, and read | hello_world | **N** | `null`
`roles/read` | **list** of job templates team can view but not execute or modify | hello_world | **N** | `null`
`roles/execute` | **list** of job templates team can execute and read but not modify | hello_world | **N** | `null`
`organization` | what organization team belongs to.  don't use this unless you're using orgs, which you're probably not. | Default | **N** | `1`/`Default`
`users` | **list** of local and already existing users to be aded to team.  think service accounts | service_account_one | **N** | N/A

LDAP binds for the team will be found in `settings/authentication/ldap/LDAP TEAM MAP`.  These are automatically created during the `update-awx-settings` phase of `reconfigure-awx`.  There is currently an AWX bug that if you try to modify this using the UI, it will fail, as the value for `LDAP USER FLAGS BY GROUP` is default populated with something AWX considers invalid.  This was a [closed issue](https://github.com/ansible/awx/issues/1855) on GitHub but I commented on 10-8-19 and they've since reopened the issue.

TODO: Patch teams step?

# Credentials
The way credentials are handled has been changed in version 6.0.0+ of AWX.  The role has been updated to reflect this as such:

Credentials can now be supplied in three ways:

1) In `templates` file or `job_templates` variable, you can supply a `vault_credentials` list as well as a single `credential` for a machine credential:

```yaml
job_templates:
  - name: foo
    ...
    credential: <MACHINE_CREDENTIAL> (optional)
    ...
    vault_credentials:
      - <VAULT_CREDENTIAL_ONE>
      - <VAULT_CREDENTIAL_TWO>
```

2) Inside of `template_vault_credentials` variable/file you can supply all of your vaults as well as a machine credential:

```yaml
template_vault_credentials:
  - template_name: "foo"
    vault_credentials:
      - <VAULT_CREDENTIAL_ONE>
      - <VAULT_CREDENTIAL_TWO>
```

3) Kind of a combination of all of the above. Please note that `vault_credential` is not a list.

```yaml
job_templates:
  - name: foo
    ...
    ...
    credential: <MACHINE_CREDENTIAL>
    vault_credential: <VAULT_CREDENTIAL_ONE>

template_vault_credentials:
  - template_name: "foo"
    vault_credentials:
      - <VAULT_CREDENTIAL_TWO>
      - <VAULT_CREDENTIAL_THREE>
```

If a template is going to use multiple vault credentials, you must make sure you supply an identifier to each of the vault
credentials used inside of the template otherwise the reconfigure will fail.

# Sequence of Tasks
S refers to source , A refers to app i.i awx
~ S refers to not in Source
S^A refers to resources which are in source and app
S-A refers to resources which are in source not in app


| parameter | method |  resources description |
| -- | --- | --- |
`python` | update | Update python packages |
`config` | update | Update ansible config and clone callback plugins | 
`ldap` | PATCH | update ldap config |
`ldap for teams` | PATCH | update ldap binding for teams |
`inventory` | DELETE | ~ S  |
`inventory_source` | DELETE | ~ S  |
`inventory_script` | DELETE | ~ S  |
`local users` | DELETE | ~ S  |
`teams` | DELETE | ~ S  | 
`templates` | DELETE | ~ S  |
`template survey` | DELETE | ~ S  |
``projects` | DELETE | ~ S  | 
`local users` | POST |  S - A  | 
`local teams` | POST |  S - A |
`projects` | POST |  S - A | 
`inventories` | POST |  S - A | 
`inventory_scripts` | POST |  S - A
`inventory_source` | POST |  S - A |
`users` | PATCH | S ^ A |
`projects` | PATCH | S ^ A |
`inventories` | PATCH | S ^ A |
`inventory_scripts` | PATCH | S ^ A
`inventory_sources` | PATCH | S ^ A | 
`templates` | POST |  S - A | 
`templates` | PATCH | S ^ A | 
`template survey`  | POST | S | 
`credentials` | POST | Update/Delete job template credemntials as needed
`teams & roles` | GET, POST | update teams and roles |
`resync inventories` | POST | update inventory sources | 
