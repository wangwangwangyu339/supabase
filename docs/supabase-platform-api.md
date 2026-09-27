# Supabase 平台管理 API 接口手册（按接口维度）

> 本文档按 **API 接口维度** 整理 Supabase 平台管理 API（Platform Management API）的全部端点，
> 数据来源：`packages/api-types/types/{platform,api-v1,api-v2}.d.ts`（由 openapi-typescript 从 api.supabase.com 实时 OpenAPI 生成）
> 与 `apps/docs/spec/api_v{1,2}_openapi.json`（官方下载的 OpenAPI 规范，含完整 summary/description）。
> 说明栏保留官方英文原文，结构性说明为中文。

## 1. 概览

### 1.1 是什么

平台管理 API 是 Supabase 控制台（Studio）背后的管理面 API，用于**管理**（而非访问用户数据）：
项目、组织、数据库配置、认证配置、存储配置、Edge Functions、日志与分析、计费、分支、Webhook 等。
Studio 前端通过 `data/fetchers.ts`（openapi-fetch）直连该 API，类型定义即来自本手册对应文件。

### 1.2 三套规范

| 规范 | 基址 | 路径数 | 操作数 | 可见性 | 用途 |
| --- | --- | --- | --- | --- | --- |
| **Platform（内部）** | `NEXT_PUBLIC_API_URL`（托管指向 `https://api.supabase.com`） | 296 | 392 | 内部（平台后端实现，Studio 消费） | 平台运营全能力：auth 管理、pg-meta、replication、storage、billing、telemetry、warehouse 等 |
| **Management API v1** | `https://api.supabase.com/v1` | 115 | 170 | 公开（开发者/官方文档） | 面向开发者的管理 API：projects、config、database、functions、analytics、branches 等 |
| **Management API v2** | `https://api.supabase.com/v2` | 38（规范 32 + 类型新增 6） | 55（并集） | 公开（新端点持续迁移中） | 新一代端点：webhooks、compute/workers、transfers、notebooks、private-link、log drains 等 |

> 注：v2 官方规范与类型文件存在漂移（`workers`→`compute` 演进、`notebooks` 新增），本文取并集整理，见 §3。

### 1.3 鉴权

- 公开 v1/v2：**Bearer Token**（`Authorization: Bearer <PAT>`，PAT = Personal Access Token），
  支持 OAuth2 授权码流程（`/v1/oauth/authorize`、`/v1/oauth/token`）获取用户级 token。
- 内部 platform：Bearer + Cookie 会话（Studio 端 `credentials: include`，由平台网关校验）。
- 详见附录 A。

### 1.4 数据流（Studio 如何消费）

```
浏览器(Studio) --openapi-fetch--> api.supabase.com(/platform|/v1|/v2) --> 平台后端
                                     ├── pg-meta（用户库元数据/查询）
                                     ├── GoTrue / PostgREST / Storage / Realtime / Functions（用户项目运行时）
                                     └── Logflare / ClickHouse（日志分析）
```

---

## 2. Management API v1 —— `/v1`（115 路径 / 170 操作）

按 OpenAPI tag 分组。全部端点基址 `https://api.supabase.com/v1`。


### Projects

共 29 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects` | **List all projects** — Returns a list of all projects you've previously created. |
| `POST` | `/v1/projects` | **Create a project** |
| `GET` | `/v1/projects/available-regions` | **[Beta] Gets the list of available regions that can be used for a new project** |
| `POST` | `/v1/projects/{ref}/network-bans/retrieve` | **[Beta] Gets project's network bans** |
| `POST` | `/v1/projects/{ref}/network-bans/retrieve/enriched` | **[Beta] Gets project's network bans with additional information about which databases they affect** |
| `DELETE` | `/v1/projects/{ref}/network-bans` | **[Beta] Remove network bans.** |
| `GET` | `/v1/projects/{ref}/network-restrictions` | **[Beta] Gets project's network restrictions** |
| `PATCH` | `/v1/projects/{ref}/network-restrictions` | **[Alpha] Updates project's network restrictions by adding or removing CIDRs** |
| `POST` | `/v1/projects/{ref}/network-restrictions/apply` | **[Beta] Updates project's network restrictions** |
| `GET` | `/v1/projects/{ref}` | **Gets a specific project that belongs to the authenticated user** |
| `DELETE` | `/v1/projects/{ref}` | **Deletes the given project** |
| `PATCH` | `/v1/projects/{ref}` | **Updates the given project** |
| `POST` | `/v1/projects/{ref}/upgrade` | **[Beta] Upgrades the project's Postgres version** |
| `GET` | `/v1/projects/{ref}/upgrade/eligibility` | **[Beta] Returns the project's eligibility for upgrades** |
| `GET` | `/v1/projects/{ref}/upgrade/status` | **[Beta] Gets the latest status of the project's upgrade** |
| `GET` | `/v1/projects/{ref}/health` | **Gets project's service health status** |
| `POST` | `/v1/projects/{ref}/pause` | **Pauses the given project** |
| `POST` | `/v1/projects/{ref}/restart` | **Restarts the given project** |
| `GET` | `/v1/projects/{ref}/restore` | **Lists available restore versions for the given project** |
| `POST` | `/v1/projects/{ref}/restore` | **Restores the given project** |
| `POST` | `/v1/projects/{ref}/restore/cancel` | **Cancels the given project restoration** |
| `GET` | `/v1/projects/{ref}/claim-token` | **Gets project claim token** |
| `POST` | `/v1/projects/{ref}/claim-token` | **Creates project claim token** |
| `DELETE` | `/v1/projects/{ref}/claim-token` | **Revokes project claim token** |
| `GET` | `/v1/projects/{ref}/config/disk` | **Get database disk attributes** |
| `POST` | `/v1/projects/{ref}/config/disk` | **Modify database disk** |
| `GET` | `/v1/projects/{ref}/config/disk/util` | **Get disk utilization** |
| `GET` | `/v1/projects/{ref}/config/disk/autoscale` | **Gets project disk autoscale config** |
| `GET` | `/v1/organizations/{slug}/projects` | **Gets all projects for the given organization** — Returns a paginated list of projects for the specified organization.  This endpoint uses offset-based pagination. Use the `offset` parameter to skip a number of projects and the `limit` parameter to control the number of projects returned per page. |

### Database

共 46 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/snippets` | **Lists SQL snippets for the logged in user** |
| `GET` | `/v1/snippets/{id}` | **Gets a specific SQL snippet** |
| `GET` | `/v1/projects/{ref}/jit-access` | **[Beta] Get project's temporary access configuration.** |
| `PUT` | `/v1/projects/{ref}/jit-access` | **[Beta] Update project's temporary access configuration.** |
| `GET` | `/v1/projects/{ref}/ssl-enforcement` | **[Beta] Get project's SSL enforcement configuration.** |
| `PUT` | `/v1/projects/{ref}/ssl-enforcement` | **[Beta] Update project's SSL enforcement configuration.** |
| `GET` | `/v1/projects/{ref}/types/typescript` | **Generate TypeScript types** — Returns the TypeScript types of your schema for use with supabase-js. |
| `GET` | `/v1/projects/{ref}/readonly` | **Returns project's readonly mode status** |
| `POST` | `/v1/projects/{ref}/readonly/temporary-disable` | **Disables project's readonly mode for the next 15 minutes** |
| `POST` | `/v1/projects/{ref}/read-replicas/setup` | **[Beta] Set up a read replica** |
| `POST` | `/v1/projects/{ref}/read-replicas/remove` | **[Beta] Remove a read replica** |
| `POST` | `/v1/projects/{ref}/cli/login-role` | **[Beta] Create a login role for CLI with temporary password** |
| `DELETE` | `/v1/projects/{ref}/cli/login-role` | **[Beta] Delete existing login roles used by CLI** |
| `GET` | `/v1/projects/{ref}/database/migrations` | **List applied migration versions** |
| `POST` | `/v1/projects/{ref}/database/migrations` | **Apply a database migration** |
| `PUT` | `/v1/projects/{ref}/database/migrations` | **Upsert a database migration without applying** |
| `DELETE` | `/v1/projects/{ref}/database/migrations` | **Rollback database migrations and remove them from history table** |
| `GET` | `/v1/projects/{ref}/database/migrations/{version}` | **Fetch an existing entry from migration history** |
| `PATCH` | `/v1/projects/{ref}/database/migrations/{version}` | **Patch an existing entry in migration history** |
| `POST` | `/v1/projects/{ref}/database/query` | **[Beta] Run sql query** |
| `POST` | `/v1/projects/{ref}/database/query/read-only` | **[Beta] Run a sql query as supabase_read_only_user** — All entity references must be schema qualified. |
| `POST` | `/v1/projects/{ref}/database/webhooks/enable` | **[Beta] Enables Database Webhooks on the project** |
| `GET` | `/v1/projects/{ref}/database/context` | **Gets database metadata for the given project.** — This is an **experimental** endpoint. It is subject to change or removal in future versions. Use it with caution, as it may not remain supported or stable. |
| `PATCH` | `/v1/projects/{ref}/database/password` | **Updates the database password** |
| `GET` | `/v1/projects/{ref}/database/jit` | **Get user-id to role mappings for JIT access** — Mappings of roles a user can assume in the project database |
| `POST` | `/v1/projects/{ref}/database/jit` | **Authorize user-id to role mappings for JIT access** — Authorizes the request to assume a role in the project database |
| `PUT` | `/v1/projects/{ref}/database/jit` | **Updates a user mapping for JIT access** — Modifies the roles that can be assumed and for how long |
| `GET` | `/v1/projects/{ref}/database/jit/list` | **List all user-id to role mappings for JIT access** — Mappings of roles a user can assume in the project database |
| `POST` | `/v1/projects/{ref}/database/jit/invite` | **Invites an external user to a database for JIT access** — Invites the external user and sets initial roles that can be assumed and for how long |
| `POST` | `/v1/projects/{ref}/database/jit/invite/accept` | **Accepts invitation for JIT database access** — Accepts the invitation to JIT database access |
| `DELETE` | `/v1/projects/{ref}/database/jit/invite/{invite_id}` | **Deletes the invite for an external user to a database for JIT access** — Revokes and deletes the invitation |
| `DELETE` | `/v1/projects/{ref}/database/jit/{user_id}` | **Delete JIT access by user-id** — Remove JIT mappings of a user, revoking all JIT database access |
| `GET` | `/v1/projects/{ref}/database/openapi` | **Get PostgREST OpenAPI spec** — Returns the PostgREST OpenAPI specification for the project. This is the replacement for querying `/rest/v1/` directly with the anon key. |
| `GET` | `/v1/projects/{ref}/config/database/pgbouncer` | **Get project's pgbouncer config** |
| `GET` | `/v1/projects/{ref}/config/database/pooler` | **Gets project's supavisor config** |
| `PATCH` | `/v1/projects/{ref}/config/database/pooler` | **Updates project's supavisor config** |
| `GET` | `/v1/projects/{ref}/config/database/postgres` | **Gets project's Postgres config** |
| `PUT` | `/v1/projects/{ref}/config/database/postgres` | **Updates project's Postgres config** |
| `GET` | `/v1/projects/{ref}/database/backups` | **Lists all backups** |
| `POST` | `/v1/projects/{ref}/database/backups/restore-pitr` | **Restores a PITR backup for a database** |
| `POST` | `/v1/projects/{ref}/database/backups/restore-point` | **Initiates a creation of a restore point for a database** |
| `GET` | `/v1/projects/{ref}/database/backups/restore-point` | **Get restore points for project** |
| `POST` | `/v1/projects/{ref}/database/backups/restore` | **Restores a physical backup for a database** |
| `GET` | `/v1/projects/{ref}/database/backups/schedule` | **Gets the backup schedule for a project** |
| `PATCH` | `/v1/projects/{ref}/database/backups/schedule` | **Updates the backup schedule time for a project** — Sets the time at which the daily backup runs. The change takes effect on the next backup window that includes the new time. If the new time has already passed for today, the first backup at the new time will occur the following day. It can only be updated 3 times per 24 hours. |
| `POST` | `/v1/projects/{ref}/database/backups/undo` | **Initiates an undo to a given restore point** |

