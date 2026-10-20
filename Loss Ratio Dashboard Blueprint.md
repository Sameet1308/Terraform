Architecture blueprint · Snowflake + Unity Catalog · read-only sources

# One loss ratio dashboard in Sigma, Strategy ONE and Qlik

Five large tables are split between Snowflake and Databricks Unity Catalog. Nothing new can be created in either platform, so every transformation has to happen inside the BI tool. This page shows how each tool handles that, what it costs the warehouses, and how to prove the numbers are right.

SigmaStrategy ONE (MSTR)Qlik Cloud

The scenario

## Five tables, two platforms, no new objects

| Table | Platform | What it holds | Transformation needed |
| --- | --- | --- | --- |
| `policy` | Snowflake | Policy versions with line, state, agency | Keep the latest version per policy |
| `premium_txn` | Snowflake | Premium transactions | Sum earned premium to policy × month |
| `agency` | Snowflake | Agency attributes | Dimension |
| `claim` | Unity Catalog | Claims with policy key | Join to payments |
| `claim_payment` | Unity Catalog | Paid amounts and reserve changes | Sum incurred loss to policy × month |

**Constraint:** read-only access to these five tables. No new tables, views, foreign catalogs or materializations in Snowflake or Databricks. The loss ratio needs premium from Snowflake and loss from Databricks in the same calculation, so the join must happen in the BI tool.

The architecture

## Push aggregation down, join in the tool

The lean pattern is the same for every tool that can do it: send one aggregating query to each source, bring back only policy × month rows, and join those two small results inside the tool. The raw tables never leave the warehouses.

Where the transformation runs

## Two read-only queries carry all the logic

Query A runs in Snowflake and Query B runs in Databricks. Both are SELECT statements, so they respect the constraint. Strategy and Qlik send these queries to the sources and join the results. Sigma can run each one, but only on its own connection.

```
-- Query A · Snowflake · latest policy version + earned premium by policy × month
WITH policy_cur AS (
  SELECT policy_id, lob, state, agency_id
  FROM core.policy
  QUALIFY ROW_NUMBER() OVER (PARTITION BY policy_id ORDER BY version_ts DESC) = 1
)
SELECT pc.policy_id, pc.lob, pc.state, pc.agency_id,
       DATE_TRUNC('MONTH', t.txn_date) AS month_start,
       SUM(t.earned_premium)           AS earned_premium
FROM core.premium_txn t
JOIN policy_cur pc ON pc.policy_id = t.policy_id
WHERE t.txn_date >= DATEADD('MONTH', -36, CURRENT_DATE)   -- bound the window
GROUP BY 1, 2, 3, 4, 5;

-- Query B · Databricks SQL · incurred loss by policy × month
SELECT c.policy_id,
       date_trunc('MONTH', p.payment_date)  AS month_start,
       SUM(p.paid_amount + p.reserve_change) AS incurred_loss
FROM main.claims.claim c
JOIN main.claims.claim_payment p ON p.claim_id = c.claim_id
WHERE p.payment_date >= add_months(current_date(), -36)
GROUP BY 1, 2;
```

The tool then joins A and B on `policy_id` and `month_start` (full outer, so months with premium but no loss, and the reverse, both stay). Line, state and agency come from Query A. Keep the two amounts additive and let each tool divide them in the metric.

Build

## The same dashboard, tool by tool

### Sigma

1. Create a Snowflake connection and a Databricks connection.
2. Build one data model per connection: the premium logic (Query A) on Snowflake, the loss logic (Query B) on Databricks. Sigma compiles the model into SQL that runs in each warehouse \[40\].
3. Stop here for loss ratio. Sigma cannot join tables across connections \[1\], and its docs state that because data cannot move across connections, cross-table calculations are not possible \[2\].
4. What can still be built: premium elements and loss elements side by side, each from its own source, with no ratio.

