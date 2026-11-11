# Failure Handling

RBG supports `None` and `RecreateRoleInstanceOnPodRestart`. For the recreate policy,
`restartPolicyConfig.activateAfter` decides when its triggers are armed.

![failure-handling](../img/failure-handling.png)

## Restart Policy Types

| Policy | Description |
|--------|-------------|
| `None` | No automatic restart action; rely on default pod restart behavior. Failed pods are replaced through normal reconciliation. |
| `RecreateRoleInstanceOnPodRestart` | Recreate the whole role instance when a member fails, restarts unexpectedly, or is lost while terminating. `activateAfter` decides whether that starts before or after the instance is first Ready. |

### Triggers

`RecreateRoleInstanceOnPodRestart` recreates the instance on any of:

- a desired member Pod that is **Terminating**,
- a Pod in **Failed** phase,
- a **container restart** that an in-place update does not explain.

A Terminating member is the earliest signal after an eviction: with an unreachable node the
Pod stays Terminating until the Node object is deleted, an out-of-service taint is applied,
or it is force-deleted, and no same-name replacement exists before that. A member that
disappears without ever being observed as Terminating (force delete, PodGC, controller
downtime) is not covered by this trigger.

The existing protections stay in force: a rollout in progress (`currentRevision` not yet
equal to `updateRevision`) never triggers, any spec change bumps the generation and disarms
the triggers, Pending or NotReady alone is not a failure, init-container and native-sidecar
restarts do not count, and an in-place update exempts only its own expected container
restarts — it cannot mask a Failed Pod or a lost member.

### Arming: `activateAfter`

| Value | Meaning |
|-------|---------|
| `Ready` (default) | Triggers are armed once the instance has been Ready at its current generation. Creation and scale-up keep their protection. |
| `Created` | Triggers are armed as soon as the instance exists, which also covers failures before the instance first becomes Ready. |

The stability fact is persisted as `status.lastReadyGeneration`. Gating on the live
`RoleInstanceReady` condition instead would be unsound: status is written after scaling, so
a fault flips the condition to False before the restart signal becomes visible, and the
policy would then never fire again.

`activateAfter` is supported by native RoleInstanceSet workloads. LeaderWorkerSet-backed
roles keep the policy they had; their behavior is already ungated (LWS recreates the group on
`ContainerRestarted` or `PodDeleted` with no start or Ready gate) and the vendored LWS v0.7
adapter exposes no field to express the distinction. StatefulSet and Deployment roles have no
instance-level recreate path.

**Deleting a single member now rebuilds the whole RoleInstance.** That is the intended
semantics for this policy — the members of a multi-rank workload agree on a one-shot
membership at startup, so replacing one rank in place does not recover it — and it matches
LeaderWorkerSet. Use the `restart-trigger-policy: Ignore` annotation on auxiliary components
whose failures should not affect the main workload.

Both served RBG APIs accept the policy and `activateAfter`. In v1alpha2 use the pattern's
`restartPolicyConfig`; in v1alpha1 use `spec.roles[].restartPolicy`, which cannot express
`activateAfter` and therefore always resolves to the default `Ready`. Reading an object back
through v1alpha1 drops an explicitly configured `activateAfter`, so use v1alpha2 when the
arming point matters.

## Configuration

Set the `restartPolicyConfig` field on the pattern (`leaderWorkerPattern` or `customComponentsPattern`).
`standalonePattern` has no restart policy field — it is always `None` (single pod, recreating the instance is equivalent to normal pod replacement).

`restartPolicyConfig` is an object with the following fields:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | `RecreateRoleInstanceOnPodRestart` | Restart policy type: `None` or `RecreateRoleInstanceOnPodRestart`. |
| `activateAfter` | string | `Ready` | When the recreate triggers are armed: `Ready` or `Created`. Omit it to keep the default. |
| `baseDelaySeconds` | int | `30` | Base delay (seconds) for exponential backoff between restart attempts. Set to `0` with `maxDelaySeconds: 0` to disable backoff entirely. |
| `maxDelaySeconds` | int | `600` | Cap (seconds) for the exponential backoff delay. |

### Deprecated `restartPolicy` field

The pattern also accepts a deprecated `restartPolicy` string field, which is the shape v0.7.0
used before the backoff configuration was introduced. It is still honoured so that workloads
written for v0.7.0 keep working:

1. If `restartPolicyConfig.type` is set, it is used.
2. Otherwise, if `restartPolicy` is set, its value is used.
3. Otherwise the pattern default (`RecreateRoleInstanceOnPodRestart`) applies.

The two fields are resolved at runtime — the controller and the conversion webhook always read them through the precedence above, and stored objects are not modified. New workloads should only set `restartPolicyConfig`.

### Upgrading

- **From v0.7.0**: no action required. The `restartPolicy` wire format is unchanged.
- **From v0.8.0-alpha.3 / v0.8.0-alpha.4**: the released images for these two pre-releases briefly defined `restartPolicy` as an *object* instead of a string. That shape is no longer decodable by v0.8.0+ controllers — leaving even one object in the old shape prevents the new controller from listing that resource type at all, not just the offending object. These two versions are unsupported and should not be used. If you deployed them, delete all affected objects (`RoleBasedGroup`, `RoleBasedGroupSet`, `RoleInstance`, `RoleInstanceSet`) and recreate them with v0.8.0+.

