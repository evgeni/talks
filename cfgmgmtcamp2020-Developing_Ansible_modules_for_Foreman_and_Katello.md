<!--
date: 2020-02-04
link: https://cfp.cfgmgmtcamp.org/2020/talk/KZCMLR/
-->
# Developing Ansible modules for Foreman and Katello

---

## `$ whoami`

Evgeni Golov

Senior Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

## Foreman + Ansible = <span class="emoji">♥</span>

* Foreman has an API
* Everyone loves writing YAML instead of clicking in a GUI
* So we wrote modules to allow that
* They have bugs, missing features or we miss whole modules
* This is how everyone can help

---

# Foreman Ansible Modules

---

## Foreman Ansible Modules

* A collection of Ansible modules to interact with the Foreman API
* Also supports Foreman plugins like Katello, Remote Execution, SCC
* Provide an abstraction layer, so you don't have to repeat yourself

---

## An example

```yaml=
- name: Create ACME Organization
  foreman_organization:
    username: admin
    password: changeme
    server_url: https://foreman.example.com
    name: ACME
    state: present
```

Note:
* `username`/`password`/`server_url`/`validate_certs` for all modules
* `name` to refer to an entity

---

## Under the hood

* Connect to the API
* Search for an entity (usually by name)
* Create/Update/Delete depending on current state and user input
* Report to the user

---

# Writing Foreman Ansible Modules

---

## Organization Module

```python=
class ForemanOrganizationModule(ForemanEntityAnsibleModule):
    pass

module = ForemanOrganizationModule(
    entity_spec=dict(
        name=dict(required=True),
        description=dict(),
        label=dict(),
    ),
)

with module.api_connection():
    module.run()
```

Note:
* `entity_spec` describes the data
* No special handling, all data searching/updating is handled by the framework
* FIXME `find_resource_by_name` searches for the resource if it already exists
* FIXME `ensure_entity` updates the found resource (or `None`) to the state the user asked for

---

## Reference an Organization (list) from another module

```python=
module = ForemanLocationAnsibleModule(
    entity_spec=dict(
        …
        organizations=dict(type='entity_list'),
    ),
)

with module.api_connection():
    module.run()
```

Note:
* most modules will just inherit from `ForemanTaxonomicEntityAnsibleModule`

---

## Using taxonomies in modules

```python=
class ForemanDomainModule(ForemanTaxonomicEntityAnsibleModule):
    pass

module = ForemanDomainModule(
    entity_spec=dict(
      name=dict(required=True),
      …
  ),
)

with module.api_connection():
    module.run()
```

Note:
* This adds optional `organizations` and `locations` parameters

---

## Renaming entities

```python=
module = ForemanDomainModule(
    argument_spec=dict(
        updated_name=dict(),
    ),
    entity_spec=dict(
        name=dict(required=True),
        …
    ),
)

with module.api_connection():
    module.run()
```

Note:
* The `updated_name` is "magic"

---

## Custom data handling

```python=
entity_dict = module.clean_params()
with module.api_connection():
    entity_dict, scope = 
      module.handle_organization_param(entity_dict)
    entity = module.find_resource_by_name(
      'content_credentials', name=entity_dict['name'],
      params=scope, failsafe=True)

    module.ensure_entity('content_credentials',
      entity_dict, entity, params=scope)
```

Note:
```python=
module = KatelloEntityAnsibleModule(
    entity_spec=dict(
        name=dict(required=True),
        content_type=dict(required=True, 
          choices=['gpg_key', 'cert']),
        content=dict(required=True),
    ),
)
```

---

## Custom workflow handling

```python=
entity_dict = module.clean_params()
with module.api_connection():
    params = {'id': entity_dict['name']}
    power_state = module.resource_action('hosts', 
      'power_status', params=params)
    if module.state == 'state':
        module.exit_json(power_state=power_state['state'])
    elif (module.state == power_state['state']):
        module.exit_json()
    else:
        params['power_action'] = module.state
        module.resource_action('hosts', 'power',
          params=params)
```

Note:
```python=
module = ForemanEntityAnsibleModule(
    entity_spec=dict(
        name=dict(required=True),
        state=dict(default='state',
          choices=['on', 'off', 'reboot', 'state']),
    ),
)
```

---

### Available helpers

* `list_resource`
* `show_resource`
* `find_resource` / `find_resource_by_{name,title,id}`
* `find_resources` / `find_resources_by_{name,title,id}`
* `ensure_entity`
* `resource_action`

---

# Testing Foreman Ansible Modules

---

## our test suite

* Ansible playbooks for each module
    * Handle setup, tests, teardown
    * Ensure idempotency by checking the `changed` property
* VCRpy is used to record API interaction
    * Tests can be run in `test` or `record` mode
    * CI always runs `test` mode
    * Developers need `record` mode when API requests change

---

## test execution

* `make test` runs tests for *ALL* modules
* `make test_<module>` only for that one module
* `make record_<module>` when a new recording is needed

---

## example: `organization.yml`

```yaml=
  - include: tasks/organization.yml
    vars:
      organization_state: present
      expected_change: true
  - include: tasks/organization.yml
    vars:
      organization_state: present
      expected_change: false
```

---

## example: `tasks/organization.yml`

