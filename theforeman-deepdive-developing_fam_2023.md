# Deep Dive: Foreman Ansible Modules

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

## Foreman + Ansible = ♥️

* Foreman has an API
* Everyone loves writing YAML instead of clicking in a GUI
* So we wrote modules to allow that
* They have bugs and are missing features
* This is how everyone can help

---

# Foreman Ansible Modules

---

## Foreman Ansible Modules

* A collection of Ansible modules to interact with the Foreman API
* Also supports Foreman plugins like Katello, Remote Execution, etc
* Provide an abstraction layer, so you don't have to repeat yourself

---

## An example

```yaml=
- name: Create ACME Organization
  theforeman.foreman.organization:
    username: admin
    password: changeme
    server_url: https://foreman.example.com
    name: ACME
    state: present
```

Note:
* `username`/`password`/`server_url`/`validate_certs` for all modules
* `name` to refer to an entity
* I hope everone seen something like this before ;-)

---

## Under the hood

* Connect to the API
* Search for an entity (usually by name)
* Create/Update/Delete depending on current state and user input
* Report to the user

Note:
In a perfect world, every action you can do via UI can also be done in hammer and FAM.

---

## Common issues

* bad/incomplete docs and examples
* missing parameters
* missing modules

---

## How can you help?

* Ideally, a PR at [theforeman/foreman-ansible-modules](https://github.com/theforeman/foreman-ansible-modules)
  * Please not at [RedHatSatellite/satellite-ansible-collection](https://github.com/RedHatSatellite/satellite-ansible-collection)
* GitHub Issues / BZs with descriptions also help :-)

Note:
* We won't teach you how to open a good issue here, but the easier it is for us to reproduce an issue, the higher are the chances for a timely fix.

---

# Today: code contributions - from typo to new module

---

## repository structure

```console
.
├── changelogs
├── docs
├── meta
├── plugins
│   ├── callback
│   ├── doc_fragments
│   ├── filter
│   ├── inventory
│   ├── modules
│   └── module_utils
├── roles
└── tests

```

---

## `changelogs/`

* We use [antsibull-changelog](https://github.com/ansible-community/antsibull-changelog) for changelog generation
* This directory holds snippets that will be used for the next release's changelog
* We'll take care for you if needed
* Example:

```yaml
bugfixes:
  - repository - don't fail when removing a content
    credential from a repository
    (https://bugzilla.redhat.com/show_bug.cgi?id=2224122)

```

---

## `docs/`

* Doesn't actually contain most of the docs
* Just config and templates for Sphinx
* Can be safely ignored for today's excercise

---

## `meta/`

* `runtime.yml`
  * Minimum supported Ansible version
  * Module redirects
  * Action groups
* `execution-environment.yml`
  * Configuration for Ansible Builder
  * To build EEs for Ansible Automation Platform
* Again, nothing trully interesting for today

Note:
* Action groups need adjustment when adding new modules, but we have it automated

---

## `plugins/`

* Here's *everything*
* Actual plugins (`callback`, `filter`, `inventory`, `modules`)
* Documentation
  * Inside the actual plugin files
  * Shared snippets in `doc_fragments`
* Shared code in `module_utils`

---

## `roles/`

* We also have roles
* They are mostly thin wrappers to allow easier bulk actions, e.g.:
  * Create Products/Repos from a list
  * Publish a list of Content Views
  * etc
* Nothing Foreman special

---

## `tests/`

* A few unit tests for select parts of the shared code
* Mostly integration tests for the modules:
  * Create something
  * Create it again (assert no changes)
  * Update it
  * Update it again (assert no changes)
  * Delete it
  * Delete it again (assert no changes)
* Test fixtures are recorded using VCR, so no Foreman is required to re-run the tests

---

# Excercise 1: update documentation

---

```diff
Author: Kenny Tordeurs <ktordeur@redhat.com>
this needs to run as a loop to recursively go
over the directory so needs to be correctly indented

--- plugins/modules/job_template.py
+++ plugins/modules/job_template.py
@@ -243,8 +243,8 @@ EXAMPLES = '''
     - SKARO
     organizations:
     - DALEK INC
-    with_fileglob:
-     - "./arsenal_templates/*.erb"
+  with_fileglob:
+    - "./arsenal_templates/*.erb"

 # If the templates are stored locally and the ansible module is executed on a remote host
 - name: Ensure latest version of all your Job Templates
```
[6a64db5062750b98728667a5ef67b091d8ab6883](https://github.com/theforeman/foreman-ansible-modules/commit/6a64db5062750b98728667a5ef67b091d8ab6883)

---

```diff
clarify that we need the *name* of the content source, not URL

--- plugins/doc_fragments/foreman.py
+++ plugins/doc_fragments/foreman.py
@@ -256,7 +256,7 @@ options:
     type: str
   content_source:
     description:
-      - Content source.
+      - Content Source (Smart Proxy with Content) name.
       - Only available for Katello installations.
     required: false
     type: str
```

[b00b247e674ee214703bf9a68880e51a29fb1c63](https://github.com/theforeman/foreman-ansible-modules/commit/b00b247e674ee214703bf9a68880e51a29fb1c63)

---

# Excercise 2: new parameters for existing modules

---

```patch
--- plugins/modules/content_view.py
+++ plugins/modules/content_view.py
@@ -68,6 +68,11 @@ options:
     description:
       - Solve RPM dependencies by default on Content View publish
     type: bool
+  import_only:
+    description:
+      - Designate this Content View for importing from upstream servers only.
+    type: bool
+    version_added: 3.14.0
   composite:
     description:
       - A composite view contains other content views.
```

[6cc6283ed3c4a9b51c01a43210ac6809ebaf4bba](https://github.com/theforeman/foreman-ansible-modules/commit/6cc6283ed3c4a9b51c01a43210ac6809ebaf4bba)

---

```patch
--- plugins/modules/content_view.py
+++ plugins/modules/content_view.py
@@ -167,6 +172,7 @@ def main():
             composite=dict(type='bool', default=False),
             auto_publish=dict(type='bool', default=False),
             solve_dependencies=dict(type='bool'),
+            import_only=dict(type='bool'),
             components=dict(type='nested_list', foreman_spec=cvc_foreman_spec, resolve=False),
             repositories=dict(type='entity_list', elements='dict', resolve=False, options=dict(
                 name=dict(required=True),
```

[6cc6283ed3c4a9b51c01a43210ac6809ebaf4bba](https://github.com/theforeman/foreman-ansible-modules/commit/6cc6283ed3c4a9b51c01a43210ac6809ebaf4bba)

---

# Excercise 3: new modules

---

## `organization`

```python
class ForemanOrganizationModule(ForemanEntityAnsibleModule):
    pass

def main():
    module = ForemanOrganizationModule(
        foreman_spec=dict(
            name=dict(required=True),
        ),
    )
    with module.api_connection():
        module.run()

if __name__ == '__main__':
    main()
```

---

* The [reality](https://github.com/theforeman/foreman-ansible-modules/blob/develop/plugins/modules/organization.py) is slightly more complicated
* Documentation needs to be static strings (`ansible-doc` requirement)
* Many modules have slightly more complex logic in the `run()` part

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
