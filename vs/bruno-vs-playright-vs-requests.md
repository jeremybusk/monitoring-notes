# API Testing in CI/CD: Tools, Execution Architecture, and AKS Examples

## 1. Tool Selection

| Tool | Best fit | Strengths in CI/CD | Considerations |
| --- | --- | --- | --- |
| **Playwright Test + `APIRequestContext`** | Code-first API regression suites, especially alongside browser tests | Fixtures, configurable test retries, parallel workers, HTML/JUnit reporting, shared tooling for API and UI tests | API-only tests need no browser binaries. Browser fixtures or browser-based setup do require browsers. Test-runner retries and reporting are separate from the HTTP client. |
| **Bruno CLI** (`@usebruno/cli`) | Teams maintaining Bruno collections for local and automated testing | Git-friendly collections, scripting, assertions, environments, JSON/JUnit/HTML reports, no required cloud service | Complex fixtures and test isolation may be easier in a general-purpose test framework. Verify parallel-execution behavior for the pinned CLI version and keep dependent requests ordered. |
| **Python** (`pytest` + `httpx` or `requests`) | Teams comfortable with Python and data-heavy testing | Fixtures, parametrization, assertion introspection, built-in JUnit XML output | Parallel execution and HTML reporting typically use plugins. Set explicit HTTP timeouts and manage test data isolation. |
| **k6** | Performance testing and lightweight smoke scenarios | Load generation, metrics, thresholds, CLI execution | Configure thresholds to fail CI; failed `check()` results alone do not necessarily produce a failing exit code. Its JavaScript runtime is not Node.js. |
| **Go** (`testing` + `net/http`) | Go-centric services and teams | Standard-library HTTP client and tests; compiled test binaries can simplify execution | Code-first workflow; CI report conversion may require another tool. Static portability depends on the build, including CGO and system dependencies. |

Choose the tool that fits the team's existing tests and skills:

- Keep Bruno collections as the source of truth when the team already maintains them.
- Choose Playwright Test for TypeScript/JavaScript suites needing reusable fixtures and API/browser integration.
- Choose pytest for Python-based suites and extensive data processing.
- Add k6 when performance behavior is part of the acceptance criteria.
- Keep one authoritative suite for each behavior to avoid maintaining duplicate assertions across tools.

Playwright and pytest are not inherently better CI tools than Bruno. The main distinction is collection-based authoring versus general-purpose test-framework organization.

## 2. Execution Architecture

### Option A: Run on the CI/CD Runner

Use this as the default when the runner can reach the application's intended endpoint.

**Preferred target:** the real private or public HTTPS endpoint through the application's Gateway or Ingress. This covers routing and TLS in addition to application behavior.

**Port-forward fallback:** use `kubectl port-forward` for application-level testing when the runner can reach the Kubernetes API but cannot reach the application endpoint directly.

Port-forwarding selects a backing Pod. It does not validate the normal Gateway/Ingress path or Service load balancing, and it stops when the selected Pod terminates.

The deployment sequence should be:

1. Build and publish an immutable application image.
2. Deploy that image, or update GitOps configuration and wait for the intended Argo CD sync.
3. Verify the intended revision/image and successful rollout.
4. Run API tests.
5. Publish reports even when tests fail.
6. Promote only after the required tests pass.

A standalone workflow triggered by the same `push` as deployment can race with deployment and test the previous version. Use a deployment-dependent job or a reusable workflow called after successful deployment.

### Option B: Run an In-Cluster Kubernetes Job

Use a Job when tests need Service DNS or execution from within the cluster network.

Example target: `http://api-service.staging.svc.cluster.local:80`.

An in-cluster Job exercises networking from the test Pod's identity and location. Its namespace, labels, service account, and NetworkPolicies should represent the caller you intend to test.

The pipeline still needs Kubernetes API access and permission to create and inspect Jobs. Alternatively, Argo CD can create a test Job as part of synchronization. Neither OIDC nor a Job automatically provides connectivity from a runner to a private AKS control plane.

Use a new Job for every run, an immutable test-image digest, a runtime deadline, and a deliberate report-export strategy.

### Option C: Azure Monitor and Application Insights

Use availability testing and alerts for continuous monitoring and post-deployment observation.

