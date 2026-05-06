---
layout: post
title: "See What Your Cluster Is Really Serving: TLS Visibility with the tls-compliance-operator"
date: 2026-05-06
tags: [kubernetes, openshift, tls, security, pqc]
excerpt: "A Kubernetes operator that auto-discovers every TLS endpoint in your cluster and tells you exactly what it's serving -- TLS versions, cipher grades, certificate health, and post-quantum readiness."
---

You know your apps are running. But do you know what TLS versions they're negotiating? What cipher suites they're using? When their certificates expire? Whether they're ready for post-quantum cryptography?

Most teams don't. Not because they don't care, but because there's no easy way to get a cluster-wide view of TLS health. The [tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator) changes that.

## What It Does

The tls-compliance-operator is a Kubernetes operator that auto-discovers every TLS endpoint in your cluster (Services, Ingresses, OpenShift Routes, and Pods) and creates a `TLSComplianceReport` custom resource for each one. No configuration needed. Deploy it and start seeing what your cluster is actually serving.

It also evaluates endpoints against [OpenShift TLS security profiles](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/tls-security-profiles), so you can see whether your workloads meet the same standards as the platform itself.

## Getting Started

One command:

```
kubectl apply -f https://github.com/sebrandon1/tls-compliance-operator/releases/download/v1.0.3/install.yaml
```

Within minutes, the operator discovers all TLS-speaking endpoints across the cluster and starts creating reports.

## What You See Immediately

Here's what `kubectl get tlsreport` shows on a live OpenShift 4.22 cluster with a mix of platform services and demo workloads:

```
$ kubectl get tlsreport
NAME                                             HOST                                       PORT  SOURCE  COMPLIANCE  GRADE  FS    TLS1.3  TLS1.2  PQC           CERTEXPIRY
router-internal-default-openshift-ingress-...    router-internal-default.openshift-ingress   443   Service Compliant   A      true  true    true    PQCReady      703
console-openshift-console-443-...                console.openshift-console                   443   Service Compliant   A      true  true    true    PQCReady      703
web-tls13-tls-blog-demo-443-...                  web-tls13.tls-blog-demo                     443   Service Compliant   A      true  true    false   TLS13Capable  364
web-tls12-tls-blog-demo-443-...                  web-tls12.tls-blog-demo                     443   Service Compliant   A      true  false   true    LegacyTLS     364
web-expired-tls-blog-demo-443-...                web-expired.tls-blog-demo                   443   Service Compliant   A      true  true    true    TLS13Capable  -489
web-http-tls-blog-demo-443-...                   web-http.tls-blog-demo                      443   Service NoTLS              false false   false
```

Every column is a question answered at a glance. Is the endpoint compliant? What grade are its ciphers? Does it support forward secrecy? Is TLS 1.3 negotiating? Is the certificate about to expire? Is it ready for post-quantum crypto?

And notice: the operator auto-discovered the OpenShift platform services (router, console, API server) alongside the demo workloads. Zero configuration.

## Digging Deeper

`kubectl describe` gives you the full picture. Here's a TLS 1.3-only endpoint:

```
$ kubectl describe tlsreport web-tls13-tls-blog-demo-443-23323ee0
Status:
  Compliance Status:     Compliant
  Overall Cipher Grade:  A
  Forward Secrecy:       true
  Tls Versions:
    tls12:  false
    tls13:  true
  Cipher Suites:
    TLS 1.3:
      TLS_AES_128_GCM_SHA256
  Key Exchange Types:
    TLS 1.3:  TLS13
  Negotiated Curves:
    TLS 1.3:  X25519
  Certificate Info:
    Issuer:            CN=tls-demo
    Days Until Expiry: 364
    Not After:         2027-05-06T20:41:44Z
  Conditions:
    Type: TLSCompliant      Status: True   Reason: Compliant
    Type: CertificateValid  Status: True   Reason: Valid
    Type: PQCCompliant      Status: False  Reason: TLS13Capable
```

TLS version support, the exact cipher suite negotiated, key exchange type, curve used, certificate details with expiry countdown, and structured conditions you can query programmatically. All in one place.

## Catching Problems

The operator doesn't just report. It alerts. Here's what happens when a certificate is expired:

```
$ kubectl describe tlsreport web-expired-tls-blog-demo-443-d38a652c
  Certificate Info:
    Days Until Expiry:  -489
    Is Expired:         true
    Issuer:             CN=expired-demo
    Not After:          2025-01-02T00:00:00Z
  Conditions:
    Type: CertificateValid  Status: False  Reason: Expired
Events:
  Warning  CertificateExpired  8m  tls-compliance-controller  TLS certificate has expired for web-expired.tls-blog-demo:443
```

