---
layout: post
title: Kubernetes Gatekeeper
date: 2023-08-04 17:00:00 +0100
categories: learning
tags: kubernetes, k8s, gatekeeper
---

## Introduction

In the early day, Kubernetes offer [PodSecurityPolicy](https://kubernetes.io/docs/concepts/security/pod-security-policy/). However, it was marked deprecated in version 1.21. Kubernetes team introduce [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) but it's still in beta and there's no customize policy without involving other components as of the writing date (2023-08-04).

One of the recommendation from kubernetes is Gatekeeper.

This document is the result of my self-study regarding the Gatekeeper.

## Prerequisite

You should have the basic knowledge of:

- [Kubernetes](https://kubernetes.io/docs/tutorials/)
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/)

## Install

In this documentation we will follow [the official documentation](https://open-policy-agent.github.io/gatekeeper/website/docs/install).

Before install anything we need to have the admin permission:

```bash
  kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole cluster-admin \
    --user <YOUR USER NAME>
```

In this document, we will install via helm.

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace
```

## How to use Gatekeeper

Gatekeeper is using [OPA Constraint Framework](https://github.com/open-policy-agent/frameworks/tree/master/constraint), which comprise of `constraint` and `constraint template`.

### Constraint Templates

Before you can define a constraint, you must first define a
[`Constraint Template`](https://open-policy-agent.github.io/gatekeeper/website/docs/constrainttemplates),
which describes both the [Rego](https://www.openpolicyagent.org/docs/latest/#rego)
that enforces the constraint and the schema of the constraint.[^1]

Constraint Template is the a place where we define the rule (what and where to check) and receive the requirement value as an input from `constraint` via `parameters` field. The desired input schema of `constraint` file is specified the `validation` field as shown below.

#### Example Constraint template

```yaml
apiVersion: gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: foosystemrequiredlabels

spec:
  crd:
    spec:
      names:
        kind: FooSystemRequiredLabel
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items: string

  targets:
    - target: admission.k8s.gatekeeper.sh
      libs:
        - |
          package lib.helpers

          make_message(missing) = msg {
            msg := sprintf("you must provide labels: %v", [missing])
          }

      rego: |
        package foosystemrequiredlabels

        import data.lib.helpers

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
           provided := {label | input.request.object.metadata.labels[label]}
           required := {label | label := input.parameters.labels[_]}
           missing := required - provided
           count(missing) > 0
           msg := helpers.make_message(missing)
        }
```

From the example above, In the `constraint` file, it must contain `labels` key which contain `array` of `string`. Parameters explanation is below.

- `validation`, which provides the schema for the `parameters` field for the constraint
- `targets`, which specifies what "target" (defined later) the constraint applies to.
Note that currently constraints can only apply to one target.
- `rego`, which defines the logic that enforces the constraint.
- `libs`, which is a list of all library functions that will be available to the `rego` package.
Note that all packages in `libs` must have `lib` as a prefix (e.g. `package lib.<something>`)

#### Rego rules

```rego
package loremIpsum

violation[{"msg": msg, "details": {}}] {
  # rule body
}
```

In this framework, there are some specific rules that are not with traditional rego.

- the rule name must be `violation`
- `msg` is the string message returned to the violator. This is a required parameter
- `details` is the optional parameter without pre-defined schema which allows
for custom values to be returned. This helps support uses like automated remediation.

### Constraint

Constraint is the location where the requirement value is set.
Please find the corresponding constraint below.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-gk
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["gatekeeper"]
```

From the example above, once this constraint is enforced,
all objects in the `expensive` namespace will be required to have a `billing` label.

To decide which resources will be enforced, there's a match field.
Please find the details about this field
[here](https://open-policy-agent.github.io/gatekeeper/website/docs/howto/#the-match-field).
We can also specify exception here as well.

### PodSecurityPolicy replacement rules

In Gatekeeper library website, there are pod-security-policies equivalent rule that we can take it as a primer and customize to our need [here](https://open-policy-agent.github.io/gatekeeper-library/website/pspintro)

## Reference

1. [UID and GID best practices](https://www.redhat.com/sysadmin/user-account-gid-uid)
2. [PodSecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#podsecuritycontext-v1-core)
3. [SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#securitycontext-v1-core)

[^1]: [Constraint Template](https://open-policy-agent.github.io/gatekeeper/website/docs/howto#constraint-templates)
