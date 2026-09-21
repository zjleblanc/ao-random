# TLS certificate addendum: a trusted certificate for AO on MicroShift

This is an addendum to [`INSTRUCTIONS.md`](INSTRUCTIONS.md) for an AI agent (or a person). It
picks up once AO is running (`INSTRUCTIONS.md` Phase 11) and the router is serving MicroShift's
self-signed default certificate for `<AO_HOST>`. It replaces that certificate with one browsers
trust, using either a certificate the human already has, or one issued automatically by
**Let's Encrypt through the cert-manager Operator for Red Hat OpenShift**.

> **Note on confidence.** Unlike `INSTRUCTIONS.md`, this addendum was written from the cert-manager
> Operator for Red Hat OpenShift documentation (versions 1.19-1.20) and upstream cert-manager
> provider docs, not from a verified build. Treat every **Verify** step as load-bearing, and if
> something doesn't match what's documented here (an API field, a namespace, an RBAC subject),
> check the cluster directly rather than assuming this doc is right.

## Rules for the agent

These carry over from `INSTRUCTIONS.md`; the two most relevant here:

1. **Ask the human, don't guess,** for everything in [Inputs](#inputs) below, and at both decision
   points: which path ([A](#path-a-bring-your-own-certificate) vs
   [B](#path-b-automated-issuance-with-lets-encrypt-and-cert-manager)), and which DNS provider.
2. **Never put a secret (cert private key, DNS API token) in a command line, a log, a chat
   reply, or a git commit.** Pipe them over SSH on stdin into files with `umask 077`, as in
   `INSTRUCTIONS.md`. Don't echo them back.

## Inputs

| Placeholder | What it is | Who provides it |
|---|---|---|
| `<AO_HOST>` | Already set in `INSTRUCTIONS.md`; the hostname the Route serves | Carried forward |
| `<CERT_DOMAIN>` | Domain the certificate should cover. Usually equals `<AO_HOST>`; could be a wildcard (`*.apps.example.com`) if you want one cert for multiple hosts | Human |
| **Path choice** | [Bring your own](#path-a-bring-your-own-certificate) or [Let's Encrypt via cert-manager](#path-b-automated-issuance-with-lets-encrypt-and-cert-manager) | Human (decision) |

**Path A only:**

| Placeholder | What it is | Who provides it |
|---|---|---|
| Certificate + key files | PEM-encoded full chain certificate and matching private key for `<CERT_DOMAIN>` | Human (secret) |

**Path B only:**

| Placeholder | What it is | Who provides it |
|---|---|---|
| `<ACME_EMAIL>` | Email for the Let's Encrypt account (expiry notices) | Human |
| **DNS provider** | Where `<CERT_DOMAIN>`'s DNS is hosted: Cloudflare, or [another provider](#other-dns-providers) | Human (decision) |
| DNS API credential | Provider-specific token/key scoped to the zone for `<CERT_DOMAIN>` (see [Phase B2](#phase-b2-create-the-dns-credential-secret)) | Human (secret) |
| **Staging or production** | Whether to test against Let's Encrypt's staging CA first (recommended) before switching to production | Human (decision) |

---

## Decision: pick a path

**Ask the human which path they want**, and if Path B, which DNS provider.

| Path | What it does | Renews itself? | Effort |
|---|---|---|---|
| **A. Bring your own** | Human supplies a cert + key (e.g. from an internal CA, or a cert already issued for this domain). Agent installs it. | No — human re-supplies files before expiry | Low |
| **B. Let's Encrypt via cert-manager** | Installs the cert-manager operator, requests a certificate from Let's Encrypt via a DNS-01 challenge, and lets cert-manager renew it automatically | Yes | Higher, one-time |

Both paths converge on the same mechanism for actually serving the certificate: a
`kubernetes.io/tls` Secret referenced from the AO Route via `spec.tls.externalCertificate` (see
[Phase C](#phase-c-point-the-route-at-the-secret)). This means Path B's certificate renews with no
further action — cert-manager rewrites the Secret in place, and the router picks it up
automatically without the Route needing to change.

---

## Path A: bring your own certificate

### Phase A1: Collect and validate the certificate

Get the certificate chain and private key from the human without letting them pass through a
command line or chat:

```bash
# from the human's workstation:
scp fullchain.pem privkey.pem <SSH_USER>@<VM_IP>:/tmp/
# on the VM:
umask 077
sudo mv /tmp/fullchain.pem /tmp/privkey.pem ~/
```

Validate before installing anything:

```bash
openssl x509 -in ~/fullchain.pem -noout -subject -issuer -dates
openssl x509 -in ~/fullchain.pem -noout -ext subjectAltName
# the modulus hashes must match, confirming the key belongs to the cert:
openssl x509 -in ~/fullchain.pem -noout -modulus | openssl md5
openssl rsa  -in ~/privkey.pem   -noout -modulus | openssl md5   # or: openssl ec -in ~/privkey.pem -noout ... for EC keys
```

**Verify:** `notAfter` is in the future, the SAN list includes `<CERT_DOMAIN>`, and the two
modulus hashes match.

### Phase A2: Create the TLS secret

```bash
oc -n automation-orchestrator create secret tls ao-custom-tls \
  --cert=$HOME/fullchain.pem --key=$HOME/privkey.pem \
  --dry-run=client -o yaml | oc apply -f -
shred -u ~/fullchain.pem ~/privkey.pem
```

**Verify:** `oc -n automation-orchestrator get secret ao-custom-tls -o jsonpath='{.type}'` prints
`kubernetes.io/tls`.

Continue to [Phase C](#phase-c-point-the-route-at-the-secret) with `<TLS_SECRET_NAME>` =
`ao-custom-tls`.

---

## Path B: automated issuance with Let's Encrypt and cert-manager

### Phase B1: Install the cert-manager operator

Uses the catalog already added in `INSTRUCTIONS.md` Phase 9. The operator lives in
`cert-manager-operator`; it deploys the actual cert-manager controller, webhook, and CA injector
into a separate `cert-manager` namespace.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

**Verify:**

```bash
oc -n cert-manager-operator get csv                       # Succeeded
oc get crd certificates.cert-manager.io issuers.cert-manager.io clusterissuers.cert-manager.io
oc -n cert-manager get pods                                # cert-manager, cert-manager-webhook, cert-manager-cainjector all Running
```

### Phase B2: Create the DNS credential secret

**Ask the human which DNS provider hosts `<CERT_DOMAIN>`.** The credential secret and the
`solvers` block in Phase B3 both depend on it. This uses a namespaced `Issuer`, so the credential
secret lives in `automation-orchestrator`, alongside the Route it will end up securing.

#### Cloudflare

Automatable end-to-end: Cloudflare's API can create the DNS-01 `_acme-challenge` TXT record
without a human touching the DNS console for each issuance or renewal.

Ask the human for an **API Token** (not the older Global API Key) from
**My Profile > API Tokens** in the Cloudflare dashboard, scoped to:
- Permission: `Zone > DNS > Edit`
- Zone Resources: the specific zone containing `<CERT_DOMAIN>`

```bash
ssh <SSH_USER>@<VM_IP> 'umask 077; cat > ~/cf-token.txt'   # human pastes the token, then Ctrl-D
oc -n automation-orchestrator create secret generic cloudflare-api-token-secret \
  --from-file=api-token=$HOME/cf-token.txt \
  --dry-run=client -o yaml | oc apply -f -
shred -u ~/cf-token.txt
```

> Red Hat's cert-manager operator docs officially test and support Route 53, Azure DNS, Google
> Cloud DNS, and a small set of webhook providers. Cloudflare is not on that list, but it is a
> built-in provider in the upstream cert-manager this operator packages, so the `cloudflare` field
> under `dns01` is present and functional — it's just not a Red Hat-tested combination. Use the
> [staging issuer](#phase-b3-create-the-issuer) to confirm it works in this environment before
> relying on it.

#### Other DNS providers

Red Hat officially tests: **Amazon Route 53**, **Azure DNS**, **Google Cloud DNS**, and DNS
providers reachable through a **webhook** (e.g. `cert-manager-webhook-ibmcis`). Ask the human
which one applies, then look up that provider's `solvers` block and credential shape in the
[cert-manager Operator for Red Hat OpenShift documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift)
("Configuring an ACME issuer using DNS-01 validation"), or the
[upstream cert-manager DNS01 provider docs](https://cert-manager.io/docs/configuration/acme/dns01/)
for any other provider not listed there. The pattern is the same regardless of provider:

1. Create a namespaced credential Secret in `automation-orchestrator` (as above).
2. Reference it from the `Issuer`'s `solvers[].dns01.<provider>` block in Phase B3.

If the human doesn't know or can't automate their DNS provider, fall back to Path A, or use an
HTTP-01 `Issuer` with an `ingress` solver instead of DNS-01 — but that needs port 80 reachable
from the internet for the challenge, which the DNS-01 path avoids.

**Verify:** `oc -n automation-orchestrator get secret <credential-secret-name>` exists.

### Phase B3: Create the Issuer

Ask the human: **staging first (recommended), or straight to production?** Let's Encrypt rate
limits production issuance; staging certs aren't trusted by browsers but prove the DNS-01 flow
works.

| | Server URL |
|---|---|
| Staging | `https://acme-staging-v02.api.letsencrypt.org/directory` |
| Production | `https://acme-v02.api.letsencrypt.org/directory` |

Cloudflare example (swap the `solvers` block for another provider from
[Phase B2](#phase-b2-create-the-dns-credential-secret) if not using Cloudflare):

```bash
cat <<'EOF' | oc apply -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ao-letsencrypt
  namespace: automation-orchestrator
spec:
  acme:
    server: <STAGING_OR_PRODUCTION_SERVER_URL>
    email: <ACME_EMAIL>
    privateKeySecretRef:
      name: ao-letsencrypt-account-key
    solvers:
      - dns01:
          cloudflare:
            email: <CLOUDFLARE_ACCOUNT_EMAIL>
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
EOF
```

**Verify:** `oc -n automation-orchestrator get issuer ao-letsencrypt -o jsonpath='{.status.conditions[0]}'`
shows `"status":"True","type":"Ready"`. If not, `oc -n automation-orchestrator describe issuer ao-letsencrypt`
— a common cause is the ACME account registration itself failing (bad email, or a leftover account
key from a different server URL; delete `ao-letsencrypt-account-key` and reapply to register fresh).

### Phase B4: Request the certificate

```bash
cat <<'EOF' | oc apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ao-letsencrypt
  namespace: automation-orchestrator
spec:
  secretName: ao-letsencrypt-tls
  commonName: <CERT_DOMAIN>
  dnsNames:
    - <CERT_DOMAIN>
  issuerRef:
    kind: Issuer
    name: ao-letsencrypt
EOF
```

### Phase B5: Wait for issuance

```bash
until [ "$(oc -n automation-orchestrator get certificate ao-letsencrypt \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = True ]; do
  sleep 10
done
oc -n automation-orchestrator get secret ao-letsencrypt-tls
```

**Verify:** the loop exits and `ao-letsencrypt-tls` (type `kubernetes.io/tls`) exists. This
usually takes 1-3 minutes: cert-manager creates the DNS TXT record, waits for DNS propagation
(self-checked before it asks Let's Encrypt to verify), then requests the certificate.

If it never reaches `Ready`, see [Troubleshooting](#troubleshooting) before improvising —
`oc -n automation-orchestrator get challenge` and `describe` on the stuck challenge is the fastest
way to see exactly where it's stuck.

Continue to [Phase C](#phase-c-point-the-route-at-the-secret) with `<TLS_SECRET_NAME>` =
`ao-letsencrypt-tls`.

**Switching staging to production later:** edit the `Issuer`'s `server` to the production URL and
`privateKeySecretRef.name` to a new name (a new server needs a new ACME account), then
`oc -n automation-orchestrator delete secret ao-letsencrypt-tls` to force cert-manager to reissue
against the new issuer. The Route needs no changes — it already points at the secret name, not
its contents.

---

## Phase C: point the Route at the secret

Common to both paths. `<TLS_SECRET_NAME>` is `ao-custom-tls` (Path A) or `ao-letsencrypt-tls`
(Path B).

The Route API can reference a Secret directly (`spec.tls.externalCertificate`) instead of having
the PEM content inlined in the Route object. This means Path B's renewals need no further action
on the Route — the router reads the current Secret contents on every request.

First, find the namespace the default router's service account actually runs in — don't assume:

```bash
oc get pods -A -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default \
  -o jsonpath='{.items[0].metadata.namespace}{"\n"}'
```

This is almost always `openshift-ingress`; the rest of this phase assumes that. Grant the router's
service account read access to just this one secret, then reference it from the Route:

```bash
NS=automation-orchestrator
oc -n $NS create role secret-reader --verb=get,list,watch \
  --resource=secrets --resource-name=<TLS_SECRET_NAME>
oc -n $NS create rolebinding secret-reader-binding \
  --role=secret-reader --serviceaccount=openshift-ingress:router

oc -n $NS get route -l app.kubernetes.io/part-of=automation-orchestrator   # confirm the Route name
oc -n $NS patch route <ROUTE_NAME> --type=merge \
  -p '{"spec":{"tls":{"termination":"edge","externalCertificate":{"name":"<TLS_SECRET_NAME>"}}}}'
```

**Verify:**

```bash
oc -n automation-orchestrator get route <ROUTE_NAME> -o jsonpath='{.spec.tls}{"\n"}'
curl -sv https://<AO_HOST>/ 2>&1 | grep -E 'subject:|issuer:|expire'
curl -s -o /dev/null -w "%{http_code}\n" https://<AO_HOST>/       # expect 200, no -k needed now
```

The `subject:` line should show `<CERT_DOMAIN>`, and for Path B the `issuer:` line should show
Let's Encrypt (or `(STAGING) Let's Encrypt` if still on the staging server).

If the Route rejects `externalCertificate` (a validation error on `oc patch`, or the router keeps
serving the old cert), your MicroShift build may not support it yet — see
[Troubleshooting](#troubleshooting) for the inline-PEM fallback.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `oc patch route ... externalCertificate` fails validation, or has no effect | This MicroShift/router build predates the route external-certificate feature | Fall back to inlining the PEM: `oc -n automation-orchestrator patch route <ROUTE_NAME> --type=merge -p "{\"spec\":{\"tls\":{\"termination\":\"edge\",\"certificate\":\"$(cat fullchain.pem)\",\"key\":\"$(cat privkey.pem)\"}}}"`. This needs re-running after every renewal, since it copies the PEM instead of referencing the secret. |
| Router still serves the old/self-signed cert after patching | RoleBinding missing or wrong service account, so the router can't read the secret | Re-check the router namespace and SA name (Phase C); `oc -n automation-orchestrator get rolebinding secret-reader-binding -o yaml` |
| `Issuer` never reaches `Ready` | ACME account registration failed (bad `<ACME_EMAIL>`, or a private key reused across server URLs) | `oc -n automation-orchestrator describe issuer ao-letsencrypt`; delete the `privateKeySecretRef` secret and reapply |
| `Certificate` stuck, `Challenge` stuck `pending` for minutes | DNS-01 TXT record not created or not yet propagated; DNS API token missing permissions | `oc -n automation-orchestrator describe challenge`; for Cloudflare confirm the token has `Zone:DNS:Edit` on the right zone; `dig TXT _acme-challenge.<CERT_DOMAIN>` from the VM |
| `Order` shows an ACME error about rate limits | Too many production issuance attempts for this domain | Switch to the staging server while testing; Let's Encrypt production allows 5 duplicate certs per week per domain set |
| `CatalogSource redhat-operators` not `READY`, so the operator Subscription never installs | Same catalog used by AO itself; see `INSTRUCTIONS.md` Phase 9 | Confirm Phase 9's Verify still passes before retrying here |
| cert-manager operator CSV `Succeeded` but no pods in `cert-manager` namespace | Operator still reconciling the `CertManager` cluster CR | Wait a minute; `oc get certmanager cluster -o yaml` for its status |

---

## Renewal

- **Path A (bring your own):** manual. Track the certificate's expiry and repeat
  [Phase A1](#phase-a1-collect-and-validate-the-certificate)–[A2](#phase-a2-create-the-tls-secret)
  with new files before it expires; the Route needs no change since it already references the
  secret by name.
- **Path B (cert-manager):** automatic. cert-manager renews roughly 30 days before expiry by
  default (`spec.renewBefore` on the `Certificate` can override this) and rewrites
  `ao-letsencrypt-tls` in place. The router picks up the new content on its own. Spot-check
  occasionally with `oc -n automation-orchestrator get certificate ao-letsencrypt` and confirm
  `cert-manager` pods stay `Running` — a renewal failure is silent from the Route's perspective
  until the old certificate actually expires.

---

## Removing it

```bash
NS=automation-orchestrator
oc -n $NS delete rolebinding secret-reader-binding
oc -n $NS delete role secret-reader
oc -n $NS patch route <ROUTE_NAME> --type=json -p '[{"op":"remove","path":"/spec/tls/externalCertificate"}]'

# Path A:
oc -n $NS delete secret ao-custom-tls

# Path B:
oc -n $NS delete certificate ao-letsencrypt
oc -n $NS delete issuer ao-letsencrypt
oc -n $NS delete secret ao-letsencrypt-tls ao-letsencrypt-account-key cloudflare-api-token-secret
oc delete subscription openshift-cert-manager-operator -n cert-manager-operator
oc delete csv -n cert-manager-operator --all
oc delete namespace cert-manager-operator cert-manager
```

Revoke the DNS provider API token (Cloudflare: delete it from **My Profile > API Tokens**) once
you're done with it.