### Auth

共 18 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/v1/projects/{ref}/config/auth/signing-keys/legacy` | **Set up the project's existing JWT secret as an in_use JWT signing key. This endpoint will be removed in the future always check for HTTP 404 Not Found.** |
| `GET` | `/v1/projects/{ref}/config/auth/signing-keys/legacy` | **Get the signing key information for the JWT secret imported as signing key for this project. This endpoint will be removed in the future, check for HTTP 404 Not Found.** |
| `POST` | `/v1/projects/{ref}/config/auth/signing-keys` | **Create a new signing key for the project in standby status** |
| `GET` | `/v1/projects/{ref}/config/auth/signing-keys` | **List all signing keys for the project** |
| `GET` | `/v1/projects/{ref}/config/auth/signing-keys/{id}` | **Get information about a signing key** |
| `DELETE` | `/v1/projects/{ref}/config/auth/signing-keys/{id}` | **Remove a signing key from a project. Only possible if the key has been in revoked status for a while.** |
| `PATCH` | `/v1/projects/{ref}/config/auth/signing-keys/{id}` | **Update a signing key, mainly its status** |
| `GET` | `/v1/projects/{ref}/config/auth` | **Gets project's auth config** |
| `PATCH` | `/v1/projects/{ref}/config/auth` | **Updates a project's auth config** |
| `POST` | `/v1/projects/{ref}/config/auth/third-party-auth` | **Creates a new third-party auth integration** |
| `GET` | `/v1/projects/{ref}/config/auth/third-party-auth` | **Lists all third-party auth integrations** |
| `DELETE` | `/v1/projects/{ref}/config/auth/third-party-auth/{tpa_id}` | **Removes a third-party auth integration** |
| `GET` | `/v1/projects/{ref}/config/auth/third-party-auth/{tpa_id}` | **Get a third-party integration** |
| `POST` | `/v1/projects/{ref}/config/auth/sso/providers` | **Creates a new SSO provider** |
| `GET` | `/v1/projects/{ref}/config/auth/sso/providers` | **Lists all SSO providers** |
| `GET` | `/v1/projects/{ref}/config/auth/sso/providers/{provider_id}` | **Gets a SSO provider by its UUID** |
| `PUT` | `/v1/projects/{ref}/config/auth/sso/providers/{provider_id}` | **Updates a SSO provider by its UUID** |
| `DELETE` | `/v1/projects/{ref}/config/auth/sso/providers/{provider_id}` | **Removes a SSO provider by its UUID** |

### Environments

共 17 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/branches/{branch_id_or_ref}` | **Get database branch config** — Fetches configurations of the specified database branch |
| `PATCH` | `/v1/branches/{branch_id_or_ref}` | **Update database branch config** — Updates the configuration of the specified database branch |
| `DELETE` | `/v1/branches/{branch_id_or_ref}` | **Delete a database branch** — Deletes the specified database branch. By default, deletes immediately. Use force=false to schedule deletion with 1-hour grace period (only when soft deletion is enabled). |
| `POST` | `/v1/branches/{branch_id_or_ref}/push` | **Pushes a database branch** — Pushes the specified database branch |
| `POST` | `/v1/branches/{branch_id_or_ref}/merge` | **Merges a database branch** — Merges the specified database branch |
| `POST` | `/v1/branches/{branch_id_or_ref}/reset` | **Resets a database branch** — Resets the specified database branch |
| `POST` | `/v1/branches/{branch_id_or_ref}/restore` | **Restore a scheduled branch deletion** — Cancels scheduled deletion and restores the branch to active state |
| `GET` | `/v1/branches/{branch_id_or_ref}/diff` | **[Beta] Diffs a database branch** — Diffs the specified database branch |
| `HEAD` | `/v1/projects/{ref}/actions` | **Count the number of action runs** — Returns the total number of action runs of the specified project. |
| `GET` | `/v1/projects/{ref}/actions` | **List all action runs** — Returns a paginated list of action runs of the specified project. |
| `GET` | `/v1/projects/{ref}/actions/{run_id}` | **Get the status of an action run** — Returns the current status of the specified action run. |
| `PATCH` | `/v1/projects/{ref}/actions/{run_id}/status` | **Update the status of an action run** — Updates the status of an ongoing action run. |
| `GET` | `/v1/projects/{ref}/actions/{run_id}/logs` | **Get the logs of an action run** — Returns the logs from the specified action run. |
| `GET` | `/v1/projects/{ref}/branches` | **List all database branches** — Returns all database branches of the specified project. |
| `POST` | `/v1/projects/{ref}/branches` | **Create a database branch** — Creates a database branch from the specified project. |
| `DELETE` | `/v1/projects/{ref}/branches` | **Disables preview branching** — Disables preview branching for the specified project |
| `GET` | `/v1/projects/{ref}/branches/{name}` | **Get a database branch** — Fetches the specified database branch by its name. |

### Secrets

共 12 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/api-keys` | **Get project api keys** |
| `POST` | `/v1/projects/{ref}/api-keys` | **Creates a new API key for the project** |
| `GET` | `/v1/projects/{ref}/api-keys/legacy` | **Check whether JWT based legacy (anon, service_role) API keys are enabled. This API endpoint will be removed in the future, check for HTTP 404 Not Found.** |
| `PUT` | `/v1/projects/{ref}/api-keys/legacy` | **Disable or re-enable JWT based legacy (anon, service_role) API keys. This API endpoint will be removed in the future, check for HTTP 404 Not Found.** |
| `PATCH` | `/v1/projects/{ref}/api-keys/{id}` | **Updates an API key for the project** |
| `GET` | `/v1/projects/{ref}/api-keys/{id}` | **Get API key** |
| `DELETE` | `/v1/projects/{ref}/api-keys/{id}` | **Deletes an API key for the project** |
| `GET` | `/v1/projects/{ref}/pgsodium` | **[Beta] Gets project's pgsodium config** |
| `PUT` | `/v1/projects/{ref}/pgsodium` | **[Beta] Updates project's pgsodium config. Updating the root_key can cause all data encrypted with the older key to become inaccessible.** |
| `GET` | `/v1/projects/{ref}/secrets` | **List all secrets** — Returns all secrets you've previously added to the specified project. |
| `POST` | `/v1/projects/{ref}/secrets` | **Bulk create secrets** — Creates multiple secrets and adds them to the specified project. |
| `DELETE` | `/v1/projects/{ref}/secrets` | **Bulk delete secrets** — Deletes all secrets with the given names from the specified project |

### Domains

共 9 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/custom-hostname` | **[Beta] Gets project's custom hostname config** |
| `DELETE` | `/v1/projects/{ref}/custom-hostname` | **[Beta] Deletes a project's custom hostname configuration** |
| `POST` | `/v1/projects/{ref}/custom-hostname/initialize` | **[Beta] Updates project's custom hostname configuration** |
| `POST` | `/v1/projects/{ref}/custom-hostname/reverify` | **[Beta] Attempts to verify the DNS configuration for project's custom hostname configuration** |
| `POST` | `/v1/projects/{ref}/custom-hostname/activate` | **[Beta] Activates a custom hostname for a project.** |
| `GET` | `/v1/projects/{ref}/vanity-subdomain` | **[Beta] Gets current vanity subdomain config** |
| `DELETE` | `/v1/projects/{ref}/vanity-subdomain` | **[Beta] Deletes a project's vanity subdomain configuration** |
| `POST` | `/v1/projects/{ref}/vanity-subdomain/check-availability` | **[Beta] Checks vanity subdomain availability** |
| `POST` | `/v1/projects/{ref}/vanity-subdomain/activate` | **[Beta] Activates a vanity subdomain for a project.** |

### Organizations

