# Connect Fourtkeh deployment

**V1 requires exactly one backend replica.** Games, rooms, presence and matchmaking live in memory; pod restart loses active games. Analytics has exclusive filesystem ownership. The schema rejects replica counts other than 1, and upgrades use `Recreate` to avoid overlapping writers. Expect a brief interruption on upgrade.

The root Dockerfile builds React with Node 24, tests/builds the Java 25 application and copies only the executable JAR into a Temurin JRE image. One Java process serves HTTP, React, WebSocket and Admin. No separate frontend or Admin service is needed.

## Build, run and transfer

From the repository root with a Linux Docker engine:

```sh
docker build -t connect-fourtkeh:1.0.0 .
docker run --rm --name connect-fourtkeh -p 8080:8080 \
  --mount type=volume,src=connect-fourtkeh-analytics,dst=/data/analytics \
  connect-fourtkeh:1.0.0 \
  --connect-fourtkeh.analytics.storage.path=/data/analytics
curl http://localhost:8080/actuator/health
```

For a simple Compose deployment using the locally built image:

```sh
docker build --build-arg SKIP_TESTS=true -t connect-fourtkeh:1.0.0 .
docker compose -f deploy/compose.yaml up -d
docker compose -f deploy/compose.yaml logs -f
docker compose -f deploy/compose.yaml down
```

Compose exposes port 8080 by default and keeps analytics in a named volume. Set `CONNECT_FOURTKEH_PORT` to use another host port, and `CONNECT_FOURTKEH_ADMIN_PASSWORD` in your environment before starting to enable Admin; without a password, gameplay remains available. Ordinary image builds still run tests by default; `SKIP_TESTS=true` skips them when explicitly desired.

The image defaults to non-root UID 10001/group 0; OpenShift may assign another UID. No chart `runAsUser` is specified. Mounted storage must be writable by the assigned UID/group. There is no root chmod init container. Docker's ordinary local volume initialization and OpenShift's volume group policy differ; check your storage driver/SCC when using an existing PVC. The Java heap uses 65% of the detected memory limit; requests/limits are initial budgets, not benchmark claims.

```sh
docker save -o connect-fourtkeh-1.0.0.tar connect-fourtkeh:1.0.0
# Optional compressed archive (POSIX shell):
docker save connect-fourtkeh:1.0.0 | gzip > connect-fourtkeh-1.0.0.tar.gz
# Transfer the archive to the destination machine, then:
docker load -i connect-fourtkeh-1.0.0.tar
# Or:
gzip -dc connect-fourtkeh-1.0.0.tar.gz | docker load
```

A `docker save` tar is an image archive, not an application directory. Import it with `docker load`; do not manually unpack it to run the application. PowerShell users should use the uncompressed `-o`/`-i` commands to preserve binary data.

```sh
docker tag connect-fourtkeh:1.0.0 artifactory.example.com/docker-local/connect-fourtkeh:1.0.0
docker login artifactory.example.com
docker push artifactory.example.com/docker-local/connect-fourtkeh:1.0.0
```

Podman uses the same `build`, `run`, `save -o`, `load -i`, `tag`, `login` and `push` commands. Helm never logs into the registry. Configure an existing Kubernetes docker-registry Secret through `imagePullSecrets: [{name: artifactory-pull}]`.

## Application configuration and credentials

Edit `helm/connect-fourtkeh/config/application.yaml` **before linting, packaging or deploying**. This is ordinary Spring YAML, merged over packaged defaults, mounted at `/config/application.yaml`. Its local schema reference provides IDE completion/validation for project-owned properties and the explicit Spring settings. Cross-field constraints (for example allowed/default turn times, heartbeat/TTL and query duration ceilings) remain validated by the application at startup. Helm validates `values.schema.json`; it does not validate arbitrary application YAML against the separate IDE schema.

Application settings never belong in values. Do not change server.port or the analytics path without updating the deployment contract: chart probes use 8080 and analytics is mounted at `/data/analytics`. Production cookies are Secure, requiring HTTPS at an ingress/Route. For a local HTTP smoke test, override `--server.servlet.session.cookie.secure=false`.

The username, enabled flag and session timeout are application config. Supply the password only through an existing Secret, injected as `CONNECT_FOURTKEH_ADMIN_PASSWORD`. Without a password Admin stays unavailable and gameplay starts. Use a private password file outside the repository to avoid placing a credential in shell history:

```sh
oc create secret generic connect-fourtkeh-admin --from-file=password=/private/path/admin-password
# kubectl create secret generic ... works identically
```