Azure Monitor alerts can participate in Azure Pipelines deployment checks, including deciding whether a staged rollout should advance. Combine those checks with direct functional tests that validate the intended release.

Monitoring supplements the test suite: an absence of alerts alone does not prove that a particular release passed its functional requirements.

## 3. Pattern 1: GitHub Actions + Bruno + AKS Port-Forward

This example is a reusable/manual validation workflow. It does not deploy the application. Call it after deployment succeeds, or run it manually against an already-deployed image.

### Prerequisites

- A Linux runner labeled `aks-staging`, with connectivity to the AKS API and current GitHub Actions runner software.
- Azure CLI, a cluster-compatible pinned `kubectl`, pinned `kubelogin`, `jq`, and `curl` installed on that runner.
- GitHub environment `staging`, with variables `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AKS_RESOURCE_GROUP`, and `AKS_CLUSTER_NAME`.
- An Azure federated identity credential matching the GitHub environment, permission to retrieve AKS user credentials, and namespace-scoped access to inspect deployments/Pods/Services and create Pod port-forward sessions. Do not use cluster-admin credentials.
- A Bruno collection under `collections/core-api`, a `staging` environment, and request URLs using `{{baseUrl}}`. Supply any API credentials at runtime.
- A repository-root `package.json` and committed `package-lock.json` containing an exact Bruno CLI version. To add Bruno once, run `npm install --save-dev --save-exact @usebruno/cli`, review it, and commit both files. CI uses `npm ci`.
- An application readiness endpoint. Change `/health/ready`, namespace, deployment name, container name, Service name, and port to match the application.

The caller must supply the exact image reference used by the Deployment, including its SHA-256 digest. Serialize deployment and validation for a shared environment; the concurrency group below only serializes this validation workflow.

Save as `.github/workflows/api-smoke.yml`:

```yaml
name: API Smoke Tests

on:
  workflow_dispatch:
    inputs:
      expected_image:
        description: Deployed application image reference including @sha256 digest
        required: true
        type: string
  workflow_call:
    inputs:
      expected_image:
        required: true
        type: string

permissions:
  contents: read
  id-token: write

concurrency:
  group: api-smoke-staging
  cancel-in-progress: false

jobs:
  test:
    runs-on: [self-hosted, linux, aks-staging]
    environment: staging
    timeout-minutes: 15
    defaults:
      run:
        shell: bash
    env:
      NAMESPACE: staging
      DEPLOYMENT: api-service
      APP_CONTAINER: api
      SERVICE: api-service
      EXPECTED_IMAGE: ${{ inputs.expected_image }}
      READINESS_PATH: /health/ready

    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false

      - uses: actions/setup-node@v7
        with:
          node-version: '24'
          package-manager-cache: false

      - name: Install locked test dependencies
        run: npm ci

      - name: Sign in to Azure using OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Set AKS user context
        uses: azure/aks-set-context@v5
        with:
          resource-group: ${{ vars.AKS_RESOURCE_GROUP }}
          cluster-name: ${{ vars.AKS_CLUSTER_NAME }}
          admin: 'false'
          use-kubelogin: 'true'

      - name: Verify the expected deployment and run tests
        timeout-minutes: 8
        run: |
          set -euo pipefail
          results="$GITHUB_WORKSPACE/test-results"
          mkdir -p "$results"

          if [[ ! "$EXPECTED_IMAGE" =~ @sha256:[a-f0-9]{64}$ ]]; then
            echo "expected_image must include a SHA-256 image digest" >&2
            exit 1
          fi

          get_deployment() {
            kubectl --request-timeout=15s -n "$NAMESPACE" \
              get deployment "$DEPLOYMENT" -o json
          }
          verify_image() {
            jq -e --arg name "$APP_CONTAINER" --arg image "$EXPECTED_IMAGE" \
              'any(.spec.template.spec.containers[];
                   .name == $name and .image == $image)' >/dev/null
          }

          before=$(get_deployment)
          if ! verify_image <<< "$before"; then
            echo "Expected image is not deployed; finish deployment first" >&2
            exit 1
          fi
          generation=$(jq -r '.metadata.generation' <<< "$before")

          kubectl -n "$NAMESPACE" rollout status \
            "deployment/$DEPLOYMENT" --timeout=180s

          ready=$(get_deployment)
          verify_image <<< "$ready"
          [[ "$(jq -r '.metadata.generation' <<< "$ready")" == "$generation" ]]

          kubectl -n "$NAMESPACE" port-forward \
            --address=127.0.0.1 "svc/$SERVICE" 18080:80 \
            >"$results/port-forward.log" 2>&1 &
          pf_pid=$!
          trap 'kill "$pf_pid" 2>/dev/null || true' EXIT

          deadline=$((SECONDS + 45))
          while true; do
            if ! kill -0 "$pf_pid" 2>/dev/null; then
              cat "$results/port-forward.log"
              exit 1
            fi
            if curl --fail --silent --max-time 2 \
              "http://127.0.0.1:18080$READINESS_PATH" >/dev/null; then
              break
            fi
            if (( SECONDS >= deadline )); then
              echo "API readiness timed out" >&2
              cat "$results/port-forward.log"
              exit 1
            fi
            sleep 1
          done

          cd "$GITHUB_WORKSPACE/collections/core-api"
          npx --no-install bru run \
            --env staging \
            --env-var baseUrl=http://127.0.0.1:18080 \
            --reporter-json "$results/bruno.json" \
            --reporter-junit "$results/bruno.xml" \
            --reporter-skip-all-headers \
            --reporter-skip-body

          test -s "$results/bruno.json"
          test -s "$results/bruno.xml"

          after=$(get_deployment)
          verify_image <<< "$after"
          [[ "$(jq -r '.metadata.generation' <<< "$after")" == "$generation" ]]

      - name: Upload test reports and port-forward diagnostics
        if: always()
        uses: actions/upload-artifact@v7
        with:
          name: api-smoke-${{ github.run_id }}-${{ github.run_attempt }}
          path: test-results/
          if-no-files-found: warn
          retention-days: 7
```

