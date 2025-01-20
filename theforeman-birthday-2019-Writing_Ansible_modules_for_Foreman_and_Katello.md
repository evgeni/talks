<!--
date: 2019-07-25
link: https://community.theforeman.org/t/foreman-birthday-party/13731
-->
# Writing Ansible modules for Foreman and Katello

---

## `$ whoami`

Evgeni Golov

Senior Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian and Grml Developer

♥ FOSS ♥

♥ automation ♥

---

## WTF?!

* 15 minute version of 45 minute talk
* how to best automate Foreman/Katello using Ansible
* spoiler: `command: hammer` is not the answer!

---

## Why not `X`?!

* [`ansible-module-foreman`](https://github.com/Nosmoht/ansible-module-foreman) by Thomas Krahn ([@Nosmoht](https://github.com/Nosmoht)) is probably the oldest
    * Well maintained
    * Supports only Foreman
* Upstream Ansible `foreman` and `katello` modules
    * Deprecated since Ansible 2.8
    * "one" module for everything

Note:
* Nosmoth does not use `apidoc.json`
* Upstream uses `nailgun`, to be removed in 2.12

---

## Foreman Ansible Modules

* Started June 2017 as a repository to clean up upstream modules
* One module per Foreman entity or action
* Extensive test-suite
* Abstraction framework for common tasks (connect, search, create, update, delete)

---

## Foreman Ansible Modules

* Initially, we still used `nailgun`
    * `nailgun` releases are Satellite version specific
    * Plugins not in Satellite are not supported
    * Doesn't work without Katello installed
* Recent switch to `apypie`
    * Consumes the `apidoc.json` published by Foreman / `apipie-rails`
* Migration quite easy thanks to the existing framework and tests

---

## Foreman Ansible Modules - Stats

* 43 <span class="emoji">🌟</span> on GitHub
* 24 Contributors (8 Red Hat, 7 ATIX)
* 8 new Contributors in 2019

---

<img src="https://people.redhat.com/egolov/fam/fam-new-contributors.png" alt="new contributors" style="width:80%; height:80%"/>

Note:
* almost no new contributors in 2018 (well, 5, but…)
* none in between February and September
* Patrick (in September) is a one-time contributor so far
* yes, the slide only has 23 contributors…

---

<img src="https://people.redhat.com/egolov/fam/fam-pr-activity.png" alt="PR activity" style="width:80%; height:80%"/>

Note:
* we made Module writing easy in January/February 2019
* before that we had 7.74 PRs on average per month
* after it's 12.33 PRs!

---

## Foreman Ansible Modules - Outlook

* Collection on Ansible Galaxy
* RPM on yum.theforeman.org

---

# Let's write a module!

---

## Under the hood

Most modules manage objects/entities in Foreman

1. Find an existing entity
2. Compare existing entity with the data provided by the user
3. Save the entity

We have a framework to support that

---

First a wrapper around `AnsibleModule`:

```python
from ansible.module_utils.foreman_helper import
  ForemanEntityApypieAnsibleModule

module = ForemanEntityApypieAnsibleModule(
 argument_spec=dict(name=dict(required=True)))
```

---

Load user provided parameters and connect to the API:

```python
entity_dict = module.clean_params()
module.connect()
```

---

Find the entity and ensure it looks like the user wanted:

```python
entity = module.find_resource_by_name('architectures',
  name=entity_dict['name'], failsafe=True)
changed = module.ensure_resource_state('architectures',
  entity_dict, entity, name_map)
module.exit_json(changed=changed)
```

Note:
* resource name is whatever is in the docs
* `name_map` is still undefined
* `entity_dict` is the filtered user input
* `entity` entity the existing entity, or None

---

Translate Ansible params to Foreman API params:

```python
name_map = { 'name': 'name' }
```

---

```python=
from ansible.module_utils.foreman_helper import
  ForemanEntityApypieAnsibleModule

name_map = { 'name': 'name' }
module = ForemanEntityApypieAnsibleModule(
 argument_spec=dict(name=dict(required=True)))
entity_dict = module.clean_params()
module.connect()

entity = module.find_resource_by_name('architectures',
  name=entity_dict['name'], failsafe=True)
changed = module.ensure_resource_state('architectures',
  entity_dict, entity, name_map)
module.exit_json(changed=changed)
```

---

```python
if not module.desired_absent:
  if 'operatingsystems' in entity_dict:
    entity_dict['operatingsystems'] = 
      module.find_resources_by_title('operatingsystems',
        entity_dict['operatingsystems'], thin=True)
```

Note:
* boring without optional params
* here we add a list (!) of OSes to an architecture

---

```python
if not module.desired_absent:
  if 'operatingsystems' in entity_dict:
    search_list = ["title~{}".format(title) for title
                   in entity_dict['operatingsystems']]
    entity_dict['operatingsystems'] =
      module.find_resources('operatingsystems', search_list,
                            thin=True)
```

Note:
* We can also be more flexible in searching
* OSes sometimes want fuzzy search :/

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-twitter" aria-hidden="true"></i> [@zhenech](https://twitter.com/zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)

<i class="fa fa-stack-exchange" aria-hidden="true"></i> [zhenech](https://stackexchange.com/users/1107433/zhenech)