Kubernetes events fire for expired and expiring certificates. On our test cluster, the operator also caught real issues: `cert-manager-webhook` expiring in 5 days and `kubelet` expiring in 19 days:

```
$ kubectl get events --field-selector reason=CertificateExpiring
Warning  CertificateExpiring  cert-manager-webhook-cert-manager-443-...  TLS certificate expires in 5 days
Warning  CertificateExpiring  kubelet-kube-system-10250-...              TLS certificate expires in 19 days
Warning  CertificateExpiring  web-expiring-tls-blog-demo-443-...         TLS certificate expires in 6 days
```

These are real findings on a running cluster, the kind of visibility you don't have until something breaks.

The operator also detects non-TLS endpoints (like our plain HTTP service showing `NoTLS`) and ports that aren't responding (`Closed`, `Timeout`). This helps you understand what's actually listening across your cluster.

## Post-Quantum Readiness

With [OpenShift driving post-quantum cryptography adoption](https://www.redhat.com/en/blog/road-to-quantum-safe-cryptography-red-hat-openshift) and [PQC support landing in the OCP 4.20 control plane](https://www.redhat.com/en/blog/deeper-look-post-quantum-cryptography-support-red-hat-openshift-420-control-plane), knowing where you stand today is critical. The operator already detects PQC key exchange algorithms.

Here's what a PQC-ready endpoint looks like on our OCP 4.22 cluster:

```
  Negotiated Curves:
    TLS 1.2:  X25519
    TLS 1.3:  X25519MLKEM768
  Pqc Readiness:  PQCReady
  Quantum Ready:  true
```

`X25519MLKEM768` is the hybrid post-quantum key exchange, combining classical X25519 with [ML-KEM](https://www.redhat.com/en/topics/security/post-quantum-cryptography), the NIST-standardized lattice-based algorithm. The operator categorizes every endpoint into one of four PQC readiness levels:

- **PQCReady**: Already negotiating post-quantum key exchange
- **TLS13Capable**: Supports TLS 1.3 (upgrade path exists) but not yet negotiating PQC
- **LegacyTLS**: Only supports TLS 1.2, no path to PQC without a protocol upgrade
- **NoPQC**: No TLS at all

On our test cluster, every OCP platform service is already PQCReady. Our TLS 1.3-only demo service is TLS13Capable: it speaks 1.3 but the server's crypto library hasn't enabled ML-KEM yet. And our TLS 1.2-only service is LegacyTLS, needing a TLS 1.3 upgrade before PQC is even possible.

This is exactly the inventory you need for a [PQC migration plan](https://www.redhat.com/en/blog/if-how-year-post-quantum-reality). As [RHEL 10.1 makes PQC generally available](https://www.redhat.com/en/blog/whats-new-post-quantum-cryptography-rhel-101) across Go, OpenSSL, GnuTLS, and NSS, the gap between "platform ready" and "workload ready" is the one you need to close.

## Prometheus Integration

For teams running Prometheus, the operator exposes metrics you can alert on:

```promql
# What percentage of endpoints are compliant?
sum(tls_compliance_endpoints_total{status="Compliant"}) / sum(tls_compliance_endpoints_total) * 100

# Which certificates expire within 7 days?
tls_compliance_certificate_expiry_days < 7

# Any endpoints still on TLS 1.0?
tls_compliance_version_support{version="1.0"} == 1

# Which endpoints are PQC-ready?
tls_compliance_pqc_readiness{readiness="PQCReady"} == 1
```

Pre-built alerting rules catch non-compliant endpoints, expiring certificates, and slow scan cycles. A Grafana dashboard JSON is included in the repo for immediate visualization.

## Wrapping Up

The tls-compliance-operator gives you immediate, continuous visibility into the TLS health of your entire cluster. One deploy command, zero configuration, and you go from "I assume our TLS is fine" to "I can see exactly what every endpoint is serving."

Check it out at [github.com/sebrandon1/tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator).

### Further Reading

- [Configuring TLS security profiles in OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/tls-security-profiles)
- [The road to quantum-safe cryptography in Red Hat OpenShift](https://www.redhat.com/en/blog/road-to-quantum-safe-cryptography-red-hat-openshift)
- [PQC support in the OpenShift 4.20 control plane](https://www.redhat.com/en/blog/deeper-look-post-quantum-cryptography-support-red-hat-openshift-420-control-plane)
- [What is post-quantum cryptography?](https://www.redhat.com/en/topics/security/post-quantum-cryptography)
- [What's new in PQC in RHEL 10.1](https://www.redhat.com/en/blog/whats-new-post-quantum-cryptography-rhel-101)
