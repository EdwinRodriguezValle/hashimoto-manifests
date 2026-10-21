# Hashimoto Platform DevOps Roadmap

## Purpose

This document is a practical learning and implementation plan for building a professional, repeatable delivery path for the current Hashimoto Platform. It covers this application repository and the separate `hashimoto-manifests` GitOps repository.

The first deployment target is a Kubernetes cluster running on the developer's own machine (K3d). GitHub Actions will build, check, and publish application images. Argo CD, running inside the local cluster, will watch `hashimoto-manifests` and reconcile the cluster to the desired state. GitHub-hosted runners cannot normally reach a cluster behind a home router or firewall, so the cluster should pull changes outward rather than requiring an inbound connection from GitHub.

This is an incremental plan, not a requirement to install every tool immediately. Start with a working, understandable path and add controls as the project grows.

## Current repository baseline

The repository currently contains:

- **Frontend:** React, TypeScript, Vite, ESLint, and an Nginx container image in `hashimoto-tracker-app/`.
- **Backend:** a Maven parent project with `api-gateway/` and `patient-service/`, using Java 21 and Spring Boot. The parent includes Spring test dependencies.
- **Container builds:** Dockerfiles for the frontend, gateway, and patient service.
- **Kubernetes:** K3d-oriented manifests under `k8s/` and `grafana/` for application services, infrastructure, and observability.
- **Local deployment:** `Makefile` targets that build images, import them into K3d, and apply manifests.
- **Existing checks:** the frontend has `lint` and `build` npm scripts. Maven test dependencies exist, but CI should first establish which tests currently run and pass.

The first delivery milestone should use the existing services and local cluster rather than redesigning the platform. In particular, retain the current local `make` workflow while introducing CI and GitOps in small steps.

## Target delivery flow

```text
Code repository (this repo)
  └─ Pull request: lint, build, tests, security checks, manifest validation
  └─ Merge to main: build and scan immutable container images; publish to GHCR
                    └─ Open/update a pull request in hashimoto-manifests

hashimoto-manifests
  └─ Review and merge image tag + deployment configuration
  └─ Argo CD detects the Git change and reconciles the local K3d cluster
       └─ Kubernetes pulls the image from GHCR and rolls out the workloads
```

The application repository owns source code and image builds. The manifests repository owns deployment configuration, image versions, environment-specific settings, and Argo CD `Application`/`ApplicationSet` resources. Do not copy application source code into the manifests repository.

## Ordered implementation plan

### Phase 0 — Record the current behavior

1. Make sure `README.md` and `docs/k8s-onboarding.md` describe the known-good local deployment.
2. Confirm the current branch, default branch, and required Java/Node versions in GitHub and in the project documentation.
3. Identify which values in Kubernetes manifests are intentionally local and which must become environment-specific. Do not commit real credentials; treat health-related data and access tokens as sensitive.
4. Write down the current local image names, tags, namespaces, ingress hostnames, storage needs, and startup dependencies.
5. Keep `make deploy-all` working as the manual recovery and learning path during CI/CD development.

**Exit condition:** a developer can follow the local guide and explain how the three application images and Kubernetes resources are deployed.

### Phase 1 — Establish CI checks in this repository

Add GitHub Actions workflows under `.github/workflows/`. Begin with pull-request validation only; CI must not deploy to the home cluster.

Recommended checks:

1. **Frontend:** `npm ci`, `npm run lint`, and `npm run build` in `hashimoto-tracker-app/`.
2. **Java:** set up JDK 21, use the Maven wrapper, and run the root Maven `verify` lifecycle so both backend modules are checked together.
3. **Unit tests:** add focused tests for meaningful behavior, beginning with API authorization, request validation, and service logic. Do not add tests just to increase a coverage percentage.
4. **Formatting and static analysis:** add a consistent Java formatter/check (for example Spotless with a documented format command) once the team agrees on the style. Keep ESLint for the frontend.
5. **Workflow hygiene:** use least-privilege `permissions`, pin third-party actions to full commit SHAs, set timeouts, avoid running privileged jobs on untrusted pull-request code, and cache dependencies carefully.
6. **Branch protection:** require the CI checks to pass before merging to the default branch.

At this stage, run the equivalent commands locally and fix existing failures. The workflow should report the same checks a developer can run before opening a pull request.

**Exit condition:** a pull request receives repeatable frontend and backend results, and merging is blocked when a required check fails.

### Phase 2 — Add baseline repository security checks

Add these incrementally to pull-request or scheduled workflows:

