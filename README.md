As of 2.0 version of `django-haystack`, the compatibility layer with Solr backend lacks support for the newest 5.* version of Solr out of the box. But the two can still happily work together with a few bits of extra configuration. Here you can find steps for doing just that.

You can skip the first step if you already have your Solr-as-a-**service** installed.

### 1. Install Solr as a service

* Install Java if needed.
* Download the `tgz` file from [Solr Releases](http://www.us.apache.org/dist/lucene/solr/).
* Extract the installation script from the archive and run it:

```sh
tar xzf solr-<version>.tgz solr-<version>/bin/install_solr_service.sh --strip-components=2
sudo bash ./install_solr_service.sh solr-<version>.tgz
```

* When installed as a service, the Solr server will start automatically on every system boot.

### 2. Create a non-schemaless Solr core

* `django-haystack` currently only supports Solr indexes based on a schema, which is stored in `schema.xml` file of a Solr core, so create a core from the non-schemaless configuration called "basic_configs" and predefined in the Solr's installation:

```sh
sudo su - solr -c '/opt/solr/bin/solr create -c <core_name> -d basic_configs'
```

* In the command above, core creation needs to be performed on behalf of `solr` user because Solr would run into permission problems otherwise.

### 3. Override `django-haystack` default template for `schema.xml`

* Grab `solr.xml` file from this repository and put it into your Django project where `django-haystack` will be able to locate the template, such as:

```
<project_name>/templates/search_configuration/solr.xml
```

* `solr.xml` is how `django-haystack` likes to call `schema.xml` templates.
* You can see the `django-haystack`-specific config near the top of the template, so if you ever need to use *another* initial template, make sure to remove from it the two declarations for `<field name="id" ...` and for `<uniqueKey>` as these will be declared by the `django-haystack`-specific config and copy the `django-haystack`-specific config into the same spot in your own template.

### 4. Fine-tune `django-haystack` settings

* Modify the settings for `django-haystack` in your Django settings to make it talk to a specific core:

```python
HAYSTACK_CONNECTIONS = {
    "default": {
        "ENGINE": "haystack.backends.solr_backend.SolrEngine",
        "URL": "http://127.0.0.1:8983/solr/<core_name>"
    },
    # ... your other settings ...
}
```

### 5. Make it easy to build Solr schema

* There is no need for manual `schema.xml` copying and Solr restarting whenever you rebuild your schema for `django-haystack` as the schema can be put directly into the core's config and the core can then be reloaded all with a single command:

```sh
python manage.py build_solr_schema --filename=/var/solr/data/<core_name>/conf/schema.xml && curl 'http://localhost:8983/solr/admin/cores?action=RELOAD&core=<core_name>&wt=json&indent=true'
```

* As before, change `<core_name>` to your Solr core's name.
* You will likely need to change the permissions on `/var/solr/data/<core_name>` to more liberal ones for the above line to execute.

### 6. Fix `pysolr.py` if needed

* If you are in 2015, there is a tiny bug in `pysolr` that still isn't fixed, so if you are getting an exception from `pysolr.py` about an argument in `startswith` not being a binary string then, assuming your are running Django in its own virtual environment as you should, you would need to open

```
<virtual_environment_dir>/lib/python<python_version>/site-packages/pysolr.py
```

and change

```
if response.startswith('<?xml'):
```

on line 445 to

```
if response.startswith(b'<?xml'):
```

or better yet, make a pull request.
