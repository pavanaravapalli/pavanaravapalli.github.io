---
layout: post
title: "Licensing for distributed control planes"
date: 2021-08-01 11:30:00 +0530
categories: [infrastructure,architecture,design]
---

A licensing design for multi-node control planes that works online and air-gapped, and degrades to read-only instead of taking the platform down.

This post describes a generic, vendor-agnostic architecture for any multi-node platform, such as a cloud management engine or a hybrid orchestration framework. It separates three concerns: evaluating entitlements, monitoring connectivity, and enforcing policy. That split lets one set of rules run on top of both an online REST sync and offline cryptographic validation.

This post covers the licensing control flow and enforcement model. It leaves out pricing models, the license-issuing side and UI details. If you've handled this differently, I'd like to hear how.

> **About this post.** This article presents a general approach to licensing in distributed control planes, based on challenges I’ve worked on. The examples and component names are illustrative.

## The problem

Four constraints shape the design.

| Constraint | What it requires |
|---|---|
| **Air-gapped sites** | No outbound internet. A pure SaaS "phone-home" check makes the product unusable in financial, defense and government environments. |
| **Mixed topologies** | The same compliance rules for a connected hybrid cloud (online) and an isolated private cloud (offline). |
| **Metered entitlements** | Capabilities are checked against cluster topology (CPU count, sockets, memory, deployment count), not a simple on/off flag. |
| **Operational continuity** | A license expiry or a network blip must not crash the control plane or disturb running workloads. A synchronous check in the request path turns a transient drop into an outage. |

## Architecture

Three modules, each with one job:

<figure class="lic-diagram">
<svg viewBox="0 0 600 380" role="img" aria-labelledby="lic-title lic-desc" xmlns="http://www.w3.org/2000/svg">
<title id="lic-title">Licensing architecture for a distributed control plane</title>
<desc id="lic-desc">Online REST polling and an offline license key file feed the License Sentinel. The Sentinel passes entitlement state, and Pulse Guard passes reachability, to the Policy Interceptor, which allows read and list calls and blocks mutating calls.</desc>
<style>
.lic-diagram{margin:1.8em 0;overflow-x:auto}
.lic-diagram svg{display:block;width:100%;min-width:520px;height:auto}
.lic-diagram .box{fill:var(--card);stroke:var(--border);stroke-width:1.5}
.lic-diagram .hl{fill:color-mix(in srgb,var(--accent) 14%,var(--card));stroke:color-mix(in srgb,var(--accent) 55%,var(--border));stroke-width:1.5}
.lic-diagram .zone{fill:none;stroke:var(--muted);stroke-width:1.2;stroke-dasharray:6 6;opacity:.6}
.lic-diagram .t{fill:var(--text);font:600 15px var(--font)}
.lic-diagram .s{fill:var(--muted);font:400 12.5px var(--font)}
.lic-diagram .z{fill:var(--muted);font:500 11px var(--mono);letter-spacing:.08em}
.lic-diagram .ln{stroke:var(--muted);stroke-width:1.6;fill:none}
.lic-diagram marker path{fill:var(--muted)}
.lic-diagram figcaption{color:var(--muted);font-size:.9rem;text-align:center;margin-top:.7em}
</style>
<defs><marker id="lic-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0 L10 5 L0 10 z"/></marker></defs>
<rect class="box" x="20" y="8" width="270" height="60" rx="12"/>
<text class="t" x="155" y="34" text-anchor="middle">Online mode</text>
<text class="s" x="155" y="53" text-anchor="middle">REST polling · Bearer token</text>
<rect class="box" x="310" y="8" width="270" height="60" rx="12"/>
<text class="t" x="445" y="34" text-anchor="middle">Offline mode</text>
<text class="s" x="445" y="53" text-anchor="middle">Uploaded license key file</text>
<rect class="zone" x="8" y="100" width="584" height="268" rx="16"/>
<text class="z" x="26" y="360">CONTROL NODE CLUSTER</text>
<rect class="hl" x="20" y="136" width="560" height="70" rx="12"/>
<text class="t" x="300" y="164" text-anchor="middle">License Sentinel</text>
<text class="s" x="300" y="186" text-anchor="middle">Evaluates entitlements · persists state in the shared cluster database</text>
<rect class="box" x="20" y="270" width="300" height="72" rx="12"/>
<text class="t" x="170" y="299" text-anchor="middle">Policy Interceptor</text>
<text class="s" x="170" y="320" text-anchor="middle">Allows read and list · blocks mutations</text>
<rect class="box" x="410" y="270" width="170" height="72" rx="12"/>
<text class="t" x="495" y="299" text-anchor="middle">Pulse Guard</text>
<text class="s" x="495" y="320" text-anchor="middle">Monitors reachability</text>
<line class="ln" x1="155" y1="68" x2="155" y2="132" marker-end="url(#lic-arrow)"/>
<line class="ln" x1="445" y1="68" x2="445" y2="132" marker-end="url(#lic-arrow)"/>
<line class="ln" x1="170" y1="206" x2="170" y2="266" marker-end="url(#lic-arrow)"/>
<text class="s" x="182" y="242">entitlement state</text>
<line class="ln" x1="408" y1="306" x2="324" y2="306" marker-end="url(#lic-arrow)"/>
<text class="s" x="366" y="298" text-anchor="middle">reachability</text>
</svg>
<figcaption>Evaluation, health monitoring and enforcement are separate modules.</figcaption>
</figure>

