# ansible-role-victoria_monitoring_stack

This Ansible role starts the following five Docker containers, with appropriate
configuration, resulting in a fully featured monitoring/observability setup:

- [OTel Collector][1]: The "gateway" for ingesting, tranforming and routing all incoming data.
- [vmetrics][2]: The VictoriaMetrics metrics database.
- [vtraces][3]: The VictoriaMetrics traces database.
- [vlogs][4]: The VictoriaMetrics logs database.
- [Grafana][5]: The frontend/visualizer for the data collected.


## Installation

This repository does not have any dependencies, so just move into your `roles/`
folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-victoria_monitoring_stack.git victoria_monitoring_stack
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: metrics_hub
  name: Provision a Metrics Hub
  become: true
  gather_facts: no
  roles:
    - victoria_monitoring_stack
```


## Usage

This section will go though the different components in the order they are
deployed by the role.

### Base Config
All the containers started will share a parent directory and Docker network,
which may be renamed in case the defaults aren't good enough

```yaml
victoriastack_base_dir: "/srv/victoriastack"
victoriastack_network_name: "victoriastack_network"
```

### Databases
All three VictoriaMetrics databases are grouped into the same section since
they are very "basic" in that there aren't that much to configure. Exact version
and retention time are what may be of interest to change (change the `<type>` to
one of logs/traces/metrics):

```yaml
victoriastack_<type>_container_version: "latest"

# Defaults to 3 months, else add one of the following suffixes:
#    h(ours), d(ays), w(eeks), M(onth), y(ears)
victoriastack_<type>_retention_period: "3"

# Custom CLI arguments if needed.
victoriastack_<type>_container_command: []
```

For the `command` argument the `-retentionPeriod` and `-storageDataPath`
options are already set, but other options may be included like this (if you
want to remove the `scope.*` attributes on all OTel metrics for example):

```yaml
victoriastack_metrics_container_command:
  - "-opentelemetry.promoteScopeMetadata=false"
```

### OTel Collector Gateway
In order to simplify for the clients who want to send data to this backend setup
we run the OTel Collector as a gateway/entrypoint which means that the clients
can target the same port for all types of data. This collector then routes the
data to the correct backend database based on the type.

```yaml
victoriastack_collector_container_version: "latest"
victoriastack_collector_container_command: []

victoriastack_collector_published_ports:
  - "4317:4317" # Public OTLP gRPC endpoint
  - "4318:4318" # Public OTLP HTTP endpoint
```

This setup has the added benefit of allowing us to add "global" processors that
can transform or drop data so that anything that is finally stored is actually
wanted and follows the same format.

**Example**
```yaml
victoriastack_collector_processors:
  transform/collector_version_remover: # Remove collector version information from metrics
    metric_statements:
      - context: scope
        statements:
          - set(scope.attributes["scope.name"], nil)
          - set(scope.attributes["scope.version"], nil)

# Don't forget to add the processor in the desired position in the pipeline.
victoriastack_collector_pipeline_metrics_processors: [
  "transform/collector_version_remover"
]
```

> NOTE: There is already a "memory_limiter" and "batch" processors placed in
> the first and last position in the pipeline for performance reasons. Check
> out the [template](./templates/otel_collector.yml.j2) for details.

### Grafana
The final component to the stack is Grafana, which is the combined
frontend/visualizer for the data collected in the databases. From the base
settings for this container it is just the `password` one that you must
configure by yourself:

```yaml
victoriastack_grafana_enabled: true
victoriastack_grafana_container_version: "latest"
victoriastack_grafana_container_command: []

victoriastack_grafana_admin_user: "admin"
victoriastack_grafana_admin_password: "admin"
```

This will initialize the first admin user with the name "admin" and the password
"admin". This should be changed to something that is more secure.

It is possible to provide more enviroment variables to the container via

```yaml
victoriastack_grafana_container_env_extra: {}
```

but if you define anyone of the keys listed below they will be overwritten with
the following values:

```yaml
GF_PLUGINS_PREINSTALL: "{{ (['victoriametrics-logs-datasource'] + victoriastack_grafana_plugins_preinstall) | join(',') }}"
GF_USERS_ALLOW_SIGN_UP: "false"
GF_SECURITY_ADMIN_USER: "{{ victoriastack_grafana_admin_user }}"
GF_SECURITY_ADMIN_PASSWORD: "{{ victoriastack_grafana_admin_password }}"
```

These are some core values that need to be defined, so should not be modified
outside the dedicated ansible variables.

Finally you _can_ edit the `victoriastack_grafana_datasources_<type>` variables
inside the [datasources template](./templates/grafana_datasources.yml.j2), but
they should be pretty sane as they are.




[1]: https://hub.docker.com/r/otel/opentelemetry-collector-contrib
[2]: https://hub.docker.com/r/victoriametrics/victoria-metrics
[3]: https://hub.docker.com/r/victoriametrics/victoria-traces
[4]: https://hub.docker.com/r/victoriametrics/victoria-logs
[5]: https://hub.docker.com/r/grafana/grafana
