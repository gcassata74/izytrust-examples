# IzyTrust examples

Helm values and a local Kubernetes demo for **IzyTrust**, a self-hosted MCP gateway that
controls what AI agents can do on your systems.

IzyTrust exposes one MCP endpoint in front of your MCP servers. An agent (Claude Code,
or any MCP client) connects to IzyTrust only. When a prompt first needs a tool, the user
signs in with Keycloak. On every tool call IzyTrust:

1. checks the user's token and the scope the tool requires (a tool that is not in the
   registry is denied);
2. exchanges the user's token for one that is valid only for that upstream
   (OAuth 2.0 Token Exchange, RFC 8693);
3. forwards the call, so the upstream never sees the user's IzyTrust token;
4. writes user, upstream, tool, arguments, scope and outcome to an audit log.

The agent holds no credentials of its own, and every action is traceable to a person.

Product page: https://izylifesolutions.com/products/izytrust/

## What is in this repository

| Path | Contents |
|---|---|
| `k3s-demo/` | Keycloak with a demo realm, two example upstream MCP servers, a demo app and a NetworkPolicy |
| `k3s-demo/values.yaml` | Helm values for the demo, including the upstream registry |

The Helm chart (`oci://ghcr.io/gcassata74/charts/izytrust`) and the images
`ghcr.io/gcassata74/izytrust` and `ghcr.io/gcassata74/izytrust-kube-mcp` are signed and
stored on GitHub Container Registry. The IzyTrust source code is not in this repository.

## Get access to the images

The chart and the images are private. To try IzyTrust:

1. Ask for access in one of two ways:
   - [open a "Request demo access" issue](https://github.com/gcassata74/izytrust-examples/issues/new?template=demo-access.yml)
     (public; access goes to the account that opens it), or
   - use the [contact form](https://izylifesolutions.com/?service=izytrust-demo#contacts)
     and write your GitHub username in the message.

   You will get read access to the three packages.
2. Create a GitHub personal access token (classic) with the `read:packages` scope only:
   https://github.com/settings/tokens/new?scopes=read:packages
3. Log in to the registry with Helm:

```bash
export GHCR_USER=<your-github-username>
export GHCR_TOKEN=<your-token>
echo "$GHCR_TOKEN" | helm registry login ghcr.io -u "$GHCR_USER" --password-stdin
```

To read the chart's templates and default values:

```bash
helm pull oci://ghcr.io/gcassata74/charts/izytrust --version 0.1.1 --untar
```

## Run the demo on k3d (k3s in Docker)

You need Docker, [k3d](https://k3d.io), kubectl, Helm and access to the images (see above).

```bash
# 1. Cluster with ports 8001 (IzyTrust) and 8080 (Keycloak) on localhost
k3d cluster create izytrust-demo \
  -p "8001:30001@server:0" -p "8080:30080@server:0"

# 2. Namespaces and secrets (ghcr: pull secret for the private images)
kubectl apply -f k3s-demo/namespaces.yaml
kubectl -n izytrust create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username="$GHCR_USER" --docker-password="$GHCR_TOKEN"
kubectl -n izytrust create secret generic izytrust-oidc \
  --from-literal=client-secret="$(openssl rand -hex 24)"
kubectl -n izytrust create secret generic keycloak-admin \
  --from-literal=password="$(openssl rand -hex 12)"

# 3. Keycloak, example upstreams, demo app
kubectl apply -k k3s-demo
kubectl -n izytrust rollout status deploy/keycloak --timeout=5m

# 4. IzyTrust
helm install izytrust oci://ghcr.io/gcassata74/charts/izytrust --version 0.1.1 \
  -n izytrust -f k3s-demo/values.yaml
kubectl -n izytrust rollout status deploy/izytrust
```

On an existing k3s node, skip step 1 and use NodePorts 30001 and 30080 directly, or
change `publicUrl`, `KC_HOSTNAME` and `IZYTRUST_PUBLIC_URL` to match your host.

## Try it with Claude Code

```bash
claude mcp add --transport http izytrust http://localhost:8001/mcp
```

Then ask: *"scale demo-app to 3 replicas"*. IzyTrust answers with a login link. The realm
has two users:

| User | Password | Roles | Can call |
|---|---|---|---|
| `alice` | `alice` | reader, writer | every tool, including `kube_scale_deployment` |
| `bob` | `bob` | reader | `kube_list_pods`, `kube_get_deployment`, `notes_list_notes` |

Sign in as `bob` and the scale request is denied by IzyTrust before it reaches Kubernetes.
Sign in as `alice` and it goes through. In both cases the decision is in the audit log:

```bash
kubectl -n izytrust logs deploy/izytrust | grep audit
```

The example Kube MCP can only touch the `demo` namespace: its ServiceAccount has a Role
there and nowhere else, and it scales `demo-app` between 0 and 5 replicas only.

## Connect your own MCP servers

Add an entry to `upstreams` in the values file and a client with the same name as the
`audience` in Keycloak. No code changes are needed.

```yaml
upstreams:
  - name: tickets
    url: http://tickets-mcp.tools.svc:8000/mcp
    audience: tickets-mcp
    tools:
      search_tickets: read
      close_ticket: write
```

Agents see the tools as `tickets_search_tickets` and `tickets_close_ticket`.

## Status and licence

IzyTrust is in early access (0.x). The files in this repository are released under the
Apache License 2.0. The IzyTrust chart and container images are proprietary software of
Izylife Solutions s.r.l. and are licensed under a written agreement. For access, or for
an assessment of your use case, contact us at https://izylifesolutions.com/?service=izytrust-demo#contacts.

The demo uses Keycloak in dev mode with fixed passwords. Do not expose it to a network.