### License Sentinel: evaluation and lifecycle

Runs as a core service on the control node (CN).

- **Registration.** Supports two paths. Online, it takes the endpoint URL, credentials, license key and a deployment identifier (customer ID, MAC address or deployment key), then authenticates with the Sentinel Hub to obtain a Bearer token. Offline, the admin uploads the license key file issued by the vendor. The file carries the subscription and vendor details in encrypted form, and the control node validates it locally with no outbound call.
- **Polling.** Calls `checkEntitlements` and `fetchLicenseDetails` on a schedule using that token.
- **Evaluation.** Compares the returned capability metadata with the local topology: `daysRemaining`, CPU count, sockets, memory limits and deployment count. State is persisted atomically in the shared cluster database.
- **Clustering.** Distributed coordination ensures only one CN polls per interval. Every node reads the same database state, so there is no duplicate outbound traffic.

### Pulse Guard: reachability

Watches connectivity between the control node and Sentinel Hub.

- **Starts** on CN restart, on credential or key updates, and as a renewal window approaches.
- **Retries** up to a configured threshold before raising anything, so a brief drop does not trigger alarms.
- **Escalates** to the Policy Interceptor only when reachability stays broken past that threshold, which then applies its grace-period policy.

### Policy Interceptor: enforcement

Sits in the control plane's request pipeline and acts on what the other two modules report.

| Condition | Enforcement |
|---|---|
| **Valid license**: `daysRemaining` is not negative and usage is within entitlement | No restrictions |
| **Hub unreachable**: Pulse Guard is past its retry threshold | Grace-period policy |
| **Expired**: `daysRemaining < 0` | Read and list APIs keep working; mutating operations are blocked |
| **Over capacity**: sockets, memory or counts above the licensed tier | Specific operations such as adding hosts or nodes are blocked, and the UI asks the admin to reduce capacity or upgrade |
| **Renewed**: a valid license is applied | All restrictions are lifted immediately |

## Two modes, one policy

### Online: REST synchronization

1. **Register.** Credentials (URL, username, password, license key) and the deployment key are submitted through the UI or API.
2. **Exchange.** Sentinel Hub validates the request and returns a session Bearer token.
3. **Poll.** The license evaluator calls `checkEntitlements` and `fetchLicenseDetails` periodically and parses the JSON capability metadata. A positive `daysRemaining` means valid, and a negative value means expired.
4. **Share.** In a clustered setup, one node does the polling and writes the result to the shared database, so all nodes enforce the same state.

### Offline: license key file validation

For dark sites, validation happens entirely on the control node.

- **Upload.** The vendor issues a license key file when the customer buys a subscription. The admin uploads it through the UI or API. The file holds all subscription entitlements and vendor details in encrypted form.
- **Signature check.** The file is protected with asymmetric cryptography, for example a signature using RS256 or ES256. The CN verifies it with key material embedded in the product, so the vendor's private key never leaves the vendor and the contents cannot be edited.
- **Binding.** The payload carries a deployment identifier, such as a MAC address or system UUID. The Sentinel compares it with the local value, which discourages copying a license to another cluster.
- **Local checks.** Entitlements (CPU count, sockets, memory) and the expiry date are validated against the cluster topology. Each check-in records a timestamp atomically in the database so a clock set backwards can be detected.

## Lessons

> An expired license should never stop operators from seeing what their cloud is doing.

- **Separate evaluation from enforcement.** The Sentinel decides, and the Interceptor acts. Because enforcement does not care how the decision was made, online and offline modes follow identical rules.
- **Never break monitoring.** Read-only mode protects the business interest without risking infrastructure stability. A hard stop on expiry is an outage you caused.
- **Absorb network flaps.** Intermittent partitions are normal in enterprise data centers. A dedicated probe with a retry threshold prevents enforcement on a false positive.
- **Keep the cluster consistent.** Use distributed locking and atomic persistence so that licensing jobs cannot race across nodes.

## Trade-offs to plan for

- **Hardware binding is a deterrent, not a lock.** MAC addresses and UUIDs change when a virtual control node is cloned or a NIC is replaced. Bind to a cluster-level identifier where you can, and provide a support path for re-binding a legitimate move.
- **Clock-rollback detection raises the bar, nothing more.** Persisting the latest observed timestamp and rejecting an earlier wall clock stops casual tampering. An administrator with full control of the host can still defeat it, and offline expiry always depends on the local clock.
- **Give the polling lock a lease.** If the node holding the lock crashes, its lock should expire on its own so another node can take over, instead of licensing going stale until someone intervenes.

---

*Written by Pavan Aravapalli, based on hands-on development experience with licensing in distributed systems. Generic overview; not a product specification. If you have a different approach, I’d like to hear about it.*
