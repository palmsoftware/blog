---
layout: post
title: "Running OpenShift on GitHub Actions Free Runners"
date: 2026-03-17 10:00:00
author: brandon
tags: [openshift, github-actions, ci, devops]
excerpt: "Test OpenShift operators and platform features in CI without cloud costs—using free GitHub Actions runners."
---

## Introduction

If you're building operators, controllers, or applications for Red Hat OpenShift, you need real integration tests. Mocking the OpenShift API misses edge cases, and spinning up cloud clusters for every PR burns time and money. What if you could run full OpenShift clusters directly in GitHub Actions on free-tier runners?

That's exactly what [quick-ocp](https://github.com/palmsoftware/quick-ocp) does. It provisions ephemeral OpenShift Local clusters in your CI workflows, giving you production-like testing without the infrastructure overhead. Here's why you should use it.

## Why OpenShift-Specific Testing Matters

Vanilla Kubernetes isn't enough when you're targeting OpenShift. The platform adds:

- **OpenShift-specific APIs**: Route, SecurityContextConstraints, Project, ImageStream
- **Built-in operators**: OpenShift Ingress, DNS, monitoring stack, internal registry
- **Different RBAC and security models**: SCCs, project-level isolation, stricter defaults
- **Platform conventions**: How networking, storage, and authentication work in production

If your operator uses Routes instead of Ingress, relies on OpenShift's built-in monitoring, or needs to validate against SCCs, you can't test it properly on plain Kubernetes. You need the real platform.

## What quick-ocp Gives You

### 1. Production-Like OpenShift Clusters in CI

Every workflow run gets a fresh OpenShift Local cluster with:
- Full OpenShift API (not just Kubernetes + add-ons)
- Built-in operators and platform components
- OpenShift's RBAC, security, and networking models
- Configurable OpenShift versions (4.18, 4.19, 4.20, 4.21, or latest)

No mocks, no compromises—just real OpenShift running in GitHub Actions.

### 2. Fast Startup with Bundle Caching

First run: ~10-12 minutes to download the OpenShift bundle and start the cluster.

With caching enabled:
```yaml
with:
  bundleCache: true
```

Subsequent runs: ~4-6 minutes. The 8GB bundle restores from GitHub's cache in seconds instead of re-downloading.

### 3. Automatic Resource Optimization

GitHub Actions free-tier runners for public repositories provide ([source](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)):
- **4 vCPU cores**
- **16 GB RAM**
- **14 GB free disk space**

OpenShift Local needs 31GB of disk space. `quick-ocp` automatically:
- Clears 25-30GB by removing unused pre-installed software (Android SDK, .NET, language toolchains)
- Relocates Docker and CRC storage to the larger `/mnt` partition
- Creates swap space to handle memory spikes
- Protects critical processes from the OOM killer

You don't have to think about disk cleanup, memory tuning, or resource constraints—it just works.

### 4. Version Pinning Prevents Broken Releases

OpenShift Local releases occasionally ship with issues. CRC 2.55.0 and 2.55.1 with OpenShift 4.19 had expired kube-scheduler certificates, causing cryptic startup failures.

`quick-ocp` includes a version pinning mechanism that:
- Maps OpenShift versions to known-good CRC releases
- Documents known issues with broken versions
- Automatically avoids problematic releases

Your CI workflows stay stable even when upstream releases break.

### 5. Flexible Configuration

Pin a specific OpenShift version:
```yaml
with:
  desiredOCPVersion: 4.19
```

Wait for all operators to be ready:
```yaml
with:
  waitForOperatorsReady: true
```

Adjust resources if needed:
```yaml
with:
  crcCpu: 4
  crcMemory: 10752
  crcDiskSize: 31
```

Override CRC version for testing:
```yaml
with:
  crcVersion: 2.54.0
```

### 6. Zero Cloud Costs