BlockedUnder this constraint Sigma cannot deliver the loss ratio. The smallest exception that would unblock it is a Lakehouse Federation foreign catalog over Snowflake in Unity Catalog: a read-only mirror with no data copied, which gives Sigma one connection \[39\]. It is still a new Unity Catalog object, so it needs governance approval. Materialization and input tables are ruled out because they write tables to the warehouse \[6\].

### Strategy ONE

1. Create one database instance for Snowflake and one for Databricks \[13\].
2. Create one Multi-Table Data Import (MTDI) cube. Add Query A from the Snowflake instance and Query B from the Databricks instance using "type a query" \[15\]. MTDI combines tables from different sources into one Intelligent Cube \[14\].
3. Link the two tables on Policy and Month. Define attributes (Line, State, Agency, Month, Policy) and metrics; Loss Ratio = Incurred Loss ÷ Earned Premium.
4. Apply a security filter on State \[20\]. Schedule the cube refresh after the source loads.
5. Point every dashboard at this one cube \[16\], then certify the cube \[22\].

The warehouses see two aggregating queries per refresh, regardless of how many users open the dashboards. The cube must fit in Intelligence Server memory, which is why Query A and B bound the window to 36 months.

### Qlik Cloud

1. Create a Snowflake connection and a Databricks connection in the space.
2. In the load script, run Query A and Query B with `SQL SELECT`. Qlik sends the SQL to each database to execute \[25\].
3. Join the two results in the script on `policy_id` and `month_start`. Store each result as a QVD so later reloads pull only new months \[27\]. QVDs live in Qlik storage, not in the data platforms.
4. Create master measures for Earned Premium, Incurred Loss and Loss Ratio \[32\]. Add Section Access on STATE \[29\].
5. Schedule the reload, or chain it with qlik-cli \[33\].

Direct Query is not an option here, because a Direct Query app can use only one database \[26\]. The joined data must fit the app memory tier, up to 50 GB with large app support \[28\].

Sigma POC · Database impact analysis

## Cost impact on Snowflake and Databricks

The POC measures four themes. Sigma uses a live connection, so these themes describe its ongoing cost. For Strategy and Qlik, they describe one refresh or reload.

| Theme | Sigma | Strategy ONE | Qlik Cloud |
| --- | --- | --- | --- |
| Query complexity | Highest per request: the dedupe window function, join and aggregation from the data model run as SQL on each uncached request \[40\] | Same SQL (Query A and B), but run once per cube refresh | Same SQL, run once per reload; incremental QVD loads read only new months \[27\] |
| Query frequency | Grows with users × dashboard elements × filter changes, minus cache hits | Number of cube refreshes (for example, one per source load) \[16\]Lowest | Number of reloads Lowest |
| Cache protection | Browser cache and Alpha Query: per browser only. Warehouse caches: shared, with limits. Snowflake reuses results for 24 hours when the role can read all tables \[35\]; Databricks' shared remote cache is serverless only and resets when tables change \[37\]. Manual and scheduled refresh bypass all caches \[4\]. On OAuth connections, the query ID cache and universal result cache are used only with service account credentials \[4\]\[5\] | The cube is the protection: one in-memory copy shared by many reports and users \[16\] | The loaded app is the protection; the engine also keeps a result cache built from all users' selections \[30\] |
| Warehouse selection | Matters on every interaction. On Snowflake, user attributes can assign a different warehouse or role per user or team, with key pair authentication only, not OAuth \[8\]. Compare small and medium warehouses for the same workbook | Matters only at refresh. A small warehouse with a longer refresh is usually enough | Matters only at reload. Same as Strategy |
| Expected cost driver | Warehouse compute that scales with usage | Intelligence Server memory; warehouse cost near-flat | App memory tier and capacity; warehouse cost near-flat |

### How to measure it (read-only system views)

