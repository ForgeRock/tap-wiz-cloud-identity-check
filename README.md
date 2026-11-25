# Wiz Cloud Identity Check Node

A custom authentication node for PingAM that checks whether the authenticating user exists as an identity in the Wiz platform, routing the journey based on the result.

## Overview

During authentication, the node resolves the user's identity from shared state, authenticates to the Wiz API with client credentials, and searches Wiz `directoryUsers` for a match. It writes the match count to shared state and routes to `hasCloudIdentities` or `none`.

Use this node to:

- Branch journeys based on whether the user has a Wiz cloud identity (e.g. enforce extra controls for users with Wiz access)
- Gate Wiz-specific enrichment or privileged-access checks to users who actually exist in Wiz
- Record the Wiz identity-match count in shared state for claims or downstream decisions

### Resolution logic

The node searches `directoryUsers(first: 1, search: <userId>)`:

| Result | Outcome |
|---|---|
| `totalCount > 0` | `hasCloudIdentities` |
| `totalCount = 0` | `none` |
| Any error | `error` |

The match count is written to shared state before branching — a count of `0` is written on the `none` outcome, so downstream nodes and claims can always distinguish "checked" from "not checked".

## Compatibility

| Product | Versions |
|---|---|
| PingAM | 8.0.2 and later |
| PingOne Advanced Identity Cloud | All current tenants |

Built against the PingAM 8.0.2 node SDK (Java 17 bytecode).

## Outcomes

| Outcome | Description |
|---|---|
| `hasCloudIdentities` | The user matched at least one Wiz directory user |
| `none` | The user has no Wiz cloud identity |
| `error` | The user id was missing, the Wiz configuration was incomplete, or the API call failed |

## Configuration

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| Auth URL | String | No | `https://auth.app.wiz.io/oauth/token` | Wiz OAuth 2.0 token endpoint. |
| API URL | String | Yes | `https://api.us17.app.wiz.io/graphql` | Tenant-specific Wiz GraphQL endpoint (varies by tenant region — copy from your Wiz tenant settings). |
| Client ID | String | Yes | *(empty)* | Wiz service-account client ID. |
| Client Secret | String | Yes | *(empty)* | Wiz service-account client secret. |

A fresh OAuth token (`grant_type=client_credentials`, `audience=wiz-api`) is obtained per node invocation; there is no token caching.

## Shared state

### Reads

| Key | Description |
|---|---|
| `mail` | User id used for the Wiz search — preferred |
| `username` | Fallback user id when `mail` is absent |

### Writes

| Key | Type | When | Description |
|---|---|---|---|
| `wizCloudIdentityCount` | Number | Success (including `none`) | Number of matching Wiz directory users |

Nothing is written on the `error` outcome.

## Error handling

| Condition | Behavior |
|---|---|
| Neither `mail` nor `username` in shared state | `error` outcome, no outbound call |
| Blank auth/API URL, client ID, or secret | `error` outcome (validated at token-request time) |
| Token or GraphQL HTTP failure (response body appended to the log for diagnosis) | `error` outcome |
| GraphQL response contains `errors` | `error` outcome |
| Any other exception | `error` outcome |

On every error path the node logs the cause and takes the `error` outcome without writing shared state or a failure-node message.

Fail open (continue to `Success`) or fail closed (route to `Failure`) is journey wiring, not node behavior.

## Journey example

```
Start → [Username/Password collection] → [Data Store Decision] → [Wiz Cloud Identity Check]
                                                                    ├─ hasCloudIdentities → [Wiz-enriched path / extra controls]
                                                                    ├─ none               → [standard path]
                                                                    └─ error              → Success or Failure (your policy)
```

## Wiz service account

This node requires a Wiz service account with API access and permission to read directory users. Create one in your Wiz tenant under **Settings → Service Accounts**, then copy the **API URL** (tenant-specific, e.g. `https://api.us17.app.wiz.io/graphql`) and the client credentials into the node configuration.

## Installation

1. Obtain the node jar (`wiz-gti-marketplace-nodes-<version>.jar` — this jar also contains the companion nodes; see Related nodes).
2. Copy it into `webapps/openam/WEB-INF/lib/` (PingAM) or deploy per your PingAIC packaging process.
3. Restart the AM web application container.
4. The node appears in the journey designer as **Wiz cloud identity check for the authenticated user**.

## Building from source

Requirements: JDK 17+, Maven 3.8+, and Ping Identity repository credentials (Backstage subscription) configured in `~/.m2/settings.xml` for `https://maven.forgerock.org/artifactory/private-releases`.

```bash
mvn clean package
```

The jar is produced in `target/`.

## Related nodes

- [Wiz Risk Enrichment](https://github.com/ForgeRock/tap-wiz-risk-enrichment)
- [Wiz Privileged Gate](https://github.com/ForgeRock/tap-wiz-privileged-gate)
- [GTI IP Reputation](https://github.com/ForgeRock/tap-gti-ip-reputation)

## Support

This integration is provided as a sample journey node. Review with your Ping Identity team before production use. For general PingAM/PingAIC node development, see the Ping Identity documentation on custom authentication nodes.
