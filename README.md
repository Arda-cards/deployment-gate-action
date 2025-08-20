# gradle build

... Grab the badge for the CI build here, see
[![ci](https://github.com/Arda-cards/deployment-gate-action/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/Arda-cards/deployment-gate-action/actions/workflows/ci.yaml?query=branch%3Amain)
[CHANGELOG.md](CHANGELOG.md)

This action implements a deployment gating process.

## Design

### Data Structure: the permission request

Every infrastructure has a namespace `deployment-gate-action`, name configurable with the parameter `deployment_gate_namespace`.

The namespace has a ConfigMap for every component for every partition present in the infrastructure.

The ConfigMap name follows the component's namespace name pattern: *purpose* - *component name*.

Under the key `requests`, every ConfigMap contains a simple data structure:

```yaml
{
"version", # string, the version schema of this data structure, start at "1"
  "granted": { # a map of
    chart_version: { # string, the version of the chart that can be deployed
      "grant_date", # string, ISO 8601 of when the request was approved
      "request_date", # string, ISO 8601 of when the entry was added
      "run_id", # string, the unique identifier of the workflow run.
    }
  }
}
```

Example:

```yaml
{
  "version": "1",
  "granted": {
    "0.3.0": {
      "request_date": "2025-08-20T17:07:01+00:00",
      "run_id": "17084014751"
    }
    }
}
```

### Usage

The action's input `mode` must be set to one of `request` or `grant`.

#### Request

The action aborts if its input `deployment_gate` is not equals to `manual`.

Otherwise, it fetches the ConfigMap and proceeds if it finds an entry for the `chart_version` to be deployed
with its `run_id` and the `grant_date` set.

If not, it creates, or updates, an entry for the `chart_version` with its `run_id`, and fails the deployment.

#### Grant

Every component has a simple manually triggered workflow with a GitHub UI that let a human select a purpose
and enter a chart version.

In the ConfigMap of the partition for the purpose, the workflow updates the entry for the `chart_version`
with the `grant_date`, then reads the `run_id` and triggers the deployment workflow.

The workflow fails if it doesn't find an entry for the `chart_version` and displays all the chart versions
pending approval; they have a `run_id` but no grant_date.

## Arguments

See [action.yaml](action.yaml).

## Usage

```yaml
- name: "Process request to deploy ${{ inputs.chart_version }} for ${{ inputs.purpose }}"
  uses: Arda-cards/deployment-gate-action@dna/1
  with:
    aws_region: "${{ steps.purpose_config.outputs.aws_region }}"
    chart_name: "${{ inputs.chart_name }}"
    chart_version: "${{ inputs.chart_version }}"
    cluster_name: "${{ steps.purpose_config.outputs.cluster_name }}"
    mode: "grant"
    purpose: "${{ inputs.purpose }}"
```

## Permission Required

```yaml
permissions:
  actions: write
```