```yaml=
- name: "Testing organization"
  vars:
    - organization_name: "Test Organization"
    - organization_description: "A test organization"
  foreman_organization:
    name: "{{ organization_name }}"
    description: "{{ organization_description }}"
    state: "{{ organization_state }}"
  register: result
- assert:
    fail_msg: "Testing organization failed"
    that:
      - result.changed == expected_change
  when: expected_change is defined
```

Note:
* Lines shortened, real tests have better messages to know what failed

---

# Development environment

---

## Python environment for users

* Modules and dependencies are available as [RPM](https://github.com/theforeman/foreman-ansible-modules#installation-via-rpm)
* And from Ansible Galaxy (modules) / PyPI (dependencies)

Note:

When just consuming existing modules, the easiest way to get a working Python environment with all the dependencies is to [install Foreman Ansible Modules and their dependencies (mostly `apypie` and `requests`) via RPM](https://github.com/theforeman/foreman-ansible-modules#installation-via-rpm) on the machine you execute your playbooks against (usually `localhost` for modules that talk to an API).

---

## Python environment for developers

* A devel setup has more dependencies
* Using a virtualenv is highly recommended!
    * `ansible_python_interpreter = "/usr/bin/env python"`
* The tests also require a configuration file

```bash
python3 -m venv ./venv
source ./venv/bin/activate
make test-setup
```

Note:

However, as soon as one wants to run or adjust the tests or re-record the test fixtures, one needs more Python libraries which might not be available from the usual packaging repositories. To get around, it is recommended to use a Python virtual environment:

```bash
python3 -m venv ./venv # or python2 -m virtualenv ./venv
source ./venv/bin/activate
make test-setup
```

`make test-setup` will ensure all required test dependencies are installed in the virtual environment and generate a configuration file that the tests use (more about that in the next section).

You should now be able to execute `make test` to run the test-suite.

Please be aware, that when you want to use the Python virtual environment for other playbooks, you will need to use the `bin/python` from the virtual environment as the `ansible_python_interpreter`, and not `/usr/bin/python`. The tests use an own inventory file, which sets `ansible_python_interpreter="/usr/bin/env python"`, to achieve that.

---

## Foreman/Katello environment

* (re-)running existing tests (`make test`) uses recorded fixtures
    * this is great to ensure API requests didn't change after refactoring
    * real behavior changes will yield "cannot match request" errors
* behavior changes require new recording
    * need to run `make record_<testname>`
    * requires running Foreman/Katello

---

## Foreman/Katello environment

* on Linux, the easiest way is [`forklift`](https://github.com/theforeman/forklift/#quickstart)
* any instance that can be destroyed is fine
* set URL and admin credentials in `tests/test_playbooks/vars/server.yml`

Note:
The main part of our tests are integration tests that are described as regular Ansible playbooks.
When you execute `make test`, the tests are run against recorded fixtures using `VCRpy`, but when you want to re-record the fixtures because you changed how the module interacts with the API or added/adjusted a test you will require a working Foreman or Katello environment..

If you happen to have a working `Libvirt` and `Vagrant` setup, you can use [`forklift`](https://github.com/theforeman/forklift/) to set up the environment for you. It's best to use the latest available `centos7-foreman-X.Y` or `centos7-katello-X.Y` boxes for the tests.

Any other uptodate installation that you have admin permissions for will work too, but probably don't try to use your production setup.

Whichever path you select, the URL and credentials for the environment need to be added to `tests/test_playbooks/vars/server.yml` (which `make test-setup` created earlier).

Now you should be able to run `make record_<testname>` (e.g. `record_organization`) which should execute the `organization.yml` playbook against your environment and see the files under `tests/test_playbook/fixtures/<testname>-*` change.

---

## Debugging Modules

If you're used to `print`-based debugging, Ansible will hide all interesting information from you and you'll need a different approach.

---

### `raise Exception` and `module.warn`

* `raise Exception("the message")`
* `module.warn("the message")`
* not nice, but gets the job done

Note:

The simplest way to get output from an Ansible module mid-execution is to either raise an Exception with the output you want, or call `module.warn("the message")`. Neither is nice, but gets the job done quickly ;-)

---

### q

[`q`](https://pypi.org/project/q/) is the Quick-and-dirty debugging output for tired programmers.

* `q("the message")`
* output goes to `/tmp/q`

Note:

After importing it, you can just call `q("the message")` and it will get logged to `/tmp/q`.

See the PyPI page for more documentation.

---

### a real debugger

* `pdb` is the default Python debugger, but [doesn't play nice with Ansible](https://docs.ansible.com/ansible/latest/dev_guide/debugging.html)
    * mostly because Ansible forks another Python process
* a debugger with remote debugging feature is useful: `epdb`, `remote-pdb`

Note:

The Ansible Developer Guide has an example how to [use `pdb` with Ansible modules](https://docs.ansible.com/ansible/latest/dev_guide/debugging.html), but the direct `pdb` route has two disadvantages: you need to call the module directly, so you can't just reuse your playbook and you need to jump though extra steps when debugging a remotely running module.

Thankfully, there are extensions to `pdb` that allow native remote debugging: `epdb`, `remote-pdb`, `web-pdb` (there are also others, but these are the ones I tried).

---

# Demo

---

let's fix [#586](https://github.com/theforeman/foreman-ansible-modules/issues/586) together

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-twitter" aria-hidden="true"></i> [@zhenech](https://twitter.com/zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
