> See also: [Secret events](/t/7191)

[note type=information]
This feature is available starting with `ops` 2.0.0, but only when using Juju `3.0.2` or greater.
[/note]

As of version 3.0, Juju supports **secrets**. Secrets allow charms to securely exchange data via a channel mediated by the Juju controller. In this document we will guide you through how to use the new secrets API as exposed by the Charmed Operator Framework `ops`. 

We are going to assume:
- Surface knowledge of the [`ops`](/t/5527) library
- Familiarity with secrets terminology and concepts (see [Secret events](/t/7191))

**Contents:**
- [Setup](#heading--setup)
- [Owner: Add a secret](#heading--owner-add-a-secret)
- [Observer: Get a secret](#heading--observer-get-a-secret)
- [Manage secret revisions](#heading--manage-secret-revisions)
    - [Owner: Create a new revision](#heading--owner-create-a-new-revision)
    - [Observer: Update to a new revision](#heading--observer-update-to-a-new-revision)
        - [Peek at a secret’s payload](#heading--peek-at-a-secrets-payload)
    - [Label secrets](#heading--label-secrets)
        - [When to use labels](#heading--when-to-use-labels)
    - [Owner: Manage rotation](#heading--owner-manage-rotation)
- [Manage a secret's end of life](#heading--manage-a-secret's-end-of-life)
    - [Remove a secret](#heading--remove-a-secret)
    - [Remove a revision](#heading--remove-a-revision)
    - [Revoke a secret](#heading--revoke-a-secret)
- [Conclusion](#heading--conclusion)


<a href="#heading--setup"><h2 id="heading--setup">Setup</h2></a>

Let us assume that you have two charms that need to exchange a secret. For example, a web server and a database. To access the database, the web server needs a username/password pair.
Without secrets, charmers were forced to exchange the credentials in clear text via relation data, or manually add a layer of (presumably public key) encryption to avoid broadcasting that data (because, remember, anyone with the Juju CLI and access to the controller can run `juju show-unit`).

In secrets terminology, we call the database the **owner** of a secret, and the web server its **observer**.

Starting from the owner, let's see what changes are necessary to the codebase to switch from using plain-text relation data to one backed by Juju secrets.


<a href="#heading--owner-add-a-secret"><h2 id="heading--owner-add-a-secret">Owner: Add a secret</h2></a>

Presumably, the owner code (before using secrets) looked something like this:

```python
class MyDatabaseCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.database_relation_joined, 
                               self._on_database_relation_joined)

    ...  # other methods and event handlers
   
    def _on_database_relation_joined(self, event: ops.RelationJoinedEvent):
        event.relation.data[self.app]['username'] = 'admin' 
        event.relation.data[self.app]['password'] = 'admin'  # don't do this at home   
```

Using the new secrets API, this can be rewritten as:

```python
class MyDatabaseCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.database_relation_joined,
                               self._on_database_relation_joined)

    ...  # other methods and event handlers

    def _on_database_relation_joined(self, event: ops.RelationJoinedEvent):
        content = {
            'username': 'admin',
            'password': 'admin',
        }
        secret = self.app.add_secret(content)
        secret.grant(event.relation)
        event.relation.data[self.app]['secret-id'] = secret.id
```

Note that:
- We call `add_secret` on `self.app` (the application). That is because we want the secret to be owned by this application, not by this unit. If we wanted to create a secret owned by the unit, we'd call `self.unit.add_secret` instead.
- The only data shared in plain text is the secret ID (a locator URI). The secret ID can be publicly shared. Juju will ensure that only remote apps/units to which the secret has explicitly been granted by the owner will be able to fetch the actual secret payload from that ID.
- The secret needs to be granted to a remote entity (app or unit), and that always goes via a relation instance. By passing a relation to `grant` (in this case the event's relation), we are explicitly declaring the scope of the secret -- its lifetime will be bound to that of this relation instance.


<a href="#heading--observer-get-a-secret"><h2 id="heading--observer-get-a-secret">Observer: Get a secret</h2></a>

Before secrets, the code in the secret-observer charm may have looked something like this:

```python
class MyWebserverCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.database_relation_changed,
                               self._on_database_relation_changed)

    ...  # other methods and event handlers

    def _on_database_relation_changed(self, event: ops.RelationChangedEvent):
        username = event.relation.data[event.app]['username']
        password = event.relation.data[event.app]['password']
        self._configure_db_credentials(username, password)
```

With the secrets API, the code would become:

```python
class MyWebserverCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.database_relation_changed,
                               self._on_database_relation_changed)

    ...  # other methods and event handlers

    def _on_database_relation_changed(self, event: ops.RelationChangedEvent):
        secret_id = event.relation.data[event.app]['secret-id']
        secret = self.model.get_secret(id=secret_id)
        content = secret.get_content()
        self._configure_db_credentials(content['username'], content['password'])
```

Note that:
- The observer charm gets a secret via the model (not its app/unit). Because it's the owner who decides who the secret is granted to, the ownership of a secret is not an observer concern. The observer code can rightfully assume that, so long as a secret ID is  shared with it, the owner has taken care to grant and scope the secret in such a way that the observer has the rights to inspect its contents.
- The charm first gets the secret object from the model, then gets the secret's content (a dict) and accesses individual attributes via the dict's items.


<a href="#heading--manage-secret-revisions"><h2 id="heading--manage-secret-revisions">Manage secret revisions</h2></a>

If your application never needs to rotate secrets, then this would be enough. However, typically you want to rotate a secret periodically to contain the damage from a leak, or to avoid giving hackers too much time to break the encryption.

Creating new secret revisions is an owner concern. First we will look at how to create a new revision (regardless of when a charm decides to do so) and how to observe it. Then, we will look at the built-in mechanisms to facilitate periodic secret rotation.

<a href="#heading--owner-create-a-new-revision"><h3 id="heading--owner-create-a-new-revision">Owner: Create a new revision</h3></a>

To create a new revision, the owner charm must call `secret.set_content` and pass in the new payload:

```python
class MyDatabaseCharm(ops.CharmBase):

    ... # as before

    def _rotate_webserver_secret(self, secret):
        content = secret.get_content()
        secret.set_content({
            'username': content['username'],              # keep the same username
            'password': _generate_new_secure_password(),  # something stronger than 'admin'
        })
```

This will inform Juju that a new revision is available, and Juju will inform all observers tracking older revisions that a new one is available, by means of a `secret-changed` hook.


<a href="#heading--observer-update-to-a-new-revision"><h3 id="heading--observer-update-to-a-new-revision">
Observer: Update to a new revision</h3></a>

To update to a new revision, the web server charm will typically subscribe to the `secret-changed` event and call `get_content` with the "refresh" argument set (refresh asks Juju to start tracking the latest revision for this observer).

```python
class MyWebserverCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.secret_changed,
                               self._on_secret_changed)

    ...  # as before

    def _on_secret_changed(self, event: ops.SecretChangedEvent):
        content = event.secret.get_content(refresh=True)
        self._configure_db_credentials(content['username'], content['password'])
```

<a href="#heading--peek-at-a-secrets-payload"><h4 id="heading--peek-at-a-secrets-payload">Peek at a secret’s payload</h4></a>

Sometimes, before reconfiguring to use a new credential revision, the observer charm may want to peek at its contents (for example, to ensure that they are valid). Use `peek_content` for that:

```python
    def _on_secret_changed(self, event: ops.SecretChangedEvent):
        content = event.secret.peek_content()
        if not self._valid_password(content.get('password')):
           logger.warning('Invalid credentials! Not updating to new revision.')
           return
        content = event.secret.get_content(refresh=True)
        ...
```

<a href="#heading--label-secrets"><h3 id="heading--label-secrets">Label secrets</h3></a>


Sometimes a charm will observe multiple secrets. In the `secret-changed` event handler above, you might ask yourself: How do I know which secret has changed?
The answer lies with **secret labels**: a label is a charm-local name that you can assign to a secret. Let's go through the following code:

```python
class MyWebserverCharm(ops.CharmBase):

    ...  # as before

    def _on_database_relation_changed(self, event: ops.RelationChangedEvent):
        secret_id = event.relation.data[event.app]['secret-id']
        secret = self.model.get_secret(id=secret_id, label='database-secret')
        content = secret.get_content()
        self._configure_db_credentials(content['username'], content['password'])

    def _on_secret_changed(self, event: ops.SecretChangedEvent):
        if event.secret.label == 'database-secret':
            content = event.secret.get_content(refresh=True)
            self._configure_db_credentials(content['username'], content['password'])
        elif event.secret.label == 'my-other-secret':
            self._handle_other_secret_changed(event.secret)
        else:
            pass  # ignore other labels (or log a warning)
```

As shown above, when the web server charm calls `get_secret` it can specify an observer-specific label for that secret; Juju will attach this label to the secret at that point. Normally `get_secret` is called for the first time in a relation-changed event; the label is applied then, and subsequently used in a secret-changed event.

Labels are unique to the charm (the observer in this case): if you attempt to attach a label to two different secrets from the same application (whether it's the on the observer side or the owner side) and give them the same label, the framework will raise a `ModelError`.

Whenever a charm receives an event concerning a secret for which it has set a label, the label will be present on the secret object exposed by the framework.

The owner of the secret can do the same. When a secret is added, you can specify a label for the newly-created secret:

```python
class MyDatabaseCharm(ops.CharmBase):

    ...  # as before

    def _on_database_relation_joined(self, event: ops.RelationJoinedEvent):
        content = {
            'username': 'admin',
            'password': 'admin',
        }
        secret = self.app.add_secret(content, label='secret-for-webserver-app')
        secret.grant(event.relation)
        event.relation.data[event.unit]['secret-id'] = secret.id
```

If a secret has been labelled in this way, the charm can retrieve the secret object at any time by calling `get_secret` with the "label" argument. This way, a charm can perform any secret management operation even if all it knows is the label. The secret ID is normally only used to exchange a reference to the secret *between* applications. Within a single application, all you need is the secret label.

So, having labelled the secret on creation, the database charm could add a new revision as follows:

```python
    def _rotate_webserver_secret(self):
        secret = self.model.get_secret(label='secret-for-webserver-app')
        secret.set_content(...)  # pass a new revision payload, as before
```

<a href="#heading--when-to-use-labels"><h4 id="heading--when-to-use-labels">When to use labels</h4></a>

When should you use labels? A label is basically the secret's *name* (local to the charm), so whenever a charm has, or is observing, multiple secrets you should label them. This allows you to distinguish between secrets, for example, in the `SecretChangedEvent` shown above.

Most charms that use secrets have a fixed number of secrets each with a specific meaning, so the charm author should give them meaningful labels like `database-credential`, `tls-cert`, and so on. Think of these as "pets" with names.

In rare cases, however, a charm will have a set of secrets all with the same meaning: for example, a set of TLS certificates that are all equally valid. In this case it doesn't make sense to label them -- think of them as "cattle". To distinguish between secrets of this kind, you can use the [`Secret.unique_identifier`](https://ops.readthedocs.io/en/latest/#ops.Secret.unique_identifier) property, added in ops 2.6.0.

Note that [`Secret.id`](https://ops.readthedocs.io/en/latest/#ops.Secret.id), despite the name, is not really a unique ID, but a locator URI. We call this the "secret ID" throughout Juju and in the original secrets specification -- it probably should have been called "uri", but the name stuck.


<a href="#heading--owner-manage-rotation"><h3 id="heading--owner-manage-rotation">Owner: Manage rotation</h3></a>

We have seen how an owner can create a new revision and how an observer can update to it (or peek at it). What remains to be seen is: when does the owner decide to rotate a secret?

Juju exposes two separate mechanisms for owner charms to rotate a secret: `rotation` and `expiration`. The observer-side mechanism for peeking at or updating to new revisions does not change, so this section will only discuss owner code.

A charm can configure a secret, at creation time, to have one or both of:

- A rotation policy (weekly, monthly, daily, and so on).
- An expiration date (for example, in two months from now).

Here is what the code would look like:

```python
class MyDatabaseCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.secret_rotate,
                               self._on_secret_rotate)

    ...  # as before

    def _on_database_relation_joined(self, event: ops.RelationJoinedEvent):
        content = {
            'username': 'admin',
            'password': 'admin',
        }
        secret = self.app.add_secret(content,
            label='secret-for-webserver-app',
            rotate=SecretRotate.DAILY)

    def _on_secret_rotate(self, event: ops.SecretRotateEvent):
        # this will be called once per day.
        if event.secret.label == 'secret-for-webserver-app':
            self._rotate_webserver_secret(event.secret)
```

Or, for secret expiration:

```python
class MyDatabaseCharm(ops.CharmBase):
    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.secret_expired,
                               self._on_secret_expired)

    ...  # as before

    def _on_database_relation_joined(self, event: ops.RelationJoinedEvent):
        content = {
            'username': 'admin',
            'password': 'admin',
        }
        secret = self.app.add_secret(content,
            label='secret-for-webserver-app',
            expire=datetime.timedelta(days=42))  # this can also be an absolute datetime

    def _on_secret_expired(self, event: ops.SecretExpiredEvent):
        # this will be called only once, 42 days after the relation-joined event.
        if event.secret.label == 'secret-for-webserver-app':
            self._rotate_webserver_secret(event.secret)
```

<a href="#heading--manage-a-secret's-end-of-life"><h2 id="heading--manage-a-secret's-end-of-life">Manage a secret's end of life</h2></a>

No matter how well you keep them, secrets aren't forever.
If the relation holding the two charms together is removed, the owner charm might want to clean things up and **remove** the secret as well (the observer won't be able to access it anyway).

Also, suppose that the owner charm has some config variable that determines who is to be granted access to the db. If that were to change after the charm already has granted access to some remote entity, the database charm will need to **revoke** access.

Finally, if the owner rotates the charm and all observers update to track the new (latest) revision, the old revision will become dead wood and can be removed.

We show you how to handle all these scenarios below.

<a href="#heading--remove-a-secret"><h3 id="heading--remove-a-secret">Remove a secret</h3></a>

To remove a secret (effectively destroying it for good), the owner needs to call `secret.remove_all_revisions`. Regardless of the logic leading to the decision of when to remove a secret, the code will look like some variation of the following:

```python
class MyDatabaseCharm(ops.CharmBase):
    ...

    # called from an event handler
    def _remove_webserver_secret(self):
        secret = self.model.get_secret(label='secret-for-webserver-app')
        secret.remove_all_revisions()
```

After this is called, the observer charm will get a `ModelError` whenever it attempts to get the secret. In general, the presumption is that the observer charm will take the absence of the relation as indication that the secret is gone as well, and so will not attempt to get it.


<a href="#heading--removing-a-revision"><h3 id="heading--removing-a-revision">Removing a revision</h3></a>

Removing a single secret revision is a more common (and less drastic!) operation than removing all revisions.

Typically, the owner will remove a secret revision when it receives a `secret-remove` event -- that is, when that specific revision is no longer tracked by any observer. If a secret owner did remove a revision while it was still being tracked by observers, they would get a `ModelError` when they tried to get the secret.

A typical implementation of the `secret-remove` event would look like:

```python
class MyDatabaseCharm(ops.CharmBase):

    ...  # as before

    def __init__(self, *args, **kwargs):
        ...  # other setup
        self.framework.observe(self.on.secret_remove,
                               self._on_secret_remove)

    def _on_secret_remove(self, event: ops.SecretRemoveEvent):
        # all observers are done with this revision, remove it
        event.secret.remove_revision(event.revision)
```


<a href="#heading--revoke-a-secret"><h3 id="heading--revoke-a-secret">Revoke a secret</h3></a>

For whatever reason, the owner of a secret can decide to revoke access to the secret to a remote entity. That is done by calling `secret.revoke`, and is the inverse of `secret.grant`.

An example of usage might look like:
```python
class MyDatabaseCharm(ops.CharmBase):

    ...  # as before

    # called from an event handler
    def _revoke_webserver_secret_access(self, relation):
        secret = self.model.get_secret(label='secret-for-webserver-app')
        secret.revoke(relation)
```

Just like when the owner granted the secret, we need to pass a relation to the `revoke` call, making it clear what scope this action is to be applied to.


<a href="#heading--conclusion"><h2 id="heading--conclusion">Conclusion</h2></a>

In this guide we have taken a tour of the new `ops` secrets API. Hopefully you will find that the API is intuitive and provides a clear wrapper around the new `juju` hook tools. If you find bugs, have suggestions, or want to contribute to the discussion, you can use [the tracker on github](https://github.com/canonical/operator/issues).

Of course, we have also added facilities in the testing Harness to help charmers test their secrets. This is however a matter for another document.