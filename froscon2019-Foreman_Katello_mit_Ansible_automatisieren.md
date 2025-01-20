<!--
date: 2019-08-11
link: https://programm.froscon.org/2019/events/2418.html
-->
# Foreman/Katello mit Ansible automatisieren

---

## `$ whoami`

Evgeni Golov

Senior Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian and Grml Developer

♥ FOSS ♥

♥ automation ♥

---

## Agenda

* Motivation / WTF
* Warum nicht `X`?!
* Foreman Ansible Modules
* Workflow Beispiele
* Selber Module schreiben!

---

# Motivation / WTF

---

## Was ist Foreman?

* Tool zur Verwaltung von physikalischen und virtuellen Servern
* Power Management, Installation, Konfiguration
* Bare-Metal, VMware, RHV, OpenStack, GCE, Azure, etc
* Erweiterbar durch Plugins (zB Katello, Monitoring, Ansible)

---

## Was ist Katello?

* Plugin für Foreman
* Content Management (RPM, DEB, Puppet, Containers, Files)
* Content kann grupiert und gefiltert an Clients ausgeliefert werden
* Erlaubt Snapshots von Content zur Versionierung
* Patch Management

---

## Was ist Ansible?

* "radically simple IT automation engine"
* bringt eine enorme Zahl an Modulen für unterschiedliche Einsatzzwecke mit
* kann leicht durch eigene Module erweitert werden
* lässt sich gut mit REST APIs integrieren

---

## Warum automatisieren?

* Jeder kann mitmachen
* Peer Review der Änderungen
* Rollback bei Problemen
* Reproducibility
* Daten lesbar speichern und versionieren

Note:
* PR ist einfacher als in der UI rumklicken

---

## Wie automatisieren?

### Foreman hat eine WebUI!

* Kein Review der Änderungen möglich
* Zurückspringen zu älteren Einstellungen aufwändig
* Reproducibility ist eher nicht gegeben

---

## Wie automatisieren?

### Foreman hat eine CLI!

* Ich hab da mal schnell was mit `sed` und `awk` gebaut. Nein!
* Kann auch CSV/JSON Output und `jq`/`jo` sind toll, aber immer noch nein!
* Ansible `command`/`shell` Module <span class="emoji">🙄</span>

---

## Wie automatisieren?

### Foreman hat eine API!

* <span class="emoji">😻</span>
* Daten (Zustand) in einer Datenstruktur (YAML)
* Ein API Client übernimmt die Arbeit
* API Client selber schreiben? Später!
* Ansible!

---

# Warum nicht `X`?!

---

## `foreman` und `katello` Module

* Ansible Upstream seit 2.3 (2016)
* Deprecated seit 2.8
* Wird in 2.12 entfernt
* Ein Modul für alles, dadurch kompliziert zu bedienen
* Benutzt die `nailgun` Bibliothek

Note:
* `nailgun` ist eigentlich für Satellite geschrieben
* ersions spezifisch
* hat Probleme mit non-Katello Installationen

---

```yaml=
- name: Create Organization
  foreman:
    username: admin
    password: admin
    server_url: https://foreman.example.com
    entity: organization
    params:
      name: My Cool New Organization
```

---

```yaml=
- name: Enable RHEL Product
  katello:
    username: admin
    password: admin
    server_url: https://katello.example.com
    entity: repository_set
    params:
      name: Red Hat Enterprise Linux 7 Server (RPMs)
      product: Red Hat Enterprise Linux Server
      organization: Default Organization
      basearch: x86_64
      releasever: 7Server
```

Note:
* Man muss sich `entity` merken
* Die Module sind schwierig zu dokumentieren

---

## `ansible-module-foreman`

