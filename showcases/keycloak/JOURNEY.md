# Keycloak — YarraMate discovery journey

## Provenance

| | |
| --- | --- |
| Source | <https://github.com/keycloak/keycloak> |
| Tag | `26.7.1` |
| Commit | `73f08b397f193712b26d317210dce99898129709` |
| Clone | `git clone --depth 1 --branch 26.7.1` (shallow, detached HEAD); HEAD SHA verified equal to the commit above |
| Analyzed | 2026-08-07 |
| Toolchain | yarramate 0.15.0 (`yarramate`, `yarramate-likec4`) |
| Profile | `yarramate/core@0.1` |

## Journey and question

**Journey:** Discover an existing project (yarramate-architecture skill).

**Question answered:** How is Keycloak organised as an enterprise IAM
server at 26.7.1 — its significant actors, the server core and its
realm/client/user/role model, the protocol endpoints (OIDC, OAuth 2.0,
SAML 2.0), identity brokering to external providers, user federation
(LDAP/Active Directory), the SPI extension model, authorization
services, and the relational store — as observable from the repository
at the commit above? Current state only; no target intent was inferred.

## Observations vs interpretive proposals

### Direct observations (locator-backed)

- **Server process:** the Quarkus entry point
  `quarkus/runtime/.../KeycloakMain.java` (`@QuarkusMain`) boots the CLI
  and the server through `QuarkusKeycloakApplication`; the JAX-RS
  application is `services/.../resources/KeycloakApplication.java`. Build
  and distribution live under `quarkus/server` and `quarkus/dist`.
- **Protocol routing:** `services/.../resources/RealmsResource.java`
  maps `/realms/{realm}/protocol/{protocol}` and `/realms/{realm}/login-actions`,
  `clients-registrations`, and `clients-managements`; `AdminRoot.java`
  is rooted at `/admin` with `/admin/realms` and `/admin/{realm}/console`.
- **OIDC / OAuth 2.0:** `services/.../protocol/oidc/OIDCLoginProtocolService.java`
  exposes the `auth`, `token`, `registrations`, `logout`, introspection,
  and revocation endpoints; grant types
  (`services/.../protocol/oidc/grants/`) cover authorization code,
  client credentials, resource-owner password, refresh, device flow,
  token exchange, and CIBA; `endpoints/TokenEndpoint.java` +
  `TokenManager.java` mint tokens; mappers under `protocol/oidc/mappers/`.
- **SAML 2.0:** `services/.../protocol/saml/SamlService.java` and
  `SamlProtocol.java` handle SSO, artifact resolution, single logout,
  and the IdP metadata descriptor.
- **Authentication flows:** `services/.../resources/LoginActionsService.java`
  drives `authenticate`/`resetCredentials`/`requiredAction`; execution
  in `services/.../authentication/AuthenticationProcessor.java` and
  `DefaultAuthenticationFlow.java`; the SPI is
  `server-spi-private/.../authentication/AuthenticatorSpi.java`.
- **Identity brokering:** `services/.../resources/IdentityBrokerService.java`
  serves `/realms/{realm}/broker/{provider_alias}/login` and callbacks
  (`authenticated(BrokeredIdentityContext)`, `afterFirstBrokerLogin`,
  `afterPostBrokerLoginFlow`); OIDC/SAML provider impls under
  `services/.../broker/{oidc,saml}/`; social providers (google, github,
  gitlab, microsoft, …) under `services/.../social/`; the SPI is
  `IdentityProviderSpi.java`.
- **User federation:** the `UserStorageProvider` SPI
  (`model/storage/.../storage/UserStorageProvider.java`) with the LDAP
  bridge `federation/ldap/.../LDAPStorageProvider.java` +
  `LDAPStorageProviderFactory.java` (edit modes, `ImportSynchronization`,
  `IMPORT_ENABLED`) and Kerberos (`federation/kerberos`).
- **Authorization services:** `services/.../authorization/AuthorizationService.java`
  + `protection/` Protection API; resources/scopes/policies persisted as
  `model/jpa/.../authorization/jpa/entities/{Resource,Scope,Policy}Entity.java`.
- **Policy enforcement (client side):** `core/.../representations/adapters/config/PolicyEnforcerConfig.java`,
  `core/.../AuthorizationContext.java`, and the `authz/client`
  (`AuthzClient.java`, `ClientAuthorizationContext.java`) live in this
  repository. The enforcer *runtime* (the servlet filter) is **not** in
  this repository — it ships as a separate client-side adapter artifact.