The image and generation checks detect an unexpected application image or Deployment change. They do not identify every external configuration change or prevent another deployment from racing with the test. Serialize the full deployment/test sequence, or use an isolated environment per release.

When the runner can reach the real HTTPS endpoint, remove the port-forward process and use that endpoint for the readiness check and Bruno `baseUrl`. Keep image verification and report publication.

Action tags are shown for readability. Pin reviewed action commit SHAs in your maintained workflow, and keep runner tooling and dependencies updated. This example targets GitHub.com; check action compatibility if using GitHub Enterprise Server.

## 4. Pattern 2: In-Cluster Ephemeral Job

Run this after verifying the intended deployment and rollout. The Job does not deploy the application or verify which application revision it is calling.

The test image must contain Python, pytest, required dependencies, and `tests/` under its working directory. The tests must explicitly read `BASE_URL` and use bounded HTTP timeouts. Replace the image digest placeholder, and configure registry pull access if the image is private.

Save as `test-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: api-functional-test-
  namespace: staging
spec:
  ttlSecondsAfterFinished: 3600
  activeDeadlineSeconds: 300
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      automountServiceAccountToken: false
      containers:
        - name: test-runner
          image: ghcr.io/my-org/api-tests@sha256:REPLACE_WITH_DIGEST
          command: ["python", "-m", "pytest"]
          args: ["-v", "tests/", "-p", "no:cacheprovider"]
          env:
            - name: BASE_URL
              value: "http://api-service.staging.svc.cluster.local:80"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: "1"
              memory: 512Mi
```

Adjust resources for the suite. If tests need Kubernetes API access or a projected identity token, configure a dedicated service account and the required token projection instead of relying on the default account.

Save the following as `run-api-job.sh`. It requires Bash, `kubectl`, and `jq`:

