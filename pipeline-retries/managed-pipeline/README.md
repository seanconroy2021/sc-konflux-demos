# test-managed-pipeline

Test managed pipeline with configurable failure modes for testing retry logic.

## Data Parameters

Set in the ReleasePlanAdmission `data` field.

| Name         | Description                                              | Default      |
|--------------|----------------------------------------------------------|--------------|
| failMode     | `none`, `error`, `oom`, `pipeline-timeout`, or `task-timeout` | none         |
| failOnTask   | `publish-data` or `update-status`                        | publish-data |
| failOnStep   | Step name within the task empty = first step in task     | ""           |
| passRetries  | Number of retries before succeeding (0 = always fail)    | 0            |

## How passRetries works

The pipeline reads the Release status to count `managedPipelineAttempts`. If the
attempt count >= `passRetries`, the pipeline succeeds instead of failing.

- `passRetries: 0` - always fails (never auto-pass)
- `passRetries: 1` - fails on first attempt, succeeds on first retry
- `passRetries: 2` - fails on first 2 attempts, succeeds on second retry

## Tasks

```
collect-data > publish-data > update-status
```

| Name           | Steps                          |
|----------------|--------------------------------|
| collect-data   | collect-data                   |
| publish-data   | push-artifacts, verify-publish |
| update-status  | update-cr-status               |

## Usage

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: test-rpa
  namespace: managed-namespace
spec:
  applications:
    - my-app
  origin: tenant-namespace
  data:
    failMode: "oom"
    failOnTask: "update-status"
    passRetries: 2
  pipeline:
    pipelineRef:
      resolver: git
      params:
        - name: url
          value: https://github.com/seanconroy2021/sc-konflux-demos.git
        - name: revision
          value: main
        - name: pathInRepo
          value: pipeline-retries/managed-pipeline/managed-pipeline.yaml
    serviceAccountName: managed-service-account
  policy: slsa3
```

### Failure Types

**Success:** omit data params or set `failMode: "none"`

**Error on first step:** `failMode: "error"`, `failOnTask: "publish-data"`

**Error on specific step:** `failMode: "error"`, `failOnTask: "publish-data"`, `failOnStep: "verify-publish"`

**OOMKill:** `failMode: "oom"`, `failOnTask: "update-status"`

**Pipeline Timeout:** `failMode: "pipeline-timeout"`, `failOnTask: "publish-data"` (set `pipeline.timeouts.pipeline: 3m` in RPA)

**Task Timeout:** `failMode: "task-timeout"`, `failOnTask: "publish-data"` (task has a built-in 2m timeout)

**Fail then succeed:** add `passRetries: N` to any of the above to succeed after N retries