- **SPI provider model:** `server-spi/.../provider/{Provider,ProviderFactory,Spi}.java`,
  discovered via `META-INF/services/org.keycloak.provider.Spi` files
  (e.g. `server-spi`, `model/jpa`, `federation/ldap`, `model/storage-private`).
- **JPA store:** `model/jpa/.../models/jpa/JpaRealmProvider.java`
  (`implements RealmProvider, ClientProvider, ClientScopeProvider,
  GroupProvider, RoleProvider`), `JpaUserProvider.java`,
  `JpaClientProvider.java`; connection via
  `connections/jpa/DefaultJpaConnectionProvider.java` and
  `quarkus/runtime/.../storage/database/jpa/QuarkusJpaConnectionProviderFactory.java`.
- **Relational database:** `quarkus/runtime/.../configuration/mappers/DatabasePropertyMappers.java`
  enumerates vendors — postgres, mariadb, mysql, mssql, oracle, db2
  (H2 for dev). JPA entities include `RealmEntity`, `ClientEntity`,
  `UserEntity`, `RoleEntity`, `GroupEntity`, `ClientScopeEntity`.
- **Cache:** `model/infinispan/.../cache/infinispan/InfinispanCacheRealmProviderFactory.java`
  and `InfinispanUserCacheProviderFactory.java` front the JPA store.
- **Domain model:** `RealmModel`, `ClientModel`, `ClientScopeModel`,
  `UserModel`, `RoleModel`, `GroupModel`, `UserSessionModel`,
  `UserCredentialModel` in `server-spi/.../models/`.

### Interpretive proposals (semantic meaning added for review)

- The actors **End user** and **Administrator**, and the external
  systems **Client application / relying party**, **External identity
  provider**, **LDAP / Active Directory server**, and **Relational
  database** are interpretations; the code implies them (protocol
  endpoints, social/broker packages, federation providers, DB vendor
  config) but does not name the actors.
- Grouping the OIDC token endpoint and OAuth 2.0 grant handling into one
  **OpenID Connect / OAuth 2.0 endpoint** is accurate: in Keycloak the
  OAuth 2.0 grant types are served by the OIDC `openid-connect`
  protocol service; there is no separate OAuth 2.0 endpoint.
- The **Policy enforcer** concept is evidenced only by its config,
  context, and client library in this repository; whether a given
  application embeds the enforcer is **not observable** here (the single
  `unknown` reconciliation finding).
- The **Infinispan cache** is modelled as fronting the JPA store
  (`cache-fronts-store`) rather than serving each component, to avoid
  over-claiming the cache's per-component reach at discovery depth.
- No ownership and no constraints were declared: the repository contains
  no ownership or governance sources to support them. Current state only;
  no target intent was inferred.

## Model

Canonical inputs (this directory):

- `.yarramate/workspace.yaml` — workspace `keycloak`
- `.yarramate/architecture/keycloak.yaml` — 30 concepts, 53
  relationships, 0 states (1 document)
- `.yarramate/evidence/repository.yaml` — 55 observations
- `.yarramate/projections/*.yaml` — 5 projections
- `.yarramate/integrations/likec4/subject-mapping.yaml` — 83 mappings
  (populated by `map --sync`)
- `.yarramate/likec4-project.yaml` — 5 views (2 dynamic, 3 static)

## Evidence results and reconciliation

`yarramate reconcile` summary (provider `repository-inspection`,
evidence document `keycloak-repository@1.0`):

| result | count |
| --- | --- |
| confirmed | 54 |
| contradicted | 0 |
| unknown | 1 |
| not-observed | 0 |

The single `unknown` is deliberate: claim `keycloak#enforcer-protects-client`
(the policy enforcer protects client applications) — the
`PolicyEnforcerConfig`, `AuthorizationContext`, and `authz/client` exist
in this repository, but the enforcer runtime and which client
applications actually embed it are not observable from the server
repository. Evidence was never promoted into declared intent.

## Projections and views

