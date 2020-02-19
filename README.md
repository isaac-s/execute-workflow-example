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

Next, create a deployment (this command-line example is for OpenStack. For AWS,
GCP or Azure, replace `openstack` with `aws`, `gcp` or `azure`,
respectively):

```bash
cfy deployments create dep_1 -b execute_workflow -i infra_name=openstack
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
cfy executions start execute_operation -d dep_1 -p operation=maintenance.poll -p node_ids=[app]
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
cfy executions start execute_operation -d dep_1 -p operation=maintenance.poll -p node_instance_ids=[app_123456]
```

## Executing a Custom Workflow

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