- **Secret scanning:** Gitleaks (open source) to catch accidentally committed credentials. Also enable GitHub secret scanning where available. If a real secret is found, revoke and rotate it; deleting the line alone is not enough.
- **Dependency updates:** enable Dependabot version updates and security alerts for Maven, npm, GitHub Actions, and Docker where applicable.
- **Source analysis:** enable GitHub CodeQL if the repository visibility and GitHub plan support code scanning. Otherwise, keep local/CI static checks and evaluate SonarQube Cloud's current free eligibility for the repository.
- **Container and filesystem scanning:** use Trivy (open source) for container image and configuration vulnerability checks. Start by reporting findings; after reviewing existing base images and false positives, define which severities block a release.
- **Workflow permissions:** use the minimum token permissions per job. Do not place registry or cluster credentials in pull-request jobs.

Do not turn on every possible blocking rule before looking at current findings. Establish a baseline, fix meaningful high-severity issues, and document any temporary exception with an owner and revisit date.

**Exit condition:** pull requests are checked for likely leaked secrets, vulnerable dependencies, and common source/container risks, with an actionable report.

### Phase 3 — Make Kubernetes configuration reviewable in CI

The application repository currently contains Kubernetes files, and the manifests repository will eventually become the deployment source of truth. Choose one canonical owner for each deployed manifest before enabling Argo CD; avoid having two repositories independently change the same live resources.

For the manifests repository, add pull-request validation for:

- YAML syntax and Kubernetes schema checks (for example `kubeconform`).
- Kustomize build/render checks for every environment overlay.
- Kubernetes policy/lint checks (for example `kube-linter`), introduced in report mode first.
- Secret checks that reject plaintext credentials. Use placeholders or external secret references in Git; never put actual credentials in Git.
- Diff review of rendered manifests so a change to a namespace, ingress, persistent volume, or broad RBAC permission is visible.

Keep environment overlays small and explicit, for example `overlays/local`. Use Kustomize, already used in this project, unless Helm becomes a clear fit for a specific component.

**Exit condition:** each manifests pull request can render the intended local configuration and catch invalid or unsafe changes before merge.

### Phase 4 — Publish immutable images to GHCR

After CI checks pass on the default branch:

1. Build the three application images: frontend, API Gateway, and Patient Service.
2. Tag each image with the full Git commit SHA. Do not deploy mutable `latest` tags.
3. Push to GitHub Container Registry (GHCR), using the workflow's `GITHUB_TOKEN` and only the package permissions needed by that job.
4. Scan the built image with Trivy and keep the scan result with the workflow run. Decide whether the gate blocks based on reviewed severity and exploitability.
5. Add OCI labels such as source repository and revision. Consider signing images and publishing provenance/SBOM attestations after the basic pipeline is reliable.
6. Decide whether images/packages are public or private. For a private package, configure a Kubernetes pull secret in the local cluster without committing its value. Verify GHCR package permissions and current free-tier/storage terms before relying on them.

For a learning project, publish one image per service from a shared workflow, but keep independent tags so a frontend change does not force an unnecessary backend rollout.

**Exit condition:** a successful default-branch build produces three identifiable, pullable images tied to a source commit.

### Phase 5 — Connect image delivery to `hashimoto-manifests`

Use a pull-request-based promotion flow:

1. On successful image publication, automation opens or updates a change in `hashimoto-manifests` that changes the relevant image tag to the commit SHA.
2. Prefer a narrowly scoped GitHub App installation token. A fine-grained token restricted to the manifests repository is a simpler learning alternative; store it as a GitHub Actions secret, never in source code.
3. The automation should modify only image references. It must not rewrite unrelated configuration or merge its own deployment change without review at first.
4. Review and merge the manifests pull request. This records exactly which code revision is intended for deployment and allows rollback by reverting the Git change.
5. Later, add an explicit promotion process from `local` to `staging`/`production` overlays when those environments exist.

**Exit condition:** merging code creates a reviewable manifests change, and merging that change selects the image version for local deployment.

### Phase 6 — Install Argo CD in local K3d

1. Create/start the existing K3d cluster and confirm the intended Kubernetes context. Keep a context guard for manual commands.
2. Install Argo CD into a dedicated namespace using its official installation method, then pin and document the selected version.
3. Configure Argo CD with read-only access to `hashimoto-manifests`. Use a deploy key or GitHub App credentials stored as a Kubernetes Secret; never put credentials in a committed `Application` manifest.
4. Add an Argo CD `Application` (or a small app-of-apps root) in the manifests repository that points at the local Kustomize overlay.
5. Initially use **manual sync** and inspect the diff. Once the rendered resources and ownership are correct, enable automated sync and pruning deliberately. Consider `selfHeal` only after normal edits are expected to flow through Git.
6. Configure GHCR image pulls (public packages need no pull secret; private packages do). Check image architecture and storage behavior on the local machine.
7. Add health checks and resource requests/limits where missing, then check rollout status and application health in Argo CD.

