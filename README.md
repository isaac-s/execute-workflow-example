# Example: Workflow Executions

This example shows:

* Binding a script implementation to a standard lifecycle operation
* Binding a script implementation to a custom operation
* Running scripts via Fabric (SSH)
* Implementing a workflow as a Python script
* Running a workflow
* Resuming a workflow

## Deployment Creation

First, upload the blueprint:

```bash
cfy blueprints upload blueprint.yaml -b execute_workflow
```

Next, create a YAML file containing inputs:

```yaml
ip: 10.0.0.24
ssh_user: centos
private_key_path: /etc/cloudify/default_key.pem
```

Next, create a deployment:

```bash
cfy deployments create dep_1 -b execute_workflow -i inputs.yaml
```

## Installation

To install the topology:

```bash
cfy executions start install -d dep_1
```

The out-of-the-box `install` workflow runs the standard lifecycle operations, so you
should see printouts from these operations (you can find the respective scripts inside
the [scripts](scripts) directory).

## Running a Specific Operation

Often, there is a need to execute a specific operation on the topology.
This can be done by the `execute_operation` workflow. In this example,
we will run the `poll` operation on the `app` node.

```bash
cfy executions start execute_operation -p operation=maintenance.poll -p node_ids=[app]
```

This command will execute the `poll` operation of the `maintenance` interface,
on all instances of the `app` node template.

You can also execute the operation on a specific instance of the node template,
if the node template has multiple instances. For that, you will need to obtain the
*node instance ID*. You can use the following command:

```bash
cfy node-instances list -d dep_1 -n app
```

And then (assuming the node instance ID is `app_123456`):

```bash
cfy executions start execute_operation -p operation=maintenance.poll -p node_instance_ids=[app_123456]
```

### Fabric (SSH)

The `poll` operation is implemeted using the Fabric plugin. The Fabric plugin has multiple tasks,
of which `run_script` is the most popular. `run_script` pushes the script (denoted by the `script_path`
parameter) to the target VM, and executes it there.

Of significant importance is the `fabric_env` parameter, which is a dictionary passed as-is to
the Fabric library. You can read about available Fabric environment parameters in the Fabric
documentation: https://docs.fabfile.org/en/1.12.1/usage/env.html

Normally, you would need to specify the `user` and `key_filename` parameters, instructing the plugin
which user and which key file to use for SSH.

**NOTE**: the `key_filename` should point to a file on the *Cloudify Manager* VM, as the Fabric code
itself runs on the manager.

The Fabric plugin provides a shortcut: if the enclosing node template (`app` in our case) has a
`contained_in` relationship (directly or indirectly) to a node template of type `cloudify.nodes.Compute`
(or a subtype thereof), then the `host_string` parameter in `fabric_env` is defaulted to the private IP
of the containing VM. Hence, in our example, we omitted `host_string`; the plugin will calculate it automatically.

## Executing a Cusotm Workflow

The blueprint defines a custom workflow called `rollout`. The workflow is mapped
to a script.

To execute the workflow, run the following command:

```bash
cfy executions start rollout -d dep_1
```

This workflow runs the `maintenance.update` and `maintenance.commit` operations in sequence (see the
workflow implementation in `scripts/rollout.py`).

### Resuming a Workflow

The `maintenance.commit` operation contains a condition: the operation will fail if the `app` node
instance has a runtime property called `fail_commit` and its value is `True`.

By default, this runtime property doesn't exist on any node instance.

To demonstrate how to resume a workflow, we will set the runtime property's value from the command line.

First, obtain the node instance ID:

```bash
cfy node-instances list -d dep_1 -n app
```

(We will assume that the node instance ID is `app_123456`)

Next, set the runtime property value:

```bash
cfy node-instances update-runtime -p fail_commit=True app_123456
```

Now, let's run the workflow again:

```bash
cfy executions start rollout -d dep_1
```

The execution will run, however it will fail at the `maintenance.commit` operation.

Now, we will remove the runtime property and then resume the execution (we will assume that the
execution ID is `abcdefgh`):

```bash
cfy node-instances delete-runtime -p fail_commit app_123456
cfy executions resume abcdefgh
```

This will trigger the execution's resumption. You can then use the `cfy events list` command (optionally
with the `--tail` parameter) to look at the execution log:

```bash
cfy events list abcdefgh
```
