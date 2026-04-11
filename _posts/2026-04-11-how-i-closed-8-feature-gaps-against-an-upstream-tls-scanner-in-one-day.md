---
layout: post
title: "How I Closed 8 Feature Gaps Against an Upstream TLS Scanner in One Day"
date: 2026-04-11
tags: [go, kubernetes, tls, security]
excerpt: "A walkthrough of adding PQC compliance, SSLv3 detection, forward secrecy reporting, and more to a Kubernetes TLS compliance operator — inspired by the upstream OpenShift tls-scanner."
---

## Introduction

One of the best ways to improve a project is to study what others are building in the same space. The [openshift/tls-scanner](https://github.com/openshift/tls-scanner) is a well-designed batch-mode TLS auditing tool for OpenShift clusters, and comparing it feature-by-feature against my [tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator) revealed several capabilities worth adopting.

This post walks through how I added 8 features to the operator in a single session, inspired by the upstream scanner's approach — and how the two tools complement each other for different use cases.

## Two Tools, Two Approaches

The upstream tls-scanner and the tls-compliance-operator solve the same core problem — auditing TLS endpoints across a Kubernetes cluster — but they take fundamentally different approaches.

The scanner runs as a one-shot Kubernetes Job, leveraging `testssl.sh` for deep protocol analysis. It excels at point-in-time audits with rich output formats (CSV, JSON, JUnit). The operator runs continuously as a Kubernetes controller, creating CRD-based reports with Prometheus metrics, Kubernetes events, and automatic periodic rescanning. Each approach has strengths the other doesn't, and organizations benefit from having both options.

By studying the scanner's feature set, I identified 10 capabilities worth bringing to the operator side. Here's what I built.

## What I Added

### 1. Post-Quantum Cryptography Compliance Classification

The upstream scanner classifies endpoints by PQC readiness — whether the server supports TLS 1.3 with ML-KEM, the NIST-standardized post-quantum key exchange algorithm. I adopted the same classification model as a `PQCReadiness` enum with four levels: `PQCReady`, `TLS13Capable`, `LegacyTLS`, and `NoPQC`. This is paired with a Kubernetes condition, Prometheus metric, and events on readiness changes.

### 2. Health Probe Port Filtering

The scanner flags health probe ports to avoid false positives from plaintext health check endpoints. I brought the same concept to the operator: inspect each container's liveness, readiness, and startup probe definitions, resolve named ports, and skip HTTP/TCP/gRPC probe ports during discovery. HTTPS probes are still scanned but flagged for visibility.

### 3. Forward Secrecy Status

The scanner reports forward secrecy status explicitly. The operator already collected the cipher data needed to determine this — ephemeral key exchange (ECDHE/DHE) was implicit in the A/B cipher grades — but it wasn't surfaced as its own field. Adding a `ForwardSecrecy` boolean with a dedicated metric made this information immediately queryable.

### 4. Key Exchange Type Details

Similarly, the key exchange algorithm (ECDHE, DHE, RSA, or TLS 1.3's implicit ephemeral exchange) was derivable from cipher suite names but never reported separately. I added `KeyExchangeTypes` as a per-TLS-version map, complementing the cipher grade and forward secrecy fields.

### 5. SSLv3 Detection via Raw Socket Probe

This was the most interesting implementation challenge. Go's `crypto/tls` library removed SSLv3 support entirely, so you can't detect it through normal TLS dialing. The solution: craft a raw SSLv3 ClientHello at the byte level, send it over a TCP socket, and parse the ServerHello response.

```go
var ssl30ClientHello = buildSSL30ClientHello()

func (c *TLSChecker) ProbeSSL30(ctx context.Context, addr string) bool {
    conn, err := dialer.DialContext(ctx, "tcp", addr)
    // ... send pre-built ClientHello, check for SSLv3 ServerHello ...
}
```

The upstream scanner handles this through `testssl.sh`, which has its own SSL probing built in. Going the raw socket route keeps the operator dependency-free.

### 6. IPv6 and Dual-Stack Support

Pod endpoint extraction was only reading the singular `pod.Status.PodIP`. On dual-stack clusters, `pod.Status.PodIPs` contains both IPv4 and IPv6 addresses. The fix creates endpoints for each IP, converts IPv6 colons to hyphens in CR names for readability, and uses `net.JoinHostPort` throughout for correct `[::1]:443` formatting.

### 7–8. Bonus: Image Tagging Cleanup

While working on the SSLv3 PR, I noticed the build workflow was creating a per-commit SHA tag on every push to main — cluttering the container registry with 70+ tags. I replaced this with a single `unstable` tag that gets overwritten, keeping only release version tags and `latest`.

## The Review Process

Each feature went through an automated `/simplify` code review — three parallel agents checking for code reuse opportunities, quality issues, and efficiency concerns. These reviews consistently caught real problems:

- A **subtle operator precedence bug** in the SSLv3 byte assembly where `len*2>>8` was computing `len*(2>>8)` instead of `(len*2)>>8`
- A **stdlib reimplementation** where a custom `readFull` function duplicated `io.ReadFull`
- **Redundant string parsing** where three functions independently checked the same cipher name prefixes
- A **data loss bug** where key exchange type extraction only examined the first cipher suite per TLS version

## Results

Six PRs merged as [v0.0.14](https://github.com/sebrandon1/tls-compliance-operator/releases/tag/v0.0.14). The operator and the upstream scanner now share a common feature vocabulary — PQC readiness levels, health probe port awareness, forward secrecy status — which benefits users of both tools. Someone using the scanner for periodic audits and the operator for continuous monitoring will see consistent classification across both.

## Key Takeaways

- **Study adjacent projects** — comparing against tools in the same space reveals features and patterns you wouldn't discover in isolation, and both projects benefit from the cross-pollination.
- **Different architectures serve different needs** — batch scanning and continuous monitoring are complementary, not competing. The best TLS compliance strategy uses both.
- **Post-quantum readiness is here** — TLS 1.3 with ML-KEM is already negotiated by major endpoints. Compliance tooling that classifies PQC readiness is no longer premature.
- **Automated code review catches real bugs** — the precedence bug in the SSLv3 byte assembly would have been invisible in testing since it happened to produce correct output for small inputs.

## Conclusion

The upstream OpenShift tls-scanner set a high bar for TLS auditing features, and studying it made the tls-compliance-operator significantly better. Both tools are open source, and I'd encourage anyone working on cluster security to try both — the scanner for deep one-off audits, and the operator for continuous compliance monitoring.

Check out the operator at [github.com/sebrandon1/tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator), and the upstream scanner at [github.com/openshift/tls-scanner](https://github.com/openshift/tls-scanner).