共 7 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/organizations` | **List all organizations** — Returns a list of organizations that you currently belong to. |
| `POST` | `/v1/organizations` | **Create an organization** |
| `GET` | `/v1/organizations/{slug}/entitlements` | **Get entitlements for an organization** — Returns the entitlements available to the organization based on their plan and any overrides. |
| `GET` | `/v1/organizations/{slug}/members` | **List members of an organization** |
| `GET` | `/v1/organizations/{slug}` | **Gets information about the organization** |
| `GET` | `/v1/organizations/{slug}/project-claim/{token}` | **Gets project details for the specified organization and claim token** |
| `POST` | `/v1/organizations/{slug}/project-claim/{token}` | **Claims project for the specified organization** |

### Edge Functions

共 8 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/functions` | **List all functions** — Returns all functions you've previously added to the specified project. |
| `POST` | `/v1/projects/{ref}/functions` | **Create a function** — This endpoint is deprecated - use the deploy endpoint. Creates a function and adds it to the specified project. |
| `PUT` | `/v1/projects/{ref}/functions` | **Bulk update functions** — Bulk update functions. It will create a new function or replace existing. The operation is idempotent. NOTE: You will need to manually bump the version. |
| `POST` | `/v1/projects/{ref}/functions/deploy` | **Deploy a function** — A new endpoint to deploy functions. It will create if function does not exist. |
| `GET` | `/v1/projects/{ref}/functions/{function_slug}` | **Retrieve a function** — Retrieves a function with the specified slug and project. |
| `PATCH` | `/v1/projects/{ref}/functions/{function_slug}` | **Update a function** — Updates a function with the specified slug and project. |
| `DELETE` | `/v1/projects/{ref}/functions/{function_slug}` | **Delete a function** — Deletes a function with the specified slug from the specified project. |
| `GET` | `/v1/projects/{ref}/functions/{function_slug}/body` | **Retrieve a function body** — Retrieves a function body for the specified slug and project. |

### Analytics

共 6 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/analytics/endpoints/logs.all` | **Gets project's logs** — Executes a SQL query on the project's logs.  Either the `iso_timestamp_start` and `iso_timestamp_end` parameters must be provided. If both are not provided, only the last 1 minute of logs will be queried. The timestamp range must be no more than 24 hours and is rounded to the nearest minute. If the range is more than 24 hours, a validation error will be thrown.  Note: Unless the `sql` parameter is provided, only edge_logs will be queried. See the [log query docs](https://supabase.com/docs/guides/monitoring-and-debugging/logs#logs-explorer) for all available sources. |
| `GET` | `/v1/projects/{ref}/analytics/endpoints/logs` | **Gets all project's logs in a single log stream** — Executes an SQL or LQL query on the project's unified logs stream.  Either the `iso_timestamp_start` and `iso_timestamp_end` parameters must be provided. If both are not provided, only the last 1 minute of logs will be queried. The timestamp range must be no more than 24 hours and is rounded to the nearest minute. If the range is more than 24 hours, a validation error will be thrown.  Filter by the `source` column to specify specific log sources, such as edge_logs, postgres_logs, etc.  Note: SQL must be written in **ClickHouse SQL dialect**. |
| `GET` | `/v1/projects/{ref}/analytics/endpoints/usage.api-counts` | **Gets project's usage api counts** |
| `GET` | `/v1/projects/{ref}/analytics/endpoints/usage.api-requests-count` | **Gets project's usage api requests count** |
| `GET` | `/v1/projects/{ref}/analytics/endpoints/functions.combined-stats` | **Gets a project's function combined statistics** |
| `GET` | `/v1/projects/{ref}/analytics/endpoints/metrics` | **Scrape a project's metrics** — Prometheus scrape endpoint. Returns metrics of a customer project in the Prometheus open exposition format. |

### Billing

共 3 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/billing/addons` | **List billing addons and compute instance selections** — Returns the billing addons that are currently applied, including the active compute instance size, and lists every addon option that can be provisioned with pricing metadata. |
| `PATCH` | `/v1/projects/{ref}/billing/addons` | **Apply or update billing addons, including compute instance size** — Selects an addon variant, for example scaling the project’s compute instance up or down, and applies it to the project. |
| `DELETE` | `/v1/projects/{ref}/billing/addons/{addon_variant}` | **Remove billing addons or revert compute instance sizing** — Disables the selected addon variant, including rolling the compute instance back to its previous size. |

### Advisors

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/advisors/performance` | **Gets project performance advisors.** — This is an **experimental** endpoint. It is subject to change or removal in future versions. Use it with caution, as it may not remain supported or stable. |
| `GET` | `/v1/projects/{ref}/advisors/security` | **Gets project security advisors.** — This is an **experimental** endpoint. It is subject to change or removal in future versions. Use it with caution, as it may not remain supported or stable. |

### Storage

共 3 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/storage/buckets` | **Lists all buckets** |
| `GET` | `/v1/projects/{ref}/config/storage` | **Gets project's storage config** |
| `PATCH` | `/v1/projects/{ref}/config/storage` | **Updates project's storage config** |

### Rest

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/postgrest` | **Gets project's postgrest config** |
| `PATCH` | `/v1/projects/{ref}/postgrest` | **Updates project's postgrest config** |

### Realtime

共 3 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/projects/{ref}/config/realtime` | **Gets realtime configuration** |
| `PATCH` | `/v1/projects/{ref}/config/realtime` | **Updates realtime configuration** |
| `POST` | `/v1/projects/{ref}/config/realtime/shutdown` | **Shutdowns realtime connections for a project** |

### OAuth

共 4 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/oauth/authorize` | **[Beta] Authorize user through oauth** |
| `POST` | `/v1/oauth/token` | **[Beta] Exchange auth code for user's access and refresh token** — Supports `authorization_code`, `refresh_token`, and `urn:ietf:params:oauth:grant-type:jwt-bearer` grant types. The `jwt-bearer` grant type (IDJAG — identity-directed JWT assertion) is in beta and available on Team and Enterprise plans only. |
| `POST` | `/v1/oauth/revoke` | **[Beta] Revoke oauth app authorization and it's corresponding tokens** |
| `GET` | `/v1/oauth/authorize/project-claim` | **Authorize user through oauth and claim a project** — Initiates the OAuth authorization flow for the specified provider. After successful authentication, the user can claim ownership of the specified project. |

### Profile

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v1/profile` | **Gets the user's profile** |

---

## 3. Management API v2 —— `/v2`（合并整理 38 路径 / 55 操作）

新一代公开管理 API。官方 OpenAPI 规范下载版为 **32 路径 / 45 操作**，
本地生成的类型文件（`api-v2.d.ts`）更新为 34 路径 / 50 操作，两版存在漂移：
规范中的 `workers` 端点（Alpha）在类型文件中已演进为 `compute` 端点，`notebooks` 为类型文件新增。
下表取两者**并集**（官方描述优先），按路径语义分组。


### Project Config

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/config` | **[Alpha] Get a project's service configuration** — Returns the project's database, pooler, Auth, Data API, Realtime and Storage configuration — the same configuration a branch inherits from its base project. Each is the effective config, so a setting the project has never overridden is reported at its platform default rather than as null. Auth secrets are returned as an HMAC of their value. `storage` is read live from the storage service; the rest come from this platform's own records. |

### Project Analytics

共 4 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/analytics/log-drains` | **List project log drains** |
| `POST` | `/v2/projects/{ref}/analytics/log-drains` | **Create a log drain for a project** |
| `PUT` | `/v2/projects/{ref}/analytics/log-drains/{id}` | **Update a project log drain** |
| `DELETE` | `/v2/projects/{ref}/analytics/log-drains/{id}` | **Delete a project log drain** |

### Project Branches

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/v2/projects/{ref}/branches` | **Create a database branch** — Creates a database branch from the specified project. Compute and disk size can be set here so the branch is provisioned at the requested size, instead of being resized after creation. |

### Project Compute

共 5 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/compute` | 内部端点（实现 `v2-list-all-compute-instances`） |
| `GET` | `/v2/projects/{ref}/compute/{name}` | 内部端点（实现 `v2-get-a-compute-instance`） |
| `DELETE` | `/v2/projects/{ref}/compute/{name}` | 内部端点（实现 `v2-delete-a-compute-instance`） |
| `POST` | `/v2/projects/{ref}/compute/{name}/deploy` | 内部端点（实现 `v2-deploy-a-compute-instance`） |
| `POST` | `/v2/projects/{ref}/compute/{name}/uploads` | 内部端点（实现 `v2-create-compute-instance-upload`） |

### Project Workers (Alpha)

共 5 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/workers` | **[Alpha] List all workers** — Returns all workers you've previously deployed to the specified project. |
| `GET` | `/v2/projects/{ref}/workers/{name}` | **[Alpha] Retrieve a worker** — Returns a worker along with its instance tally. Poll this after a deploy until `build_state` leaves `building`. |
| `DELETE` | `/v2/projects/{ref}/workers/{name}` | **[Alpha] Delete a worker** — Tombstones the worker. Its instances and image are torn down asynchronously. |
| `POST` | `/v2/projects/{ref}/workers/{name}/uploads` | **[Alpha] Mint a presigned slot for a build-context upload** — PUT the `.tar.gz` build context to the returned `url` before `expires_at`, then deploy with the upload id as `context_upload_id`. The bytes go straight to storage — no management API request carries them. |
| `POST` | `/v2/projects/{ref}/workers/{name}/deploy` | **[Alpha] Deploy a worker** — Creates the worker if it does not exist, building from a context staged through the uploads endpoint. The build runs asynchronously: this answers 202 and the worker reaches `build_state` `active` or `failed` later. |

### Project Notebooks

共 5 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/notebooks` | 内部端点（实现 `v2-list-notebooks`） |
| `POST` | `/v2/projects/{ref}/notebooks` | 内部端点（实现 `v2-create-notebook`） |
| `GET` | `/v2/projects/{ref}/notebooks/{id}` | 内部端点（实现 `v2-get-notebook`） |
| `DELETE` | `/v2/projects/{ref}/notebooks/{id}` | 内部端点（实现 `v2-delete-notebook`） |
| `PATCH` | `/v2/projects/{ref}/notebooks/{id}` | 内部端点（实现 `v2-update-notebook`） |

### Project Private Link

共 4 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/private-link/associations` | **List AWS accounts attached to the project PrivateLink share** |
| `POST` | `/v2/projects/{ref}/private-link/associations` | **Add an AWS account to the project PrivateLink share** — Adds an AWS account to the project's PrivateLink configuration and schedules the AWS resources to be created. |
| `DELETE` | `/v2/projects/{ref}/private-link/associations/aws-account/{aws_account_id}` | **Remove an AWS account from the project PrivateLink share** — Removes an AWS account from the project's PrivateLink configuration (targeting the primary database). Cleans up the associated AWS resources. |
| `DELETE` | `/v2/projects/{ref}/private-link/associations/aws-account/{aws_account_id}/database/{database_identifier}` | **Remove an AWS account from a specific database PrivateLink share** — Removes an AWS account from the project's PrivateLink configuration for the given read replica. Cleans up the associated AWS resources. |

### Project Transfers

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/v2/projects/{ref}/transfers/previews` | **Previews transferring a project to a different organizations, shows eligibility and impact** |
| `POST` | `/v2/projects/{ref}/transfers` | **Transfers a project to a different organization** |