```bash
#!/usr/bin/env bash
set -euo pipefail

namespace=staging
results="${RESULTS_DIR:-test-results}"
mkdir -p "$results"

# generateName + create produces a new Job on every invocation.
job_ref=$(kubectl --request-timeout=15s -n "$namespace" \
  create -f test-job.yaml -o name)
printf '%s\n' "$job_ref" > "$results/job-name.txt"

collect_diagnostics() {
  rc=$?
  trap - EXIT
  kubectl --request-timeout=15s -n "$namespace" \
    logs "$job_ref" --all-containers=true --pod-running-timeout=10s \
    > "$results/job.log" 2>&1 || true
  kubectl --request-timeout=15s -n "$namespace" \
    describe "$job_ref" > "$results/job-describe.txt" 2>&1 || true
  kubectl --request-timeout=15s -n "$namespace" \
    get "$job_ref" -o json > "$results/job.json" 2>&1 || true
  cat "$results/job.log" || true
  exit "$rc"
}
trap collect_diagnostics EXIT

deadline=$((SECONDS + 360))
while (( SECONDS < deadline )); do
  state=$(kubectl --request-timeout=15s -n "$namespace" \
    get "$job_ref" -o json)

  if jq -e 'any(.status.conditions[]?;
      (.type == "Failed" or .type == "FailureTarget")
      and .status == "True")' <<< "$state" >/dev/null; then
    echo "Test Job failed: $job_ref" >&2
    exit 1
  fi

  if jq -e 'any(.status.conditions[]?;
      .type == "Complete" and .status == "True")' \
      <<< "$state" >/dev/null; then
    echo "Test Job completed: $job_ref"
    exit 0
  fi

  sleep 2
done

echo "Timed out waiting for $job_ref" >&2
exit 1
```

The script preserves the failing exit status while collecting diagnostics. It leaves the Job available for inspection until TTL cleanup. The Job's own deadline bounds execution even if the runner disconnects.

In GitHub Actions, run the script and upload `test-results/` in a separate step with `if: always()`, as in Pattern 1.

This example exports logs and Job metadata. For JUnit/HTML files, have the test container upload them to artifact storage before exiting, or write them to storage that a separate collector can access. `kubectl cp` cannot reliably retrieve files from an already-terminated container because it relies on executing a command inside the container.

## 5. Argo CD Integration

When Argo CD owns deployment, choose one of these approaches:

- **Runner-owned tests:** wait for the intended Git revision, successful sync operation, and healthy application, then run the suite.
- **Argo CD-owned tests:** use an explicit `PostSync` Job that runs after application resources are healthy.

For a PostSync Job, replace the generated-name metadata from Pattern 2 with:

```yaml
metadata:
  name: api-functional-test
  namespace: staging
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

Let Argo CD manage this hook instead of running `run-api-job.sh` for the same resource. `BeforeHookCreation` recreates the named hook on a subsequent applicable sync. Export reports before cleanup; the next sync can replace the previous Job.

For this Argo-managed variant, remove `ttlSecondsAfterFinished` and use the hook deletion policy to control cleanup. Avoid selective sync when relying on hooks, because hooks do not run during selective sync.

Argo CD does not automatically execute Helm test hooks. Use an Argo CD hook explicitly or run Helm tests through a separate Helm-managed process. A failed PostSync test marks the sync operation as failed; it does not automatically roll back the application. Configure the pipeline to inspect operation success as well as application health before promotion.

## 6. Operational Guidelines

- Keep deployment verification tied to an immutable image and the intended configuration revision.
- Prefer the real application endpoint for release acceptance; use port-forwarding for narrower application checks.
- Bound readiness waits, HTTP requests, test execution, and Kubernetes Jobs.
- Keep retries explicit. Retrying state-changing tests requires isolated data and cleanup.
- Publish diagnostics on failure and treat missing test execution or missing required reports as failures.
- Keep credentials and sensitive request/response data out of published reports.
- Use continuous monitoring alongside deployment tests to observe behavior after release.

## References

- [Playwright API testing](https://playwright.dev/docs/api-testing)
- [Playwright test retries](https://playwright.dev/docs/test-retries)
- [Bruno CLI options](https://docs.usebruno.com/bru-cli/run/options)
- [Bruno collection execution](https://docs.usebruno.com/bru-cli/run/overview)
- [pytest reporting](https://docs.pytest.org/en/stable/how-to/output.html)
- [k6 thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
- [Kubernetes port-forward](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [GitHub OIDC with Azure](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-azure)
- [AKS context action](https://github.com/Azure/aks-set-context)
- [Azure Pipelines deployment checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals)
- [Argo CD sync phases and waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Argo CD Helm integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