This runs on GitHub Actions free-tier runners. No AWS, GCP, or Azure clusters to provision. No ARO, ROSA, or cloud marketplace charges. Just your existing GitHub Actions minutes.

For public repositories, those minutes are unlimited.

## Real-World Use Cases

### Operator Testing

Test your OpenShift operator's full lifecycle:
```yaml
steps:
  - uses: palmsoftware/quick-ocp@v0.0.16
    with:
      ocpPullSecret: ${{ secrets.OCP_PULL_SECRET }}
      waitForOperatorsReady: true
  
  - name: Install operator
    run: |
      oc apply -f deploy/operator.yaml
      oc wait --for=condition=ready pod -l app=my-operator --timeout=300s
  
  - name: Test operator functionality
    run: |
      oc apply -f examples/custom-resource.yaml
      # Verify operator handles CR correctly
```

### Route and Ingress Testing

Validate OpenShift Routes (not just Kubernetes Ingress):
```yaml
steps:
  - uses: palmsoftware/quick-ocp@v0.0.16
    with:
      ocpPullSecret: ${{ secrets.OCP_PULL_SECRET }}
  
  - name: Test Route creation
    run: |
      oc expose service my-app --name=my-route
      oc get route my-route -o jsonpath='{.spec.host}'
```

### SCC Validation

Test SecurityContextConstraints compatibility:
```yaml
steps:
  - uses: palmsoftware/quick-ocp@v0.0.16
    with:
      ocpPullSecret: ${{ secrets.OCP_PULL_SECRET }}
  
  - name: Test SCC policies
    run: |
      oc apply -f manifests/restricted-deployment.yaml
      # Verify deployment works with OpenShift's default restricted SCC
```

## Getting Started

### 1. Get an OpenShift Pull Secret

Visit [console.redhat.com](https://console.redhat.com/openshift/install/azure/aro-provisioned), click "Download Pull Secret", and add it to your repository's secrets as `OCP_PULL_SECRET`.

### 2. Add to Your Workflow

```yaml
name: Test Operator

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up OpenShift
        uses: palmsoftware/quick-ocp@v0.0.16
        with:
          ocpPullSecret: ${{ secrets.OCP_PULL_SECRET }}
          bundleCache: true
      
      - name: Run tests
        run: |
          oc get nodes
          # Your tests here
```

That's it. The action handles cluster provisioning, resource optimization, and cleanup automatically.

## When to Use quick-ocp vs quick-k8s

**Use quick-ocp when:**
- Testing OpenShift operators or platform features
- Using OpenShift-specific APIs (Route, SCC, Project, ImageStream)
- Validating against OpenShift's security and RBAC models
- Need built-in operators (monitoring, registry, DNS)

**Use [quick-k8s](https://github.com/palmsoftware/quick-k8s) when:**
- Building generic Kubernetes applications
- Testing with KinD or Minikube
- Need faster startup (<2 minutes vs 4-6 minutes)
- Don't require OpenShift-specific features

Both actions work on the same free-tier runners and can be used together in different workflows.

## Key Takeaways

- **Test real OpenShift in CI** without cloud infrastructure or costs
- **Bundle caching** makes subsequent runs 2x faster
- **Automatic resource management** handles disk cleanup and memory optimization
- **Version pinning** prevents broken releases from breaking your CI
- **Works on free-tier runners** for public repositories

## Conclusion

If you're building for OpenShift and want proper integration testing in CI, [quick-ocp](https://github.com/palmsoftware/quick-ocp) eliminates the choice between "mock it and hope" or "pay for cloud clusters." You get real OpenShift, real testing, and zero infrastructure overhead.

Get started by adding your pull secret to GitHub Secrets and using the action in your workflows. If you hit issues or have feature requests, [open an issue](https://github.com/palmsoftware/quick-ocp/issues) or contribute a PR.

See the [README](https://github.com/palmsoftware/quick-ocp#readme) for complete documentation and advanced configuration options.