### Project Webhooks

共 10 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/projects/{ref}/webhooks/endpoints` | **List endpoints** — List all Webhook endpoints based on a project's ref or an organization's slug. |
| `POST` | `/v2/projects/{ref}/webhooks/endpoints` | **Create endpoint** — Create new endpoint configuration to subscribe to specific webhook events. |
| `DELETE` | `/v2/projects/{ref}/webhooks/endpoints` | **Delete all endpoints** — Delete all endpoints including all events and deliveries.  Any in-flight webhooks will result in a no-op. |
| `GET` | `/v2/projects/{ref}/webhooks/endpoints/{id}` | **Get endpoint** — Get details of a specific endpoint. |
| `PATCH` | `/v2/projects/{ref}/webhooks/endpoints/{id}` | **Update endpoint** — Update endpoint's configuration. |
| `DELETE` | `/v2/projects/{ref}/webhooks/endpoints/{id}` | **Delete endpoint** — Delete the endpoint including all events and deliveries  Any in-flight webhooks will result in a no-op. |
| `GET` | `/v2/projects/{ref}/webhooks/endpoints/{id}/deliveries` | **List deliveries** — List all deliveries for a specific endpoint in descending order (newest first).  Deliveries which has expired are no longer available and will not be listed. |
| `POST` | `/v2/projects/{ref}/webhooks/endpoints/{id}/test` | **Send test event** — Ingests and schedules a test webhook event to be published out this endpoints.  Which event type to use can be specified in the request body, otherwise it will use any matching type the endpoint is listening for.  The event will contain `is_test: true` in it's payload.  This endpoint is heavy rate-limited to allow for 10 request within 60 seconds. |
| `GET` | `/v2/projects/{ref}/webhooks/deliveries/{id}` | **Get delivery** — Get details of a specific delivery attempt. |
| `POST` | `/v2/projects/{ref}/webhooks/deliveries/{id}/retry` | **Retry delivery** — Retry delivering the same event again.  Automatic retries are not applicable to manual retries - if the delivery fails, there won't be any automatic retries attempted.  This endpoint is heavy rate-limited to allow for 10 request within 60 seconds. |

### Project Advisors

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/v2/projects/{ref}/advisors/run` | **Runs the project advisors with the given names** |

### Organizations

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/organizations/{slug}/projects` | **List projects of an organization** — Returns a cursor-paginated list of projects for the specified organization, including their databases.  Use `page[after]` and `page[before]` to navigate pages and `page[size]` to control the number of projects returned per page. |

### Organizations Members

共 4 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/organizations/{slug}/members` | **List members of an organization** — Returns a cursor-paginated list of organization members including their roles and project-scoped permissions. |
| `PATCH` | `/v2/organizations/{slug}/members/{user_id}/roles` | **Assign or change an organization member role** — Assigns an org-wide role when projects is omitted, or creates a project-scoped assignment when projects is provided. Uses an org-level role template id from GET /v2/organizations/{slug}/roles. Stale role assignments are automatically cleaned up: if a role no longer has any projects, it is deleted; overlapping project assignments in other roles are automatically removed to avoid duplication. |
| `POST` | `/v2/organizations/{slug}/members/invitations` | **Creates organization invitations** — Creates member invitations for an organization. Each invitation can have different role and project scope settings. |
| `DELETE` | `/v2/organizations/{slug}/members/invitations` | **Deletes organization invitations by email** — Bulk delete member invitations for an organization by email address. |

### Organizations Roles

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/organizations/{slug}/roles` | **List roles of an organization** — Returns a list of org-level roles for the organization. |

### Organizations Integrations

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/organizations/{slug}/integrations/github/connections` | **List GitHub connections of an organization** — Returns a cursor-paginated list of the GitHub connections of the organization's projects.  Use `page[after]` and `page[before]` to navigate pages and `page[size]` to control the page size. Paging walks the organization projects, so a page holds at most `page[size]` connections and can hold fewer (or none) when some of its projects are not connected. Follow `links.next` until it is `null` rather than stopping on a short page.  Use `filter[project_ref]` to narrow the list down to a single project. |

### Organizations Webhooks

