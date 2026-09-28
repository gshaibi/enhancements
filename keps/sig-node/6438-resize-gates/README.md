# KEP-6438: Resize Gates for In-Place Pod Resize

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Why denying at admission is not enough](#why-denying-at-admission-is-not-enough)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API changes](#api-changes)
  - [Kubelet contract](#kubelet-contract)
  - [Integrator flow](#integrator-flow)
  - [Scheduler interaction](#scheduler-interaction)
  - [Example sequences](#example-sequences)
    - [Accepted resize](#accepted-resize)
    - [Held, then canceled by the user](#held-then-canceled-by-the-user)
    - [Second resize while the first is held](#second-resize-while-the-first-is-held)
    - [Approved, then infeasible on the node](#approved-then-infeasible-on-the-node)
    - [Approved, then deferred on the node (KEP-5836)](#approved-then-deferred-on-the-node-kep-5836)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
    - [Deprecation](#deprecation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Persistent gates with approvals in status](#persistent-gates-with-approvals-in-status)
  - [Optimistic admission (actuate, then reclaim)](#optimistic-admission-actuate-then-reclaim)
  - [Comparison](#comparison)
  - [Open questions](#open-questions)
    - [Gate mutability and authorization](#gate-mutability-and-authorization)
    - [Scope of the hold](#scope-of-the-hold)
    - [Field name](#field-name)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

In-place pod resize ([KEP-1287], GA in v1.35) lets a running Pod grow CPU and memory through the
`pods/resize` subresource without passing through any point where an out-of-tree quota or
capacity controller can hold or reject the growth. This KEP adds `pod.spec.resizeGates`,
mirroring `schedulingGates` ([KEP-3521]): while a gate is present, the kubelet does not allocate
resource growth for the pod. External controllers install gates on the mutation paths they care
about, observe the proposed growth, admit it against their own capacity model, and remove their
gate to let the kubelet proceed. Enforcement lives at the kubelet allocation step and applies
only to growth, so the same gate covers any mutation path that grows a running pod, while
down-sizes and resource-neutral updates are never held.

[KEP-1287]: /keps/sig-node/1287-in-place-update-pod-resources
[KEP-3521]: /keps/sig-scheduling/3521-pod-scheduling-readiness
[KEP-5836]: /keps/sig-scheduling/5836-scheduler-preemption-for-ippr
[KEP-5465]: https://github.com/kubernetes/enhancements/issues/5465
[KEP-5972]: https://github.com/kubernetes/enhancements/pull/6169

## Motivation

One of the stated use cases of [KEP-3521] was out-of-tree quota: gate the pod, admit it against
external capacity, release the gate. In-place resize breaks the underlying assumption that
resources are only committed through scheduling. After bind, `pods/resize` can grow a pod past
any externally enforced budget, and the controller has no point at which to hold or reject:

| Project | How it enforces quota today | How resize bypasses it |
| :---- | :---- | :---- |
| **AAQ** (KubeVirt) | Holds new pods with a scheduling gate until they fit its quota | Resize happens after the pod is scheduled, so the gate is never consulted again |
| **Kueue** | Admits a workload before its pods run; deletes pods whose spec drifts from what was admitted | Can only delete a resized pod, never approve a resize that fits; eviction defeats the point of in-place resize |
| **KAI Scheduler** | Checks queue capacity (`limit`, non-preemptible deserved `quota`) when it schedules each pod | A resized pod is not rescheduled, so the capacity check never runs; a `pods/resize` webhook can only check best-effort and races |

In-tree ResourceQuota is not affected: its admission-time ledger already accounts for resize
deltas ([KEP-1287]). The gap is exclusively out-of-tree.

Three projects hit this independently ([kubernetes/kubernetes#131835], [kueue#5257]), and VPA's
in-place modes (`InPlaceOrRecreate`, `InPlace`) will widen it: growth that today arrives through
pod recreation, and therefore through external admission, becomes automated `pods/resize`
traffic.

[kubernetes/kubernetes#131835]: https://github.com/kubernetes/kubernetes/issues/131835
[kueue#5257]: https://github.com/kubernetes-sigs/kueue/issues/5257

### Why denying at admission is not enough

Denying resource growth on `pods/resize` with a validating webhook or a
ValidatingAdmissionPolicy was suggested on [kubernetes/kubernetes#131835] as a workaround. KAI
Scheduler ships this as an interim mitigation ([KAI-Scheduler#1997]), but it is not sufficient
long term:

- Races: two concurrent resizes each fit the remaining budget; together they exceed it. A
  reservation ledger behind the webhook can serialize those, at the cost of consistent shared
  state and cleanup for admissions that never land, and every project rebuilds it. It still
  cannot see the scheduler binding new pods against the same headroom on its own path. With a
  hold, the component that admits new pods also decides when held resizes proceed. Both kinds of
  consumption are decided in one place, against one view of usage, so they cannot race.
- No hold semantics: admission can only accept or deny at request time. Quota controllers often
  need to defer until capacity frees, which is the same reason scheduling gates exist instead of
  deny-at-create.
- Decision latency: a webhook must answer synchronously, within admission timeouts, on the API
  request path. A real capacity decision can be expensive (hierarchical queue state, fair share)
  and belongs in an asynchronous controller loop, not inside an API call.
- Operational cost: every quota project ships its own racing webhook on the pod write path,
  adding latency at the API server and duplicating effort.

[KAI-Scheduler#1997]: https://github.com/kai-scheduler/KAI-Scheduler/pull/1997

### Goals

- Give out-of-tree controllers a point to observe, hold, and admit or reject `pods/resize`
  growth before the kubelet allocates it.
- Keep upstream minimal: the kubelet honors gates; installation, admission logic, and which
  paths to gate are integrator policy.
- Keep the kubelet the source of truth for node-level feasibility (unchanged).

### Non-Goals

- Node-capacity preemption for `Deferred` resizes; that is [KEP-5836].
- A generic in-tree quota API; desirable but much larger in scope.
- Making in-tree ResourceQuota a gate owner. ResourceQuota rejects growth that exceeds quota at
  admission today; it could instead add a gate and release it when quota allows, so one
  mechanism serves in-tree and out-of-tree quota. This is the resize counterpart of [KEP-5465]
  (deferred ResourceQuota enforcement for gated pods), which stalled. It is a separate change
  owned by SIG API Machinery; this proposal neither requires nor precludes it.

## Proposal

[KEP-1287] separates *desired* resources (the spec, written through `/resize`) from *allocated*
resources (checkpointed by the kubelet and reported in status), with the `PodResizePending`
condition (`Deferred` or `Infeasible`) in between. This proposal inserts an externally clearable
hold into that existing state machine:

1. A new pod spec field, `resizeGates`, lists named gates, with the same shape as
   `schedulingGates`.
2. While any gate is present, the kubelet does not allocate a spec whose resources exceed what
   is currently allocated, and reports `PodResizePending` with a new reason, `Gated`.
3. An external controller adds its gate in the same write that grows the pod, decides against
   its own capacity model, and removes the gate to admit the growth.

The gate does not block the API write: a resize against a gated pod is accepted and stored in
the spec, so controllers can see the requested growth. What is held is the kubelet's allocation
of that growth. Down-sizes and updates that do not change resources are never held.

### Notes/Constraints/Caveats (Optional)

- The predicate is growth only. Growth is measured against allocated resources, never against a
  previous desired value: shrinking a pending request (desired 4 → 2 while allocated is 1) is
  still pending growth and stays held. A mixed resize (cpu up, memory down) is treated as growth.
- A gate with no pending growth is dormant and harmless. Gates do not affect initial admission: a
  gate present at pod creation is dormant until the first growth after the pod has been admitted
  to its node.
- The kubelet never mutates gates; only their owners add or remove them.
- Gates are per controller, not per resize. A second growth write while the first is held
  re-adds the same gate name, which is a no-op, and the controller evaluates the latest desired
  resources.
- A mutating webhook is the common way to install a gate, but any actor that mutates the growth
  write can add it: a MutatingAdmissionPolicy, or the resizing controller itself. This document
  says "webhook" for brevity.

### Risks and Mitigations

- **Self-approval.** Removing a gate needs pod update permission, which many users hold, so a
  user with resize permission could remove a gate and approve their own resize. This is the
  same trust model as `schedulingGates` today. Integrators can defend with the arming webhook
  (re-add the gate while growth is pending unless the requester is the controller) or with a
  ValidatingAdmissionPolicy. Whether upstream needs per-gate ownership is an api-review
  question; the status-based alternative avoids the risk by construction (see
  [Alternatives](#alternatives)).
- **Stale controller.** If a gate owner stops running, pending growth on its pods hangs. This
  matches `schedulingGates` today. The `Gated` condition names the pending gates, a metric
  counts held resizes, and the gate owner is responsible for its gates. Growth-only scope bounds
  the damage to upsizes: downsizes always proceed.
- **Webhook unavailability.** With `failurePolicy: Ignore`, a growth write made while the
  integrator's webhook is down is not gated. With `failurePolicy: Fail`, resizes are rejected
  while the webhook is down. This is the integrator's trade-off, as for scheduling gates
  installed by webhook today. The status-based alternative moves this risk to pod creation: a
  pod created while the webhook is down is never gated.
- **Scheduler accounting.** The scheduler sizes a running pod at the maximum of desired and
  allocated resources. Without a change, a gated resize would reserve node room for growth that
  no controller has approved. See [Scheduler interaction](#scheduler-interaction).

## Design Details

### API changes

- New field `pod.spec.resizeGates`, a map-type list of named gates keyed by `name`, with the
  same shape and validation as `schedulingGates` (duplicate names are rejected). Gates can be
  removed through a pod update and added through `/resize`, so a gate lands in the same write
  that grows resources.
- New `PodResizePending` reason `Gated`, alongside `Deferred` and `Infeasible`. The condition
  message names the gates present.
- The kubelet does not allocate resource growth while any gate is present.
- The shared effective-request helper (`PodRequests` in `k8s.io/component-helpers`, used by the
  scheduler) treats `Gated` like `Infeasible` and sizes the pod at its allocated resources while
  gated.

```yaml
# A pod held on a 1 -> 4 cpu resize, before the controller approves
spec:
  containers:
  - name: main
    resources:
      requests:
        cpu: "4"          # desired, written via /resize
  resizeGates:
  - name: example.com/quota   # added by the integrator's webhook in the same write
status:
  containerStatuses:
  - name: main
    allocatedResources:
      cpu: "1"          # unchanged until the gate is removed
  conditions:
  - type: PodResizePending
    status: "True"
    reason: Gated
    message: "resize gated by: example.com/quota"
```

### Kubelet contract

While any gate is present, the kubelet does not allocate a spec whose effective resources exceed
what is currently allocated, and reports `PodResizePending: Gated` with a message naming the
pending gates. The predicate is growth only: down-sizes, container removals, and
resource-neutral updates (for example, image-only) are allocated normally even while gated. Once
the last gate is removed, the kubelet runs its normal admission: node fit, then allocate and
actuate, or `Deferred`/`Infeasible` exactly as today.

Gates are evaluated before node admission: while any gate is present the condition is `Gated`,
even if the resize would otherwise be `Deferred`, and the resize is not part of the kubelet's
deferred-retry set until the last gate is removed.

### Integrator flow

Step numbers in the diagram match the list below.

```mermaid
sequenceDiagram
    participant U as User / VPA
    participant A as API server
    participant W as Integrator webhook
    participant C as Integrator controller
    participant K as Kubelet

    Note over U,W: Step 1: growth write, gate injected
    U->>A: PATCH pods/resize (cpu 1 -> 4)
    A->>W: mutating admission
    W-->>A: add resizeGate example.com/quota
    A-->>U: 200 (spec updated, gate present)

    Note over A,K: Step 2: kubelet holds
    A->>K: watch: spec updated (desired 4, gate present)
    K->>K: desired > allocated, gate present: hold
    K->>A: status: PodResizePending=Gated

    Note over A,C: Step 3: controller decides
    A->>C: watch: pod gated, delta +3
    C->>C: capacity check
    alt fits
        Note over A,C: Step 4a: admit
        C->>A: remove gate (generation precondition)
        Note over A,K: Step 5: kubelet admission
        A->>K: watch: gate removed
        K->>K: node admission
        K->>A: status: allocated cpu=4, condition cleared
    else no headroom
        Note over C: Step 4b: hold
        C-->>C: leave gate, retry when capacity frees
    end
```

1. The controller's mutating webhook on `pods/resize` adds its gate in the same write that grows
   resources (atomic; no bypass window while the webhook is up).
2. The kubelet sees desired above allocated with a gate present, holds, and reports
   `PodResizePending: Gated`.
3. The controller watches pods that carry its gate and have pending growth (reported by the
   kubelet as `Gated`), and runs its capacity logic.
4. Admit: remove its gate, conditional on the pod still being at the spec generation it
   evaluated, so an approval computed for one spec cannot land after a newer, larger growth.
   Hold: leave the gate until capacity frees.
5. The kubelet runs its normal node admission and allocates. If it reports `Infeasible` instead,
   the controller drops the ask from its accounting; the upstream effective-request formula
   already excludes the desired spec for infeasible resizes. If `Deferred`, the resize may still
   actuate later, so the booking stays.

The spec can still change between gate removal and allocation, but no approval goes stale in
the unsafe direction: new growth passes the arming webhook and is re-gated, and a shrink stays
within what was approved.

Hold reasons are surfaced by the gate owner, as for scheduling gates: a pod condition named
after the gate (as readiness gates do), an event, or the owner's own status object. The
kubelet's `Gated` condition names the gates, so the user knows which controller to consult.

### Scheduler interaction

The scheduler sizes a running pod at the maximum of desired and allocated ([KEP-1287]), so a
pending resize keeps its node room. For a `Gated` resize that would reserve capacity for growth
no controller has approved, and a quota controller that relies on the upstream scheduler for
placement (Kueue, for example) could not place other pods into that room while it holds the
resize. `Gated` is therefore treated like `Infeasible` in the shared effective-request helper:
the pod is sized at allocated while any gate is present. A quota controller books the growth
when it approves, regardless of gates other controllers still hold.

This cannot oversubscribe a node, since the kubelet's own admission decides fit. Once the last
gate is removed, the kubelet runs node admission; if the room is gone, the resize becomes
`Deferred` and [KEP-5836] preemption applies as today. Preemption never runs for growth that has
not been approved.

### Example sequences

Desired is `spec.containers[0].resources.requests.cpu`; allocated is
`status.containerStatuses[0].allocatedResources.cpu`; condition is `PodResizePending`.

#### Accepted resize

| Step | Actor | Desired | `spec.resizeGates` | Allocated | Condition |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 0 | Pod running | 1 | none | 1 | none |
| 1 | User patches `/resize` cpu=4; webhook adds gate in the same write | 4 | `example.com/quota` | 1 | none |
| 2 | Kubelet: desired > allocated, gate present | 4 | `example.com/quota` | 1 | `Gated` |
| 3 | Controller: +3 fits, removes its gate | 4 | none | 1 | `Gated` |
| 4 | Kubelet: no gates, node fit ok, allocates and actuates | 4 | none | 4 | none |

#### Held, then canceled by the user

| Step | Actor | Desired | `spec.resizeGates` | Allocated | Condition |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 0-2 | As above | 4 | `example.com/quota` | 1 | `Gated` |
| 3 | Controller: no headroom, leaves the gate | 4 | `example.com/quota` | 1 | `Gated` (message names the gate) |
| 4 | User patches `/resize` cpu=1; a decrease, so no gate is added | 1 | `example.com/quota` | 1 | none (no pending growth; gate dormant) |

If capacity frees later instead, the controller removes the gate and the flow continues as in
step 3 of the accepted case.

#### Second resize while the first is held

| Step | Actor | Desired | `spec.resizeGates` | Allocated | Condition |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 0-2 | As in the accepted case | 4 | `example.com/quota` | 1 | `Gated` |
| 3 | User patches `/resize` cpu=8; webhook re-adds the gate (no-op, already present) | 8 | `example.com/quota` | 1 | `Gated` |
| 4 | Controller evaluates the latest delta (+7) | 8 | `example.com/quota` | 1 | `Gated` |

The same gate covers any number of writes; gates are per controller, not per resize.

#### Approved, then infeasible on the node

| Step | Actor | Desired | `spec.resizeGates` | Allocated | Condition |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 0-3 | As in the accepted case | 4 | none | 1 | `Gated` |
| 4 | Kubelet: 4 exceeds node capacity | 4 | none | 1 | `Infeasible` |
| 5 | Controller sees `Infeasible`, drops the ask from its accounting | 4 | none | 1 | `Infeasible` |

`Deferred` is the same shape, but the resize may still actuate later, so the controller keeps
the booking.

#### Approved, then deferred on the node (KEP-5836)

| Step | Actor | Desired | `spec.resizeGates` | Allocated | Scheduler sizes pod at | Condition |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 0-2 | As in the accepted case | 4 | `example.com/quota` | 1 | 1 (gated: allocated) | `Gated` |
| 3 | Controller removes its gate | 4 | none | 1 | 4 (pending: desired) | `Gated` |
| 4 | Kubelet: node has no room | 4 | none | 1 | 4 | `Deferred` |
| 5 | Scheduler preempts lower-priority pods for the `Deferred` resize ([KEP-5836]) | 4 | none | 1 | 4 | `Deferred` |
| 6 | Kubelet retries, allocates | 4 | none | 4 | 4 | none |

No preemption happens at steps 0-2: the resize is `Gated`, not `Deferred`, and the scheduler
sizes the pod at allocated.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

None identified.

##### Unit tests

Coverage figures to be filled in before the KEP is marked implementable.

- `k8s.io/kubernetes/pkg/apis/core/validation`: validation of `resizeGates` on create, update,
  and `/resize`; duplicate names rejected; field dropped when the feature gate is disabled.
- `k8s.io/kubernetes/pkg/registry/core/pod`: `/resize` strategy allows `resizeGates` changes.
- `k8s.io/kubernetes/pkg/kubelet/allocation`: growth held while a gate is present; downsizes
  and resource-neutral updates allocated; `Gated` takes precedence over `Deferred`; gated
  resizes excluded from the deferred-retry set; release after the last gate is removed.
- `k8s.io/kubernetes/pkg/kubelet/status`: `PodResizePending` condition with reason `Gated` and
  gate names in the message.
- `k8s.io/component-helpers/resource`: `PodRequests` sizes a gated pod at allocated.

##### Integration tests

- API server: `resizeGates` accepted on create and `/resize`, remove-only on pod update,
  duplicate names rejected, field dropped with the feature gate disabled, and preserved on
  update of pods that already have it when the gate is disabled.
- Scheduler: a pod with a gated resize is sized at allocated for node fit.

##### e2e tests

Node e2e (`test/e2e_node`):

- Gated growth is held: allocated resources unchanged, `PodResizePending: Gated`.
- Removing the last gate releases the resize; a downsize proceeds while gated.
- A second growth while held is covered by the same gate.
- After release, an oversized resize reports `Infeasible`, and a resize that no longer fits
  reports `Deferred`.

### Graduation Criteria

#### Alpha

- `resizeGates`, the `Gated` reason, the kubelet hold, and the effective-request change
  implemented behind the `PodResizeGates` feature gate.
- Unit, integration, and node e2e tests listed above.
- A reference integration (KAI Scheduler) validating the flow end to end.

#### Beta

- Feedback from at least one other integrator (Kueue or AAQ).
- The field name and the choice between the proposal and the status-based alternative settled
  with api-review.
- Metric for held resizes exposed by the kubelet.
- `kubectl describe pod` shows the `Gated` condition with gate names.

#### GA

- At least two releases in beta with no open issues on gate semantics.
- Real-world usage by at least two integrators.

The field is optional; pods without gates are unaffected, so conformance tests are not required
for the feature itself.

#### Deprecation

Not applicable.

### Upgrade / Downgrade Strategy

No changes are required on upgrade to keep existing behavior: pods without `resizeGates` behave
as today. To use the feature, the feature gate must be enabled on the API server, kubelet, and
scheduler, and an integrator must install gates.

On downgrade or when the feature gate is disabled, the kubelet ignores `resizeGates` and
allocates pending growth as it does today, so resizes that were held proceed. Integrators must
treat a kubelet without the feature as one that does not enforce gates. The API server drops the
field on writes when the gate is disabled, except for pods that already carry it.

### Version Skew Strategy

- New API server, old kubelet: the kubelet ignores the unknown field and allocates growth as
  today. Gates are not enforced on that node. Integrators should check the kubelet version or
  fall back to their existing mitigation for such nodes.
- New kubelet, old API server: the field is never persisted, so nothing is gated.
- Old scheduler: it sizes a gated pod at its desired resources (the current rule), which
  over-reserves node room but is safe.
- Old controllers: unaffected; they do not read the field.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `PodResizeGates`
  - Components depending on the feature gate: kube-apiserver, kubelet, kube-scheduler

###### Does enabling the feature change any default behavior?

No. Pods without `resizeGates` behave exactly as today. Behavior changes only for pods on which
an integrator has installed a gate.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the gate on the kubelet makes it ignore `resizeGates`, so held resizes proceed
through normal node admission. Disabling it on the API server drops the field from new writes.
Existing gates on pods are preserved but have no effect.

###### What happens if we reenable the feature if it was previously rolled back?

Gates still present on pods are enforced again. Growth that was allocated while the feature was
disabled is not undone.

###### Are there any tests for feature enablement/disablement?

Unit tests for the API server dropping and preserving the field with the gate on and off, and
for the kubelet ignoring gates with the gate off, will be added with the implementation.

### Rollout, Upgrade and Rollback Planning

To be completed when targeting beta.

###### How can a rollout or rollback fail? Can it impact already running workloads?

A partial rollout across nodes means some kubelets enforce gates and others do not; integrators
see growth actuated without approval on nodes without the feature. Running workloads are not
disrupted: the feature only adds a hold before allocation.

###### What specific metrics should inform a rollback?

A high or growing count of held resizes with no corresponding gate removals may indicate a
stuck integrator rather than a problem with the feature; disabling the feature releases them.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

To be done before beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

To be completed when targeting beta.

###### How can an operator determine if the feature is in use by workloads?

Pods with a non-empty `spec.resizeGates`, and the `kubelet_pod_gated_resizes_total` metric.

###### How can someone using this feature know that it is working for their instance?

- [x] API .status
  - Condition name: `PodResizePending` with reason `Gated` while a resize is held; cleared or
    replaced by `Deferred`/`Infeasible` once the last gate is removed.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Once the last gate is removed, the kubelet acts on the resize within the same bounds as an
ungated resize today.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - Metric name: `kubelet_pod_gated_resizes_total`
  - Components exposing the metric: kubelet

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

A gauge of currently held resizes per node may be added if the counter proves insufficient.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No in-tree dependency. Its usefulness depends on an out-of-tree controller installing and
removing gates.

### Scalability

###### Will enabling / using this feature result in any new API calls?

One pod update per admitted resize (gate removal) by the integrator, and one mutating admission
call per growth write for integrators that use a webhook. Integrators already watch pods.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Pods gain a `resizeGates` list, roughly the size of `schedulingGates`: tens of bytes per gate,
only on pods where an integrator installs one.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No. Resize actuation for gated pods waits for the integrator by design; ungated pods are
unaffected.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No. The kubelet performs one additional comparison per resize evaluation.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

The kubelet keeps holding gated resizes based on its cached pod spec. Controllers cannot remove
gates until the API server is back, so held resizes stay held. Ungated behavior is unchanged.

###### What are other known failure modes?

- Stale gate owner
  - Detection: `PodResizePending: Gated` persists; `kubelet_pod_gated_resizes_total` grows.
  - Mitigations: remove the gate with a pod update; fix or disable the controller.
  - Diagnostics: the condition message names the gate.
- Integrator webhook down with `failurePolicy: Ignore`
  - Detection: growth allocated on pods that should have been gated.
  - Mitigations: integrator-side reconciliation; use `failurePolicy: Fail` if resize
    availability is less important than enforcement.

###### What steps should be taken if SLOs are not being met to determine the problem?

Check the `Gated` condition on the pod for the gate names, then the state of the named
controllers.

## Implementation History

- 2025-05: [kubernetes/kubernetes#131835] opened by the AAQ maintainers.
- 2026-08: Issue reopened and accepted for triage; design outline shared with SIG Node and
  SIG Scheduling participants.
- 2026-09: Presented at the SIG Scheduling meeting.
- 2026-09-28: Tracking issue [kubernetes/enhancements#6438](https://github.com/kubernetes/enhancements/issues/6438) opened; KEP opened as provisional.

## Drawbacks

- Adds a field to the Pod spec for a use case served only by out-of-tree controllers.
- Integrators need a mutating admission step on the resize path to install gates, with the
  usual webhook availability trade-offs.
- Gate removal uses pod update permission, so the self-approval risk of `schedulingGates` is
  inherited (see [Risks and Mitigations](#risks-and-mitigations)).

## Alternatives

### Persistent gates with approvals in status

Gate names are declared in spec at creation (remove-only, like `schedulingGates`); controllers
approve by writing `status.resizeApprovals: [{name, approvedGeneration}]`, and the kubelet holds
pending growth while any gate's approved generation differs from `metadata.generation`. The
kubelet only reads, so there is no per-resize webhook and a single writer, and approvals never
need clearing: any later spec change makes them stale. That includes unrelated changes, which
then need a re-approval. Binding the approval to the approved pod requests instead avoids this,
since only a larger request then invalidates it, but the kubelet must clear approvals it has
allocated so a later downsize and re-upsize is re-approved.

Approval is a `pods/status` write, which users with resize permission don't normally hold, so a
user cannot approve their own resize; in the main proposal, removing a gate needs only pod
update, the same permission as removing a `schedulingGate`, which users usually have.

```yaml
spec:
  containers:
  - name: main
    resources:
      requests:
        cpu: "8"          # desired, written via /resize (generation 5)
  resizeGates:
  - name: example.com/quota   # static, set at creation
status:
  resizeApprovals:
  - name: example.com/quota
    approvedGeneration: 4     # written by the controller for the previous request
  conditions:
  - type: PodResizePending
    status: "True"
    reason: Gated
    message: "approvedGeneration is 4, but metadata.generation is 5"
```

| Step | Actor | Desired | Generation | `status.resizeApprovals` | Allocated | Condition |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 0 | Pod created with `spec.resizeGates: [example.com/quota]` | 1 | 1 | none | 1 | none |
| 1 | User patches `/resize` cpu=4; no webhook | 4 | 2 | none | 1 | none |
| 2 | Kubelet: desired > allocated, no approval for generation 2 | 4 | 2 | none | 1 | `Gated` |
| 3 | Controller approves generation 2 | 4 | 2 | `approvedGeneration: 2` | 1 | `Gated` |
| 4 | Kubelet: approval matches the generation, allocates | 4 | 2 | `approvedGeneration: 2` | 4 | none |

### Optimistic admission (actuate, then reclaim)

The kubelet actuates immediately and the controller checks quota in parallel, shrinking the pod
back if it rejects. This needs no API and is available today by not gating a pod, so integrators
can use it for latency-sensitive workloads. It is not suitable as the default for hard quota:
the pod runs over quota until the controller reacts; the shrink is best-effort for memory (per
[KEP-1287], the kubelet skips a memory limit decrease below current usage), so the reject may
never take effect; and the controller overwrites the requested size, which a resizing controller
such as VPA would then re-apply.

### Comparison

|  | Proposal: gates re-armed per resize | Persistent gates, approvals in status | Optimistic admission |
| :---- | :---- | :---- | :---- |
| New API | `spec.resizeGates`, `Gated` reason | same, plus `status.resizeApprovals` | none |
| Webhook on the resize path | yes | no (only at creation) | no |
| Approval write | remove a gate (pod update) | write to `pods/status` | shrink the pod (`/resize`) |
| Self-approval by a resize-permitted user | possible with pod update (as for `schedulingGates`); mitigate with the arming webhook or a VAP | not possible without `pods/status` permission | n/a |
| Re-arm for the next resize | webhook re-adds the gate | any spec change stales the approval, including unrelated ones | n/a |
| Safe for hard quota | yes | yes | no: over quota until the controller reacts, revert not guaranteed |
| Symmetry with `schedulingGates` | exact | partial | none |

### Open questions

#### Gate mutability and authorization

Unlike `schedulingGates`, `resizeGates` must be addable after creation. Any pod writer can also
remove a gate, so a resize-permitted user could self-approve; integrators can defend with the
arming webhook or a ValidatingAdmissionPolicy. Whether upstream needs per-gate ownership is for
api-review. The status-based alternative avoids this by construction.

#### Scope of the hold

Growth only (proposed), or every resize while gated? Growth-only keeps downsizes immediate
(including a user canceling a pending request), bounds a stale controller's damage to upsizes,
and spares integrators approving capacity-safe shrinks. Static floors on downsizing are already
served by a ValidatingAdmissionPolicy, LimitRange, or VPA's `minAllowed`. A shrink that must
wait for something (draining in-flight work, reconfiguring the app) is a hold rather than a
floor and would need a gate; a per-gate scope (`Growth` by default, `All`) could serve it later,
added without breaking anything, at the cost of a stale controller blocking reclamation.

#### Field name

`resizeGates` is the working name, consistent with the `PodResizePending` and
`PodResizeInProgress` conditions, which already cover container additions under the dynamic
containers proposal ([KEP-5972]). Alternatives raised: `allocationGates` and
`reallocationGates` (kubelet-internal, and the former suggests gating initial admission),
`kubeletAllocationGates`, `updateGates` (too generic), and `resourceChangeGates` (general and
admission-neutral, longer). Final name for api-review.

## Infrastructure Needed

None.
