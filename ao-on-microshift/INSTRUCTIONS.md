# Build automation orchestrator on MicroShift and connect it to an existing AAP

This is a runbook for an AI agent (or a person) to stand up Red Hat Ansible Automation Platform
**automation orchestrator (AO)** on **MicroShift** in a single RHEL VM, then connect it to an
**existing AAP**. It needs no OpenShift cluster.

It was written from a real build and includes every fix that build needed. Follow the phases in
order. Each phase ends with a **Verify** step; don't start the next phase until it passes.

> **Support status.** Red Hat documents AO on OpenShift only. MicroShift is not a supported
> platform for AO, so use this for labs and demos, not production. Everything below uses
> supported components (RHEL, MicroShift, OLM, the AO operator from Red Hat's catalog); only the
> combination is unsupported.

---

## Rules for the agent

1. **Ask the human, don't guess,** for everything in [Inputs](#inputs). Several steps need their
   Red Hat login or AAP credentials; the human does those, or supplies the secret for you to store.
2. **Never put a secret in a command line, a log, a chat reply, or a git commit.** Write secrets
   to files on the VM with `umask 077` by piping them over SSH on stdin, as shown below. Don't
   echo them back. Scan before committing anything.
3. **Start read-only.** Phase 1 changes nothing. Get the human's go-ahead before writing to their
   AAP (Phase 12 with demo content).
4. **Verify every phase** before moving on. If a Verify step fails, use
   [Troubleshooting](#troubleshooting) before improvising.
5. **Run commands on the VM over SSH** as a sudo-capable user. Commands below assume you're in a
   shell on the VM; wrap them in `ssh <SSH_USER>@<VM_IP> '...'` as needed.

---

## Inputs

Collect these before starting. Placeholders in `<ANGLE_BRACKETS>` are used throughout.

| Placeholder | What it is | Who provides it |
|---|---|---|
| `<VM_IP>` | IP of a fresh RHEL VM (see [sizing](#vm-requirements)) | Human |
| `<SSH_USER>` | A sudo user on the VM with your SSH key in `~/.ssh/authorized_keys` | Human |
| `<AO_HOST>` | Hostname for the AO UI, with DNS pointing at `<VM_IP>`. If there's no DNS, use `ao.<VM_IP>.nip.io`, which resolves automatically. | Human |
| `<AAP_URL>` | Existing AAP gateway URL, for example `https://aap.example.com` (AAP 2.5 or later) | Human |
| AAP token | OAuth token with **write** scope for an AAP user with admin rights in the target organization. Create it in AAP under **Access Management > Users > (user) > Tokens**. | Human (secret) |
| Red Hat pull secret | From https://console.redhat.com/openshift/install/pull-secret | Human (secret) |
| Red Hat subscription | The VM must be registered with `subscription-manager` | Human (needs their login) |
| **Demo option** | Which of the three [connection modes](#phase-12-connect-ao-to-aap) to use | Human (decision) |

### VM requirements

| | Minimum | Notes |
|---|---|---|
| OS | RHEL 9.8 | Pairs with MicroShift 4.22. See [version pairing](#phase-3-pick-versions). |
| vCPU | 8 | AO requests about 1.5 CPU; MicroShift and PostgreSQL need the rest. |
| RAM | 16 GB | AO requests about 1.6 GB, and Temporal may use up to 4 GB. |
| Disk | 50 GB root | AO needs no persistent volumes, so no extra LVM space is required. |
| Network | Outbound HTTPS to `registry.redhat.io`, `quay.io`, `cdn.redhat.com`, `github.com`, and `<AAP_URL>` | |

---

## Phase 1: Read-only recon

```bash
id; hostname; cat /etc/redhat-release; uname -m
nproc; free -g | head -2; df -h / | tail -1
sudo -n true && echo "passwordless sudo OK"
sudo subscription-manager status | head -8
sudo dnf repolist 2>&1 | tail -5
rpm -q microshift podman 2>&1
curl -s -o /dev/null -w "registry.redhat.io %{http_code}\n" https://registry.redhat.io/v2/
curl -sk -o /dev/null -w "AAP %{http_code}\n" <AAP_URL>/api/controller/v2/ping/
```

**Verify:** RHEL 9.x on x86_64, resources at or above the table, sudo works,
`registry.redhat.io` answers `401` (reachable, needs auth), AAP answers `200`.

Also check the hostname is not `localhost`. MicroShift generates certificates from it, so set a
real one first if needed: `sudo hostnamectl set-hostname <name>`.

---

## Phase 2: Subscription and repositories

If `dnf repolist` shows repositories, skip to Phase 3.

A common trap: `subscription-manager status` says Simple Content Access is enabled, yet
`dnf repolist` shows **no repositories**. Try a refresh first:

```bash
sudo subscription-manager refresh && sudo dnf repolist
```

If `refresh` fails with `Unknown or expired client certificate (HTTP error code 401)`, the
registration on the VM is stale. No local fix works; the human must re-register, since it needs
their Red Hat login:

```bash
sudo subscription-manager clean
sudo subscription-manager register --username <RH_LOGIN>          # prompts for the password
# or: sudo subscription-manager register --org <ORG_ID> --activationkey <KEY_NAME>
```

Then enable the base repositories if they aren't already:

```bash
sudo subscription-manager repos \
  --enable rhel-9-for-$(uname -m)-baseos-rpms \
  --enable rhel-9-for-$(uname -m)-appstream-rpms
```

**Verify:** `sudo dnf repolist` lists BaseOS and AppStream.

---

## Phase 3: Pick versions

MicroShift supports specific RHEL minor versions. From the MicroShift 4.22 documentation:

| RHEL | MicroShift | Status |
|---|---|---|
| 9.8 | 4.22 | Supported. **Use this.** |
| 9.6 | 4.20 or 4.21 | Supported |
| 10.2 | 4.22 | Technology Preview only |

If the VM is on 9.6, either update it (`sudo dnf update -y && sudo reboot`, which moves to the
latest 9.x) and use 4.22, or stay on 9.6 and use 4.21. The rest of this runbook uses
`<MS_VERSION>` for the choice.

**Verify:** `cat /etc/redhat-release` matches the row you picked.

---

## Phase 4: Install MicroShift and OLM

```bash
sudo subscription-manager repos \
  --enable rhocp-<MS_VERSION>-for-rhel-9-$(uname -m)-rpms \
  --enable fast-datapath-for-rhel-9-$(uname -m)-rpms
sudo dnf install -y microshift microshift-olm openshift-clients
```

`fast-datapath` supplies OVN, which MicroShift needs. `microshift-olm` is the optional Operator
Lifecycle Manager; AO is installed as an operator, so it's required. Don't start the service yet.

**Verify:** `rpm -q microshift microshift-olm` shows both at `<MS_VERSION>`.

---

## Phase 5: Pull secret

The human downloads their pull secret. Put it on the VM without it passing through a command
line, then lock it down:

```bash
# from the human's workstation:
scp pull-secret.txt <SSH_USER>@<VM_IP>:/tmp/ps.json
# on the VM:
sudo install -o root -g root -m 600 /tmp/ps.json /etc/crio/openshift-pull-secret && rm -f /tmp/ps.json
```

This file is used by CRI-O for **every** image pull, so it also authenticates the AO operator
and AO images from `registry.redhat.io`. Podman can use it too with `--authfile`.

**Verify** (prints registry names only, never the secret):

```bash
sudo python3 -c 'import json;print(sorted(json.load(open("/etc/crio/openshift-pull-secret"))["auths"]))'
```

It should include `registry.redhat.io` and `quay.io`.

---

## Phase 6: Firewall

```bash
# mandatory for MicroShift: pod network and host gateway
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=169.254.169.1
# AO UI (80/443) and the Kubernetes API (6443)
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp --add-port=443/tcp --add-port=6443/tcp
sudo firewall-cmd --reload
```

Trusting the pod network also lets pods reach PostgreSQL on the host in Phase 8.

**Verify:** `sudo firewall-cmd --list-all --zone=trusted` shows both sources, and the public zone
lists ports 80, 443 and 6443.

---

## Phase 7: Start MicroShift

```bash
sudo systemctl enable --now microshift
sudo microshift healthcheck -v=1 --timeout=600s     # first start pulls images: a few minutes
mkdir -p ~/.kube && sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config && chmod 600 ~/.kube/config
oc get nodes; oc get pods -A
```

**Verify:** `healthcheck` ends with `MicroShift is ready`, the node is `Ready`, and
`olm-operator`, `catalog-operator` and `router-default` are `Running`.

---

## Phase 8: PostgreSQL 15

AO requires **PostgreSQL 15** and three databases: one for AO, one for Temporal, and
`temporal_visibility`.

> **Documentation discrepancy.** AO's system requirements page says the operator creates
> `temporal_visibility` automatically. It does not. If it's missing, the Temporal migration job
> fails with `database "temporal_visibility" does not exist` and AO never becomes ready. Create
> all three databases up front.

Run as root. It generates passwords on the VM into a root-only file and never prints them:

```bash
sudo bash <<'EOF'
set -e
if [ ! -f /root/ao-db-creds.env ]; then
  umask 077
  cat > /root/ao-db-creds.env <<CREDS
PG_ADMIN_PASSWORD=$(openssl rand -hex 16)
BACKEND_DB=orchestrator
BACKEND_USER=orchestrator
BACKEND_PASSWORD=$(openssl rand -hex 16)
TEMPORAL_DB=temporal
TEMPORAL_USER=temporal
TEMPORAL_PASSWORD=$(openssl rand -hex 16)
CREDS
fi
. /root/ao-db-creds.env

podman pull --authfile /etc/crio/openshift-pull-secret registry.redhat.io/rhel9/postgresql-15:latest

# Quadlet: a systemd-managed container that restarts on boot
cat > /etc/containers/systemd/ao-postgres.container <<UNIT
[Unit]
Description=PostgreSQL 15 for automation orchestrator
After=network-online.target
Wants=network-online.target

[Container]
ContainerName=ao-postgres
Image=registry.redhat.io/rhel9/postgresql-15:latest
PublishPort=5432:5432
Volume=ao-pgdata:/var/lib/pgsql/data:Z
Environment=POSTGRESQL_ADMIN_PASSWORD=${PG_ADMIN_PASSWORD}
Environment=POSTGRESQL_MAX_CONNECTIONS=300

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target default.target
UNIT
chmod 600 /etc/containers/systemd/ao-postgres.container

systemctl daemon-reload
systemctl start ao-postgres.service
for i in $(seq 1 30); do
  podman exec ao-postgres bash -c "PGPASSWORD=${PG_ADMIN_PASSWORD} psql -h 127.0.0.1 -U postgres -c 'SELECT 1'" >/dev/null 2>&1 && break
  sleep 2
done

q() { podman exec ao-postgres bash -c "PGPASSWORD=${PG_ADMIN_PASSWORD} psql -h 127.0.0.1 -U postgres -tAc \"$1\""; }
q "SELECT 1 FROM pg_roles WHERE rolname='${BACKEND_USER}'"  | grep -q 1 || q "CREATE USER ${BACKEND_USER} PASSWORD '${BACKEND_PASSWORD}'"
q "SELECT 1 FROM pg_roles WHERE rolname='${TEMPORAL_USER}'" | grep -q 1 || q "CREATE USER ${TEMPORAL_USER} PASSWORD '${TEMPORAL_PASSWORD}' CREATEDB"
for db in "${BACKEND_DB}:${BACKEND_USER}" "${TEMPORAL_DB}:${TEMPORAL_USER}" "temporal_visibility:${TEMPORAL_USER}"; do
  name=${db%%:*}; owner=${db##*:}
  q "SELECT 1 FROM pg_database WHERE datname='${name}'" | grep -q 1 || q "CREATE DATABASE ${name} OWNER ${owner}"
done
q "SELECT datname, pg_get_userbyid(datdba) FROM pg_database WHERE datistemplate=false"
EOF
```

`POSTGRESQL_MAX_CONNECTIONS=300` follows AO's guidance to raise the default of 100 when several
AO replicas share one instance. `sslMode: disable` is used in Phase 11 because this container has
no TLS; that's acceptable for a lab where PostgreSQL only listens on the VM.

**Verify:** the last query lists `orchestrator` (owner `orchestrator`), and `temporal` and
`temporal_visibility` (owner `temporal`). `systemctl is-active ao-postgres` returns `active`.

---

## Phase 9: Operator catalog

Unlike OpenShift, **MicroShift ships no operator catalogs.** Add Red Hat's operator index. The
`securityContextConfig: restricted` setting is required on MicroShift.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: registry.redhat.io/redhat/redhat-operator-index:v<MS_VERSION>
  displayName: Red Hat Operators
  publisher: Red Hat
  grpcPodConfig:
    securityContextConfig: restricted
  updateStrategy:
    registryPoll:
      interval: 30m
EOF
```

MicroShift has no `packagemanifests` API, so check the index contents directly:

```bash
until [ "$(oc -n openshift-marketplace get catalogsource redhat-operators -o jsonpath='{.status.connectionState.lastObservedState}')" = READY ]; do sleep 5; done
POD=$(oc -n openshift-marketplace get pods -l olm.catalogSource=redhat-operators -o jsonpath='{.items[0].metadata.name}')
oc -n openshift-marketplace exec "$POD" -- ls /configs | grep automation-orchestrator
```

**Verify:** the catalog reaches `READY` (usually within 2 minutes) and the last command prints
`automation-orchestrator-operator`.

---

## Phase 10: Install the AO operator

The OperatorGroup **must** be AllNamespaces (`spec: {}`); AO supports no other scope.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: automation-orchestrator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: automation-orchestrator-operator
  namespace: automation-orchestrator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: automation-orchestrator-operator
  namespace: automation-orchestrator
spec:
  channel: stable
  name: automation-orchestrator-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

**Verify:** within a couple of minutes, `oc -n automation-orchestrator get csv` shows the
operator `Succeeded`, and `oc get crd automationorchestrators.aap.ansible.com` exists.

---

## Phase 11: Create the AO instance

The database secrets are built from the root-only credentials file, so no password appears on a
command line the agent types:

```bash
sudo KUBECONFIG=$HOME/.kube/config bash <<'EOF'
set -e
. /root/ao-db-creds.env
NS=automation-orchestrator
oc -n $NS create secret generic orchestrator-pg-credentials \
  --from-literal=database="$BACKEND_DB" --from-literal=username="$BACKEND_USER" --from-literal=password="$BACKEND_PASSWORD" \
  --dry-run=client -o yaml | oc apply -f -
oc -n $NS create secret generic temporal-pg-credentials \
  --from-literal=database="$TEMPORAL_DB" --from-literal=username="$TEMPORAL_USER" --from-literal=password="$TEMPORAL_PASSWORD" \
  --dry-run=client -o yaml | oc apply -f -
EOF

cat <<'EOF' | oc apply -f -
apiVersion: aap.ansible.com/v1alpha1
kind: AutomationOrchestrator
metadata:
  name: automation-orchestrator
  namespace: automation-orchestrator
spec:
  postgres:
    host: <VM_IP>          # pods reach the host's published PostgreSQL port
    port: 5432
    sslMode: disable
    backendDatabase:
      secretRef:
        name: orchestrator-pg-credentials
    temporalDatabase:
      secretRef:
        name: temporal-pg-credentials
  ingress:
    type: Route            # MicroShift includes the OpenShift router
    host: <AO_HOST>
EOF
```

Wait for it:

```bash
until [ "$(oc -n automation-orchestrator get automationorchestrator automation-orchestrator \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = True ]; do sleep 10; done
oc -n automation-orchestrator get pods
```

**Verify:** `Ready=True` (reason `AllComponentsReady`), usually 1 to 5 minutes. Pods running:
backend ×2, worker ×2, ui ×2, background-worker, temporal, redis. Migration jobs show
`Completed`. Then:

```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://<AO_HOST>/          # expect 200
```

The admin password is generated by the operator. Retrieve it on the VM only when needed; don't
paste it into chat unless the human asks for it:

```bash
oc -n automation-orchestrator get secret automation-orchestrator-initial-admin-password -o jsonpath='{.data.password}' | base64 -d
```

---

## Phase 12: Connect AO to AAP

### 12a. Let AO reach a private-network AAP

AO blocks integration URLs that aren't on an allow-list, and blocks OIDC to private networks.
These are environment variables on the AO deployments, not fields in the AutomationOrchestrator
resource. The Red Hat demo platform sets the same two values on its own AO installs.

```bash
NS=automation-orchestrator
AAP_HOST=$(echo "<AAP_URL>" | sed -E 's#https?://([^/:]+).*#\1#')
ALLOWED="[\"${AAP_HOST}\",\"<AO_HOST>\"]"
for d in automation-orchestrator-backend automation-orchestrator-worker automation-orchestrator-background-worker; do
  oc -n $NS set env deploy/$d APP_INTEGRATION_URL_ALLOWED_HOSTS="$ALLOWED"
done
oc -n $NS set env deploy/automation-orchestrator-backend APP_OIDC_ALLOW_PRIVATE_NETWORKS=true
for d in automation-orchestrator-backend automation-orchestrator-worker automation-orchestrator-background-worker; do
  oc -n $NS rollout status deploy/$d --timeout=180s
done
```

**Verify:** after 90 seconds (one operator reconcile) the variables are still present, and a pod
can reach AAP:

```bash
oc -n $NS get deploy automation-orchestrator-backend -o yaml | grep -A1 -E "APP_INTEGRATION_URL_ALLOWED_HOSTS|APP_OIDC_ALLOW_PRIVATE_NETWORKS"
POD=$(oc -n $NS get pods --no-headers | grep 'backend-' | grep Running | head -1 | awk '{print $1}')
oc -n $NS exec "$POD" -- curl -sk -o /dev/null -w "%{http_code}\n" <AAP_URL>/api/controller/v2/ping/   # expect 200
```

An AO operator upgrade may reset these. If AO later can't reach AAP, check them first.

### 12b. Choose a connection mode

**Ask the human which mode they want.** Modes 2 and 3 write to their AAP only in mode 3.

| Mode | What it does | Writes to AAP? |
|---|---|---|
| **1. Connect only** | Creates an AAP credential and a global AAP integration in AO. No workflows. | No |
| **2. Connect + demo workflows as drafts** | Mode 1, plus imports four example workflows. They stay drafts because their job templates don't exist on AAP. | No |
| **3. Full demo** | Mode 2, plus creates an `AO Demo Content` project (syncing this repo), an `AO Demo Inventory` (localhost), and five `AO Demo \| ...` job templates on AAP, then **publishes the Fleet Health Demo workflow** so it runs end to end. | **Yes** |

All three use [`ao_setup.yml`](../ao_setup.yml) from this repository with `--tags aoconfig`,
run on the VM itself (it only needs `ansible-core`; tested with the RHEL 9 package, 2.14).

### 12c. Store the credentials on the VM

Pipe them over stdin into a mode-600 file. Replace the values locally; nothing here should end up
in shell history or a repository:

```bash
ssh <SSH_USER>@<VM_IP> 'umask 077; cat > ~/aap-creds.yml' <<'EOF'
demo_aap_url: <AAP_URL>
demo_aap_token: <AAP_TOKEN>
ao_url: https://<AO_HOST>
EOF
```

On the VM, append the AO admin password straight from the cluster secret, so it's never typed:

```bash
PW=$(oc -n automation-orchestrator get secret automation-orchestrator-initial-admin-password -o jsonpath='{.data.password}' | base64 -d)
printf 'ao_password: "%s"\n' "$PW" >> ~/aap-creds.yml; unset PW
```

The `demo_aap_*` variable names are historical: they mean "the AAP you're connecting to".

### 12d. Run the playbook

```bash
sudo dnf install -y ansible-core git
git clone https://github.com/gregsowell/ao-random.git ~/ao-random
cd ~/ao-random
```

**Mode 1: connect only**

```bash
ansible-playbook ao_setup.yml --tags aoconfig -e @~/aap-creds.yml \
  -e demo_content_enabled=false -e '{"ao_workflows": []}'
```

**Mode 2: connect + demo workflows as drafts**

```bash
ansible-playbook ao_setup.yml --tags aoconfig -e @~/aap-creds.yml \
  -e demo_content_enabled=false -e ao_publish_workflows=false \
  -e '{"ao_workflows": [
        {"name": "Disk Utilization Demo 101", "url": "https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/disk-utilization/ao/disk-demo-101.json"},
        {"name": "RHEL CVE Remediation - Intelligent Patching", "url": "https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/cve-remediation/ao/rhel-cve-remediation.json"},
        {"file": "workflows/build-ee.json"},
        {"file": "workflows/fleet-health-demo.json"}]}'
```

**Mode 3: full demo** (the playbook's defaults)

```bash
ansible-playbook ao_setup.yml --tags aoconfig -e @~/aap-creds.yml
```

To have the demo project sync a fork instead of this repository, add
`-e demo_content_scm_url=https://github.com/<you>/<fork>.git`.

**Verify:** `PLAY RECAP` shows `failed=0`, and the final summary lists the credential and
integration (and, in modes 2 and 3, the workflows). The playbook can be rerun safely; it updates
what already exists.

---

## Phase 13: Verify the integration, and run the demo (mode 3)

In the AO UI at `https://<AO_HOST>`, **Configuration > Integrations** should show the AAP
integration as **Available**. That status comes from AO's own health check against AAP.

For mode 3, run **Workflows > Fleet Health Demo** with an input `scenario`:

| Scenario | Expected path |
|---|---|
| `healthy` | ends at **Notify: All Clear** |
| `degraded` | remediates two hosts, ends at **Notify: Auto-Remediated** |
| `critical` | one host needs a person, ends at **Notify: Page On-Call** |

A run takes about a minute and launches five jobs on AAP, all named `AO Demo | ...`.

---

## Optional: a trusted certificate

The router serves MicroShift's self-signed default certificate (`CN=*.apps.example.com`), so
browsers warn on every visit to `<AO_HOST>`.

**Ask the human:** would they like a trusted certificate installed? There are two options: bring
one they already have, or have one issued automatically by Let's Encrypt. If yes, follow
[TLS.md](TLS.md), which covers both, including DNS provider automation (e.g. Cloudflare) for the
Let's Encrypt path. It picks up from here and returns you to this point when done. If no, skip to
Phase 12.

To serve AO under a second hostname, add another Route to the `automation-orchestrator-ui`
service (edge TLS, target port `http`), and add that hostname to
`APP_INTEGRATION_URL_ALLOWED_HOSTS`. Changing `spec.ingress.host` on the AutomationOrchestrator
resource renames the primary Route.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dnf repolist`: "No repositories available" despite Simple Content Access | Entitlement certificate never downloaded | `subscription-manager refresh` |
| `refresh` fails: `Unknown or expired client certificate (HTTP 401)` | Stale or cloned registration | `subscription-manager clean`, then the human re-registers (Phase 2) |
| MicroShift won't start | Missing or malformed pull secret | Check `/etc/crio/openshift-pull-secret` (Phase 5 Verify); `journalctl -u microshift` |
| `oc get packagemanifests`: resource type not found | Normal on MicroShift | Inspect the catalog pod's `/configs` (Phase 9) |
| CatalogSource stuck in `TRANSIENT_FAILURE` for minutes | Index image still pulling, or missing `securityContextConfig: restricted` | Wait; confirm the setting; `oc -n openshift-marketplace describe pod` |
| AO `Degraded=True`, `MigrationFailed`; Temporal migration pods in `Error` | `temporal_visibility` database missing | Create it owned by the Temporal user, then `oc -n automation-orchestrator delete job -l app.kubernetes.io/part-of=automation-orchestrator` so the operator retries |
| AO pods can't reach PostgreSQL | Pod network not trusted by firewalld | Phase 6 trusted-zone sources |
| Browsing `<AO_HOST>` gives a router 503 | Route host doesn't match the name | Set `spec.ingress.host` or add a Route (Optional section above) |
| AO integration can't reach AAP, or OIDC fails | Allow-list missing or reset by an operator upgrade | Reapply Phase 12a |
| Playbook: credential save fails with HTTP 422 unknown fields | Credential type fields differ from what's sent | The playbook prints the type's fields; set `ao_aap_credential_inputs` to match |
| Background worker never scales | No metrics API on MicroShift | Expected; it runs at its minimum replicas |

---

## Removing it

```bash
oc delete automationorchestrator automation-orchestrator -n automation-orchestrator
oc delete subscription automation-orchestrator-operator -n automation-orchestrator
oc delete csv -n automation-orchestrator --all
oc delete namespace automation-orchestrator
sudo systemctl stop ao-postgres.service && sudo rm /etc/containers/systemd/ao-postgres.container && sudo systemctl daemon-reload
sudo podman volume rm ao-pgdata          # deletes the AO database permanently
```

On AAP (mode 3 only), delete the five `AO Demo | ...` job templates, then `AO Demo Inventory`
and `AO Demo Content`. Revoke the AAP token when you're done with it.

---

## What was tested

This runbook was produced from a working build: RHEL 9.8, MicroShift 4.22.13 with
`microshift-olm`, CRI-O 1.35, PostgreSQL 15.18 in Podman, AO operator
`automation-orchestrator-operator.v2026.8.1789549281`, connected to AAP 2.7 with an OAuth token.
Mode 3 ran end to end: the integration reported Available, and the Fleet Health Demo `degraded`
scenario completed in about 45 seconds through five successful AAP jobs.