```
-- Snowflake · group repeat queries by shape; view latency up to 45 minutes [36]
SELECT query_parameterized_hash, warehouse_size,
       COUNT(*)                      AS runs,
       AVG(total_elapsed_time)/1000  AS avg_sec,
       SUM(bytes_scanned)/1e9        AS gb_scanned,
       AVG(percentage_scanned_from_cache) AS disk_cache_share
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD('DAY', -7, CURRENT_TIMESTAMP())
  AND user_name = 'SIGMA_SERVICE_USER'      -- or filter on query_tag / role_name
GROUP BY 1, 2 ORDER BY runs DESC;

-- Databricks · how often results came from cache [38]
SELECT client_application, from_result_cache,
       COUNT(*)                    AS runs,
       AVG(total_duration_ms)/1000 AS avg_sec,
       SUM(read_bytes)/1e9         AS gb_read
FROM system.query.history
WHERE start_time >= current_timestamp() - INTERVAL 7 DAYS
GROUP BY 1, 2;
```

| Complexity bucket | Definition for readouts | Example in this build |
| --- | --- | --- |
| Simple | One table, filter and aggregate | Agency list for a filter control |
| Medium | One join and aggregate | Query B (claims + payments) |
| Complex | Window function plus joins over large tables | Query A (latest policy version + premium) |

The buckets are a proposed framework for grouping readouts, not a vendor definition. Use the same Sigma service user, or a dedicated role, so Sigma traffic is easy to isolate in both system views.

Sigma POC · Value impact analysis

## Where Sigma adds value, and where the constraint limits it

The POC measures five value themes. The table shows what each tool offers per theme in this build, and what to record so the readout has numbers rather than opinions.

| Theme | Sigma | Strategy ONE | Qlik Cloud | What to record |
| --- | --- | --- | --- | --- |
| Ad-hoc analytics | Spreadsheet interface on live warehouse data, so business users can answer their own questions without a queue request Strongest. Single source per analysis under this constraint \[1\] | Self-service on certified objects; less free-form | Associative exploration, limited to data loaded in the app | Requests avoided in the DA & BID queues per month; time to first answer |
| Data analysis (engineering validation) | Engineers can profile and check a source table in a workbook instead of writing ad hoc SQL. Works within one connection | Needs a dataset or report built first | Needs a load script and reload first | Hours saved per validation task, compared with writing SQL by hand |
| License replacement (TM1, Qlik, MicroStrategy) | Can replace single-source content. Gap Cannot replace content that combines Snowflake and Databricks in one metric, such as this loss ratio, without the federation exception \[2\] | Baseline in scope for replacement | Baseline in scope for replacement | Share of existing reports that are single-source vs. cross-platform; licenses that can be retired only for the single-source share |
| Governance | Version tags with approval for production \[9\]; RLS in data models \[7\]; one metric definition per data model | Central metadata, security filters \[20\], certification \[22\], Integrity Manager \[21\] Strongest | Managed spaces with a locked data model \[31\]; Section Access \[29\] | Reworked or incorrect reports traced to manual steps, before and after |
| Writeback | Input tables write to the warehouse through a connection with write access \[2\]. Conflict This needs a writable schema, which the current "no new objects" rule does not allow | Not assessed here | Write table in in-memory apps only, not in Direct Query apps \[26\] | Hours spent today on separate writeback processes and user-defined tables |

**Readout framing:** Sigma's value is clearest in ad-hoc analytics, engineering data checks and writeback. Its license replacement value depends on how much existing Qlik and MicroStrategy content is single-source. Count that share early, because it caps the savings. Writeback needs one approved writable schema before it can be tested.

Caching and reuse

## One copy, many users: who can do it

| Question | Sigma | Strategy ONE | Qlik Cloud |
| --- | --- | --- | --- |
| Where cached data lives | Each user's browser; warehouse result caches; universal result cache (beta) in your own S3, GCS or Azure storage, not used for Snowflake or BigQuery \[4\]\[5\] | Intelligent Cubes in Intelligence Server memory: OLAP cubes and MTDI cubes, which the mstrio-py library calls "super cubes" \[16\]\[17\]; Mosaic models in In-Memory mode \[18\] | The app's data model in engine RAM plus a result cache \[30\] |
| One copy, many reports | No. Each workbook element issues its own SQL | Yes. A cube is shared as a single in-memory copy among many reports by multiple users \[16\]Strongest | Yes, within the app |
| Shared across users | Browser cache: no, it is per browser \[4\]. Warehouse caches: yes, with the limits above \[35\]\[37\] | Yes, with security filters applied | Yes \[30\] |
| Reused by other BI tools | No | Yes, through Mosaic connectors for Tableau, Power BI, Excel and Google Sheets \[19\] | No. QVDs are reused by other Qlik apps \[27\] |