```yaml
apiVersion: workloads.x-k8s.io/v1alpha2
kind: RoleBasedGroup
metadata:
  name: restart-policy-demo
spec:
  roles:
    # Worker role with leaderWorkerPattern - recreate instance on pod failure (default)
    - name: worker
      replicas: 3
      leaderWorkerPattern:
        size: 2
        restartPolicyConfig:
          type: RecreateRoleInstanceOnPodRestart
        template:
          spec:
            containers:
              - name: worker
                image: nginx:latest

    # Worker role with explicit None - no restart action
    - name: auxiliary
      replicas: 1
      leaderWorkerPattern:
        size: 2
        restartPolicyConfig:
          type: None
        template:
          spec:
            containers:
              - name: auxiliary
                image: nginx:latest

    # StandalonePattern - no restart policy field (always None)
    - name: monitor
      replicas: 1
      standalonePattern:
        template:
          spec:
            containers:
              - name: monitor
                image: nginx:latest
```

## Exponential Backoff

When the resolved restart policy type is `RecreateRoleInstanceOnPodRestart`, subsequent crash-triggered recreations within a stability window are delayed by exponential backoff:

- **First crash**: immediate recreation (no delay).
- **Second crash** (within the stability window): delayed by `baseDelaySeconds`.
- **Each subsequent crash**: delay doubles, capped at `maxDelaySeconds`.
- Formula: `delay = min(baseDelaySeconds * 2^(restartCount-1), maxDelaySeconds)`.

The stability window is `max(maxDelaySeconds * 2, 10 minutes)`. If the instance remains stable (no crashes) for longer than this window, `restartCount` resets to 0 and the next crash starts with immediate recreation again.

### Default Behavior Change

> **Note**: Before the backoff feature, pod crashes always triggered immediate recreation. With default settings (`baseDelaySeconds: 30`, `maxDelaySeconds: 600`), the second and
> subsequent crashes within the stability window now receive 30s→600s exponential delays. All existing workloads using the default `RecreateRoleInstanceOnPodRestart` policy
> automatically inherit this behavior.

### Disabling Backoff

To disable backoff entirely and restore the pre-backoff behavior (immediate recreation on every crash), set both `baseDelaySeconds` and `maxDelaySeconds` to `0`:

```yaml
restartPolicyConfig:
  type: RecreateRoleInstanceOnPodRestart
  baseDelaySeconds: 0
  maxDelaySeconds: 0
```

### Custom Backoff Configuration

```yaml
restartPolicyConfig:
  type: RecreateRoleInstanceOnPodRestart
  baseDelaySeconds: 10
  maxDelaySeconds: 120
```

With the above configuration: first crash is immediate, second crash delayed 10s, third 20s, fourth 40s, fifth 80s, sixth and beyond capped at 120s.

> **Note**: `maxDelaySeconds` must be greater than or equal to `baseDelaySeconds` (enforced by CEL validation at admission). Because `maxDelaySeconds` defaults to 600, setting `baseDelaySeconds` above 600 without explicitly setting `maxDelaySeconds` will be rejected. If you need a base delay above 600s, set `maxDelaySeconds` to the same or higher value.

## Excluding Components from Restart Policy Trigger

You can prevent specific pods from triggering the role's restart policy by setting an annotation on the pod template. This is useful for auxiliary components (e.g., monitoring, logging sidecars) whose failures should not affect the main workload. The annotation works with any deployment pattern (standalonePattern, leaderWorkerPattern, or customComponentsPattern).

Set the annotation `rbg.workloads.x-k8s.io/restart-trigger-policy: "Ignore"` in the component's `template.metadata.annotations`:

```yaml
apiVersion: workloads.x-k8s.io/v1alpha2
kind: RoleBasedGroup
metadata:
  name: inference-demo
spec:
  roles:
    - name: engine
      replicas: 2
      customComponentsPattern:
        restartPolicyConfig:
          type: RecreateRoleInstanceOnPodRestart
        components:
          # Main inference component - will trigger restart policy on pod failure
          - name: inference
            size: 1
            template:
              spec:
                containers:
                  - name: inference
                    image: inference:latest

          # Monitoring sidecar - will NOT trigger restart policy on pod failure
          - name: monitor
            size: 1
            template:
              metadata:
                annotations:
                  # This component's pod restart/delete will NOT trigger role/RBG recreation
                  rbg.workloads.x-k8s.io/restart-trigger-policy: "Ignore"
              spec:
                containers:
                  - name: monitor
                    image: monitoring:latest
```

When this annotation is set to "Ignore" on a component's pod template:
- Pod failure and container restart events from this component will NOT trigger the role's restart policy
- The restart policy (RecreateRoleInstanceOnPodRestart) remains unaffected for other components
- Only pods from components WITHOUT this annotation (or with annotation set to "Inherit") will trigger the restart policy

## Use Cases

- **`RecreateRoleInstanceOnPodRestart`**: Existing workloads that depend on the original, readiness-gated unexpected-restart and Failed-pod triggers.
- **`activateAfter: Created`**: instances that must also recover from failures before they first become Ready, such as a role whose startup can fail and then hang in a rendezvous.
- **None**: Monitoring/logging sidecars that don't affect the main workload. Failed pods are replaced individually.
- **restart-trigger-policy: "Ignore"** (annotation): Auxiliary components within a multi-component instance whose pod failures should not trigger the role's restart policy for the entire instance.

## Examples

- [Restart Policy Examples](../../examples/basic/rbg/restart-policy/restart-policy.yaml)
- [CustomComponentsPattern with restart-trigger-policy annotation](../../examples/basic/rbg/patterns/custom-components-pattern.yaml)