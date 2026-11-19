# Configuring an Identity Provider (IDP) for OpenShift Dedicated on GCP via CLI

## 1. Log in to the `ocm` CLI and Retrieve Cluster Details

Authenticate with OpenShift Cluster Manager using one of the following methods:

```bash
# Recommended: opens a browser window for Red Hat SSO login
ocm login --use-auth-code

# For remote servers, SSH sessions, or headless environments without a browser
ocm login --use-device-code

# For scripts and CI/CD pipelines (generate a token at [https://console.redhat.com/openshift/token/show](https://console.redhat.com/openshift/token/show))
ocm login --token="<offline-token>"
```

Verify your active session, find your cluster ID, and inspect its base domain and API URL:

```bash
ocm whoami
ocm list clusters
ocm describe cluster <cluster-id-or-name>
```

---

## 2. Create an Identity Provider (IDP)

### Option A: `htpasswd` (Quick Initial Access)

Create a local `htpasswd` provider and user in a single command:

```bash
ocm create idp \
  --type htpasswd \
  --cluster <cluster-id-or-name> \
  --name htpasswd-admin \
  --username admin \
  --password '<strong-password>'
```

### Option B: OpenID Connect (OIDC)

Before configuring OIDC in OCM, register the following OAuth Callback URL in your external identity provider (Keycloak, Microsoft Entra ID, Okta, Google, etc.):

```text
https://oauth-openshift.apps.<cluster_name>.<cluster_domain>/oauth2callback/<idp_provider_name>
```

* **Interactive mode** (the CLI will prompt you step-by-step for `name`, `client-id`, `client-secret`, `issuer-url`, and claims):
  ```bash
  ocm create idp --type openid --cluster <cluster-id-or-name>
  ```

* **Non-interactive mode** (for automation using the built-in OCM API client):
  ```bash
  ocm post /api/clusters_mgmt/v1/clusters/<cluster-id>/identity_providers <<EOF
  {
    "name": "corporate-oidc",
    "type": "OpenIDIdentityProvider",
    "mapping_method": "claim",
    "open_id": {
      "client_id": "<client-id>",
      "client_secret": "<client-secret>",
      "issuer": "https://<issuer-url>",
      "claims": {
        "email": ["email"],
        "name": ["name"],
        "preferred_username": ["preferred_username"],
        "groups": ["groups"]
      }
    }
  }
  EOF
  ```

Verify the configured identity providers on your cluster:

```bash
ocm get /api/clusters_mgmt/v1/clusters/<cluster-id>/identity_providers
```

---

## 3. Grant Administrator Permissions

By default, newly authenticated IDP users do not have cluster-level administrative privileges. Add the user to either the `cluster-admins` group (available for Customer Cloud Subscription / CCS clusters on GCP) or the `dedicated-admins` group:

```bash
# Grant full cluster-admin privileges
ocm post /api/clusters_mgmt/v1/clusters/<cluster-id>/groups/cluster-admins/users <<EOF
{
  "id": "<username>"
}
EOF

# Or grant standard dedicated-admin privileges
ocm post /api/clusters_mgmt/v1/clusters/<cluster-id>/groups/dedicated-admins/users <<EOF
{
  "id": "<username>"
}
EOF
```

> **Note:** The `id` value must match the exact username mapped from your IDP's `preferred_username` or `email` claim.

---

## 4. Log in to the OpenShift Cluster (`oc login`)

IDP and group membership changes typically take 1–3 minutes to reconcile. Once applied, log in to the cluster using the `oc` CLI:

```bash
# If using htpasswd:
oc login https://api.<cluster-domain>:6443 -u admin -p '<strong-password>'

# If using OIDC, open the token request endpoint in your browser:
# [https://oauth-openshift.apps](https://oauth-openshift.apps).<cluster-domain>/oauth/token/request
# Copy the displayed login command and run:
oc login --token=<sha256~...> --server=https://api.<cluster-domain>:6443

# Verify cluster access:
oc whoami
oc get nodes
```

---

## References

* [Configuring identity providers (OpenShift Dedicated 4)](https://docs.redhat.com/en/documentation/openshift_dedicated/4/html/authentication_and_authorization/sd-configuring-identity-providers)
* [Managing administration roles and users (OpenShift Dedicated 4)](https://docs.redhat.com/en/documentation/openshift_dedicated/4/html/authentication_and_authorization/osd-admin-roles)
* [OpenShift Cluster Manager CLI (`ocm`) GitHub Repository](https://github.com/openshift-online/ocm-cli)