* Seit 2015 gibt es auch [`ansible-module-foreman`](https://github.com/Nosmoht/ansible-module-foreman) von Thomas Krahn ([@Nosmoht](https://github.com/Nosmoht))
* Gut gepflegt
* Benutzt eine eigene Bibliothek um mit der API zu sprechen
  * Benutzt nicht Foremans `apidoc.json`
  * Braucht Anpassungen für Plugins
* Aktuell kein Katello Support

---

# Foreman Ansible Modules

---

## Foreman Ansible Modules

* Seit Juni 2017
* Versucht `foreman`/`katello` aufzuräumen
* Zunächst durch Aufsplittung in einzelne Module pro Objekt
* Dann durch Aufbau eines Frameworks um Module schlank zu halten
* Tests!

---

## Foreman Ansible Modules

* Die Nutzung von `nailgun` wurde irgendwann anstrengend
  * Welche Version von `nailgun`?
  * Was ist mit Plugins?
  * Funktioniert nicht ohne Katello
* `apypie` Bibliothek als Ersatz für `nailgun`
  * Liest die `apidoc.json` von Foreman
* Durch existierendes Framework und Tests ist die Migration einfach

---

```yaml=
- name: "create example.org domain"
  foreman_domain:
    name: "example.org"
    description: "Example Domain"
    server_url: "https://foreman.example.com"
    username: "admin"
    password: "secret"
    state: present
```

---

```yaml=
- name: "Enable RHEL 7 RPMs repositories"
  katello_repository_set:
    username: "admin"
    password: "changeme"
    server_url: "https://foreman.example.com"
    name: "Red Hat Enterprise Linux 7 Server (RPMs)"
    organization: "Default Organization"
    product: "Red Hat Enterprise Linux Server"
    repositories:
    - releasever: "7Server"
      basearch: "x86_64"
    state: enabled
```

---

## Foreman Ansible Modules - Stats

* 43 <span class="emoji">🌟</span> auf GitHub
* 24 Contributors (8 Red Hat, 7 ATIX)
* 8 neue Contributors in 2019

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

* Bald auf Ansible Galaxy
* Bald als RPM auf yum.theforeman.org
* Beide Wege werden Ansible 2.8 benötigen (Ansible Collections)
* Module weiterhin Ansible 2.3+ kompatibel

---

# Workflow Beispiele

Note:
* Katello sync+publish+promote
* Katello LCE+AK
* Foreman cleanup

---

## Katello sync+publish+promote

```yaml=
- name: "Sync RHEL repositories"
  katello_sync:
    product: "Red Hat Enterprise Linux Server"

- name: "Publish RHEL content view"
  katello_content_view_version:
    content_view: "RHEL"

- name: "Promote RHEL content view to Test"
  katello_content_view_version:
    content_view: "RHEL"
    current_lifecycle_environment: "Library"
    lifecycle_environments:
      - Test
```

Note:
* `organization` fehlt
* Connection data fehlt

---

## Katello sync+publish+promote

```yaml=
- name: "Sync RHEL repositories"
  katello_sync:
    product: "Red Hat Enterprise Linux Server"

- name: "Publish and promote RHEL content view"
  katello_content_view_version:
    content_view: "RHEL"
    lifecycle_environments:
      - Library
      - Test
```

Note:
* same as before, just in one step

---

## Katello Lifecycle Environment + Activation Key

```yaml=
- katello_lifecycle_environment:
    name: "{{ lifecycle_env }}"
    prior: "Library"

- name: "Copy Activation Key"
  katello_activation_key:
    name: "{{ activation_key }}"
    new_name: "{{ activation_key }}-{{ lifecycle_env }}"
    state: 'copied'

- name: "Set Lifecycle Environment for Activation Key"
  katello_activation_key:
    name: "{{ activation_key }}-{{ lifecycle_env }}"
    lifecycle_environment: "{{ lifecycle_env }}"
```

Note:
* no name in first step to save space ;-)

---

## Foreman cleanup

```yaml=
- name: "Clean all media"
  foreman_installation_medium:
    name: "*"
    state: absent
- name: "Dissociate all Provisioning templates"
  foreman_provisioning_template:
    name: "*"
    organizations: []
    locations: []
- name: "Dissociate all Partition Table templates"
  foreman_ptable:
    name: "*"
    organizations: []
    locations: []
```

---

# Selber Module schreiben!

---

## Modul Aufbau

Die meisten Module sind dazu da Objekte in Foreman zu verwalten

1. Bereits vorhandenes Objekt suchen
2. Objekt mit den vom User gegebenen Daten vergleichen
3. Objekt speichern

Dafür gibt es ein Framework…

---

Wir haben einen Wrapper um `AnsibleModule`:

```python
from ansible.module_utils.foreman_helper import
  ForemanEntityApypieAnsibleModule

module = ForemanEntityApypieAnsibleModule(
 argument_spec=dict(name=dict(required=True)))
```

---

Parameter laden und API Verbindung testen:

```python
entity_dict = module.clean_params()
module.connect()
```

---

Bereits existierendes Objekt finden und es mit den per Parameter übergebenen Daten updaten:

```python
entity = module.find_resource_by_name('architectures',
  name=entity_dict['name'], failsafe=True)
changed = module.ensure_resource_state('architectures',
  entity_dict, entity, name_map)
module.exit_json(changed=changed)
```

Note:
* resource name is whatever is in the docs
* `name_map` ist hier noch nicht erklärt
* `entity_dict` sind die gefilterten User-Angaben
* `entity` das evtl gefundene Objekt

---

Ansible Parameter in Foreman API Parameter übersetzen:

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