Argo CD should manage application workloads from `hashimoto-manifests`. Decide explicitly whether it also owns infrastructure and monitoring manifests; some local persistent services may need manual bootstrap, backup, and upgrade care.

**Exit condition:** a merged image-tag change in `hashimoto-manifests` is detected by Argo CD and rolls out successfully on the local K3d cluster without a GitHub runner connecting inbound to the machine.

### Phase 7 — Add meaningful automated tests

Add tests in this order, based on the current stack:

1. **Java unit tests:** JUnit 5 and Mockito are included through Spring Boot's test starter. Focus on business rules, validation, security decisions, and error cases.
2. **Backend integration tests:** Testcontainers (free/open-source) can start PostgreSQL and Kafka for realistic tests. Begin with PostgreSQL-backed repository/API tests; add Kafka only for behavior that depends on it. Docker is needed on the runner.
3. **Frontend tests:** add Vitest and React Testing Library for components and user interactions. Start with authentication-independent flows and form validation.
4. **End-to-end tests:** Playwright (free/open-source) can exercise the built frontend and API in a test environment. Use synthetic data and a dedicated non-production Keycloak realm/account; avoid real personal or health data in CI.
5. **Smoke tests after local deployment:** check that expected deployments are available and health endpoints respond. Keep these separate from unit tests because the local cluster may be offline.
6. **Load tests (later):** k6 has an open-source CLI for modest local performance experiments. Keep load testing opt-in until a stable test environment and safe limits exist.

Choose tests for fast feedback: unit tests on every pull request, selected integration tests on every pull request, broader end-to-end checks on the default branch or a schedule, and performance tests manually or on a dedicated environment.

**Exit condition:** the pipeline catches regressions at the cheapest useful layer and produces understandable test reports.

### Phase 8 — Operational readiness and maintainability

- Define health/readiness probes, CPU/memory requests and limits, and rollout behavior for each application deployment.
- Add a documented backup and restore procedure for persistent local data before treating the cluster as dependable.
- Review ingress exposure, Keycloak redirect URLs, TLS expectations, admin accounts, RBAC, network access, and secret rotation.
- Redact tokens, credentials, and sensitive user data from application logs. Use synthetic data in demos and tests.
- Keep Argo CD and the K3d cluster reachable only as intended. Do not expose the Argo CD admin endpoint publicly for convenience.
- Document routine operations: view sync/health, inspect events/logs, roll back through Git, recover a PVC, update Argo CD, and recreate the cluster.
- Pin container base images and workflow actions responsibly, and schedule dependency/image updates.

**Exit condition:** another person can deploy, diagnose, roll back, and restore the local environment from written instructions.

## Suggested `hashimoto-manifests` structure

Start small and expand only when a second environment or ownership boundary requires it:

```text
hashimoto-manifests/
├── README.md
├── argocd/
│   ├── projects/
│   └── applications/
│       └── hashimoto-local.yaml
├── apps/
│   ├── frontend/
│   │   ├── base/
│   │   └── overlays/local/
│   ├── api-gateway/
│   │   ├── base/
│   │   └── overlays/local/
│   └── patient-service/
│       ├── base/
│       └── overlays/local/
├── platform/
│   ├── namespaces/
│   ├── ingress/
│   └── observability/       # Optional; add when Argo CD should own it
└── .github/workflows/
    └── validate-manifests.yaml
```

Each Kustomize base should contain the deployment/service and safe defaults. The `local` overlay should contain local-only hostnames, image references/tags, replicas, and resource/storage choices. Keep secrets out of this tree; commit only references to secrets provisioned locally or by a later secret-management system.

The first `hashimoto-local.yaml` should point Argo CD at one root Kustomization (or app-of-apps directory) so the local environment can be reviewed and bootstrapped consistently. Keep the current `k8s/` manifests as the source during the transition, then migrate ownership deliberately and document when Argo CD becomes authoritative.

## Tool recommendations with free/open-source starting points

