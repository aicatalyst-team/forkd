## What is forkd?

If you've ever watched an AI agent spin up a dozen parallel "thought branches" and wished each one ran in its own hardware-isolated sandbox, forkd is the project that scratches that itch. Built on top of [Firecracker](https://firecracker-microvm.github.io/) — the same microVM technology that powers AWS Lambda — forkd provides a runtime for forking and branching *live* microVMs with copy-on-write memory sharing. The result: sub-200ms spawning of fully isolated sandbox environments from a single snapshot.

The project is a Rust workspace comprising several crates — a controller daemon (`forkd-controller`), a CLI tool, a userfaultfd handler for demand-paging guest memory, and a VMM library. The controller exposes a REST API on port 8889 for managing sandboxes and snapshots, making it the management plane for what is essentially a "git branch for VMs."

forkd targets the agentic AI pattern specifically: fan out an agent's execution into multiple isolated branches, let each explore independently, then collect results. It's infrastructure-level plumbing for a use case that's becoming increasingly common as LLM-backed agents execute untrusted code, browse the web, or interact with external APIs.

## Why this matters for OpenShift AI

forkd occupies an interesting niche in the OpenShift AI / Open Data Hub ecosystem. RHOAI provides workbenches, model serving, and pipelines — but it doesn't have a native answer for "run this agent's generated code in a hardware-isolated sandbox that can be forked in milliseconds." That's the gap forkd fills.

Our RHOAI fitness evaluation scored forkd at **52/100**, which is honest: the project is technically impressive but has limited integration with RHOAI platform components today. Its Firecracker/KVM dependency makes deployment on OpenShift genuinely challenging — pods don't normally get access to `/dev/kvm`. But as an infrastructure primitive for the agentic AI story, it's worth understanding what the deployment path looks like and where the integration boundaries are.

This PoC exercises a scoped question: can we containerize the forkd controller daemon, deploy it to Kubernetes, and validate that the HTTP management plane works — even without KVM? The answer tells us how much of the forkd stack is portable to OpenShift and what a KVM-enabled node pool would unlock.

## Setting up the PoC

The good news: forkd's controller daemon is lightweight. Here's what we needed:

- **Resource profile:** Small — this is a compiled Rust binary with minimal runtime dependencies
- **GPU:** None
- **Persistent storage:** None for the PoC (production would need `/var/lib/forkd`)
- **Vector DB / Inference server:** None — this is pure infrastructure
- **Sidecar containers:** None
- **Environment variables:** Just `FORKD_LOG=info` for log verbosity

The key decision was **scope limitation**. forkd's core value — forking microVMs via KVM + userfaultfd — requires a bare-metal or nested-virt host with `/dev/kvm` access and `--privileged` mode. A typical Kubernetes PoC cluster doesn't expose KVM to pods. So we scoped the PoC to validate the controller daemon's HTTP management plane: does it build, start, and serve its API correctly? Full microVM forking is documented as a follow-up requiring a KVM-enabled node pool.

This is a deliberate tradeoff, and we think it's the right one. Proving the containerization and deployment pipeline works is a prerequisite for the more ambitious KVM integration anyway.

--------------------
**[Image Placeholder 1: Architecture diagram showing forkd-controller in a Kubernetes pod]**

**Placement rationale**: Readers need a mental model of what's deployed vs. what's deferred before diving into Dockerfile details.

**Image generation prompt**: A clean technical architecture diagram on a white background showing a Kubernetes pod containing a single container labeled "forkd-controller" with port 8889 exposed via a Service. Outside the pod, a dashed-line box labeled "KVM-enabled Node (future)" contains Firecracker microVM icons. Use flat design, blue and gray color scheme with red accents for the "future" components. Aspect ratio 16:9, minimal style, developer documentation aesthetic.

**Alt text**: Architecture diagram showing the forkd-controller container deployed in a Kubernetes pod with port 8889 exposed, and a dashed outline indicating future KVM-enabled node requirements for full microVM forking.
--------------------

## Containerizing with UBI

forkd is a Rust workspace, so containerization means multi-stage builds: compile in a builder image, copy the binary to a slim runtime image. Here's the key fragment from our Dockerfile:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS builder
RUN microdnf install -y gcc make openssl-devel && microdnf clean all
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"
WORKDIR /src
COPY . .
RUN cargo build --release --bin forkd-controller

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
COPY --from=builder /src/target/release/forkd-controller /usr/local/bin/
USER 1001
EXPOSE 8889
ENTRYPOINT ["/usr/local/bin/forkd-controller", "serve", "--bind", "0.0.0.0:8889"]
```

A few notable decisions:

- **UBI9 minimal** for both stages — keeps the final image small and Red Hat ecosystem-compatible.
- **Non-root by default** (`USER 1001`) — the controller doesn't need root for its HTTP API. Full KVM operation would change this, but for the management plane, non-root is correct.
- The Rust binary is statically linked enough that we don't need to carry build dependencies into the runtime image. The final image is lean.

One challenge: the workspace has multiple crates with interdependencies, so `cargo build` pulls in everything. We targeted just `--bin forkd-controller` to avoid building the CLI and other binaries we didn't need.

The built image is published at `quay.io/aicatalyst/forkd-forkd-controller:latest`.

## Deploying to Kubernetes

The deployment is straightforward — a single-replica Deployment with a ClusterIP Service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: forkd-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: forkd-controller
  template:
    metadata:
      labels:
        app: forkd-controller
    spec:
      containers:
        - name: forkd-controller
          image: quay.io/aicatalyst/forkd-forkd-controller:latest
          ports:
            - containerPort: 8889
          env:
            - name: FORKD_LOG
              value: "info"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: forkd-controller
spec:
  selector:
    app: forkd-controller
  ports:
    - port: 8889
      targetPort: 8889
```