| Projection | View type | Content |
| --- | --- | --- |
| `system-overview` | static | All 30 concepts and all 53 relationships |
| `protocol-surfaces` | static | OIDC/OAuth 2.0 + SAML endpoints, admin & account surfaces, token service |
| `authorization-services` | static | Authorization services, policy enforcer, resources/scopes/policies, store |
| `direct-login-flow` | dynamic (7 steps) | client → OIDC → auth flow → store/federation → code → token → client |
| `brokered-login-flow` | dynamic (8 steps) | client → OIDC → broker → external IdP → broker → store → token → client |

Generated LikeC4 output: `.yarramate-out/likec4/` (`model.likec4`,
`specification.likec4`, `likec4.config.json`, `yarramate.generated.json`).
The generated output is current with the authored inputs.

## Rendering coverage audit

- **Concepts in no projection:** none — `system-overview` covers all 30
  concepts and all 53 relationships.
- **Ordered chains without a dynamic view:**
  - the user-federation credential path
    (`authentication-flow-engine → user-federation → ldap-directory`) is
    included as a step inside `direct-login-flow` rather than its own
    view; the standalone LDAP sync/import path
    (`user-federation → jpa-store`, `user-federation → ldap-directory`)
    is rendered statically — intentional: it converges with the direct
    login lookup immediately after the federation step.
  - the authorization decision path
    (`policy-enforcer → authorization-service → store`) — intentional: a
    static projection (`authorization-services`) covers it; no
    multi-step login-style chain warrants its own dynamic view.
- **Projections absent from the LikeC4 project:** none — all 5
  projections are listed as views.

### Intentional model omissions

Observed in the repository but deliberately left out of the smallest
useful model: the Docker registry protocol, OID4VC/verifiable
credentials, SCIM, the events/audit store, the export/import and
migration tooling, the Kubernetes operator, themes/i18n, client
registration and management services, the Infinispan clustering and
multi-site detail, FIPS/crypto providers, the JS adapter and admin/account
SPAs as separate nodes, the per-provider social and broker packages
(~12 collapsed into one **External identity provider** concept), and
deployment packaging (Quarkus distribution, container image).

## Validation

All commands used the pinned yarramate 0.15.0 toolchain, run from the
showcase directory.

| Command | Outcome |
| --- | --- |
| `yarramate check .yarramate/workspace.yaml --json` | ok, exit 0 (1 doc / 30 concepts / 53 relationships / 0 diagnostics) |
| `yarramate reconcile .yarramate/workspace.yaml` | exit 0, 54 confirmed / 0 contradicted / 1 unknown / 0 not-observed |
| `yarramate export graph .yarramate/workspace.yaml` | exit 0 |
| `yarramate ask .yarramate/workspace.yaml <each of 5 projections>` | exit 0 (×5) |
| `yarramate export markdown <system-overview, direct-login-flow>` | exit 0 |
| `yarramate-likec4 check .yarramate/likec4-project.yaml --json .yarramate/workspace.yaml` (pre-sync) | ok:false — expected: empty mapping, 83 missing entries |
| `yarramate-likec4 map --sync .yarramate/integrations/likec4/subject-mapping.yaml .yarramate/workspace.yaml` | exit 0, added 83 mappings (authored diff reviewed) |
| `yarramate-likec4 check` (post-sync) | ok:true, exit 0 |
| `yarramate-likec4 export-project .yarramate/likec4-project.yaml .yarramate-out/likec4 .yarramate/workspace.yaml` | exit 0 (model.likec4, specification.likec4, likec4.config.json, yarramate.generated.json) |
| Determinism — re-run `export-project`; SHA-256 of all 4 files | byte-identical to the first export |

⚠️ A green check is deterministic correctness, not completeness or
architecture approval. The gaps above are reported, not hidden.

## Unresolved architectural decisions

None pending for the current-state model. Open modelling questions a
maintainer might take up later:

- Whether the policy enforcer (config/client present, runtime absent
  from this repo) should be modelled as an external system rather than
  an internal component once the enforcer adapter artifact is in scope.
- Whether the Infinispan cache and clustering topology deserve
  first-class node modelling for a multi-site deployment view.
- Whether per-protocol OAuth 2.0 grant types (device, CIBA, token
  exchange) deserve individual components or remain folded into the OIDC
  endpoint.

## Status in Git

This showcase is a proposed, uncommitted artifact: the model was
authored directly under `showcases/keycloak/`. No git commands were run
in the gallery repository; the upstream Keycloak source was cloned to a
temporary directory outside the gallery and left unchanged.
