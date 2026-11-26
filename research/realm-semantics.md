# Realm semantics on FlashBlade and in this collection

- **Addresses:** issue #2 (research for #1, "objects inside realms are poorly documented")
- **Date:** 2026-11-26
- **Collection commit:** `ee83522` (branch `master`; `galaxy.yml` version 1.29.0, with unreleased fragments for 1.30.0)
- **SDK:** py-pure-client **1.99.0**, installed at `~/.local/lib/python3.13/site-packages/pypureclient`. I read `flashblade/FB_2_0` … `FB_2_30` (`client.py`, `models/__init__.py`) and `flashblade/_common/models/*`.
- **Public docs:** Portworx CSI FlashBlade prerequisites, the Everpure SMT announcement blog and the Everpure March 2026 launch blog (URLs below). I found no public Purity//FB admin guide or REST reference page that defines realms. A support.everpuredata.com PDF came back but could not be text-extracted in this environment.
- **Completeness check:** `grep -ril realm plugins/ | sort` returns 56 files: 55 modules plus `plugins/module_utils/purefb.py`. `grep -L -i realm plugins/modules/*.py | xargs grep -l '"::"\|{0}::{1}'` returns nothing, so no module handles `::` names without also mentioning realms.

**Citation conventions**
- **SDK** means a py-pure-client model or client method. These are generated from the FlashBlade OpenAPI spec, so they are the primary source for API shape.
- **Coll-doc** means a statement in a module's `DOCUMENTATION` or in a changelog fragment. These are claims the collection authors make about array behaviour. They look empirical (many say "Purity//FB 4.8 refuses …"), but no external source backs them, so this document treats them as **Unverified** unless the SDK or a public doc agrees.
- **Inferred** means my own reasoning from code or schema, not stated by any source.

---

## 1. FlashBlade realm semantics

### 1.1 What a realm is

| Claim | Source |
|---|---|
| A realm is the tenant boundary for Secure Multi-Tenancy: "a logical construct to group storage objects together and deliver multi-tenancy". | Everpure blog, *Announcing Secure Multi-tenancy for Purity* — https://blog.everpuredata.com/purely-technical/announcing-secure-application-workspaces-with-pure-storage/ |
| "FlashBlade with secure multi-tenancy (using Realms) provides isolated credentials and resource boundaries". Requires Purity 4.6.1+. | Portworx CSI docs — https://docs.portworx.com/portworx-csi/install/prepare/flash-blade |
| The REST resource is `/realms` (GET/POST/PATCH/DELETE), with `/realms/space`, `/realms/space/storage-classes` (2.19+) and `/realms/defaults` (2.20+). | SDK `FB_2_19/client.py`, `FB_2_20/client.py`: `get_realms`, `post_realms`, `patch_realms`, `delete_realms`, `get_realms_space*`, `get_realms_defaults`, `patch_realms_defaults` |
| The `Realm` model has these fields: `id`, `name`, `created`, `default_inbound_tls_policy`, `destroyed`, `space`, `time_remaining`. `RealmPatch` can change `name`, `default_inbound_tls_policy` and `destroyed`. | SDK `_common/models/realm_v_2_19.py`, `realm_patch_v_2_19.py` (unchanged through `_v_2_26`) |
| A destroyed realm "cannot be modified while it is in the destroyed state". It can be recovered with `destroyed=false` until `time_remaining` runs out. | SDK `Realm.destroyed` description |
| `post_realms` requires `without_default_access_list` ("ensures a realm is created without any resource accesses"). | SDK `FB_2_30/client.py` `post_realms` |
| The array requires `without_default_access_list=true`. | Coll-doc `purefb_realm` option `without_default_access_list`. **Unverified.** |
| Realm defaults (`RealmDefaults`: `realm`, `object_store[].server`, `context`) hold the per-realm default object-store server (2.20+). | SDK `realm_defaults_v_2_20.py` |
| `default_inbound_tls_policy` applies to realm IPs that have no more specific TLS policy. Without one, those IPs "will not be accessible using TLS". | SDK `Realm.default_inbound_tls_policy` description |
| Realms can be QoS policy members (`member_types=realms`). | SDK `get_qos_policies_members` / `post_qos_policies_members` `member_types` description |
| The REST API version that introduced realms is 2.19, which maps to Purity//FB 4.6.1. | SDK: realm methods first appear in `FB_2_19`. Portworx docs give 4.6.1. The 2.19 ↔ 4.6.1 pairing itself comes only from Coll-doc (`purefb_realm` `main`). |
| Realms are not nested; a realm name cannot contain `::` or `,`. | Coll-doc `purefb_realm` `name`, `purefb_realm_connection` `realm`. **Unverified** (no SDK constraint found). |
| Realm names are matched case-insensitively. | Coll-doc `purefb_realm` description. **Unverified.** |

### 1.2 "Realm" versus other meanings

- **Fleet `context` is not a realm.** The SDK's `context_names` / `context` fields mean "the name of an array in the same fleet or the name of the fleet itself" (SDK `FB_2_30/client.py`, any `context_names` description; `RealmDefaults.context`). The collection's `context` option means the same thing ("Name of fleet member on which to perform the operation", e.g. `purefb_bucket` option `context`). It plays no part in realm scoping.
- **Kerberos realm.** `Keytab.realm` is "The Kerberos realm that issued the keytab" (SDK `keytab_v_2_*`). `purefb_info` reports it as `realm` in the `kerberos` subset (`generate_kerb_dict`) without saying that it is not a FlashBlade realm.

### 1.3 Which object types are realm-scoped

