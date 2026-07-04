# saphnet-komodo
The main resource sync to use for Komodo in the Sapphic Homelab/Home Server

Note that this is intended to work in tandem with [saphnet-compose-configs](https://github.com/AnarchoBooleanism/saphnet-compose-configs), and be initially introduced to host systems in [saphnet-nixos-configs](https://github.com/AnarchoBooleanism/saphnet-nixos-configs).

**NOTE** (Last tested with Komodo v2.2.0): `resource-sync` resources seem to have no effect on a Komodo server, so, for now, try to manually add each Resource Sync!

Documentation on Komodo resources and resource syncs:
- Sync Resources: https://komo.do/docs/automate/sync-resources
- Procedures and Actions: https://komo.do/docs/automate/procedures
- Schedule: https://komo.do/docs/automate/schedules
- Build: https://komo.do/docs/build
- Swarm: https://komo.do/docs/swarm
- Docker Compose: https://komo.do/docs/deploy/compose
- Containers: https://komo.do/docs/deploy/containers
- Automatic Updates: https://komo.do/docs/deploy/auto-update

Note that you can create Komodo resources in the UI, and then have it generate the corresponding TOML config, to use in your resource files!

## Rationale for this being separate from `saphnet-compose-configs` and `saphnet-nixos-configs`
`saphnet-compose-configs` is mainly intended for the Docker Compose stacks being run, with their Compose files, other config files, and Komodo resource files directly related to the stacks; having a separate repository dedicated to configurations for Komodo servers and for Komodo as a whole allows `saphnet-compose-configs` to solely focus on its Compose stacks and their corresponding resources, and it helps prevent any issues with circular dependencies when it comes to the Procedures defined in this repository. Furthermore, the contents of the Compose stacks in `saphnet-compose-configs` are very highly subject to change, meaning that having a separation of concerns will allow for increased stability.

While `saphnet-nixos-configs` is designed for the complete (and thorough) configuration of a host, all the way to the programs being run, a deployment of such a system can only be updated with a build procedure that builds and updates everything all at once; for our purposes, it is preferable to have Komodo be able to update itself in a lightweight manner, as Komodo mostly runs independently to the underlying operating system anyway. So, the configuration for Komodo in the `control-server` configuration is designed for ensuring Komodo only has what it needs to start running, with just enough resource files and plumbing to bootstrap the process of pulling this repository and synchronizing the resources within it. This also allows for the ability to run Komodo Core outside of NixOS, as long as it is given the values needed to effectively run everything within this repository.

As well, since `saphnet-nixos-configs` handles the bootstrapping process for Komodo, all Repo resources (mapping to Git repositories, like `saphnet-compose-configs` and this repository) are defined in the bootstrap resource file instead of `saphnet-komodo`, in order to avoid circular dependency issues in Komodo.

## Adding new servers

To add new Komodo servers to the repository for Komodo Core to look for, you will need to add entries for their corresponding Server resources, Resource Syncs resources for server-specific Stacks, and, depending on the type of server, Builder resources, to the `servers.toml` file.

Here is an example of what a portion of a configuration would look like for a server named `example-server`, in `servers.toml`:

```toml
##

[[server]]
name = "example-server"
tags = ["gpu", "high-availability"]
[server.config]
address = "https://example.server.saphnet.xyz:8120"
region = "Example region"
enabled = true

[[resource-sync]]
name = "example-server_stack-sync"
tags = ["stack-sync", "iac"]
[resource-sync.config]
linked_repo = "saphnet-compose-configs"
resource_path = ["example-server.toml"]
```

Firstly, note that each server's configuration is separated from each other by lines that consist of `##`.

In this example, the corresponding Server resource is defined with `[[server]]`, with all configuration under that line. It is named as `example-server`, and has the tags `gpu` and `high-availability`; generally, tags are used to describe a server's main function and abilities. Under `[server.config]`, is the URL through which Komodo Periphery on the server can be accessed (`https://example.server.saphnet.xyz:8120`), the name for the region that it falls under (`Example region`), and then the fact that the Server resource is enabled.

Furthermore, since, for our purposes, Komodo servers are generally for running Stacks, we have a definition for a Resource Sync resource, tied to a resource file in the `saphnet-compose-configs` repository, for synchronizing the state of Stack resources to the state defined in the `saphnet-compose-configs`. For our purposes, we only need to care about having this type of Resource Sync for each server, as it is the job of the resource file being referred to care about handling individual Stack resources.

Each Resource Sync resource is defined with `[[resource-sync]]`. In this definition, the Resource Sync is named as `example-server_stack-sync`; generally, these types of Resource Syncs are named after their target servers, with `_stack-sync` affixed at the end. This resource, then, has the tags `stack-sync` and `iac` associated with it; all Resource Syncs of this kind should have the `stack-sync` tag, and all non-Server resources managed through GitOps, like in this repository, should have the `iac` tag associated with them.

Under `[resource-sync.config]`, is the name of the Repo resource under which resource files are found, `saphnet-compose-configs`. After the Repo being linked to is a list of resource files to use for the Resource Sync, which is simply `example-server.toml`; generally, the resource file used for this type of Server-specific Resource Sync is at the root of `saphnet-compose-configs` and is named after the server that is being targeted.

In addition, depending on the server (based on whether the server is able to dedicate extra resource to it), a server can also have a Builder resource associated with it; this means that the server can be used for building Docker images from Dockerfiles:

```toml
##

[[builder]]
name = "example-server"
tags = ["iac"]
[builder.config]
type = "Server"
params.server_id = "example-server"
```

Again, each server's Builder resource configuration is separated from each other by lines that consist of "##".

Each Builder resource is defined with `[[builder]]`. In this definition, the Builder resource is named `example-server`, after the Server resource, and is given the `iac` tag (again, all non-Server resources should be given `iac` tags, to indicate that they are being managed through GitOps).

Under `[builder.config]`, the type is defined as `Server` (this needs to be specified, as Builders can be Server resources or AWS instances), and then the server ID of the Server being used is specified, as `example-server`, in this case.

These three configuration fragments (or the first two) should be enough to properly set up a server for our Komodo setup.

## On procedures

Procedures, in Komodo, allow us to run multi-stage workflows that operate on resources in our setup; they are also able to be scheduled to run automatically. The Procedures defined in `procedures.toml` should generally be enough for our day-to-day needs, mostly when it comes to Stack resources, but if you have other specific needs, you are welcome to add more Procedures to `procedures.toml`.

Here is an example of a Procedure, `saphnet-repo-sync`:
```toml
## 

[[procedure]]
name = "saphnet-repo-sync"
description = "Update both the saphnet-compose-configs and saphnet-komodo repositories."
tags = ["main", "iac"]
config.schedule_enabled = true
config.schedule_format = "Cron"
config.schedule = "0 10 * * * *" # Every hour at minute 10

[[procedure.config.stage]]
name = "Stage 1"
enabled = true
executions = [
    { execution.type = "BatchPullRepo", execution.params.pattern = """
saphnet-compose-configs
saphnet-komodo""", enabled = true }
]
```

Like the other resources in the other files, individual Procedures are generally separated from each other by lines that consist of "##".

Each Procedure is defined with `[[procedure]]`. In this definition, the Procedure is named as `saphnet-repo-sync` and given a description. It is given the tags `main` and `iac`; `main`, in this case, means that the Procedure is a core Procedure that is expected to be used often, and `iac` is given as the Procedure is a resource managed through GitOps (with this repository).

Procedures, like in this example, can run on schedules, defined with `config.schedule_enabled`. The format of the schedule can either be in the cron format (with `Cron`) or in plain English (with `English`), which gets automatically converted into the cron format. Note that the format that Komodo expects is the 6-field format, in the order of second, minute, hour, day of month, month, and day of week. In our example, we have `0 10 * * * *`, which means to run only when the second is 0, the minute is 10, and at any hour, day, day of the month, month, and day of the week; in other words, every hour at minute 10 (e.g. at 02:10). For reference, [here is a guide for writing cron expressions](https://dev.to/arenasbob2024cell/cron-expressions-explained-from-basics-to-advanced-scheduling-215n).

Going further, each Procedure has one or more stages, which run in sequential order (not in parallel), defined with `[[procedure.config.stage]]`, under the Procedure being defined. Each stage has any number of executions, which are tasks (from a which) that Komodo runs ([this is the list of possible executions under Komodo](https://docs.rs/komodo_client/latest/komodo_client/api/execute/)), which all run in parallel. If you need to perform tasks that are more complex than what Komodo provides, [you can create and execute Actions](https://komo.do/docs/automate/procedures#actions), which are Typescript scripts that call the Komodo API.

In this example, the sole stage for the Procedure is named as `Stage 1`, and runs the `BatchPullRepo` execution, which updates multiple Repo resources (that match the pattern(s) given to it), by pulling from the remote Git servers that host the files behind the Repos. Note that `enabled` is set to `true`, so that Komodo recognizes it as a Procedure that can be run.

If there are multiple stages, they will run in the order that is defined in the resource file.

## On variables

Sometimes, there may be specific values that Stacks may need for specific functionality, but may be too fast-changing or dynamic to be sensibly defined within Compose files (or other related config files), or, for some other reason, we would prefer to define such values as environment variables to be injected at deploy-time; furthermore, these values may need to be defined multiple times across multiple Stacks, and, so, we may desire a way to have a single source of truth for these values to avoid the error-prone process of manually updating values across multiple locations. In this case, Komodo provides the Variable resource, which acts as a way to store (and change) specific values, under a specific name. Variables can be referred to, in other resources, when defining environment variables at deploy-time.

For this repository, all Variable resources are defined within `variables.toml`.

Note that Komodo also has the ability to store secrets as Variables, but for our purposes (using GitOps), we will avoid storing them here, particularly as there is no easy (and secure) way to update their values using tools such as sops.

Here is an example of a Variable defined in `variables.toml`, defining the email address to give to Certbot (something needed across multiple Stacks):
```toml
[[variable]]
name = "CERTBOT_EMAIL"
value = "example@example.com"
is_secret = false
```

Each Variable is defined with `[[variable]]`, though, for this file, each Variable resource is only separated by single empty lines. Each Variable has a name (`CERTBOT_EMAIL` in this case), and a value (`example@example.com`), which is always a string; the convention for Variable names in this repository is to have it in screaming snake case (e.g. `SCREAMING_SNAKE_CASE`). As well, you will generally want `is_secret` to be set to `false`, for Komodo to not recognize it as a secret.

**NOTE**: Since the values of environment variables, which Komodo Variables will be injected into, WILL be parsed by a shell, any `$` symbols that you want to have processed literally will need to be escaped with backslashes! For example, a value like `1234$example` will need to be written as `1234\$example` in a resource file.

Note that, unlike other resources, Variables do not have tags associated with them.

Here is an example of how Variables are used in a Stack, `n8n`:
```toml
# n8n
[[stack]]
name = "n8n"
... # Omitting for brevity
[stack.config]
...
environment = """
MAIL_PASSWORD = [[NAMECHEAP_MAIL_PASSWORD]]
TAILSCALE_IP = [[TAILSCALE_IP_CORE]]
"""
```

In `environment`, which is a multi-line string, where each line is in the format of `KEY = VALUE`, we are defining multiple environment variables that will be provided to Docker Compose when running the `docker compose up` command. The names of Komodo Variables are given, wrapped in double brackets, so that Komodo knows to replace them with the values of those Variables before passing them to Docker Compose. In this example, the `TAILSCALE_IP` environment variable is defined with the value of the Komodo Variable, `TAILSCALE_IP_CORE`.

Note that the value of an environment variable doesn't have to consist of just the value of a Komodo Variable; Komodo Variables can be one of many parts of the final value of an environment variable. For example, we can have a line that looks like `ENV_VAR = [[EXAMPLE_VAR_1]].example2.[[EXAMPLE_VAR_3]]`.