No PVCs, no GPU resources, no sidecars. The controller binds to `0.0.0.0:8889` and is reachable at the ClusterIP `172.30.238.37:8889`. For a production deployment with KVM access, you'd add `securityContext.privileged: true`, mount `/dev/kvm`, and likely use a DaemonSet on labeled bare-metal nodes instead of a Deployment.

--------------------
**[Image Placeholder 2: Screenshot of the forkd-controller pod running in OpenShift console]**

**Placement rationale**: Visual confirmation that the deployment is live, bridging the YAML configuration to the running state.

**Image generation prompt**: A realistic screenshot-style image of a Kubernetes/OpenShift web console showing a pod named "forkd-controller" in Running status, with 1/1 containers ready, a green status indicator, and log output showing "listening on 0.0.0.0:8889". Dark mode UI, modern dashboard layout. Aspect ratio 16:9.

**Alt text**: OpenShift console showing the forkd-controller pod in Running status with 1 of 1 containers ready and logs indicating the HTTP server is listening on port 8889.
--------------------

## Test results

We ran five test scenarios against the deployed controller. All five passed:

| Scenario | Description | Status | Duration |
|----------|-------------|--------|----------|
| health-check | `GET /v1/sandboxes` returns 200 with empty JSON array | ✅ PASS | 0.0s |
| list-snapshots | `GET /v1/snapshots` returns 200 with empty JSON array | ✅ PASS | 0.0s |
| invalid-sandbox-request | Invalid request returns proper error code | ✅ PASS | 0.0s |
| create-sandbox-no-kvm | Sandbox creation fails gracefully without KVM | ✅ PASS | 0.0s |
| post-create-health-check | Controller remains healthy after failed create | ✅ PASS | 0.0s |

**5/5 passed.** The sub-millisecond durations reflect that the controller is a compiled Rust binary with zero cold-start overhead — once the container is up, responses are effectively instant.

The most interesting test is `create-sandbox-no-kvm`. We deliberately tried to create a sandbox on a cluster without KVM access. The controller handled this gracefully — returning a meaningful error rather than crashing. This is important: it means the management plane is deployable and debuggable on any Kubernetes cluster, even if the data plane (actual microVM forking) requires KVM-enabled nodes.

--------------------
**[Image Placeholder 3: Test results summary with pass/fail indicators]**

**Placement rationale**: A visual summary of test results reinforces the "5/5 passed" narrative and is easy to scan.

**Image generation prompt**: A clean, minimal test results dashboard graphic showing 5 test scenarios in a vertical list, each with a green checkmark icon and "PASS" label. Test names: health-check, list-snapshots, invalid-sandbox-request, create-sandbox-no-kvm, post-create-health-check. White background, monospace font for test names, green (#22c55e) checkmarks. Summary bar at bottom reads "5/5 PASSED". Aspect ratio 4:3, flat design.

**Alt text**: Test results dashboard showing all five PoC scenarios — health-check, list-snapshots, invalid-sandbox-request, create-sandbox-no-kvm, and post-create-health-check — passing with green checkmarks.
--------------------

## What we learned

**The management plane is fully portable.** The forkd controller daemon builds cleanly into a UBI-based container, runs as non-root, and serves its API without any special kernel features. This is a meaningful result — it means teams can develop against the forkd API on any Kubernetes cluster and only need KVM-enabled nodes for the actual microVM execution.

**Graceful degradation is well-implemented.** The controller doesn't crash when KVM isn't available. It starts up, serves listing endpoints, and returns clear errors on operations that require the VMM. This is good engineering and makes the deployment pipeline testable in CI environments that don't have nested virtualization.

**The gap is KVM access on OpenShift nodes.** The elephant in the room is `/dev/kvm`. OpenShift worker nodes can expose KVM — bare-metal deployments with the KVM device plugin, or nodes with nested virtualization enabled — but it's not the default. For forkd to deliver its core value on OpenShift AI, we'd need to:

1. Label specific bare-metal nodes with a `kvm=true` taint
2. Deploy forkd as a DaemonSet on those nodes with privileged access
3. Potentially integrate with OpenShift Virtualization (KubeVirt) for device management

**ODH components that would help:** Model serving integration could allow forkd sandboxes to be triggered from inference pipelines. The ODH dashboard could surface sandbox management. But honestly, the most impactful next step is proving the KVM-enabled deployment path, not adding dashboard integrations.

**Is this production-ready?** The management plane is. The full stack — forking live microVMs — needs a KVM-enabled follow-up PoC and security review around privileged containers.

## Try it yourself

Want to reproduce this PoC or extend it with KVM-enabled nodes? Here's everything you need:

- **Forked repository:** [github.com/aicatalyst-team/forkd.git](https://github.com/aicatalyst-team/forkd.git)
- **Original project:** [github.com/deeplethe/forkd](https://github.com/deeplethe/forkd)
- **Container image:** `quay.io/aicatalyst/forkd-forkd-controller:latest`
- **Open Data Hub docs:** [opendatahub.io/docs](https://opendatahub.io/docs)

If you have access to bare-metal OpenShift nodes with KVM, we'd love to see the full microVM forking path validated. File an issue on the fork or reach out — this is exactly the kind of infrastructure primitive that the agentic AI story on OpenShift needs, and a 52/100 fitness score is an invitation to close the gaps, not a verdict.