The SDK marks membership with a `realms: List[FixedReference]` field: "Reference to the realms this resource belongs to. The value is set to empty array when the resource lives outside of a realm." That field is present in these models (the earliest SDK model version that has it is in brackets):

> active_directory [2.19], audit_file_systems_policy [2.19], audit_object_store_policy [2.20], bucket_access_policy [2.19], bucket_lifecycle_policy [2.29], certificate [2.19], certificate_group [2.19], cross_origin_resource_sharing_policy [2.19], data_eviction_policy [2.21], dns [2.19], file_system [2.19], legal_hold [2.19], local_directory_service / local_group / local_user / local_member [2.24], management_access_policy [2.19], management_authentication_policy [2.22], network_access_policy [2.19], network_interface [2.19], nfs_export_policy [2.19], object_store_access_policy [2.19], object_store_account / _account_export / _remote_credentials / _virtual_host [2.20], object_store_trust_policy [2.19], password_policy [2.19], policy (snapshot) [2.19], qos_policy [2.19], s3_export_policy [2.20], server [2.19], smb_client_policy / smb_share_policy [2.19], ssh_certificate_authority_policy [2.19], storage_class_tiering_policy [2.19], telemetry_metrics_policy [2.29], tls_policy [2.19], user_group_quota_policy [2.25], worm_data_policy [2.19]

(Command used: `grep -l '^    realms: ' _common/models/*.py`.)

**Caveat (Inferred).** Many policy models inherit `realms` from a shared base (`policy_base_*`). The field alone does not prove the array lets you *create* that type inside a realm. Examples:
- `network_access_policy` and `management_access_policy` have `realms`, yet Coll-doc says network access policies "are not realm objects" (`purefb_policy` description) and management access policies are array-level (`purefb_mgmt_policy`).
- `qos_policy.realms` probably lists the realms the policy is *attached to*, not a home realm.

**Objects without a `realms` field, whose realm comes from the name:**
- `Bucket`, `ObjectStoreUser` and `FileSystemSnapshot` have no `realms` field (SDK `bucket_v_2_29`, `object_store_user_v_2_22`, `file_system_snapshot_v_2_22`).
- Their realm comes from the owner: the bucket's account, the user's account, or the snapshot's file system.
- This is visible only through the qualified name. Coll-doc `purefb_bucket` and `purefb_info` agree.

**Array-global objects (no `realms` field in the SDK)** include subnets, link aggregation groups, targets, array connections, syslog servers, keytabs, nodes, API clients, resource accesses and admins.
- `Admin` has `context` (fleet), not `realms` (SDK `admin_v_2_26`).
- Coll-doc puts these in the same array-global category, each in its own module: users (`purefb_user`), API clients (`purefb_apiclient`), management roles and role mappings (`purefb_mgmt_role`, `purefb_dsrole`), management authentication policies (`purefb_mgmt_auth_policy`), OIDC and SAML2 IdPs (`purefb_oidc`, `purefb_saml`), NTP and syslog settings (`purefb_ntp`, `purefb_syslog_settings`).

### 1.4 Naming and qualification (`realm::name`)

| Form | Example | Source |
|---|---|---|
| `realm::name` for a realm-scoped object | `realm1::fs1` | SDK `post_object_store_access_keys` `names` description ("`realmName::accountName/userName`", "`realm1::PSFB…`"). SDK `LocalDirectoryService.domain` ("for object \"myrealm::mydomain\" the domain name will be \"mydomain\""). |
| Access key ID of a realm user: `realm::<key id>` | `realm1::PSFBSAZR…` | SDK `post_object_store_access_keys` |
| Account child objects: `realm::account/name` (users, access policies, roles) | `realm1::acct/user` | SDK (same). Coll-doc `purefb_info`, `purefb_object_role`, `purefb_userpolicy`. |
| Server child objects: `realm::server::name` (AD account, keytab prefix) | `r1::srv::ad1` | Coll-doc `purefb_ad`, `purefb_keytabs`. **Unverified.** |
| Server NFS directory service: `realm::server_nfs` | | Coll-doc `purefb_ds`, `purefb_info`. **Unverified.** |
| Snapshot: `realm::fs.suffix` | | Coll-doc `purefb_snap`, `purefb_info`. **Unverified.** |
| Account export: `realm::server::realm::account` | | Coll-doc `purefb_info`. **Unverified.** |
| Bucket access-policy rule resources omit the realm (`bar/*` for `tenant1::bar`) | | Coll-doc `purefb_bucket_access` (fragment 693). **Unverified.** |
| A bare name always means the array-level object. The array does not resolve it into the caller's realm, even for a realm-scoped admin. | | Coll-doc (repeated in about 25 modules, e.g. `purefb_fs` `name`). **Unverified**, but it is consistent with the SDK's "empty array when the resource lives outside of a realm". |
| `realm::x` and `x` are distinct objects that can coexist. Junction names are the exception: they are unique across the array and all realms. | | Coll-doc `purefb_fs`, `purefb_fs_junction` `name`. **Unverified.** |

### 1.5 What can and cannot move

- **No REST operation moves an object between realms (Inferred).** `RealmPatch` has no member list, and the object PATCH models (e.g. `FileSystemPatch`) expose `realms` only read-only. Coll-doc `purefb_fs` `realm` says "the filesystem must be destroyed and recreated".
- **Renaming within a realm:**
  - Coll-doc says the realm is kept (`purefb_policy`, `purefb_quota_policy`, `purefb_s3_export_policy`, `purefb_eviction_policy`, `purefb_virtualhost` `rename`).
  - A rename across realms is refused (Coll-doc).