**Correction to a common assumption:** Sigma's browser cache helps only the user who loaded it. Another person opening the same dashboard does not reuse it. Cross-user savings in Sigma come only from the warehouse result caches.

Validate, certify, reuse

## Prove the numbers once, then let many people use them

With no new objects allowed, accuracy checks are read-only queries run against the sources, plus a comparison of each tool's output with those control totals.

| Step | Check | How |
| --- | --- | --- |
| 1 · Source quality | No policy with two versions sharing the same latest `version_ts`; no payment without a claim | Read-only SELECT checks in each platform |
| 2 · Control totals | Premium by line × month (Snowflake); loss by month (Databricks) | Run Query A and B summed to those levels; save the output outside the platforms |
| 3 · Join check | Loss rows with no matching policy | Count them in the tool after the join; this check can only run where the join happens |
| 4 · Tool vs control | Every tile matches the control totals; loss by line sums to the Databricks total | Strategy: Integrity Manager compares SQL and grid data between development and production \[21\]. Qlik and Sigma: a hidden check sheet or element |
| 5 · Security | Each role sees only allowed states | Log in as a test user per role |
| 6 · Certify | Only tested assets are official | Strategy certification \[22\]; Sigma protected version tags with approval \[9\]; Qlik managed spaces, where the published data model is locked \[31\] |

**Cost logic:** once one cube (Strategy) or one app (Qlik) is certified, new users open it instead of building their own copy. That removes duplicate queries against the warehouses and lowers the cost per user as the audience grows.

Sigma POC evaluation criteria

## Technical and functional fit

### Technical · Architecture & Engineering

| Criterion | Sigma | Strategy ONE | Qlik Cloud |
| --- | --- | --- | --- |
| Snowflake and Unity Catalog | Native connections to both \[3\] | One database instance per source \[13\] | Connections to both |
| Combine both sources, read-only | GapNo joins or calculations across connections \[1\]\[2\] | MTDI cube from different sources \[14\]; MultiSource also runs multi-pass SQL across sources \[13\] | Load script joins both in memory \[25\]; Direct Query uses one database \[26\] |
| Transformations without new objects | In the data model, recomputed on each uncached request | In the cube's queries, computed per refresh | In the load script, computed per reload |
| Security pass-through | OAuth uses warehouse grants; RLS filters in the data model \[7\] | Security filters \[20\] | Section Access \[29\] |
| Automation platforms | REST API \[10\]; workbooks as JSON or YAML (beta) \[11\] | REST API \[23\]; mstrio-py \[17\] | REST APIs and qlik-cli \[33\] |

### Functional · Engineering & Business

| Criterion | Sigma | Strategy ONE | Qlik Cloud |
| --- | --- | --- | --- |
| User-friendliness | Spreadsheet interface on live data | Governed self-service on certified objects | Associative selections |
| SDLC+ | Version tags for test and production; protected tags need approval \[9\] | Development and production projects, Integrity Manager \[21\], certification \[22\] | Publish and republish to managed spaces \[31\] |
| Fit with D&A delivery strategy | Warehouse-native; blocked for cross-platform metrics under this constraint | Central semantic layer; Mosaic serves it to other tools \[19\] | App-centric; reuse through QVDs |
| Gen AI | Sigma Assistant / Ask Sigma (parts in beta) \[12\] | Strategy Auto 2.0 \[24\] | Qlik Answers \[34\] |

### What the Sigma POC should prove