共 10 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/v2/organizations/{slug}/webhooks/endpoints` | **List endpoints** — List all Webhook endpoints based on a project's ref or an organization's slug. |
| `POST` | `/v2/organizations/{slug}/webhooks/endpoints` | **Create endpoint** — Create new endpoint configuration to subscribe to specific webhook events. |
| `DELETE` | `/v2/organizations/{slug}/webhooks/endpoints` | **Delete all endpoints** — Delete all endpoints including all events and deliveries.  Any in-flight webhooks will result in a no-op. |
| `GET` | `/v2/organizations/{slug}/webhooks/endpoints/{id}` | **Get endpoint** — Get details of a specific endpoint. |
| `PATCH` | `/v2/organizations/{slug}/webhooks/endpoints/{id}` | **Update endpoint** — Update endpoint's configuration. |
| `DELETE` | `/v2/organizations/{slug}/webhooks/endpoints/{id}` | **Delete endpoint** — Delete the endpoint including all events and deliveries  Any in-flight webhooks will result in a no-op. |
| `GET` | `/v2/organizations/{slug}/webhooks/endpoints/{id}/deliveries` | **List deliveries** — List all deliveries for a specific endpoint in descending order (newest first).  Deliveries which has expired are no longer available and will not be listed. |
| `POST` | `/v2/organizations/{slug}/webhooks/endpoints/{id}/test` | **Send test event** — Ingests and schedules a test webhook event to be published out this endpoints.  Which event type to use can be specified in the request body, otherwise it will use any matching type the endpoint is listening for.  The event will contain `is_test: true` in it's payload.  This endpoint is heavy rate-limited to allow for 10 request within 60 seconds. |
| `GET` | `/v2/organizations/{slug}/webhooks/deliveries/{id}` | **Get delivery** — Get details of a specific delivery attempt. |
| `POST` | `/v2/organizations/{slug}/webhooks/deliveries/{id}/retry` | **Retry delivery** — Retry delivering the same event again.  Automatic retries are not applicable to manual retries - if the delivery fails, there won't be any automatic retries attempted.  This endpoint is heavy rate-limited to allow for 10 request within 60 seconds. |

---

## 4. 内部平台 API —— `/platform`（296 路径 / 392 操作）

平台后端内部管理面（Studio 全量消费，不对外公开文档）。按路径前缀分组。


### projects

共 93 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/projects` | Creates a project |
| `GET` | `/platform/projects` | 内部端点（实现 `ProjectsController_getProjects`） |
| `GET` | `/platform/projects/{ref}` | Gets a specific project that belongs to the authenticated user |
| `DELETE` | `/platform/projects/{ref}` | Deletes the given project |
| `PATCH` | `/platform/projects/{ref}` | Updates the given project |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/api_keys.last_used.otel` | Gets the project's last-used API keys |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/auth.metrics` | Gets a project's auth metrics |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/functions.combined-stats` | Gets a project's function combined statistics |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/functions.req-stats` | Gets a project's function request statistics |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/functions.resource-usage` | Gets a project's function resource usage |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/logs` | Gets project's logs from the unified logs stream |
| `POST` | `/platform/projects/{ref}/analytics/endpoints/logs` | Gets project's logs from the unified logs stream |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/logs.all` | 内部端点（实现 `LogsController_getProjectLogsViaGet`） |
| `POST` | `/platform/projects/{ref}/analytics/endpoints/logs.all` | 内部端点（实现 `LogsController_getProjectLogsViaPost`） |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/logs.all.otel` | 内部端点（实现 `LogsController_getProjectLogsOtelViaGet`） |
| `POST` | `/platform/projects/{ref}/analytics/endpoints/logs.all.otel` | 内部端点（实现 `LogsController_getProjectLogsOtelViaPost`） |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/project.metrics` | Gets a project's metrics |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/project.metrics.otel` | Gets a project's metrics from the OTel/ClickHouse backend |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/service-health` | Gets project's service health based on log levels |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/usage.api-counts` | Gets project's usage api counts |
| `GET` | `/platform/projects/{ref}/analytics/endpoints/usage.api-requests-count` | Gets project's usage api requests count |
| `GET` | `/platform/projects/{ref}/analytics/log-drains` | Lists all log drains |
| `POST` | `/platform/projects/{ref}/analytics/log-drains` | Create a log drain |
| `PUT` | `/platform/projects/{ref}/analytics/log-drains/{token}` | Update a log drain |
| `DELETE` | `/platform/projects/{ref}/analytics/log-drains/{token}` | Delete a log drain |
| `PATCH` | `/platform/projects/{ref}/analytics/log-drains/{token}` | Patch a log drain |
| `POST` | `/platform/projects/{ref}/analytics/log-drains/{token}/test` | Test a log drain connection |
| `GET` | `/platform/projects/{ref}/analytics/metrics` | 内部端点（实现 `scrape-project-metrics`） |
| `POST` | `/platform/projects/{ref}/api-keys/temporary` | Create a temporary API key |
| `POST` | `/platform/projects/{ref}/api/graphql` | Queries project Graphql |
| `GET` | `/platform/projects/{ref}/api/rest` | Gets project OpenApi |
| `GET` | `/platform/projects/{ref}/billing/addons` | Gets project addons |
| `POST` | `/platform/projects/{ref}/billing/addons` | Updates project addon |
| `DELETE` | `/platform/projects/{ref}/billing/addons/{addon_variant}` | Removes project addon |
| `PATCH` | `/platform/projects/{ref}/config/pgbouncer` | Updates project's pgbouncer config |
| `GET` | `/platform/projects/{ref}/config/pgbouncer` | 内部端点（实现 `PgbouncerConfigController_getPgbouncerConfig`） |
| `GET` | `/platform/projects/{ref}/config/pgbouncer/status` | Gets project's pgbouncer status |
| `GET` | `/platform/projects/{ref}/config/postgrest` | Gets project's postgrest config |
| `PATCH` | `/platform/projects/{ref}/config/postgrest` | Updates project's postgrest config |
| `GET` | `/platform/projects/{ref}/config/realtime` | Gets realtime configuration |
| `PATCH` | `/platform/projects/{ref}/config/realtime` | Updates realtime configuration |
| `POST` | `/platform/projects/{ref}/config/realtime/shutdown` | Shutdowns realtime connections for a project |
| `PATCH` | `/platform/projects/{ref}/config/secrets` | Updates project's secrets config |
| `GET` | `/platform/projects/{ref}/config/secrets/update-status` | Gets the last JWT secret update status |
| `GET` | `/platform/projects/{ref}/config/storage` | Gets project's storage config |
| `PATCH` | `/platform/projects/{ref}/config/storage` | Updates project's storage config |
| `GET` | `/platform/projects/{ref}/config/supavisor` | Gets project's supavisor config |
| `GET` | `/platform/projects/{ref}/content` | Gets project's content |
| `PUT` | `/platform/projects/{ref}/content` | Updates project's content |
| `DELETE` | `/platform/projects/{ref}/content` | Deletes project's contents |
| `GET` | `/platform/projects/{ref}/content/count` | Gets the user's content counts |
| `GET` | `/platform/projects/{ref}/content/folders` | Gets project's content root folder |
| `POST` | `/platform/projects/{ref}/content/folders` | Creates project's content folder |
| `DELETE` | `/platform/projects/{ref}/content/folders` | Deletes project's content folders |
| `GET` | `/platform/projects/{ref}/content/folders/{id}` | Gets project's content folder |
| `PATCH` | `/platform/projects/{ref}/content/folders/{id}` | Updates project's content folder |
| `GET` | `/platform/projects/{ref}/content/item/{id}` | Gets project's content by the given id |
| `GET` | `/platform/projects/{ref}/daily-stats` | Gets daily project stats |
| `GET` | `/platform/projects/{ref}/databases` | Gets non-removed databases of a specified project |
| `GET` | `/platform/projects/{ref}/databases-statuses` | Gets statuses of databases within a project |
| `PATCH` | `/platform/projects/{ref}/db-password` | Updates the database password |
| `GET` | `/platform/projects/{ref}/disk` | Get database disk attributes |
| `POST` | `/platform/projects/{ref}/disk` | Modify database disk |
| `GET` | `/platform/projects/{ref}/disk/custom-config` | Gets disk autoscale config |
| `POST` | `/platform/projects/{ref}/disk/custom-config` | Updates disk autoscale config |
| `GET` | `/platform/projects/{ref}/disk/util` | Get disk utilization |
| `GET` | `/platform/projects/{ref}/infra-monitoring` | Gets project's usage metrics |
| `GET` | `/platform/projects/{ref}/load-balancers` | Gets non-removed databases of a specified project |
| `GET` | `/platform/projects/{ref}/members` | Gets the list of users with access to the project |
| `GET` | `/platform/projects/{ref}/notifications/advisor/exceptions` | List advisor notification exceptions |
| `POST` | `/platform/projects/{ref}/notifications/advisor/exceptions` | Create advisor notification exceptions |
| `DELETE` | `/platform/projects/{ref}/notifications/advisor/exceptions` | Deletes advisor notification exceptions |
| `PATCH` | `/platform/projects/{ref}/notifications/advisor/exceptions/{id}` | Updates advisor notification exceptions |
| `POST` | `/platform/projects/{ref}/pause` | Pauses the project |
| `GET` | `/platform/projects/{ref}/pause/status` | Gets the latest pause event for a project if a project is paused |
| `GET` | `/platform/projects/{ref}/privatelink/associations` | Get AWS accounts attached to PrivateLink share for the project. |
| `POST` | `/platform/projects/{ref}/privatelink/associations/aws-account` | 内部端点（实现 `ProjectPrivateLinkController_addAwsAccountToPrivateLink`） |
| `DELETE` | `/platform/projects/{ref}/privatelink/associations/aws-account/{aws_account_id}` | 内部端点（实现 `ProjectPrivateLinkController_removeAwsAccountFromPrivateLink`） |
| `DELETE` | `/platform/projects/{ref}/privatelink/associations/aws-account/{aws_account_id}/database/{database_identifier}` | 内部端点（实现 `ProjectPrivateLinkController_removeAwsAccountFromPrivateLinkForDatabase`） |
| `POST` | `/platform/projects/{ref}/resize` | Resize database disk |
| `POST` | `/platform/projects/{ref}/restart` | Restarts project |
| `POST` | `/platform/projects/{ref}/restart-services` | Restarts given services |
| `POST` | `/platform/projects/{ref}/restore` | Unpauses project |
| `GET` | `/platform/projects/{ref}/restore/versions` | Retrieves versions to which a project can be restored |
| `GET` | `/platform/projects/{ref}/run-lints` | Run project lints |
| `GET` | `/platform/projects/{ref}/service-versions` | Gets service versions for a specific project |
| `GET` | `/platform/projects/{ref}/settings` | Gets project's settings |
| `PATCH` | `/platform/projects/{ref}/settings/sensitivity` | Updates the given project sensitivity |
| `GET` | `/platform/projects/{ref}/status` | Gets project's status |
| `POST` | `/platform/projects/{ref}/transfer` | Transfers a project to a different organization. |
| `POST` | `/platform/projects/{ref}/transfer/preview` | Previews transferring a project to a different organizations, shows eligibility and impact. |
| `POST` | `/platform/projects/{ref}/wake` | Wakes a specific project that belongs to the authenticated user |
| `GET` | `/platform/projects/available-regions` | Gets the list of available regions that can be used for a new project |

### organizations