- **Renaming a realm:**
  - Realm-scoped objects follow the new name. The SDK models carry `id`-based references (Inferred).
  - Coll-doc `purefb_mgmt_policy`: "A rule follows its realm through a rename."
  - Coll-doc `purefb_realm` `rename`: the reference to the default inbound TLS policy keeps the old `oldrealm::policy` name until it is cleared and set again. **Unverified array quirk.**
- **Destroying a realm:**
  - The realm must be empty, with no resource accesses and no management-access-policy rules scoped to it.
  - Coll-doc `purefb_realm` `state` says an attached QoS policy blocks the destroy. **Unverified.**

### 1.6 Sharing array-level objects into a realm (resource access)

- The `ResourceAccess` model has `resource` ("Resource to which the scope has access to") and `scope` ("Scope which has access to the resource") (SDK `resource_access_v_2_22`).
- Endpoints: `get_resource_accesses`, `post_resource_accesses_batch`, `delete_resource_accesses` (SDK `FB_2_30/client.py`).
- **Shareable resource types:**
  - Purity//FB 4.6: `subnets`, `dns`.
  - Purity//FB 4.8 adds `targets`, `sso/oidc/idps`, `sso/saml2/idps`.
  - Policies are rejected with `Invalid resource type`.
  - Source: Coll-doc `purefb_resource_access` `resource_type`. **Unverified.** The SDK does not enumerate the types.

### 1.7 Permissions

- **Realm-scoped administrator.** Access is limited by management access policies whose rule `scope` is a realm. `ManagementAccessPolicyRule.scope` is "Reference to the resource (e.g. arrays, realms) which specifies the scope that the role applies" (SDK `management_access_policy_rule_*`). Rules were added in REST 2.26 (`purefb_mgmt_policy` `MIN_RULE_API_VERSION`).
- **Array admin only.** Creating, renaming, QoS and TLS defaults on realms require an array admin. Only `array_admin` can create, manage and destroy realms (Everpure SMT blog, above). Coll-doc `purefb_realm` adds that a realm admin can set its realm's default object server. **Unverified.**
- **Refused reads.** A realm admin is refused `GET /hardware` and array-wide reads such as array connections, NTP and syslog settings.
  - Coll-doc `module_utils/purefb.py` `get_system` docstring and fragment 674 say this is "400", not 401/403.
  - Also stated in `purefb_ntp`, `purefb_fs_replica` and `purefb_syslog_settings`.
  - **Unverified.**
- **Audit.** Audit records carry `scopes`, "the realms and/or array which this audit affected and is visible to" (SDK `audit_*`). A realm-scoped identity only sees the records of its realms (Coll-doc `purefb_audits`).

### 1.8 Realm connections and cross-array realms

- **REST 2.25+** adds `/realm-connections`, `/realm-connections/connection-key` and `/remote-realms`:
  - `RealmConnection`: `local_realm`, `remote_realm`, `status` ∈ `CONNECTING`, `CONNECTED`, `DISCONNECTED`, `UNREACHABLE`.
  - `RealmConnectionPost` adds `connection_key`.
  - `RemoteRealm`: `name`, `arrays`.
  - Source: SDK `realm_connection_v_2_25`, `remote_realm_v_2_26`; methods in `FB_2_25/client.py`.
- **Replica links can target a realm.** `FileSystemReplicaLink.remote` is "Reference to a remote array or realm" from 2.25 (SDK `file_system_replica_link_v_2_25`).
- **Realm replication is a marketed feature.** The launch blog mentions "replication parity for multi-tenant realms" and recommends Purity//FB 4.6.8 — https://blog.everpuredata.com/products/everpure-launch-blog-march-2026-edition/
- **Collection-only claims (Unverified):**
  - REST 2.25 corresponds to Purity//FB 4.8.0.
  - Keys expire after 2 h and are single-use.
  - Realm names must be unique across connected arrays.
  - Source: Coll-doc `purefb_realm_connection`.

---

## 2. Collection inventory

**How to read the columns**
- **Exposure** says how the realm reaches the module: **E** means embedded in the object name (`realm::name`), **P** means a separate parameter.
- **Gate** lists the realm-specific REST version checks found in the code (`*_API_VERSION` constants used with `LooseVersion`). "none" means the module sends realm-qualified names on any version and leaves refusal to the array.
- **States** gives the `state` choices. A realm-qualified name works for all of them unless noted.
- **Docs** is my assessment of the module documentation.

