# Generating Ansible modules for REST APIs without AI

---

## `$ whoami`

Matthias Dellweg

Senior Software Engineer at Red Hat

`pulp.squeezer` - 39 modules, one API

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

`theforeman.foreman` - 82 modules, one API

---

# Motivation

* Writing Ansible modules is a tedious job
* It's a *repetitive* job when you talk to an API
* Computers are great at repeating!

Note:
* Repetitive because:
  * set of common parameters
  * same HTTP requests
  * API endpoints probably look very similar

---

## ChatGPT?

* [We actually tried!](https://chat.openai.com/share/01735ba1-6711-4d1e-9f24-998b99ebd141)
* It gets the general idea (`requests`, CRUD operations, credentials)
* Even some API interactions were working
* But we couldn't convince it to use any of the existing abstractions (`apypie`, `theforeman.foreman`)
* Confused it to use `cURL` in the end
* Also no docs, no tests

Note:
* We're no Prompt Engineers, we're YAML Engineers after all!
* Inefficient: pull all entries from the API, filter locally
* No Pagination (so actually not *all* entries)
* Hard coded API endpoints in the module

---

## Ansible Lightspeed?

* Can only generate playbooks, not modules

Note:
* We didn't test Lightspeed at all after hearing this
* Maybe Greg can enlighten (lol) us?

---

## Cookiecutter?

* "Cookiecutter is an open source library for building coding *project* templates"
* But we're looking for creating things *inside* a project
* We also need more input than "name" and "API endpoint"

Note:
* You can pass arbitrary data into cookiecutter somehow, but how will that be processed?

---

# API documentation

---

## API documentation

* So far, nothing has considered the existence of API documentation
* [OpenAPI (Swagger)](https://www.openapis.org/) is quite common
* Foreman uses [Apipie](https://github.com/Apipie/apipie-rails)
* Will usually contain parameter names, types, `required`, doc strings

Note:
* Apipie can export OpenAPI/Swagger too
* This *should* be sufficient to generate a module!

---

## API documentation (OpenAPI)

```json
"operationId": "post_webhooks",
"summary": "Create a Webhook",
"parameters": [
  {
    "name": "webhook[name]",
    "type": "string",
    "in": "formData",
    "required": true
  },
  {
    "name": "webhook[target_url]",
    "type": "string",
    "in": "formData",
    "required": true
  },
```

---

## API documentation (apipie)

```json
"api_url": "/api/webhooks",
"http_method": "POST",
"short_description": "Create a Webhook",
"params": [
    {
        "name": "name",
        "required": true,
        "expected_type": "string"
    },
    {
        "name": "target_url",
        "required": true,
        "expected_type": "string"
    }
```

---

## API documentation

this data can be consumed by code

```python
api = Api.connect(url: str, username: str, passwdword: str)
api.webhooks.create(name: str, target_url: str)
```

and this will know which URL to send the final payload to and how to construct that payload

---

## API documentation

* we can repeat this with every entity in our API
* if only there was a way to enumerate things and render similar content repeatedly…

---

# Templates

---

## Templates

* We can use API docs and some code to fill in a template!
* Python and Jinja2 are an obvious choice

---

## Templates

* Great to provide the boilerplate
  * module using common `module_utils`
  * `DOCUMENTATION`
* Needs manual work for the details anyway
  * `EXAMPLES`
  * tests
  * custom logic
* Reapplying after modifications is complicated

Note:
* API often needs "more" data to create than to update
* Lookup and Create/Update use different names/formats (bad API!)
* Reapplying requires parsing diffs, or writing code that applies "special-cases" after templating

---

## Templates for Foreman

* [apinsible](https://github.com/evgeni/apinsible)
* use Apipie JSON
* takes the URL to a Foreman server and the resource name
* can generate "normal" and "info" modules
* uses existing abstractions from `theforeman.foreman`

Note:
* `theforeman.foreman` has many (subtly) different classes, `apinsible` only knows two of them
* "some assembly required"
* you still gotta write examples and tests
* used as a start, edited by brains later on and maintained

---

## Templates for AWS and VMware

* [ansible.content_builder](https://github.com/ansible-community/ansible.content_builder)
* uses OpenAPI JSON
* takes the JSON and a YAML that describes what modules to generate
* can generate "normal" and "info" modules

Note:
* Knows a lot more Ansible details
* also generates sanity ignore files etc - a full collection actually
* knows details about specific modules, and can re-generate them
* can also generate other types of content, but we didn't look into that

---

## Templates for everything

* Can we make this more generic?
* If we want target-specific `module_utils`, the templates will also be target-specific
* And so will be the code filling out the template
* `ansible.content_builder` can be extended with new targets

Note:
* We absolutely want `module_utils`, otherwise each module has the same HTTP/parsing code!

---

# Maintainability

---

## Maintainability

* pure template regeneration will overwrite any changes humans made
* keeping the code close to the template allows for smaller diffs and easier re-application
* templates help to keep code base consistent
* you can teach your code to alter the result

Note:
* consistency is good for refactoring and mental well-being

---

## Code transformations

Minimum Python version upgrade:
```
sed -e "s/iteritems/items/g" *.py
```

Ansible 2.8 "collection" (library) support:
```
sed -i 's/ansible.module_utils/ansible_collections.theforeman\
  .foreman.plugins.module_utils/g' \
  $(COLLECTION_TMP)/plugins/modules/*.py

sed -i -e '/extends_documentation_fragment/ \
  {:1 n; s/- foreman/- theforeman.foreman.foreman/; t1}' \
  $(COLLECTION_TMP)/plugins/modules/*.py
```

---

# Conclusion

* You need a few working modules first
* Then you extract the common parts
* And template the other 42 modules
* On the way there you created a framework for creating modules

Note:
* You don't *have* to extract common parts, you can template them and re-generate things from the template if needed
* In reality you want at least some parts extracted, but it's up to your workflows to decide how many
* `module_utils` and `doc_fragments` are made to create a framework.

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-envelope" aria-hidden="true"></i> [x9c4@redhat.com](mailto:x9c4@redhat.com)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@x9c4@fosstodon.org](https://fosstodon.org/@x9c4)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)

<i class="fa fa-github" aria-hidden="true"></i> [@mdellweg](https://github.com/mdellweg)