共 90 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/organizations` | Gets user's organizations |
| `POST` | `/platform/organizations` | Creates an organization |
| `GET` | `/platform/organizations/{slug}` | Gets a specific organization that belongs to the authenticated user |
| `DELETE` | `/platform/organizations/{slug}` | Deletes organization |
| `PATCH` | `/platform/organizations/{slug}` | Updates organization |
| `GET` | `/platform/organizations/{slug}/analytics/audit-log-drains` | Lists all audit log drains for an organization |
| `POST` | `/platform/organizations/{slug}/analytics/audit-log-drains` | Create an audit log drain |
| `PUT` | `/platform/organizations/{slug}/analytics/audit-log-drains/{token}` | Update an audit log drain |
| `DELETE` | `/platform/organizations/{slug}/analytics/audit-log-drains/{token}` | Delete an audit log drain |
| `PATCH` | `/platform/organizations/{slug}/analytics/audit-log-drains/{token}` | Patch an audit log drain |
| `POST` | `/platform/organizations/{slug}/analytics/audit-log-drains/{token}/test` | Test an audit log drain connection |
| `GET` | `/platform/organizations/{slug}/apps` | List platform apps for the given organization |
| `POST` | `/platform/organizations/{slug}/apps` | Create new platform app |
| `GET` | `/platform/organizations/{slug}/apps/{app_id}` | Get organization platform app by the given id |
| `DELETE` | `/platform/organizations/{slug}/apps/{app_id}` | Delete platform app |
| `PATCH` | `/platform/organizations/{slug}/apps/{app_id}` | Update platform app |
| `GET` | `/platform/organizations/{slug}/apps/{app_id}/signing-keys` | List signing keys for the given platform app |
| `POST` | `/platform/organizations/{slug}/apps/{app_id}/signing-keys` | Create a signing key for the given platform app |
| `DELETE` | `/platform/organizations/{slug}/apps/{app_id}/signing-keys/{key_id}` | Delete a signing key for the given platform app |
| `GET` | `/platform/organizations/{slug}/apps/installations` | List platform app installations for the given organization |
| `POST` | `/platform/organizations/{slug}/apps/installations` | Install a platform app to this organization |
| `GET` | `/platform/organizations/{slug}/apps/installations/{installation_id}` | Get platform app installation with the given id |
| `DELETE` | `/platform/organizations/{slug}/apps/installations/{installation_id}` | Uninstall the given platform app installation |
| `PATCH` | `/platform/organizations/{slug}/apps/installations/{installation_id}` | Update platform app installation permissions |
| `GET` | `/platform/organizations/{slug}/audit` | Gets an organization's audit logs |
| `POST` | `/platform/organizations/{slug}/available-versions` | Retrieves a list of available Postgres versions available to the organization |
| `GET` | `/platform/organizations/{slug}/billing/credits/balance` | Gets the current credit balance |
| `POST` | `/platform/organizations/{slug}/billing/credits/preview` | Preview for credit top-up |
| `POST` | `/platform/organizations/{slug}/billing/credits/redeem` | Redeems a credit code |
| `POST` | `/platform/organizations/{slug}/billing/credits/top-up` | Tops up the credit balance |
| `GET` | `/platform/organizations/{slug}/billing/invoices` | Gets invoices for the given organization |
| `HEAD` | `/platform/organizations/{slug}/billing/invoices` | Gets the total count of invoices for the given organization |
| `GET` | `/platform/organizations/{slug}/billing/invoices/{invoice_id}` | Gets invoice with the given invoice ID |
| `GET` | `/platform/organizations/{slug}/billing/invoices/{invoice_id}/payment-link` | Gets the payment link to manually pay the given invoice |
| `GET` | `/platform/organizations/{slug}/billing/invoices/{invoice_id}/receipt` | Get the receipt PDF URL for a paid invoice |
| `GET` | `/platform/organizations/{slug}/billing/invoices/upcoming` | Gets the upcoming invoice |
| `GET` | `/platform/organizations/{slug}/billing/plans` | Gets subscription Plans |
| `GET` | `/platform/organizations/{slug}/billing/subscription` | Gets the current subscription |
| `PUT` | `/platform/organizations/{slug}/billing/subscription` | Updates subscription |
| `POST` | `/platform/organizations/{slug}/billing/subscription/confirm` | Confirm subscription change |
| `POST` | `/platform/organizations/{slug}/billing/subscription/preview` | Preview subscription changes |
| `POST` | `/platform/organizations/{slug}/billing/upgrade-request` | Request organization upgrade - notifies billing owners |
| `PUT` | `/platform/organizations/{slug}/cloud-marketplace/link` | Makes an existing organization being billed by AWS Marketplace |
| `GET` | `/platform/organizations/{slug}/cloud-marketplace/redirect` | Gets the AWS Marketplace redirect url |
| `GET` | `/platform/organizations/{slug}/customer` | Gets the Billing customer |
| `PUT` | `/platform/organizations/{slug}/customer` | Updates the billing customer |
| `GET` | `/platform/organizations/{slug}/documents/iso27001-certificate` | Get ISO 27001 certificate URL |
| `GET` | `/platform/organizations/{slug}/documents/soc2-type-2-report` | Get SOC2 Type 2 report URL |
| `GET` | `/platform/organizations/{slug}/documents/standard-security-questionnaire` | Get standard security questionnaire URL |
| `GET` | `/platform/organizations/{slug}/entitlements` | 内部端点（实现 `OrganizationEntitlementsController_getEntitlements`） |
| `GET` | `/platform/organizations/{slug}/members` | Gets organization's members |
| `DELETE` | `/platform/organizations/{slug}/members/{gotrue_id}` | Removes organization member |
| `PATCH` | `/platform/organizations/{slug}/members/{gotrue_id}` | Assign organization member with new role |
| `PUT` | `/platform/organizations/{slug}/members/{gotrue_id}/roles/{role_id}` | Update organization member role |
| `DELETE` | `/platform/organizations/{slug}/members/{gotrue_id}/roles/{role_id}` | Removes organization member role |
| `GET` | `/platform/organizations/{slug}/members/invitations` | Gets organization invitations |
| `POST` | `/platform/organizations/{slug}/members/invitations` | Creates organization invitation |
| `DELETE` | `/platform/organizations/{slug}/members/invitations/{id}` | Deletes organization invitation with given id |
| `GET` | `/platform/organizations/{slug}/members/invitations/{token}` | Gets organization invitation by token |
| `POST` | `/platform/organizations/{slug}/members/invitations/{token}` | Accepts organization invitation by token |
| `GET` | `/platform/organizations/{slug}/members/mfa/enforcement` | Gets organization MFA enforcement state |
| `PATCH` | `/platform/organizations/{slug}/members/mfa/enforcement` | Update organization MFA enforcement state |
| `GET` | `/platform/organizations/{slug}/members/reached-free-project-limit` | Gets organization members who have reached their free project limit |
| `GET` | `/platform/organizations/{slug}/oauth/apps` | List published or authorized oauth apps |
| `POST` | `/platform/organizations/{slug}/oauth/apps` | Create an oauth app |
| `GET` | `/platform/organizations/{slug}/oauth/apps/{app_id}/client-secrets` | List oauth app client secrets |
| `POST` | `/platform/organizations/{slug}/oauth/apps/{app_id}/client-secrets` | Create oauth app client secret |
| `DELETE` | `/platform/organizations/{slug}/oauth/apps/{app_id}/client-secrets/{secret_id}` | Remove oauth app client secret |
| `PUT` | `/platform/organizations/{slug}/oauth/apps/{id}` | Update an oauth app |
| `DELETE` | `/platform/organizations/{slug}/oauth/apps/{id}` | Remove a published oauth app |
| `POST` | `/platform/organizations/{slug}/oauth/apps/{id}/revoke` | Revoke an authorized oauth app |
| `POST` | `/platform/organizations/{slug}/oauth/authorizations/{id}` | 内部端点（实现 `OrganizationOAuthAuthorizationsController_approveAuthorizationRequest`） |
| `DELETE` | `/platform/organizations/{slug}/oauth/authorizations/{id}` | 内部端点（实现 `OrganizationOAuthAuthorizationsController_declineAuthorizationRequest`） |
| `GET` | `/platform/organizations/{slug}/payments` | Gets Stripe payment methods for the given organization |
| `DELETE` | `/platform/organizations/{slug}/payments` | Detach payment method with the given card ID |
| `PUT` | `/platform/organizations/{slug}/payments/default` | Mark given payment method as default for organization |
| `POST` | `/platform/organizations/{slug}/payments/setup-intent` | Sets up a payment method |
| `GET` | `/platform/organizations/{slug}/projects` | 内部端点（实现 `OrganizationProjectsController_getOrganizationProjects`） |
| `GET` | `/platform/organizations/{slug}/roles` | Gets the given organization's roles with their corresponding projects |
| `GET` | `/platform/organizations/{slug}/sso` | Get the organization's SSO Provider |
| `PUT` | `/platform/organizations/{slug}/sso` | Update the organization's SSO Provider |
| `POST` | `/platform/organizations/{slug}/sso` | Create the organization's SSO Provider |
| `DELETE` | `/platform/organizations/{slug}/sso` | Delete the organization's SSO Provider |
| `GET` | `/platform/organizations/{slug}/tax-ids` | Gets the given organization's tax ID |
| `GET` | `/platform/organizations/{slug}/usage` | Gets usage stats |
| `GET` | `/platform/organizations/{slug}/usage/daily` | Gets daily aggregated usage stats |
| `POST` | `/platform/organizations/cloud-marketplace` | Creates organization billed by AWS Marketplace |
| `POST` | `/platform/organizations/confirm-subscription` | Confirm subscription change and apply pending changes |
| `POST` | `/platform/organizations/onboarding-survey` | Submit onboarding survey for a newly created organization |
| `POST` | `/platform/organizations/preview-creation` | Preview tax breakdown for organization creation |

### storage

共 44 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/storage/{ref}/analytics-buckets` | Gets list of analytics buckets |
| `POST` | `/platform/storage/{ref}/analytics-buckets` | Create an analytics bucket |
| `DELETE` | `/platform/storage/{ref}/analytics-buckets/{id}` | Deletes an analytics bucket |
| `GET` | `/platform/storage/{ref}/analytics-buckets/{id}/namespaces` | Gets list of namespaces from a bucket |
| `POST` | `/platform/storage/{ref}/analytics-buckets/{id}/namespaces` | Create a namespace within a bucket |
| `DELETE` | `/platform/storage/{ref}/analytics-buckets/{id}/namespaces/{namespace}` | Drop a namespace within an analytics bucket |
| `GET` | `/platform/storage/{ref}/analytics-buckets/{id}/namespaces/{namespace}/tables` | Gets list of tables from a namespace |
| `POST` | `/platform/storage/{ref}/analytics-buckets/{id}/namespaces/{namespace}/tables` | Create a table within a namespace |
| `DELETE` | `/platform/storage/{ref}/analytics-buckets/{id}/namespaces/{namespace}/tables/{table}` | Drop a table within a namespace |
| `GET` | `/platform/storage/{ref}/archive` | Gets project storage archive |
| `POST` | `/platform/storage/{ref}/archive` | Creates project storage archive |
| `GET` | `/platform/storage/{ref}/buckets` | Gets list of buckets |
| `POST` | `/platform/storage/{ref}/buckets` | Create bucket |
| `GET` | `/platform/storage/{ref}/buckets/{id}` | Gets bucket |
| `DELETE` | `/platform/storage/{ref}/buckets/{id}` | Deletes bucket |
| `PATCH` | `/platform/storage/{ref}/buckets/{id}` | Updates bucket |
| `POST` | `/platform/storage/{ref}/buckets/{id}/empty` | Removes all objects inside a single bucket. |
| `GET` | `/platform/storage/{ref}/buckets/{id}/lifecycle` | 内部端点（实现 `StorageBucketLifecycleController_getBucketLifecycle`） |
| `PUT` | `/platform/storage/{ref}/buckets/{id}/lifecycle` | 内部端点（实现 `StorageBucketLifecycleController_updateBucketLifecycle`） |
| `DELETE` | `/platform/storage/{ref}/buckets/{id}/lifecycle` | 内部端点（实现 `StorageBucketLifecycleController_deleteBucketLifecycle`） |
| `DELETE` | `/platform/storage/{ref}/buckets/{id}/objects` | Deletes objects |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/copy` | Copys object |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/list` | Gets list of objects with the given bucket |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/list-v2` | Gets list of objects with the given bucket |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/move` | Move object |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/public-url` | Creates URL for an asset in a public bucket |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/sign` | Creates a signed URL |
| `POST` | `/platform/storage/{ref}/buckets/{id}/objects/sign-multi` | Gets multiple signed URLs |
| `POST` | `/platform/storage/{ref}/cdn/purge-bucket` | Purges CDN cache for an entire bucket |
| `POST` | `/platform/storage/{ref}/cdn/purge-object` | Purges CDN cache for a single object |
| `GET` | `/platform/storage/{ref}/credentials` | Gets project storage credentials |
| `POST` | `/platform/storage/{ref}/credentials` | Creates project storage credential |
| `DELETE` | `/platform/storage/{ref}/credentials/{id}` | Deletes project storage credential |
| `GET` | `/platform/storage/{ref}/jwks` | Lists project storage jwks |
| `PUT` | `/platform/storage/{ref}/jwks/{kid}` | Activates or deactivates a jwk |
| `POST` | `/platform/storage/{ref}/jwks/url-signing/standby` | Creates a standby url signing jwk |
| `POST` | `/platform/storage/{ref}/jwks/url-signing/standby/{kid}/swap` | Swaps a standby url signing jwk into the active url signing jwk |
| `GET` | `/platform/storage/{ref}/vector-buckets` | Gets list of vector buckets |
| `POST` | `/platform/storage/{ref}/vector-buckets` | Create vector bucket |
| `GET` | `/platform/storage/{ref}/vector-buckets/{id}` | Gets bucket |
| `DELETE` | `/platform/storage/{ref}/vector-buckets/{id}` | Deletes bucket |
| `GET` | `/platform/storage/{ref}/vector-buckets/{id}/indexes` | Gets bucket indexes |
| `POST` | `/platform/storage/{ref}/vector-buckets/{id}/indexes` | Create index in vector bucket |
| `DELETE` | `/platform/storage/{ref}/vector-buckets/{id}/indexes/{indexName}` | Deletes bucket index |

### replication