| Test | Pass when |
| --- | --- |
| Cross-platform metric | Loss ratio is delivered. Expected result under the current constraint: fail, unless the foreign-catalog exception is approved |
| Single-source dashboards | Premium-only and loss-only dashboards match control totals |
| Database impact | Query history for 7 days at expected concurrency stays within the agreed compute budget, grouped by complexity bucket |
| Cache protection | Repeat views of the same dashboard by different users show warehouse cache hits (`from_result_cache` in Databricks) |
| Warehouse selection | The same workbook is measured on two warehouse sizes, with time and cost recorded for each |
| Security and SDLC+ | Test users see only allowed states; a change moves to a protected production tag through approval |

Overall

## Fit under this constraint

| Criterion | Sigma | Strategy ONE | Qlik Cloud |
| --- | --- | --- | --- |
| Delivers loss ratio | No | Yes | Yes |
| Cost | Warehouse compute grows with usage | Near-flat warehouse cost Low | Near-flat warehouse cost Low |
| Reusability | Data models per connection | One certified cube for many dashboards and other tools Strongest | One app; QVDs across apps |
| Performance | Depends on warehouse and cache | Fast from the cube | Fast from memory Fastest |
| Scalability | Scales with warehouses | Bounded by Intelligence Server memory | Bounded by app memory tier \[28\] |
| Semantics | Data model layer | Central metadata Strongest | Script model per app |
| Ad hoc analytics | Spreadsheet on live data, single source only | Self-service on governed objects | Associative exploration on loaded data |

**Short version:** with read-only sources split across two platforms, Strategy ONE and Qlik can deliver the dashboard by pushing two aggregating queries down and joining in memory, with low and predictable warehouse cost. Sigma cannot calculate the cross-platform loss ratio without one approved exception that puts both sources behind one connection.

Keep it lean

## What a professional developer leaves out

- **No raw tables in the tool**Aggregate in the source query; bring back policy × month only.
- **No logic copied per dashboard**One cube or one app holds Query A and B.
- **No extra engines**The tools' own query layers are enough.
- **No cache stacked on cache**One cube or one app; add more only after a measured problem.
- **No unbounded history**A fixed window keeps cubes and apps inside memory limits.
- **No stored ratios**Keep the sums; divide in the metric.

Sources

## Official documentation used