The file must contain the password without a trailing newline. Set `admin.existingSecret: connect-fourtkeh-admin` and `admin.passwordKey: password`. `examples/admin-secret.yaml.example` contains only a placeholder. Never commit a populated copy. Secret changes require a Deployment restart because environment injection happens at process start; config-file changes roll automatically through the checksum.

Visit `/admin` directly to sign in; it appears in the main sidebar only after authentication. Sessions use HttpOnly SameSite cookies and CSRF, survive navigation/refresh until logout/expiry and are lost on process restart. The Route exposes this same web application including `/admin`; there is no separate public Actuator or Admin Service. Only health is exposed by Actuator, with details hidden.

## Helm install and upgrade

Requires Helm 3/4 and a Kubernetes namespace (OpenShift only for optional Route).

```sh
helm lint deploy/helm/connect-fourtkeh
helm template connect-fourtkeh deploy/helm/connect-fourtkeh -f deploy/helm/connect-fourtkeh/values.yaml
helm upgrade --install connect-fourtkeh deploy/helm/connect-fourtkeh \
  -f deploy/helm/connect-fourtkeh/values.yaml \
  --set image.repository=artifactory.example.com/docker-local/connect-fourtkeh \
  --set-string image.tag=1.0.0 \
  --set admin.existingSecret=connect-fourtkeh-admin
# Optional portable chart archive after editing config:
helm package deploy/helm/connect-fourtkeh
```

Prefer a deployment-specific values file for repeatability. Resources: one Deployment, one ClusterIP Service with named `http` port, one application ConfigMap, optional PVC and optional OpenShift Route. A writable `/tmp` emptyDir supports the read-only image root. Startup allows 180 seconds, then liveness/readiness use the enabled `/actuator/health/liveness` and `/actuator/health/readiness` groups. Analytics degradation does not join these probe groups.

Examples (combine with your image/Secret values):

```sh
# Internal / ordinary Kubernetes, no Route:
helm template connect-fourtkeh deploy/helm/connect-fourtkeh -f deploy/examples/internal.values.yaml
# OpenShift assigns hostname:
helm template connect-fourtkeh deploy/helm/connect-fourtkeh -f deploy/examples/generated-route.values.yaml
# Explicit DNS and edge TLS:
helm template connect-fourtkeh deploy/helm/connect-fourtkeh -f deploy/examples/dns-route.values.yaml
# Durable analytics:
helm template connect-fourtkeh deploy/helm/connect-fourtkeh -f deploy/examples/persistent.values.yaml
# Ephemeral analytics:
helm template connect-fourtkeh deploy/helm/connect-fourtkeh -f deploy/examples/ephemeral.values.yaml
```

Route is disabled by default. When enabled, blank/null host omits `spec.host`; explicit host uses your DNS. TLS supports edge termination (the backend speaks HTTP). WebSocket `/ws/**` uses the same Service and Route as HTTP; configure router idle timeout annotations to match your environment if necessary. No sticky sessions are required with one replica. DNS, certificates, router timeout behavior and PVC permissions need real cluster verification.

`persistence.enabled=true` creates a PVC at `/data/analytics`. Configure positive `size`, `accessModes` and `storageClass` (null: cluster default; empty string: no class). `existingClaim` uses your PVC without creating one. Disabled persistence mounts emptyDir at exactly the same path and loses history on pod replacement. The generated PVC has Helm's keep policy and survives uninstall; reuse it with `existingClaim` on reinstall, or deliberately delete it after backing up. StorageClass changes and shrinking a PVC are not ordinary upgrades. The optional application storage cap may delete finalized history earlier than target retention; increasing retention cannot restore deleted data.

## Operate and troubleshoot

```sh
oc get pods
oc get route
oc logs deployment/connect-fourtkeh-connect-fourtkeh
oc describe pod <pod-name>
oc get pvc
# Ordinary Kubernetes: use kubectl; skip oc get route.
kubectl port-forward service/connect-fourtkeh-connect-fourtkeh 8080:8080
curl http://localhost:8080/actuator/health/readiness
# Change config/values, then repeat helm upgrade --install above.
helm history connect-fourtkeh
helm rollback connect-fourtkeh <revision>
# After a Secret update:
oc rollout restart deployment/connect-fourtkeh-connect-fourtkeh
helm uninstall connect-fourtkeh
```

Inspect events for image pull failures, missing Secrets, pending PVCs or UID/group permission errors. Storage diagnostics in Admin expose analytics degradation while gameplay remains available. Upgrade/rollback/restart loses in-memory games and Admin sessions. Application checksum includes the exact mounted file; changing it rolls the pod without manual deletion. The retained PVC and external Secrets are not removed by uninstall; manage their lifecycle explicitly.
