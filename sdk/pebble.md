<div data-theme-toc="true"> </div>

As mentioned in the [introduction](https://juju.is/docs/sdk), the recommended way to create charms for Kubernetes is using the sidecar pattern with the workload container running [Pebble](https://github.com/canonical/pebble).

Pebble is a lightweight, API-driven process supervisor designed for use with charms. If you specify the `containers` field in a charm's `metadata.yaml`, Juju will deploy the charm code in a sidecar container, with Pebble running as the workload container's `ENTRYPOINT`.

When the workload container starts up, Juju fires a [`PebbleReadyEvent`](https://ops.readthedocs.io/en/latest/#ops.charm.PebbleReadyEvent), which can be handled using [`Framework.observe`](https://ops.readthedocs.io/en/latest/#ops.framework.Framework.observe) as shown in [Framework Constructs under "Containers"](/docs/sdk/constructs#heading--containers). This gives the charm author access to `event.workload`, a [`Container`](https://ops.readthedocs.io/en/latest/#ops.model.Container) instance.

The `Container` class has methods to modify the Pebble configuration "plan", start and stop services, read and write files, and run commands. These methods use the Pebble API, which communicates from the charm container to the workload container using HTTP over a Unix domain socket.

The rest of this document provides details of how a charm interacts with the workload container via Pebble, using the Python Operator Framework [`Container`](https://ops.readthedocs.io/en/latest/#ops.model.Container) methods.

<h2 id="heading--service-management">Service management and status</h2>

The main purpose of Pebble is to control and monitor services, which are usually long-running processes like web servers and databases.

In the context of Juju sidecar charms, Pebble is run with the `--hold` argument, which prevents it from automatically starting the services marked with `startup: enabled`. This is to give the charm full control over when the services in Pebble's configuration are actually started.

<h3 id="heading--autostart">Autostart</h3>

To start all the services that are marked as `startup: enabled` in the configuration plan, call [`Container.autostart`](https://ops.readthedocs.io/en/latest/#ops.model.Container.autostart). For example (taken from the [snappass-test](https://github.com/benhoyt/snappass-test/blob/master/src/charm.py) charm):

```python
class SnappassTestCharm(CharmBase):
    ...

    def _start_snappass(self):
        container = self.unit.containers["snappass"]
        snappass_layer = {
            "services": {
                "snappass": {
                    "override": "replace",
                    "summary": "snappass service",
                    "command": "snappass",
                    "startup": "enabled",  # enables "autostart"
                }
            },
        }
        container.add_layer("snappass", snappass_layer, combine=True)
        container.autostart()
        self.unit.status = ActiveStatus()
```

<h3 id="heading--check-container-health">Checking Container Health</h3>

The Charmed Operator Framework provides a way to ensure that your container is healthy. In the [Container](https://ops.readthedocs.io/en/latest/#ops.model.Container) class, `Container.can_connect()` can be used in a conditional statement as a guard around your code to ensure that Pebble is operational.

This provides a convenient pattern for ensuring that Pebble is ready, and obviates the need to include `try`/`except` statements around Pebble operations in every hook to account for a hook being called when your Juju unit is being started, stopped or removed: cases where the Pebble API is more likely to be still coming up, or being shutdown. It can also be used on startup to check whether Pebble has started or not outside of the `pebble_ready` hook.

`Container.can_connect()` will catch and log `pebble.ConnectionError`, `pebble.APIError`, and `FileNotFoundError` (in case the Pebble socket has disappeared as part of Charm removal). Other Pebble errors or exceptions should be handled as normal.

<h3 id="heading--start-stop">Start and stop</h3>

To start (or stop) one or more services by name, use the [`start`](https://ops.readthedocs.io/en/latest/#ops.model.Container.start) and [`stop`](https://ops.readthedocs.io/en/latest/#ops.model.Container.stop) methods. Here's an example of how you might stop and start a database service during a backup action:

```python
class MyCharm(CharmBase):
    ...

    def _on_pebble_ready(self, event):
        container = event.workload
        container.start('mysql')

    def _on_backup_action(self, event):
        container = self.unit.get_container('main')
        if container.can_connect():
            try:
                container.stop('mysql')
                do_mysql_backup()
                container.start('mysql')
            except pebble.ProtocolError, pebble.PathError:
                # handle Pebble errors
```

<h3 id="heading--get-services">Fetch service status</h3>

You can use the [`get_service`](https://ops.readthedocs.io/en/latest/#ops.model.Container.get_service) and [`get_services`](https://ops.readthedocs.io/en/latest/#ops.model.Container.get_services) methods to fetch the current status of one service or multiple services, respectively. The returned [`ServiceInfo`](https://ops.readthedocs.io/en/latest/#ops.pebble.ServiceInfo) objects provide a `status` attribute with various states, or you can use the [`ServiceInfo.is_running`](https://ops.readthedocs.io/en/latest/#ops.pebble.ServiceInfo.is_running) method.

Here is a modification to the start/stop example that checks whether the service is running before stopping it:

```python
class MyCharm(CharmBase):
    ...

    def _on_backup_action(self, event):
        container = self.unit.get_container('main')
        if container.can_connect():
            is_running = container.get_service('mysql').is_running()
            if is_running:
                container.stop('mysql')
            do_mysql_backup()
            if is_running:
                container.start('mysql')
```


<h2 id="heading--configuration">Pebble layer configuration</h2>

Pebble services are [configured by means of layers](https://github.com/canonical/pebble#layer-configuration), with higher layers adding to or overriding lower layers, forming the effective Pebble configuration, or "plan".

When a workload container is created and Pebble starts up, it looks in `/var/lib/pebble/default/layers` (if that exists) for configuration layers already present in the container image, such as `001-layer.yaml`. If there are existing layers there, that becomes the starting configuration, otherwise Pebble is happy to start with an empty configuration, meaning no services.

In the latter case, Pebble is configured dynamically via the API by adding layers at runtime.

<h3 id="heading--add-layer">Add a configuration layer</h3>

To add a configuration layer, call [`Container.add_layer`](https://ops.readthedocs.io/en/latest/#ops.model.Container.add_layer) with a label for the layer, and the layer's contents as a YAML string, Python dict, or [`pebble.Layer`](https://ops.readthedocs.io/en/latest/#ops.pebble.Layer) object.

You can see an example of `add_layer` under the ["Autostart" heading](#heading--autostart) above. The `combine=True` argument tells Pebble to combine the named layer into an existing layer of that name (or add a layer if none by that name exists). Using `combine=True` is common when dynamically adding layers.

Because `combine=True` combines the layer with an existing layer of the same name, it's normally used with `override: replace` in the YAML service configuration. This means replacing the entire service configuration with the fields in the new layer.

If you're adding a single layer *without* `combine=True` on top of an existing base layer, you may want to use `override: merge` in the service configuration. This will merge the fields specified with the service by that name in the base layer. [See an example of overriding a layer.](https://github.com/canonical/pebble#layer-override-example)

<h3 id="heading--get-plan">Fetch effective "plan"</h3>

Charm authors can also introspect the current plan using [`Container.get_plan`](https://ops.readthedocs.io/en/latest/#ops.model.Container.get_plan). It returns a [`pebble.Plan`](https://ops.readthedocs.io/en/latest/#ops.pebble.Plan) object whose `services` attribute maps service names to [`pebble.Service`](https://ops.readthedocs.io/en/latest/#ops.pebble.Service) instances.

Below is an example of how you might use `get_plan` to introspect the current configuration, and only add the layer with its services if they haven't been added already:

```python
class MyCharm(CharmBase):
    ...

    def _on_config_changed(self, event):
        container = self.unit.get_container("main")
        plan = container.get_plan()
        if not plan.services:
            layer = {"services": ...}
            container.add_layer("layer", layer)
            container.start("svc")
        ...
```

<h2 id="heading--files-api">The Files API</h2>

Pebble's files API allows charm authors to read and write files on the workload container. You can write files ("push"), read files ("pull"), list files in a directory, make directories, and delete files or directories.

<h3 id="heading--push">Push</h3>

Probably the most useful operation is [`Container.push`](https://ops.readthedocs.io/en/latest/#ops.model.Container.push), which allows you to write a file to the workload, for example, a PostgreSQL configuration file. You can use `push` as follows (note that this code would be inside a charm event handler):

```python
config = """
port = 7777
max_connections = 1000
"""
container.push('/etc/pg/postgresql.conf', config, make_dirs=True)
```

The `make_dirs=True` flag tells `push` to create the intermediate directories if they don't already exist (`/etc/pg` in this case).

There are many additional features, including the ability to send raw bytes (by providing a Python `bytes` object as the second argument) and write data from a file-like object. You can also specify permissions and the user and group for the file. See the [API documentation](https://ops.readthedocs.io/en/latest/#ops.model.Container.push) for details.

<h3 id="heading--pull">Pull</h3>

To read a file from the workload, use [`Container.pull`](https://ops.readthedocs.io/en/latest/#ops.model.Container.pull), which returns a file-like object that you can `read()`.

The files API doesn't currently support update, so to update a file you can use `pull` to perform a read-modify-write operation, for example:

```python
# Update port to 8888 and restart service
config = container.pull('/etc/pg/postgresql.conf').read()
if 'port =' not in config:
    config += '\nport = 8888\n'
container.push('/etc/pg/postgresql.conf', config)
container.stop('postgresql')
container.start('postgresql')
```

If you specify the keyword argument `encoding=None` on the `pull()` call, reads from the returned file-like object will return `bytes`. The default is `encoding='utf-8'`, which will decode the file's bytes from UTF-8 so that reads return a Python `str`.

<h3 id="heading--list-files">List files</h3>

To list the contents of a directory or return stat-like information about one or more files, use [`Container.list_files`](https://ops.readthedocs.io/en/latest/#ops.model.Container.list_files). It returns a list of [`pebble.FileInfo`](https://ops.readthedocs.io/en/latest/#ops.pebble.FileInfo) objects for each entry (file or directory) in the given path, optionally filtered by a glob pattern. For example:

```python
infos = container.list_files('/etc', pattern='*.conf')
total_size = sum(f.size for f in infos)
logger.info('total size of config files: %d', total_size)
names = set(f.name for f in infos)
if 'host.conf' not in names:
    raise Exception('This charm requires /etc/host.conf!')
```

If you want information about the directory itself (instead of its contents), call `list_files(path, itself=True)`.

<h3 id="heading--create-directory">Create directory</h3>

To create a directory, use [`Container.make_dir`](https://ops.readthedocs.io/en/latest/#ops.model.Container.make_dir). It takes an optional `make_parents=True` argument (like `mkdir -p`), as well as optional permissions and user/group arguments. Some examples:

```python
container.make_dir('/etc/pg', user='postgres', group='postgres')
container.make_dir('/some/other/nested/dir', make_parents=True)
```

<h3 id="heading--remove-path">Remove path</h3>

To delete a file or directory, use [`Container.remove_path`](https://ops.readthedocs.io/en/latest/#ops.model.Container.remove_path). If a directory is specified, it must be empty unless `recursive=True` is specified, in which case the entire directory tree is deleted, recursively (like `rm -r`). For example:

```python
# Delete Apache access log
container.remove_path('/var/log/apache/access.log')
# Blow away /tmp/mysubdir and all files under it
container.remove_path('/tmp/mysubdir', recursive=True)
```

<h2 id="heading--exec">Running commands (exec)</h2>

From Juju 2.9.17, Pebble includes an API for executing arbitrary commands on the workload container: the [`Container.exec`](https://ops.readthedocs.io/en/latest/#ops.model.Container.exec) method. It supports sending stdin to the process and receiving stdout and stderr, as well as more advanced options.

To run simple commands and receive their output, call `Container.exec` to start the command, and then use the returned [`Process`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecProcess) object's [`wait_output`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecProcess.wait_output) method to wait for it to finish and collect its output.

For example, to back up a PostgreSQL database, you might use `pg_dump`:

```python
process = container.exec(['pg_dump', 'mydb'], timeout=5*60)
sql, warnings = process.wait_output()
if warnings:
    for line in warnings.splitlines():
        logger.warning('pg_dump: %s', line.strip())
# do something with "sql"
```


<h3 id="heading--exec-error-handling">Error handling</h3>

The `exec` method raises a [`pebble.APIError`](https://ops.readthedocs.io/en/latest/#ops.pebble.APIError) if basic checks fail and the command can't be executed at all, for example, if the executable is not found.

The [`ExecProcess.wait`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecProcess.wait) and [`ExecProcess.wait_output`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecProcess.wait_output) methods raise [`pebble.ChangeError`](https://ops.readthedocs.io/en/latest/#ops.pebble.ChangeError) if there was an error starting or running the process, and [`pebble.ExecError`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecError) if the process exits with a non-zero exit code.

In the case where the process exits via a signal (such as SIGTERM or SIGKILL), the exit code will be 128 plus the signal number. SIGTERM's signal number is 15, so a process terminated via SIGTERM would give exit code 143 (128+15).

It's okay to let these exceptions bubble up: Juju will mark the hook as failed and re-run it automatically. However, if you want fine-grained control over error handling, you can catch the `ExecError` and inspect its attributes. For example:

```python
process = container.exec(['cat', '--bad-arg'])
try:
    stdout, _ = process.wait_output()
    print(stdout)
except pebble.ExecError as e:
    logger.error('Exited with code %d. Stderr:', e.exit_code)
    for line in e.stderr.splitlines():
        logger.error('    %s', line)
```

That will log something like this:

```text
Exited with code 1. Stderr:
    cat: unrecognized option '--bad-arg'
    Try 'cat --help' for more information.
```


<h3 id="heading--exec-command-options">Command options</h3>

The `Container.exec` method has various options (see [full API documentation](https://ops.readthedocs.io/en/latest/#ops.pebble.Client.exec)), including:

* `environment`: a dict of environment variables to pass to the process
* `working_dir`: working directory to run the command in
* `timeout`: command timeout in seconds
* `user_id`, `user`, `group_id`, `group`: UID/username and GID/group name to run command as

Here is a (contrived) example showing the use of most of these parameters:

```python
process = container.exec(
    ['/bin/sh', '-c', 'echo HOME=$HOME, PWD=$PWD, FOO=$FOO'],
    environment={'FOO': 'bar'},
    working_dir='/tmp',
    timeout=5.0,
    user='bob',
    group='staff',
)
stdout, _ = process.wait_output()
logger.info('Output: %r', stdout)
```

This will execute the echo command in a shell and log something like `Output: 'HOME=/home/bob, PWD=/tmp, FOO=bar\n'`.


<h3 id="heading--exec-io-options">Input/output options</h3>

The simplest way of receiving standard output and standard error is by using the [`ExecProcess.wait_output`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecProcess.wait_output) method as shown above. The simplest way of sending standard input to the program is as a string, using the `stdin` parameter to `exec`. For example:

```python
process = container.exec(['tr', 'a-z', 'A-Z'],
                         stdin='This is\na test\n')
stdout, _ = process.wait_output()
logger.info('Output: %r', stdout)
```

By default, input is sent and output is received as Unicode using the UTF-8 encoding. You can change this with the `encoding` parameter (which defaults to `utf-8`). The most common case is to set `encoding=None`, which means "use raw bytes", in which case `stdin` must be a bytes object and `wait_output()` returns bytes objects. 

For example, the following will log `Output: b'\x01\x02'`:

```python
process = container.exec(['cat'], stdin=b'\x01\x02',
                         encoding=None)
stdout, _ = process.wait_output()
logger.info('Output: %r', stdout)
```

You can also pass [file-like objects](https://docs.python.org/3/glossary.html#term-file-object) using the `stdin`, `stdout`, and `stderr` parameters. These can be real files, streams, `io.StringIO` instances, and so on. When the `stdout` and `stderr` parameters are specified, call the `ExecProcess.wait` method instead of `wait_output`, as output is being written, not returned.

For example, to pipe standard input from a file to the command, and write the result to a file, you could use the following:

```python
with open('LICENSE.txt') as stdin:
    with open('output.txt', 'w') as stdout:
        process = container.exec(
            ['tr', 'a-z', 'A-Z'],
            stdin=stdin,
            stdout=stdout,
            stderr=sys.stderr,
        )
        process.wait()
# use result in "output.txt"
```

For advanced uses, you can also perform streaming I/O by reading from and writing to the `stdin` and `stdout` attributes of the `ExecProcess` instance. For example, to stream lines to a process and log the results as they come back, use something like the following:

```python
process = container.exec(['cat'])

# Thread that sends data to process's stdin
def stdin_thread():
    try:
        for line in ['one\n', '2\n', 'THREE\n']:
            process.stdin.write(line)
            process.stdin.flush()
            time.sleep(1)
    finally:
        process.stdin.close()
threading.Thread(target=stdin_thread).start()

# Log from stdout stream as output is received
for line in process.stdout:
    logging.info('Output: %s', line.strip())

# Will return immediately as stdin was closed above
process.wait()
```

That will produce the following logs:

```
Output: 'one\n'
Output: '2\n'
Output: 'THREE\n'
```

Caution: it's easy to get threading wrong and cause deadlocks, so it's best to use `wait_output` or pass file-like objects to `exec` instead if possible.


<h3 id="heading--exec-send-signal">Sending signals to a running command</h3>

To send a signal to the running process, use [`ExecProcess.send_signal`](https://ops.readthedocs.io/en/latest/#ops.pebble.ExecProcess.send_signal) with a signal number or name. For example, the following will terminate the "sleep 10" process after one second:

```python
process = container.exec(['sleep', '10'])
time.sleep(1)
process.send_signal(signal.SIGTERM)
process.wait()
```

Note that because sleep will exit via a signal, `wait()` will raise an `ExecError` with an exit code of 143 (128+SIGTERM):

```
Traceback (most recent call last):
  ..
ops.pebble.ExecError: non-zero exit code 143 executing ['sleep', '10']
```


<h2 id="heading--pebble-client">Accessing the Pebble client directly</h2>

Occassionally charm code may want to access the lower-level Pebble API directly: the [`Container.pebble`](https://ops.readthedocs.io/en/latest/#ops.model.Container.pebble) property returns the [`pebble.Client`](https://ops.readthedocs.io/en/latest/#ops.pebble.Client) instance for the given container.

Below is a (contrived) example of an action that uses the Pebble client directly to call [`pebble.Client.get_changes`](https://ops.readthedocs.io/en/latest/#ops.pebble.Client.get_changes):

```python
from ops.pebble import ChangeState

class MyCharm(CharmBase):
    ...

    def show_pebble_changes(self):
        container = self.unit.get_container('main')
        client = container.pebble
        changes = client.get_changes(select=ChangeState.ALL)
        for change in changes:
            logger.info('Pebble change %d: %s', change.id, change.summary)
```