1. [Sigma – Overview of joining data](https://help.sigmacomputing.com/docs/joins-overview)
2. [Sigma – Connect to data sources](https://help.sigmacomputing.com/docs/connect-to-data-sources)
3. [Sigma – Supported data platforms and features](https://help.sigmacomputing.com/docs/region-warehouse-and-feature-support)
4. [Sigma – Caching and data freshness](https://help.sigmacomputing.com/docs/caching-and-data-freshness)
5. [Sigma – Manage universal result cache (Beta)](https://help.sigmacomputing.com/docs/manage-universal-result-cache)
6. [Sigma – About materialization](https://help.sigmacomputing.com/docs/materialization)
7. [Sigma – Set up row-level security](https://help.sigmacomputing.com/docs/set-up-row-level-security)
8. [Sigma – User attributes on a Snowflake connection](https://help.sigmacomputing.com/docs/configure-user-attributes-on-a-snowflake-connection)
9. [Sigma – Version tags for workbooks and data models](https://help.sigmacomputing.com/docs/version-tagging)
10. [Sigma – REST API reference](https://help.sigmacomputing.com/reference)
11. [Sigma – Manage workbooks as code (Beta)](https://help.sigmacomputing.com/docs/manage-workbooks-as-code)
12. [Sigma – Sigma Assistant (Beta)](https://help.sigmacomputing.com/docs/chat-with-sigma-assistant-on-the-homepage)
13. [Strategy – Connecting data sources to a project (MultiSource)](https://www2.microstrategy.com/producthelp/Current/ProjectDesignGuide/WebHelp/Lang_1033/Content/Connecting_data_sources_to_a_project.htm)
14. [Strategy – KB235131 Multi-Table Data Import](https://community.strategy.com/article/KB235131-New-feature-in-MicroStrategy-Web-10-Multi-Table-Data)
15. [Strategy – Import data by typing a query](https://www2.strategy.com/producthelp/Current/Workstation/en-us/Content/import_data_type_query.htm)
16. [Strategy – Sharing data among reports: Intelligent Cubes](https://www2.strategy.com/producthelp/Current/ReportDesigner/WebHelp/Lang_1033/Content/sharing_data_among_reports_intelligent_cubes.htm)
17. [Strategy – mstrio-py super_cube module](https://www2.strategy.com/producthelp/Current/mstrio-py/mstrio.project_objects.datasets.super_cube.html)
18. [Strategy – Mosaic Studio query modes](https://www2.strategy.com/producthelp/Current/Mosaic/en-us/Content/adm_query_mode.htm)
19. [Strategy – Mosaic GA press release (June 24, 2025)](https://www.strategy.com/press/strategy-announces-general-availability-of-strategy-mosaic-the-ai-powered-universal-intelligence-layer_06-24-2025)
20. [Strategy – Create and edit security filters](https://www2.microstrategy.com/producthelp/current/workstation/en-us/Content/create_security_filters.htm)
21. [Strategy – Integrity Manager](https://www2.strategy.com/producthelp/current/ProjectDesignGuide/WebHelp/Lang_1033/Content/MicroStrategy_Integrity_Manager.htm)
22. [Strategy – Certify dashboards, reports, datasets and cubes](https://www2.strategy.com/producthelp/Current/Workstation/en-us/content/Certifying_objects.htm)
23. [Strategy – REST API homepage](https://community.strategy.com/article/REST-API-Homepage)
24. [Strategy – Strategy Auto 2.0](https://www2.strategy.com/producthelp/Current/NextGenAI/en-us/Content/home_next_gen.htm)
25. [Qlik – SQL script statement](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptRegularStatements/SQL.htm)
26. [Qlik – Direct Query limitations](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/DirectQuery/limitations-direct-query.htm)
27. [Qlik – Working with QVD files](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/work-with-QVD-files.htm)
28. [Qlik – Large application support](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Admin/large-app-support.htm)
29. [Qlik – Section Access](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/Security/manage-security-with-section-access.htm)
30. [Qlik – Admin playbook: cache warming](https://help.qlik.com/en-US/qs-admin-playbook/Content/tooling/cache_warming.htm) (client-managed Qlik Sense)
31. [Qlik – Managing apps in managed spaces](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Spaces/managing-apps-in-managed-spaces.htm)
32. [Qlik – Working with master items](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Assets/work-with-master-items.htm)
33. [Qlik – qlik-cli](https://qlik.dev/toolkits/qlik-cli/) and [task chaining](https://qlik.dev/manage/automate/qlik-cli-snippets-task-chaining/)
34. [Qlik – Qlik Answers](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/QlikAnswers/Qlik-Answers.htm)
35. [Snowflake – Using persisted query results](https://docs.snowflake.com/en/user-guide/querying-persisted-results)
36. [Snowflake – ACCOUNT_USAGE.QUERY_HISTORY](https://docs.snowflake.com/en/sql-reference/account-usage/query_history)
37. [Databricks – Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching.html)
38. [Databricks – Query history system table](https://docs.databricks.com/aws/en/admin/system-tables/query-history)
39. [Databricks – Federated queries on Snowflake](https://docs.databricks.com/aws/query-federation/snowflake)
40. [Gale et al., "Sigma Workbook: A Spreadsheet for Cloud Data Warehouses," VLDB 2022](https://arxiv.org/pdf/2204.03128)

Not confirmed in official docs: where Mosaic stores In-Memory data, whether Sigma's universal result cache is shared across users, and licensing costs. Qlik's cross-user result cache is documented for client-managed Qlik Sense; Qlik Cloud internals are not documented the same way. The complexity buckets are a proposed framework, not a vendor definition.