共 39 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/replication/{ref}/destinations` | 内部端点（实现 `DestinationsController_getDestinations`） |
| `POST` | `/platform/replication/{ref}/destinations` | 内部端点（实现 `DestinationsController_createDestination`） |
| `POST` | `/platform/replication/{ref}/destinations-pipelines` | 内部端点（实现 `DestinationsPipelinesController_createDestinationPipeline`） |
| `POST` | `/platform/replication/{ref}/destinations-pipelines/{destination_id}/{pipeline_id}` | 内部端点（实现 `DestinationsPipelinesController_updateDestinationPipeline`） |
| `DELETE` | `/platform/replication/{ref}/destinations-pipelines/{destination_id}/{pipeline_id}` | 内部端点（实现 `DestinationsPipelinesController_deleteDestinationPipeline`） |
| `GET` | `/platform/replication/{ref}/destinations/{destination_id}` | 内部端点（实现 `DestinationsController_getDestination`） |
| `POST` | `/platform/replication/{ref}/destinations/{destination_id}` | 内部端点（实现 `DestinationsController_updateDestination`） |
| `DELETE` | `/platform/replication/{ref}/destinations/{destination_id}` | 内部端点（实现 `DestinationsController_deleteDestination`） |
| `POST` | `/platform/replication/{ref}/destinations/validate` | 内部端点（实现 `DestinationsController_validateDestination`） |
| `GET` | `/platform/replication/{ref}/pipelines` | 内部端点（实现 `PipelinesController_getPipelines`） |
| `POST` | `/platform/replication/{ref}/pipelines` | 内部端点（实现 `PipelinesController_createPipeline`） |
| `GET` | `/platform/replication/{ref}/pipelines/{pipeline_id}` | 内部端点（实现 `PipelinesController_getPipeline`） |
| `POST` | `/platform/replication/{ref}/pipelines/{pipeline_id}` | 内部端点（实现 `PipelinesController_updatePipeline`） |
| `DELETE` | `/platform/replication/{ref}/pipelines/{pipeline_id}` | 内部端点（实现 `PipelinesController_deletePipeline`） |
| `GET` | `/platform/replication/{ref}/pipelines/{pipeline_id}/replication-status` | 内部端点（实现 `PipelinesController_getPipelineReplicationStatus`） |
| `POST` | `/platform/replication/{ref}/pipelines/{pipeline_id}/restart` | 内部端点（实现 `PipelinesController_restartPipeline`） |
| `POST` | `/platform/replication/{ref}/pipelines/{pipeline_id}/rollback-tables` | 内部端点（实现 `PipelinesController_rollbackTables`） |
| `POST` | `/platform/replication/{ref}/pipelines/{pipeline_id}/start` | 内部端点（实现 `PipelinesController_startPipeline`） |
| `GET` | `/platform/replication/{ref}/pipelines/{pipeline_id}/status` | 内部端点（实现 `PipelinesController_getPipelineStatus`） |
| `POST` | `/platform/replication/{ref}/pipelines/{pipeline_id}/stop` | 内部端点（实现 `PipelinesController_stopPipeline`） |
| `GET` | `/platform/replication/{ref}/pipelines/{pipeline_id}/version` | 内部端点（实现 `PipelinesController_getPipelineVersion`） |
| `POST` | `/platform/replication/{ref}/pipelines/{pipeline_id}/version` | 内部端点（实现 `PipelinesController_updatePipelineVersion`） |
| `POST` | `/platform/replication/{ref}/pipelines/validate` | 内部端点（实现 `PipelinesController_validatePipeline`） |
| `GET` | `/platform/replication/{ref}/sources` | 内部端点（实现 `SourcesController_getSources`） |
| `POST` | `/platform/replication/{ref}/sources` | 内部端点（实现 `SourcesController_createSource`） |
| `GET` | `/platform/replication/{ref}/sources/{source_id}/publications` | 内部端点（实现 `SourcesController_getPublications`） |
| `POST` | `/platform/replication/{ref}/sources/{source_id}/publications` | 内部端点（实现 `SourcesController_createPublication`） |
| `POST` | `/platform/replication/{ref}/sources/{source_id}/publications/{publication_name}` | 内部端点（实现 `SourcesController_updatePublication`） |
| `DELETE` | `/platform/replication/{ref}/sources/{source_id}/publications/{publication_name}` | 内部端点（实现 `SourcesController_deletePublication`） |
| `GET` | `/platform/replication/{ref}/sources/{source_id}/publications/{publication_name}/cost-estimate` | 内部端点（实现 `SourcesController_getCostEstimate`） |
| `GET` | `/platform/replication/{ref}/sources/{source_id}/tables` | 内部端点（实现 `SourcesController_getTables`） |
| `DELETE` | `/platform/replication/{ref}/tenants` | 内部端点（实现 `TenantsController_deleteTenant`） |
| `POST` | `/platform/replication/{ref}/tenants-sources` | 内部端点（实现 `TenantsSourcesController_createTenantSource`） |
| `GET` | `/platform/replication/v2/{ref}/sources/{source_id}/publications` | 内部端点（实现 `V2SourcesController_getPublications`） |
| `GET` | `/platform/replication/v2/{ref}/sources/{source_id}/publications/{publication_name}` | 内部端点（实现 `V2SourcesController_getPublication`） |
| `PUT` | `/platform/replication/v2/{ref}/sources/{source_id}/publications/{publication_name}` | 内部端点（实现 `V2SourcesController_putPublication`） |
| `DELETE` | `/platform/replication/v2/{ref}/sources/{source_id}/publications/{publication_name}` | 内部端点（实现 `V2SourcesController_deletePublication`） |
| `GET` | `/platform/replication/v2/{ref}/sources/{source_id}/tables` | 内部端点（实现 `V2SourcesController_getTables`） |
| `GET` | `/platform/replication/v2/{ref}/sources/{source_id}/tables/{table_id}/columns` | 内部端点（实现 `V2SourcesController_getTableColumns`） |

### auth

共 14 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/auth/{ref}/config` | Gets Auth config |
| `PATCH` | `/platform/auth/{ref}/config` | Updates Auth config |
| `PATCH` | `/platform/auth/{ref}/config/hooks` | Updates Auth config hooks |
| `POST` | `/platform/auth/{ref}/invite` | Sends an invite to the given email |
| `POST` | `/platform/auth/{ref}/magiclink` | Sends a magic link to the given email |
| `POST` | `/platform/auth/{ref}/otp` | Sends an OTP to the given phone number |
| `POST` | `/platform/auth/{ref}/recover` | Sends a recovery email to the given email |
| `GET` | `/platform/auth/{ref}/templates/{template}` | Gets Auth template |
| `POST` | `/platform/auth/{ref}/templates/{template}/reset` | Resets Auth template |
| `POST` | `/platform/auth/{ref}/users` | Creates user |
| `DELETE` | `/platform/auth/{ref}/users/{id}` | Delete user with given ID |
| `PATCH` | `/platform/auth/{ref}/users/{id}` | Updates user with given ID |
| `DELETE` | `/platform/auth/{ref}/users/{id}/factors` | Delete all factors associated to a user |
| `POST` | `/platform/auth/{ref}/validate/spam` | Validate spam based on the given email content |

### integrations

共 26 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/integrations` | Gets user's integrations |
| `GET` | `/platform/integrations/{slug}` | Gets integration with the given organization slug |
| `GET` | `/platform/integrations/github/authorization` | Get GitHub authorization |
| `POST` | `/platform/integrations/github/authorization` | 内部端点（实现 `GitHubAuthorizationsController_createGitHubAuthorization`） |
| `DELETE` | `/platform/integrations/github/authorization` | 内部端点（实现 `GitHubAuthorizationsController_removeGitHubAuthorization`） |
| `GET` | `/platform/integrations/github/connections` | List organization GitHub connections |
| `POST` | `/platform/integrations/github/connections` | Connects a GitHub project to a supabase project |
| `DELETE` | `/platform/integrations/github/connections/{connection_id}` | Deletes github project connection |
| `PATCH` | `/platform/integrations/github/connections/{connection_id}` | Updates a GitHub connection for a supabase project |
| `GET` | `/platform/integrations/github/connections/{connection_id}/config` | 内部端点（实现 `GitHubConnectionsController_getGitHubConnectionConfig`） |
| `GET` | `/platform/integrations/github/repositories` | Gets GitHub repositories for user |
| `GET` | `/platform/integrations/github/repositories/{repository_id}/branches` | List GitHub repository branches |
| `GET` | `/platform/integrations/github/repositories/{repository_id}/branches/{branch_name}` | 内部端点（实现 `GitHubRepositoriesController_getRepository`） |
| `GET` | `/platform/integrations/partners/{ref}` | Lists installed marketplace integrations for the given project. |
| `GET` | `/platform/integrations/partners/{ref}/{listing_slug}` | Gets the installation status of the given marketplace integration for the given project. |
| `POST` | `/platform/integrations/partners/{ref}/{listing_slug}` | Creates a partner integration and returns the redirect URL |
| `GET` | `/platform/integrations/private-link/{slug}` | Get organization's PrivateLink configuration. |
| `PUT` | `/platform/integrations/private-link/{slug}` | Update organization's PrivateLink configuration. |
| `POST` | `/platform/integrations/vercel` | 内部端点（实现 `VercelIntegrationController_createVercelIntegration`） |
| `POST` | `/platform/integrations/vercel/connections` | Connects a Vercel project to a supabase project |
| `DELETE` | `/platform/integrations/vercel/connections/{connection_id}` | Deletes vercel project connection |
| `PATCH` | `/platform/integrations/vercel/connections/{connection_id}` | Updates a Vercel connection for a supabase project |
| `POST` | `/platform/integrations/vercel/connections/{connection_id}/enable-jit-access` | Enables JIT database access for a supabase project with given connection id |
| `POST` | `/platform/integrations/vercel/connections/{connection_id}/sync-envs` | Syncs supabase project envs with given connection id |
| `GET` | `/platform/integrations/vercel/connections/project/{ref}` | Gets all Vercel integrations (regular and marketplace) with their connections for a given project |
| `GET` | `/platform/integrations/vercel/projects/{organization_integration_id}` | Gets vercel projects with the given organization integration id |

### database

共 11 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/database/{ref}/backups` | Gets project backups |
| `POST` | `/platform/database/{ref}/backups/download` | Download project backup |
| `GET` | `/platform/database/{ref}/backups/downloadable-backups` | Gets backups that might be downloadable, but potentially not restorable. |
| `POST` | `/platform/database/{ref}/backups/enable-physical-backups` | Enable usage of physical backups |
| `POST` | `/platform/database/{ref}/backups/pitr` | Restore project to a previous point in time |
| `POST` | `/platform/database/{ref}/backups/restore` | Restore project backup |
| `POST` | `/platform/database/{ref}/backups/restore-physical` | Restore project with a physical backup |
| `GET` | `/platform/database/{ref}/clone` | List valid backups to clone from |
| `POST` | `/platform/database/{ref}/clone` | Clone the current project from a backup |
| `GET` | `/platform/database/{ref}/clone/status` | Retrieve the current status of an existing cloning process |
| `POST` | `/platform/database/{ref}/hook-enable` | Enables Database Webhooks on the project |