Most rows were touched by the late-2026 "realm-audit" fix and docs PRs (#685–#743, `changelogs/fragments/6xx_*realm*`, `7xx_*realm*`). Their docs describe realm usage in detail, but they assert array behaviour without external sources.

### 2.1 Realm management

| Module | Exposure | Gate | States / ops | Docs |
|---|---|---|---|---|
| `purefb_realm` (1.25.0) | P: `name` (plain realm name; `::` and `,` refused in `main`), `rename`, `qos_policy`. `default_object_server` and `default_inbound_tls_policy` accept a bare name or `realm::x`; a bare name is qualified with the realm (`_member_leaf`, `_plan_member`). | `MINIMUM_API_VERSION` 2.19. `REALM_QOS_API_VERSION` / `REALM_DEFAULTS_API_VERSION` 2.20 for `qos_policy` / `default_object_server`. No gate for `default_inbound_tls_policy`, which is fine because the field exists in SDK `Realm` 2.19. | present (create, recover, rename, QoS, defaults); absent (destroy, eradicate; refuses while resource accesses or mgmt-policy rules reference it, `_realm_attachments`) | Accurate and thorough. Array-quirk claims are Unverified. |
| `purefb_realm_connection` (1.30.0, unreleased) | P: `realm`, `remote_realm` (plain names). | `MIN_REQUIRED_API_VERSION` 2.25 | present/absent, `generate_key`, `invalidate_keys`, `wait` | Thorough. 2.25 ↔ 4.8.0 is Unverified. |
| `purefb_resource_access` (1.28.0) | P: `scope_type=realms`, `scope_name` (plain realm); `resource_name` is array-level. | `MINIMUM_API_VERSION` 2.19 | present/absent (immutable; no update) | Thorough. Per-version resource-type list is Unverified. |
| `purefb_info` | Output: realm objects keyed `realm::name`; per-object `realms` lists (`_realm_names`, `_policy_realms`); `_realm_qualified` re-adds the prefix where the array returns bare member names (junctions, quota, eviction and WORM policy members). Subset `realms`: `realms` (incl. `default_inbound_tls_policy`, `default_object_server`), `resource_accesses`, and on 2.25+ `realm_connections`, `remote_realms`, `realm_connection_keys`. `_guard_reads` turns reads refused to realm admins into empty results. | `REALMS_API_VERSION` 2.19, `REALM_DEFAULTS_API_VERSION` 2.20, `REALM_CONNECTION_API_VERSION` 2.25 | read-only | Mostly accurate. Gaps: no QoS policy per realm (the module reads no QoS data at all); the `kerberos` `realm` key is not flagged as a Kerberos realm. |
| `module_utils/purefb.py` `get_system` | Login only. Docstring explains that a realm-scoped admin is refused `GET /hardware` (fragment 674). | — | — | No realm-name helpers exist in `module_utils`. Every module carries its own (see 3.1). |

### 2.2 Realm-scoped objects (realm in the name, with realm-specific code)

| Module | Exposure (names that take `realm::`) | Gate | States | Docs |
|---|---|---|---|---|
| `purefb_fs` | **P + E**: `realm` (the only object module with a separate realm param, 1.25.0) or `name=realm::fs`; `_resolve_name` qualifies a short name; `policy` and `worm_policy` must be full `realm::policy` (`_check_policy_scope`) | `REALM_API_VERSION` 2.19 in `_check_realm` | present/absent (all states use the resolved name) | Accurate. "If the realm has a QoS policy, the filesystem inherits it" is Unverified. |
| `purefb_bucket` | E: `name`, `account` (must be the same realm) | none | present/absent | Accurate |
| `purefb_bucket_access` | E: `name` (bucket); rule resources compared without realm (`_access_rule_matches`) | none | present/absent | Accurate |
| `purefb_bucket_replica` | E: `name`, `credential`; `target` plain (shared via resource access) | none | present/absent | Accurate. Realm-to-realm target not supported (not stated). |
| `purefb_remote_cred` | E: `name`; `target` plain | none | present/absent | Accurate (realm-docs PR #743) |
| `purefb_s3acc` | E: `name`; `account_exports` servers: a bare server is completed to `realm::server` | `REALM_API_VERSION` 2.19 / `REALM_CONTEXT_API_VERSION` 2.24 in `_check_realm` | present/absent | Accurate |
| `purefb_s3acc_export` | E: `account`, `server`, `policy` (all explicit; no completion) | none | present/absent | Accurate |
| `purefb_s3user` | E: `account`; `key_name` / `imported_key` bare IDs completed to `realm::id` | none | present/absent/remove_key/keystate | Accurate |
| `purefb_userpolicy` | E: `account`; `policy` as `realm::account/policy` | none | present/absent/show | Accurate |
| `purefb_object_role` | E: `account`; `name` plain (refuses `::`/`/`); bare policy qualified with `account` | none | present/absent | Accurate |
| `purefb_s3_export_policy` | E: `name`; `rename` bare keeps the realm | none | present/absent | Accurate |
| `purefb_virtualhost` | E: `name`, `servers`, `rename` (never completed; bare `rename` refused for a realm host) | none | present/absent | Accurate |
| `purefb_server` | E: `name`; `dns` bare means array-level; `local_directory_service` bare is looked up in the server's realm | none | present/absent | Accurate, but the rule differs within the module (see 3.2) |
| `purefb_network` | E: `name`; `attached_server` bare is completed to the interface realm | `REALMS_API_VERSION` 2.19 (realm existence check) | present/absent | Accurate |
| `purefb_dns` | E: `name`, `source`, `ca_certificate(_group)` (same realm) | none | present/absent | Accurate |
| `purefb_certs` | E: `name` | `REALM_API_VERSION` 2.19 (realm check) | present/absent/import/export/sign | Accurate |
| `purefb_certgrp` | E: `name`, `certificates` (same realm, never completed) | `REALM_API_VERSION` 2.19 | present/absent | Accurate |
| `purefb_tls_policy` | E: `name`, `appliance_certificate`, CA group, `network_interfaces`; rewrites the realm prefix to the stored case | `REALM_API_VERSION` 2.19 | present/absent | Accurate |
| `purefb_ds` | E: `nfs_server=realm::server` (directory service `realm::server_nfs`) | none | present/absent/test | Accurate |
| `purefb_ad` | E: `name` as `realm::server::account`, or `server` + `name` (`_account_name`) | none | present/absent/test/rotate | Accurate |
| `purefb_keytabs` | E: `prefix`/`name` as `realm::server::prefix` (text before the last `::` is the server) | none | absent/rotate/import/export | Accurate. Note that `realm::prefix` alone does not address a realm. |
| `purefb_local_ds` | E: `name` | none (module min 2.24) | present/absent | Accurate (PR #736). No realm code. |
| `purefb_local_user` / `purefb_local_group` | E: `local_directory_service`; `name` plain (`::` refused) | none (min 2.24) | present/absent | Accurate |
| `purefb_export` | E: `filesystem`, `server`, policies (explicit; export `name` never qualified) | none | present/absent | Accurate |
| `purefb_fs_junction` | E: `name`, `origin_fs`, `destination_fs` (same realm) | none (min 2.25) | present/absent | Accurate |
| `purefb_fs_lock` | E: `file_system` | none | absent/query | Accurate |
| `purefb_fs_replica` | E: `name`, `policy`; `target_array` must be an array (remote realm not supported) | none | present/absent | Accurate (PR #742); documents the replication gap |
| `purefb_snap` | E: `name` (snapshot `realm::fs.suffix`), `target_fs` | none | present/absent/restore/copy | Accurate |
| `purefb_legal_hold` | E: `name`, `file_system` (same realm) | `REALM_API_VERSION` 2.19 | present/absent | Accurate |
| `purefb_policy` | E: `name` for snapshot/NFS/SMB/WORM; access policies via `account`; `filesystem`, `replica_link`, `target`; `rename` bare keeps the realm | none | present/absent/copy | Mostly accurate. "Network access policies are not realm objects" conflicts with the SDK `NetworkAccessPolicy.realms` field (see 3.4). |
| `purefb_quota_policy` | E: `name`, `file_systems`, `rename` | none (min 2.25) | present/absent | Accurate |
| `purefb_eviction_policy` | E: `name`, `file_systems`, `rename` | none (min 2.21) | present/absent | Accurate |

### 2.3 Array-level objects whose docs or code address realms

| Module | Realm handling | States | Docs |
|---|---|---|---|
| `purefb_user` | Refuses `realm::name` client-side; confinement via `management_access_policies` | present/absent | Accurate (PR #737) |
| `purefb_mgmt_policy` | Realm is a rule scope (`rules[].resource_type=realms`, plain name; existence- and case-checked); `name` `::` left to the array | present/absent | Accurate (min policy 2.19, rules 2.26) |
| `purefb_mgmt_role` | Docs only: roles are array-level; grant in realm via mgmt-policy rule. No realm code. | present/absent | Accurate |
| `purefb_mgmt_auth_policy` | Docs only (PR #739); `::` refused by the array, not the module | present/absent | Accurate |
| `purefb_apiclient` | Docs only (PR #741); `::` "is refused", but by the array (no code check) | present/absent | Accurate but ambiguous about who refuses |
| `purefb_dsrole` | Refuses `realm::` in `name`/`policies` client-side | present/absent | Accurate (PR #735) |
| `purefb_oidc` | Refuses `realm::` `name`/`rename`; share via resource access | present/absent | Accurate |
| `purefb_saml` | Refuses `realm::` name and certificate refs | present/absent/test | Accurate |
| `purefb_subnet` | Refuses `realm::name`; share via resource access | present/absent | Accurate |
| `purefb_target` | Refuses create under `realm::name`; share via resource access | present/absent | Accurate |
| `purefb_syslog_settings` | Explains that the settings object is array-only; message for a realm-qualified `name` | present/absent | Accurate |

### 2.4 Passing mention only (no realm-specific code)

| Module | Mention |
|---|---|
| `purefb_audits` | `scopes` filter takes realm names (passed through; docs explain wildcard and eradicated realms). Gate `AUDIT_SCOPES_VERSION` 2.20 is for the filter, not realms. |
| `purefb_ntp` | Docs say a realm-scoped admin reads an empty server list (PR #738). |
| `purefb_groupquota`, `purefb_userquota` | `name` (file system) as `realm::fs` (PRs #733, #734) |
| `purefb_bucket_lifecycle`, `purefb_lifecycle` | `bucket` as `realm::bucket`; rule name unqualified |
| `purefb_ssh_ca_policy` | Policy is array-level; `signing_authority` certificate can be `realm::cert` |

**Counts:** 55 modules (4 in 2.1 + 33 in 2.2 + 11 in 2.3 + 7 in 2.4 = 55) plus 1 `module_utils` file, which reconciles with the 56-file grep. `purefb_local_ds` is listed in 2.2 because it is realm-scoped, though its realm handling is docs-only. `purefb_pingtrace` got a "realm-docs" branch (#740) but contains no realm text; its fragment `740_purefb_pingtrace_source_docs.yaml` is about `source`.

**History:**
- Realms first shipped in 1.25.0 (2026-05-22): `purefb_realm`, `purefb_fs` `realm` (fragment `529_realm_support.yaml`).
- 1.27.0 added `without_default_access_list`.
- 1.28.0 added `purefb_resource_access`.
- Unreleased (fragments): `purefb_realm_connection`, `default_object_server`, `default_inbound_tls_policy` and about 40 realm-audit fixes.

---

## 3. Inconsistencies and suspected bugs

### 3.1 No shared realm helpers; private copies diverge

- **Where:** 21 modules each define `_realm_of`, and many also define `_leaf` and `_same`/`_same_name`. Examples: `purefb_fs._realm_of`, `purefb_s3acc._realm_of`, `purefb_legal_hold._realm_of`.
- **Divergence:**
  - Most use `name.rpartition("::")[0]`, which returns everything before the *last* `::`.
  - `purefb_legal_hold._realm_of` uses `split("::")[0]` and `purefb_certs._realm_of` uses `rsplit("::", 1)[0]`.
  - `purefb_info._name_realm` / `_realm_qualified` use `split("::", 1)[0]`, the *first* segment.
  - For three-part names (`realm::server::account`), "realm of" means different things in different modules.
  - Some versions strip whitespace (`purefb_bucket`, `purefb_tls_policy`, `purefb_remote_cred`); others do not.
- **Repro:** `_realm_of("r1::srv::acct")` returns `r1::srv` in `purefb_fs` but `r1` in `purefb_legal_hold`.
- **Today's risk:** low. The modules that see three-part names (`purefb_ad`, `purefb_keytabs`) use their own `_server_of`.

### 3.2 Bare-name completion rules differ by module and option

| Completes a bare name into the realm | Never completes (explicit `realm::x` required) |
|---|---|
| `purefb_realm` `default_object_server`, `default_inbound_tls_policy` (`_member_leaf`) | `purefb_export` `server`, policies (bare `server` for a realm FS is refused in its scope check) |
| `purefb_s3acc` `account_exports` servers | `purefb_s3acc_export` `server`, `policy` |
| `purefb_network` `attached_server` | `purefb_virtualhost` `servers` ("A name is never completed with a realm") |
| `purefb_s3user` `key_name`, `imported_key` | `purefb_fs` `policy`, `worm_policy` ("I(realm) does not qualify it") |
| `purefb_object_role` access policies (qualified with `account`) | `purefb_certgrp` `certificates`; `purefb_tls_policy` certificates; `purefb_dns` `source` |
| `purefb_server` `local_directory_service` | `purefb_server` `dns` ("A name without a realm always means the array-level DNS configuration") |
| `rename` in `purefb_policy`, `purefb_quota_policy`, `purefb_s3_export_policy`, `purefb_eviction_policy` (bare keeps the realm) | `rename` in `purefb_virtualhost` (bare new name refused for a realm host) |

- **Repro 1:** Run `purefb_export` with `filesystem: r1::fs1, server: srv1`; it is refused. Run `purefb_s3acc` with `name: r1::acct, account_exports: [srv1]`; it uses `r1::srv1`.
- **Repro 2:** Run `purefb_virtualhost` with `name: r1::vh, rename: vh2`; it is refused. Run `purefb_policy` with `name: r1::snap, rename: snap2`; it becomes `r1::snap2`.
- Some of these may be forced by the array: shared DNS and built-in policies can legitimately be bare. Even so, the user-facing rule is not predictable.

### 3.3 `purefb_fs` is the only object module with a separate `realm` parameter

- **What differs:** `purefb_fs` has `realm` (resolved by `_resolve_name`). Every other module requires `realm::name`, including the closest siblings, `purefb_snap`, `purefb_export`, `purefb_fs_replica` and `purefb_quota_policy`. A playbook can therefore create `realm: r1, name: fs1` but must refer to it as `r1::fs1` everywhere else.
- **Dead code:** `_delete_fs`, `delete_fs` and `eradicate_fs` re-derive and re-qualify the realm from `module.params["realm"]`. `main` has already replaced `name` with `_resolve_name(module)`, so this logic never runs.

### 3.4 Realm precheck helpers have different semantics

Seven modules check, before creating, that the target realm exists and is live. Each has its own helper:

| Helper | Empty listing (`200`, no items) | Destroyed | Other read error |
|---|---|---|---|
| `purefb_fs._check_realm` | fail | fail | **fail** ("Failed to read realm") |
| `purefb_certs` realm check / `purefb_certgrp` | fail | fail | pass (left to the array) |
| `purefb_legal_hold` | fail | fail | fail on 400 only |
| `purefb_s3acc._check_realm` | **pass** | fail | pass |
| `purefb_tls_policy` | **pass** | fail | pass |
| `purefb_network` (`_is_missing`) | fail | **not checked** | pass |

- **Suspected bug:** `purefb_fs` with an identity whose role may not read `/realms` fails the create, even though the array might accept it. `purefb_realm_connection` docs say such roles exist ("a user whose role may not read realms").
- **Repro:** Create a filesystem with a custom role that is refused `GET /realms`. `purefb_fs` fails, while `purefb_certs` and `purefb_s3acc` proceed.
- **Second effect:** `purefb_network` reports a change in check mode for a new interface in a destroyed realm.

### 3.5 Client-side refusal of `realm::` on array-level objects is uneven

- **Refused before any request:** `purefb_user`, `purefb_dsrole`, `purefb_oidc`, `purefb_saml`, `purefb_subnet`, `purefb_target`, `purefb_local_user`/`_group` (`name`).
- **Left to the array:** `purefb_apiclient` (no `::` code at all), `purefb_mgmt_auth_policy`, `purefb_mgmt_role`, `purefb_mgmt_policy` `name`.
- **Repro:** `purefb_apiclient name=r1::c1 state=present` in check mode reports `changed=true` (`create_client` is skipped under `check_mode`). The real run is refused by the array. The docs say "is refused" without saying by whom.

### 3.6 `purefb_realm_connection` name matching and validation

- **Case:** `_connection_filter` builds `local_realm.name='X' and remote_realm.name='Y'` from the user input verbatim. `purefb_realm` says the array matches realm names case-insensitively. If the filter comparison is case-sensitive (**Unverified**), a task with a differently-cased realm will not find the existing connection and will try to create it again on every run.
- **Validation:** Unlike `purefb_realm` `main`, there is no refusal of `,`, `'` or `*` in `realm` / `remote_realm`. A quote breaks the filter expression.
- **Same pattern elsewhere:** `purefb_realm._realm_attachments` uses `filter="scope.name='...'"`. It is harmless there because it only produces a better error message.

### 3.7 `purefb_ntp` acts on a refused read for realm admins

- **What happens:** The docs say a realm-scoped admin reads an empty NTP list, so `state=present` tries to set servers that are already configured and `state=absent` reports no change.
- **Why it is a bug:** This is documented but not fixed. Other modules now detect restricted reads, e.g. `purefb_info._guard_reads`, `purefb_syslog_settings` and `purefb_fs_replica`'s connection hint.
- **Repro:** Run `purefb_ntp ntp_servers=[x] state=present` as a realm admin on an array that already has `x`.

### 3.8 Realm-prefix letter case on create is handled in some modules only

- **What differs:**
  - Coll-doc says the array accepts the realm prefix on create only in its stored spelling (`purefb_fs_junction` `name`, `purefb_server` `dns`, `purefb_tls_policy` fragment 712).
  - `purefb_tls_policy` rewrites the prefix to the stored case (in its realm check), and `purefb_mgmt_policy` does the same for rule scopes.
  - `purefb_server` and `purefb_fs_junction` only document the restriction.
  - `purefb_fs`, `purefb_bucket`, `purefb_s3acc`, `purefb_network`, `purefb_certs` and others neither document nor normalise it.
- **Repro (Inferred, needs an array):** Run `purefb_fs name=R1::fs` when the realm is stored as `r1`.

### 3.9 Docs that conflict with or go beyond the sources

- **`purefb_policy`:** "Network access policies are not realm objects" conflicts with SDK `NetworkAccessPolicy.realms` (2.19+). The field may be inherited from a base model (1.3 caveat), so this needs a check against an array.
- **`purefb_fs` `realm`:** "If the realm has a QoS policy, the filesystem inherits it" has no source.
- **`purefb_info` `kerberos` subset:** the `realm` key is a Kerberos realm, and the docs do not say so (`generate_kerb_dict`).
- **Purity ↔ REST mapping:** 2.19 = 4.6.1, 2.20 = 4.6.3, 2.25 = 4.8.0, 2.27 = 4.8.3 are stated in the docs of several modules with no source. Only 4.6.1 is corroborated (Portworx).
- **The `context` option:** shared by about 30 modules, it is never contrasted with realms. A user can reasonably guess `context: realm1`.

### 3.10 Functional gaps

- **Realm replication.**
  - `purefb_realm_connection` says a connection is "the prerequisite for realm-level replication".
  - No module consumes it. `purefb_fs_replica` refuses remote realms (`target_array`, "not supported as the target"), and `purefb_bucket_replica` and `purefb_remote_cred` refuse them too.
  - The SDK allows `FileSystemReplicaLink.remote` to be a realm from 2.25.
- **QoS.**
  - `purefb_realm` attaches an existing QoS policy, but no module creates QoS policies.
  - `purefb_info` reports no QoS data, so the realm's `qos_policy` cannot be read back.
- **Realm space.** `purefb_info` `realms` reports `space` but not `/realms/space/storage-classes` (SDK `get_realms_space_storage_classes`). This is minor.

### 3.11 Minor: literal `''` sent as `default_exports`

- **Where:** `purefb_fs.create_fs` sends `post_kwargs["default_exports"] = ["''"]`, the two-character string `''`. The comment says "empty string".
- **Status:** **Unverified.** This may be the REST convention for an empty list in a query parameter. It needs confirming against the API (SDK `post_file_systems` `default_exports` is "A comma-separated list of export types…").

---

## 4. Scope recommendation

**Recommendation: docs + behaviour changes/bug fixes.**

**Reasons**
1. **The complaint in #1 is about the whole picture, and that gap is documentation.** Per-module docs are now detailed: the realm-audit PRs #685–#743 rewrote nearly every module. What is missing is a single place that explains the naming, the bare-name rule, sharing, admin scoping and bring-up order, plus a glossary. Docs work is necessary either way.
2. **Documentation alone cannot make the behaviour predictable.**
   - The bare-name completion rule (3.2) and the `realm` param in `purefb_fs` (3.3) differ for no reason that the sources give. Writing that down produces a per-option exception table, which is the opposite of easy to figure out.
   - Unifying the behaviour, or at least adding one shared helper, is the real fix.
3. **There are concrete defects with repro sketches:**
   - precheck semantics (3.4),
   - check-mode false positives for array-level names (3.5),
   - realm-connection matching (3.6),
   - NTP for realm admins (3.7),
   - realm-prefix case (3.8).
4. **Much of the behaviour is still unverified on an array.** Any behaviour change should come with a verification step on Purity//FB 4.6.x and 4.8.x.

### Proposed follow-up issues (not filed)

**Docs**
1. **Add a collection-level realm guide.** One page covering what realms are, `realm::name` and the nested forms, bare names, sharing via resource access, realm admins, bring-up and tear-down order (interface → DNS → server → objects), and the module map.
2. **Create GLOSSARY.md with realm terms.** Seed it from §5 of this document.
3. **Distinguish fleet `context` from realms.** Add one sentence to every module's `context` option (or a shared doc fragment).
4. **Flag the Kerberos `realm` key.** Label it in the `purefb_info` `kerberos` subset and in the `purefb_keytabs` and `purefb_ad` docs.
5. **Verify unsourced array claims.** Check on an array, or remove: the QoS inheritance in `purefb_fs`, the network-access-policy realm claim in `purefb_policy`, the resource types per Purity version, the 2.25 = 4.8.0 mapping, and realm-name uniqueness across connected arrays.
6. **Document realm replication support.** State clearly what the collection supports today: realm connection yes, realm replica links no.

**Behaviour / bug fixes**
1. **Add shared realm helpers to module_utils.** `realm_of`, `leaf`, `qualify`, `same_name` and `check_realm_usable`, with defined semantics for three-part names, replacing the 21 private copies (3.1).
2. **Unify realm precheck semantics.** Settle what an empty listing, a destroyed realm and a refused read mean before a create, and apply it in fs, certs, certgrp, legal_hold, s3acc, tls_policy and network (3.4).
3. **Decide one bare-name completion policy.** Choose "never complete" or "complete into the owning object's realm" for references, and apply it consistently, including `rename` in `purefb_virtualhost` and `dns` in `purefb_server` (3.2).
4. **Decide the fate of `purefb_fs` `realm`.** Either deprecate it in favour of `realm::name`, or add it to the other object modules. Remove the dead re-qualification in the delete and eradicate paths (3.3).
5. **Refuse `realm::` in the remaining array-level modules.** Do it client-side in `purefb_apiclient`, `purefb_mgmt_auth_policy`, `purefb_mgmt_role` and the `purefb_mgmt_policy` `name`, so check mode matches the real run (3.5).
6. **Fix matching in `purefb_realm_connection`.** Match case-insensitively and refuse `,` `'` `*` in realm names (3.6).
7. **Make `purefb_ntp` detect restricted reads.** Fail or warn for a realm-scoped admin instead of acting on an empty list (3.7).
8. **Normalise the realm prefix on create.** Rewrite it to the stored spelling everywhere, following the `purefb_tls_policy` pattern (3.8).
9. **Report realm QoS policies in `purefb_info`.** Read them through `get_qos_policies_members(member_types=["realms"])` (3.10).
10. **Support remote-realm targets in `purefb_fs_replica`.** Feature work: use `FileSystemReplicaLink.remote` as a realm on REST 2.25+ (3.10).
11. **Confirm `default_exports=["''"]`.** Check it against the REST spec and fix or comment it (3.11).

---

## 5. Glossary seed

- **Realm.** A named, isolated tenant space on a FlashBlade (REST `/realms`, REST 2.19 / Purity//FB 4.6.1+). It groups realm-scoped objects such as file systems, object store accounts, servers, network interfaces, policies and certificates. A realm can be destroyed, recovered and eradicated, can carry a QoS policy and per-realm defaults, and is managed by an array administrator. Realms are not nested. *Not to be confused with a Kerberos realm, or with a fleet context.*
- **Realm-scoped object.** An object that lives inside exactly one realm, reported in its `realms` field or, for buckets, users and snapshots, implied by its owner. It can reference only objects of its own realm, or array-level objects shared with the realm. It cannot be moved to another realm; it must be destroyed and recreated.
- **Array-level object.** An object outside any realm (`realms` empty). Its name has no realm prefix. Some types can never be in a realm, for example users, API clients, management policies and roles, subnets, targets, and OIDC or SAML IdPs.
- **Qualified name (realm-qualified name).** The `realm::name` form that addresses a realm-scoped object. Child objects nest: `realm::account/user`, `realm::server::account`, `realm::fs.suffix`, `realm::<access key id>`. A bare name always means the array-level object. The array does not resolve it into the caller's realm.
- **Bare name (short name).** A name without a realm prefix. Whether a module completes it into a realm depends on the module and option today (see §3.2).
- **Resource access.** A grant that shares an array-level resource with a realm (scope) without moving it (REST `/resource-accesses`), e.g. a subnet, a DNS configuration, an S3 target or an IdP. It is immutable: delete and recreate to change it. While one exists it blocks destroying the realm.
- **Realm administrator (realm-scoped administrator).** An admin whose management access policies contain only rules scoped to realms (`scope.resource_type=realms`). Such an admin sees and manages only those realms plus the array-level objects shared with them, and is refused array-wide reads.
- **Realm connection.** A pairing of a local realm with a realm on a connected FlashBlade (REST `/realm-connections`, REST 2.25+). It is established with a single-use connection key generated on the remote side, and is the prerequisite for realm-level replication. Its status is one of `CONNECTING`, `CONNECTED`, `DISCONNECTED` or `UNREACHABLE`.
- **Remote realm.** A realm on a connected array, as listed by `/remote-realms`.
- **Connection key.** A one-time secret generated for a realm (`/realm-connections/connection-key`) and consumed by the peer array to create a realm connection.
- **Realm defaults.** Per-realm defaults (`/realms/defaults`, REST 2.20+). Today this holds the default object store server that accounts and virtual hosts of the realm are exported through.
- **Default inbound TLS policy.** The realm's TLS policy for its network interfaces that have no TLS policy of their own. Without one, those interfaces cannot be reached over TLS.
- **Context (fleet context).** *Not a realm.* The fleet member array, or the fleet itself, that a request is executed on (`context_names`; the collection's `context` option).

---

### Sources not found / gaps

- **No public Purity//FB admin guide or REST reference page on realms** was found. The SDK schema is the only primary API source used, and most array-behaviour statements rest on collection docs (**Unverified**).
- **The support.everpuredata.com PDF** (`/api/khub/documents/v01YXt1zRNBXs2AjEHVQiw/content`) came back but could not be text-extracted here.
- **The Purity//FB ↔ REST version table** is unsourced except 4.6.1 ↔ realms (Portworx).
- **Case sensitivity** of REST `filter` expressions, and the meaning of `realms` on policy types that inherit it from a base model, are unverified.
