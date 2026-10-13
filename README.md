# wso2is-scim2-mcp-ai-gateway

**WSO2 Identity Server 7.3 SCIM 2.0 as an MCP server, behind the WSO2 AI Gateway.**


Specs and configuration for exposing the root-organization SCIM 2.0 APIs of WSO2 Identity Server 7.3 (Users, Groups, Roles v2, Bulk, Agents) as **21 task-shaped MCP tools**:

- the server is generated with [`arazzo-mcp-gen`](https://github.com/wso2/arazzo-mcp-generator)
- it's secured at the **WSO2 AI Gateway** with Identity Server's own tokens: `mcp-auth` policy validates the JWT, and `mcp-authz` policy requires a scope per tool

```
MCP client (e.g. Claude Desktop) ──MCP + Bearer JWT (SCIM MCP Client app)──▶ WSO2 AI Gateway :8443/scim/mcp
                                                                              │ mcp-auth · mcp-authz · token stripped
                                                                              ▼
                                                       generated SCIM MCP server (arazzo-mcp-gen, 21 tools)
                                                                              │ own client-credentials token (SCIM MCP Backend app)
                                                                              ▼
                                                       WSO2 Identity Server 7.3  /scim2/*   (also acts as the token issuer + JWKS)
```

## Index

| Path | Description |
|---|---|
| `arazzo/is-scim2.arazzo.yaml` | The 21 Arazzo 1.0.1 workflows. Each one becomes one MCP tool, and its `summary` is the tool description the MCP client sees. Every step maps the SCIM error body to `error` / `errorStatus` outputs, so the MCP client can see why a call failed. |
| `arazzo/*.yaml` | IS 7.3 OpenAPI specs, modified so `arazzo-runner` 0.9.6 can call IS endpoints with client credentials |
| `specs/` | The unmodified IS 7.3 OpenAPI specs |
| `gateway/mcp-proxy.yaml` | AI Gateway MCP proxy: `/scim`, `mcp-auth` (issuer + audience), and one `mcp-authz` rule per tool. |
| `gateway/is-keymanager.toml` | Gateway config snippet that registers Identity Server as the key manager (issuer, JWKS). |

## Tools

| Area | Tools |
|---|---|
| Users | `list_users`, `search_users`, `get_user`, `create_user`, `update_user_attribute`, `update_user_profile`, `delete_user` |
| Groups | `list_groups`, `create_group`, `add_user_to_group`, `remove_user_from_group`, `delete_group` |
| Roles (v2) | `list_roles`, `create_role`, `assign_role_to_user`, `remove_role_from_user`, `delete_role` |
| Agents | `list_agents`, `create_agent`, `delete_agent` |
| Bulk | `bulk_create_users` |

`update_user_profile` sets these attributes:
- region (`addresses.work`)
- the enterprise extension's `department`, `employeeNumber` and `costCenter`
- two custom attributes, `customerTier` and `contractEndDate` (`urn:scim:schemas:extension:custom:User`)

## Prerequisites

- **WSO2 Identity Server 7.3.0** at `https://localhost:9443` (JDK 21+).
- **Docker** with Compose v2.
- [`arazzo-mcp-gen`](https://github.com/wso2/arazzo-mcp-generator/releases) 0.1.0 and [WSO2 AI Gateway](https://github.com/wso2/api-platform/releases) 2026.09.24.

### Identity Server (Console)

1. Create two **M2M applications**: **SCIM MCP Client** (for MCP clients) and **SCIM MCP Backend** (for the MCP server).
2. For **both**, open **Authorization → Authorize resource** and add these APIs with all of their scopes:
   - SCIM2 Users, Groups, Roles (V1/V2), Bulk and Agents
   - Pick the Management APIs `/scim2/...`
3. For **both**, on the **Protocol** tab set the access token type to **JWT**. The gateway validates tokens against the JWKS.
4. Under **User Attributes & Stores → Attributes**, create `customerTier` and `contractEndDate`, mapped to the SCIM 2.0 custom schema.

## Use it

**1. Generate and run the MCP server.** The image is named after the Arazzo `info.title`: `is-scim2-mcp-mcp-server`.

```sh
arazzo-mcp-gen validate -f arazzo
arazzo-mcp-gen mcp-server generate -f arazzo -p 5000 -o generated

# Both credential pairs are the SCIM MCP Backend app.
docker run -d --name scim-mcp-server -p 127.0.0.1:5050:5000 \
  -e ARAZZO_DISABLE_TLS_VERIFY=1 \
  -e SCIM_OAUTH2_CLIENT_ID=<backend id> -e SCIM_OAUTH2_CLIENT_SECRET=<backend secret> \
  -e AI_OAUTH2_CLIENT_ID=<backend id>   -e AI_OAUTH2_CLIENT_SECRET=<backend secret> \
  is-scim2-mcp-mcp-server
```

**2. Start the AI Gateway with Identity Server as key manager.**

```sh
unzip wso2apip-ai-gateway-2026.09.24.zip && cd wso2apip-ai-gateway-2026.09.24
ADMIN_USERNAME=admin ADMIN_PASSWORD='<password>' ./scripts/setup.sh
cat <path-to-this-repo>/gateway/is-keymanager.toml >> configs/config.toml
docker compose up -d
docker network connect "$(sed -n 's/^COMPOSE_PROJECT_NAME=//p' .env)_gateway-network" scim-mcp-server
```

Set `[analytics] enabled = false` for local use.

**3. Register the proxy.** First replace `${CLIENT_APP_ID}` in `gateway/mcp-proxy.yaml` with the SCIM MCP Client app's client id.

```sh
curl -u admin:'<password>' -X POST http://localhost:9090/api/management/v1/mcp-proxies \
  -H 'Content-Type: application/yaml' --data-binary @gateway/mcp-proxy.yaml        # 201, state: deployed
```

**4. Call it.** Get a token from the SCIM MCP Client app and connect any MCP client (streamable HTTP) to `https://localhost:8443/scim/mcp`.

```sh
curl -sk -u '<client id>:<client secret>' https://localhost:9443/oauth2/token \
  -d grant_type=client_credentials \
  --data-urlencode 'scope=internal_user_mgt_list internal_user_mgt_view'
```

What to expect:

| Request | Result |
|---|---|
| No token | **401**, with a `WWW-Authenticate` header pointing at `/scim/.well-known/oauth-protected-resource` |
| Token from another IS app (wrong `aud`) | **401** |
| Read-only token, `search_users` | works |
| Read-only token, `delete_user` | **403** `insufficient_scope`, requires `internal_user_mgt_delete` |

Since the local gateway's listener certificate is self-signed. point Node-based clients at it with `NODE_EXTRA_CA_CERTS=<gateway>/resources/listener-certs/default-listener.crt`.

## Versions

| Component | Version |
|---|---|
| WSO2 Identity Server | 7.3.0 |
| WSO2 AI Gateway | 2026.09.24 (mcp-auth v1.4.0, mcp-authz v1.3.0) |
| arazzo-mcp-gen / arazzo-runner | 0.1.0 / 0.9.6 |

## License

Apache License 2.0. See `LICENSE`. The OpenAPI specs in `specs/` and `arazzo/` are from the WSO2 Identity Server documentation.