### pg-meta

共 11 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/pg-meta/{ref}/column-privileges` | Retrieve column privileges |
| `GET` | `/platform/pg-meta/{ref}/extensions` | Gets project pg.extensions |
| `GET` | `/platform/pg-meta/{ref}/foreign-tables` | Retrieve database foreign tables |
| `GET` | `/platform/pg-meta/{ref}/materialized-views` | Retrieve database materialized views |
| `GET` | `/platform/pg-meta/{ref}/policies` | Gets project pg.policies |
| `GET` | `/platform/pg-meta/{ref}/publications` | Gets project pg.publications |
| `POST` | `/platform/pg-meta/{ref}/query` | Run sql query |
| `GET` | `/platform/pg-meta/{ref}/tables` | Gets project pg.tables or pg.table with the given ID |
| `GET` | `/platform/pg-meta/{ref}/triggers` | Gets project pg.triggers |
| `GET` | `/platform/pg-meta/{ref}/types` | Gets project pg.types |
| `GET` | `/platform/pg-meta/{ref}/views` | Retrieve database views |

### warehouse

共 8 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/warehouse/{ref}/catalog` | 内部端点（实现 `WarehouseController_getCatalog`） |
| `POST` | `/platform/warehouse/{ref}/catalog` | 内部端点（实现 `WarehouseController_updateCatalog`） |
| `POST` | `/platform/warehouse/{ref}/refresh-schema` | 内部端点（实现 `WarehouseController_refreshSchema`） |
| `POST` | `/platform/warehouse/{ref}/setup` | 内部端点（实现 `WarehouseController_setup`） |
| `GET` | `/platform/warehouse/{ref}/setup-status` | 内部端点（实现 `WarehouseController_getSetupStatus`） |
| `GET` | `/platform/warehouse/{ref}/tables` | 内部端点（实现 `WarehouseController_getTables`） |
| `DELETE` | `/platform/warehouse/{ref}/tables/{schema}/{name}` | 内部端点（实现 `WarehouseController_detachTable`） |
| `GET` | `/platform/warehouse/{ref}/tables/{schema}/{name}/snapshots` | 内部端点（实现 `WarehouseController_getTableSnapshots`） |

### telemetry

共 8 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/telemetry/event` | Sends analytics server event |
| `GET` | `/platform/telemetry/feature-flags` | Call feature flags |
| `POST` | `/platform/telemetry/feature-flags/track` | Track feature flag called |
| `POST` | `/platform/telemetry/groups/identify` | Send analytics group identify event |
| `POST` | `/platform/telemetry/groups/reset` | Send analytics group reset event |
| `POST` | `/platform/telemetry/identify` | Send analytics identify event |
| `POST` | `/platform/telemetry/reset` | Reset analytics |
| `GET` | `/platform/telemetry/stream` | Stream telemetry events (local dev only) |

### profile

共 14 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/profile` | Gets the user's profile |
| `POST` | `/platform/profile` | Creates user's profile |
| `PATCH` | `/platform/profile` | Updates user's profile |
| `GET` | `/platform/profile/access-tokens` | Gets the user's access tokens |
| `POST` | `/platform/profile/access-tokens` | Creates a new access token |
| `GET` | `/platform/profile/access-tokens/{id}` | Gets the access token with the given ID |
| `DELETE` | `/platform/profile/access-tokens/{id}` | Deletes the access token with the given ID |
| `GET` | `/platform/profile/audit` | Gets a user's audit logs |
| `POST` | `/platform/profile/audit-login` | Logged into account |
| `GET` | `/platform/profile/permissions` | Gets all the user's permissions |
| `GET` | `/platform/profile/scoped-access-tokens` | Gets the user's scoped access tokens |
| `POST` | `/platform/profile/scoped-access-tokens` | Creates a new scoped access token |
| `GET` | `/platform/profile/scoped-access-tokens/{id}` | Gets the scoped access token with the given ID |
| `DELETE` | `/platform/profile/scoped-access-tokens/{id}` | Deletes the scoped access token with the given ID |

### feedback

共 7 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `PATCH` | `/platform/feedback/conversations/{conversation_id}/custom-fields` | Update custom fields for a Front conversation |
| `POST` | `/platform/feedback/conversations/escalation` | Escalate an AI support conversation in Front |
| `POST` | `/platform/feedback/conversations/messages` | Sync AI support chat messages to Front |
| `POST` | `/platform/feedback/conversations/resolve` | Resolve an AI support conversation in Front |
| `POST` | `/platform/feedback/downgrade` | Send exit survey to HubSpot and survey_responses table |
| `POST` | `/platform/feedback/send` | Send feedback |
| `POST` | `/platform/feedback/upgrade` | Send upgrade survey to survey_responses table |

### stripe

共 6 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/stripe/atlas/application` | Will return the user data for the matching stripe application. |
| `POST` | `/platform/stripe/atlas/application/complete` | 内部端点（实现 `StripeAtlasPerkApplicationController_completeStripeAtlasFlow`） |
| `GET` | `/platform/stripe/invoices/overdue` | Gets information about overdue invoices that relate to the authenticated user |
| `GET` | `/platform/stripe/projects/provisioning/account_requests/{id}` | Get account request details |
| `POST` | `/platform/stripe/projects/provisioning/account_requests/{id}/confirm` | Confirm account request (from Studio) |
| `POST` | `/platform/stripe/setup-intent` | Initiated payment method setup |

### cli

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/cli/login` | Create CLI login session |
| `GET` | `/platform/cli/login/{session_id}` | Retrieve CLI login session |

### cloud-marketplace

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/cloud-marketplace/buyers/{buyer_id}/contract-linking-eligibility` | 内部端点（实现 `ClazarController_checkContractLinkingEligibility`） |
| `GET` | `/platform/cloud-marketplace/buyers/{buyer_id}/onboarding-info` | Get info needed for AWS Marketplace onboarding |

### mcp-tools-permissions

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/mcp-tools-permissions` | 内部端点（实现 `get-mcp-tools-permissions`） |

### notifications

共 4 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/notifications` | Get notifications |
| `PATCH` | `/platform/notifications` | Update notifications |
| `PATCH` | `/platform/notifications/archive-all` | Archives all notifications |
| `GET` | `/platform/notifications/summary` | Get an aggregated data of interest across all notifications for the user |

### oauth

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/oauth/apps/register` | Dynamically register an OAuth client (RFC-7591) |
| `GET` | `/platform/oauth/authorizations/{id}` | 内部端点（实现 `OAuthAuthorizationsController_getAuthorizationRequest`） |

### plans

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/plans/features` | 内部端点（实现 `PlanFeaturesController_getPlanFeatures`） |

### projects-resource-warnings

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/projects-resource-warnings` | 内部端点（实现 `ProjectsResourceWarningsController_getProjectsResourceWarnings`） |

### reset-password

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/reset-password` | Reset password for email |

### signup

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/signup` | Sign up with email and password |

### status

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/status` | Get infrastructure status |

### support

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/platform/support/verify-email` | 内部端点（实现 `VerifyEmailController_verifyEmail`） |

### update-email

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `PUT` | `/platform/update-email` | Updates a user email address |

### vercel

共 1 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/vercel/redirect/{installation_id}` | Gets the Vercel redirect url |

### workflow-runs

共 2 个操作

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/platform/workflow-runs` | Get a list of workflow runs |
| `GET` | `/platform/workflow-runs/{workflow_run_id}/logs` | Get the logs of a workflow run |

---

## 附录

### A. 鉴权细节

- **公开 v1/v2**：`Authorization: Bearer <token>`；token 为 PAT（个人访问令牌，在 Dashboard → Account → Access Tokens 创建）。
  也可通过 OAuth2 授权码流（`/v1/oauth/authorize` + `/v1/oauth/token` + `/v1/oauth/revoke`）以用户身份授权。
- **内部 platform**：Bearer（Studio 登录态 token）或会话 Cookie；`data/fetchers.ts` 中 `constructHeaders()` 自动附加 `Authorization: Bearer` 与 `X-Request-Id`。
- pg-meta 类端点需附加 `x-connection-encrypted`（加密的数据库连接串，由平台后端注入）。

### B. 错误结构

平台 API 统一返回 `{ code, message, requestId, ... }`（Studio 侧 `data/error-patterns.ts` 负责归类），
限流时带 `Retry-After` / `X-RateLimit-Reset` 响应头。

### C. 获取完整 OpenAPI

```bash
# 公开 v1/v2（官方规范下载）
make download.api.v1   # 在 apps/docs/spec/ 下执行
make download.api.v2
# 生成的 .d.ts（本地开发网关）
# redocly bundle 自 http://localhost:8080/api/{v1,v2,platform}-json
```

### D. 与 Studio 数据层对应

| 本文档分组 | Studio 数据层目录 |
| --- | --- |
| platform/projects · organizations · storage · auth · replication · database | `data/`（projects、organizations、storage、auth、database-*、replication…） |
| platform/pg-meta | `data/pg-meta` + `lib/api/self-hosted/query.ts`（自托管直连） |
| platform/telemetry · warehouse | `data/telemetry`、`data/warehouse` |
| v1/v2 | `data/fetchers.ts`（`api-types` 类型）、`pages/api/v1/**` 代理 |

---
*本文档由仓库内 OpenAPI 类型与官方规范自动生成整理，路径/方法/说明与当前类型文件一致；内部 platform 部分端点无公开描述，以“内部端点（实现 <operationId>）”标注。*