| Need | First choice | Why it fits this project | Cost/notes |
| --- | --- | --- | --- |
| CI | GitHub Actions | Already using GitHub; supports Maven, npm, Docker, and repository workflows | Standard hosted runner use is free for public repositories; private repositories have plan quotas. Self-hosted runner usage has no GitHub Actions minute charge, but consumes your own machine resources. Check current limits. |
| Container registry | GHCR | Natural pairing with GitHub Actions and Kubernetes image pulls | Check package visibility, storage, and bandwidth terms before choosing public/private. |
| Java unit/integration tests | JUnit 5, Mockito, Testcontainers | Fits Spring Boot/Maven; Testcontainers exercises PostgreSQL/Kafka in Docker | Free/open source; integration jobs need Docker and more runner time. |
| Frontend tests | Vitest + React Testing Library | Fits Vite/React and makes component behavior testable | Free/open source. |
| Browser end-to-end tests | Playwright | Useful for checking login and a complete user workflow with synthetic data | Free/open source; run after a stable test realm/environment exists. |
| Secret scanning | Gitleaks | Lightweight check for accidental secret commits | Free/open source; also enable GitHub's available secret protection features. |
| Dependency updates | Dependabot | Native update PRs for Maven, npm, Docker, and Actions | Available through GitHub; review changes rather than auto-merging everything. |
| Static application security | GitHub CodeQL or SonarQube Cloud | CodeQL integrates with GitHub code scanning; Sonar adds maintainability and quality analysis | CodeQL code scanning is free for public repositories; private access depends on plan. SonarQube Cloud Free is oriented to public projects, while private analysis has current limits/eligibility; verify before adopting. |
| Image/config scanning | Trivy | Scans container images, dependencies, and Kubernetes configuration | Free/open source; vulnerability databases and exceptions need maintenance. |
| Kubernetes manifest checks | Kustomize + kubeconform + kube-linter | Render manifests, validate schemas, and identify common configuration risks | Free/open source. Start lint rules in report mode. |
| Metrics and dashboards | Existing Prometheus + Grafana | Already present in this repository's local platform | Keep the initial footprint small; Elasticsearch/Kibana are resource-heavy on a single workstation. |
| Local performance checks | k6 OSS | Can generate a small repeatable load against local endpoints | Free/open source CLI; defer until the test target and limits are clear. |

For an initially free path, prioritize GitHub Actions, GHCR, Maven/JUnit, Vitest, Gitleaks, Dependabot, Trivy, Kustomize validation, and Playwright later. Avoid paying for a security or test platform until the repository's visibility, code size, and actual needs are known.

## Security and design rules

- Never commit `.env`, registry passwords, GitHub tokens, kubeconfigs, Keycloak admin credentials, or real user/health data.
- Use immutable image tags (commit SHA) and keep deployment changes reviewable in Git.
- Let Argo CD pull from Git; do not expose the home Kubernetes API to GitHub-hosted runners.
- Do not run a self-hosted GitHub runner on the same machine with broad host access just to make inbound deployment easy. If a runner is ever used, isolate it and treat jobs as code execution on that machine.
- Use short-lived or narrowly scoped credentials, minimum workflow permissions, and a documented rotation path.
- Keep local development secrets and defaults separate from any shared or future cloud environment.
- Health data requires special care: use synthetic test data, restrict logs, and review privacy/security requirements before inviting real users.
- Make security gates evidence-based. A scanner is useful only when findings are triaged and fixes have an owner.

## First concrete milestone checklist

Work through these tasks in order:

1. Verify that the current local K3d deployment is repeatable from the onboarding guide.
2. Add a pull-request workflow for frontend lint/build and Maven verification.
3. Fix the failures it exposes and require those checks for merges.
4. Add Gitleaks and Dependabot; review the initial findings and configuration.
5. Add manifest render/schema validation for the future `hashimoto-manifests` layout.
6. Build the three images with commit-SHA tags and publish them to GHCR.
7. Create an automated pull request in `hashimoto-manifests` to update those tags.
8. Install Argo CD in local K3d and first sync manually.
9. Verify a Git-only image tag change rolls out and that reverting the manifests commit rolls back.
10. Add integration and browser tests once the basic build/deploy feedback loop is stable.

Do not start by adding cloud infrastructure, a public self-hosted runner, or a large testing platform. The local GitOps loop is the first goal; a later cloud migration should mostly add a new environment overlay, credentials/secret management, and environment-specific operational controls.

## Official references

- [Argo CD declarative setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/) and [Application specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [GitHub Actions billing and usage](https://docs.github.com/en/actions/concepts/billing-and-usage)
- [GitHub code scanning availability](https://docs.github.com/en/code-security/concepts/code-scanning/code-scanning-alerts)
- [SonarQube Cloud subscription plans](https://docs.sonarsource.com/sonarqube-cloud/administering-sonarcloud/managing-subscription/subscription-plans)
- [Testcontainers for Java](https://java.testcontainers.org/)
- [Playwright](https://playwright.dev/)
- [Trivy](https://trivy.dev/)
- [Gitleaks](https://github.com/gitleaks/gitleaks)
- [Kubeconform](https://github.com/yannh/kubeconform)
- [kube-linter](https://github.com/stackrox/kube-linter)
- [k6 open-source](https://grafana.com/oss/k6/)
