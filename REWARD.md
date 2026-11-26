# Reward Function Design (RLVR)

## Verification Mechanism
Grading is strictly execution-based and runs on the host **once at episode termination**. The host grader connects to the rollout's isolated namespace with host-only credentials, verifies Stage-1 content hashes, replays the submitted dbt transformations on a clean target state, queries each target model, and compares against ground truth. The agent never sees ground truth, reference hashes, or host credentials.

## Why We Use Outcome Instead of Process Rewards
  - The agent will not see ground-truth rewards during test time, so it also shouldn't see it during training. During training, the agent should only see information from its own execution feedback.
  - Process rewards invite reward hacking at every step a reward is given. Using outcome rewards significantly reduces the reward hacking surface.
  - Querying the destination warehouse after each step is not as efficient as querying once when an episode is finished.

## Reward Formula
We will not use the SRDEL and SRDT metrics from the ELT-Bench paper due to sparsity early on in training. The scalar reward $R\in [0,1]$ is computed once at episode termination:

$$
R = \alpha \cdot R_{\text{EL}} + \beta \cdot R_{\text{EL}} \cdot R_{\text{T}} + \gamma \cdot \mathbf{1}_{\text{EL passed}} \cdot \mathbf{1}_{\text{T passed}}
$$

where:
- $\alpha = 0.20,\; \beta = 0.55,\; \gamma = 0.25$ ($\alpha + \beta + \gamma = 1.0$)
- $R_{\text{EL}} \in [0,1]$ is the fraction of the expected raw source tables that are correctly extracted and loaded.
- $R_{\text{T}} \in [0,1]$ is the mean execution-verified correctness score (see [$R_{\text{T}}$: Stage 2 - Transform](#stage2-reward) for exact formula).
- $\mathbf{1}_{\text{EL passed}} \in \{0, 1\}$ is $1$ iff $R_{\text{EL}} = 1$.
- $\mathbf{1}_{\text{T passed}} \in \{0, 1\}$ is $1$ iff $\text{replay\_ok} = 1$ and $R_{\text{T}} = 1$.

### $R_{\text{EL}}$: Stage 1 — Content-Verified Extract & Load
Let $S$ be the task's expected raw source tables. A table $s \in S$ passes iff:
  - It exists in the rollout schema as a materialized `BASE TABLE` (not a view).
  - Its full content matches reference:
    - On Snowflake / Databricks / Redshift: exact row count **and** order-independent full-table + per-column `HASH_AGG` match against `stage1_reference/<warehouse>.json` following [ELT-Bench/evaluation/eva_stage1_stored.py](ELT-Bench/evaluation/eva_stage1_stored.py) — does not use row count alone, and does not use `HASH_AGG` without row count to prevent XOR cancellation on duplicate-row syncs.
    - On the local credential-free DuckDB backend: `BASE TABLE` verification via `information_schema.tables` and exact sorted DataFrame content match against local fixture reference tables.
  - Two-layer Airbyte provenance check:
    - Table carries both `_AIRBYTE_RAW_ID` and `_AIRBYTE_EXTRACTED_AT` load columns.
    - The host grader queries Airbyte directly using host-only `airbyte_credential.json` (never trusting agent-writable `terraform.tfstate`) to verify that a destination targeting `(database, physical_schema)` has a connection covering table $s$ with a `succeeded` sync job and sufficient synced rows.

$$
\begin{aligned}
R_{\text{EL}} &= \frac{\left|\{s \in S : s \text{ passes}\}\right|}{|S|} \\
\mathbf{1}_{\text{EL passed}} &= \begin{cases} 1 & \text{if } R_{\text{EL}} = 1 \\ 0 & \text{otherwise} \end{cases}
\end{aligned}
$$

<a id="stage2-reward"></a>
### $R_{\text{T}}$: Stage 2 - Transform
Stage 2 runs if **at least one Stage 1 raw table passed**.

Before grading target models:
  - The host grader drops all non-raw tables/views and any unverified raw tables in the rollout schema to preserve only the Stage-1-verified raw tables.
  - The host grader re-executes the agent's submitted transformation project cleanly against the rollout schema and records $\text{replay\_ok} \in \{0, 1\}$ ($1$ iff replay exits `0` without timing out).

Let $M$ be the task's target models. For each model $m \in M$:
  - Execute `evaluation/sql/<task>/<m>.sql` (or `SELECT *`) via the warehouse connector, read candidate $\text{df}$ and ground truth $\text{df}_{\text{gt}}$ as strings, sort both by `sort_key.json`, and run `check_correctness(df_gt, df, rule="typed")`.
  - Per-model scores $s_m \in [0, 1]$ are computed as follows:

    Let $C_m \coloneqq \{\text{columns in df}_{\text{gt}}\}$, $C_m' \coloneqq \{\text{columns in df}\}$, and $e_m \coloneqq \left|C_m' \setminus C_m\right|$.

  $$
  s_m = \begin{cases}
  \dfrac{1}{1 + 0.1 \cdot e_m} & \text{if } C_m \subseteq C_m' \text{ and all ground-truth column values match} \\[8pt]
  0 & \text{otherwise}
  \end{cases}
  $$

Aggregate across all target models:

$$
\begin{aligned}
R_{\text{T}} &= \frac{1}{|M|} \sum_{m \in M} s_m \\
\mathbf{1}_{\text{T passed}} &= \begin{cases} 1 & \text{if } \text{replay\_ok} = 1 \text{ and } R_{\text{T}} = 1 \\ 0 & \text{otherwise} \end{cases}
\end{aligned}
$$

### Anti-Reward Hacking Safeguards
- **Airbyte/dbt bypass via Python scripts:** Blocked by verifying Airbyte provenance at both the warehouse column level and directly against Airbyte's host-side control-plane API, and by dropping non-raw/unverified tables and cleanly replaying `dbt run`.
- **Connector cherry-picking (ignoring hard Airbyte sources):** Mitigated by scaling $R_{\text{T}}$ by $R_{\text{EL}}$ and gating the $+0.25$ completion bonus on $\mathbf{1}_{\text{EL passed}} \cdot \mathbf{1}_{\text{T passed}}$.
- **Column stuffing (`SELECT *` / name hedging):** Blocked by smoothly decaying $s_m$ for every extra column and withholding the $+0.25$ completion bonus unless column sets match exactly.
- **Aborted/timed-out rollouts:** Timed-out and crashed rollouts are always graded so the policy cannot avoid negative advantage by crashing or refusing to call `submit`.
- **Over-rounding / tolerance abuse:** Mitigated by using `rule="typed"` (exact integer comparison, decimal-place + 1% relative check for floats) rather than loose `rtol`.

### Known Reward-Hacking Vectors (Not Implemented)
The following loopholes are out of scope for this project but documented for future hardening:

- **Decoy Airbyte sync + Python overwrite:** Because the agent container currently shares `elt-docker_elt_network` with source databases and holds `AIRBYTE_USER` write credentials, an adversarial policy could sync a dummy LocalStack S3 file with $N$ junk rows via Airbyte and then overwrite the Snowflake table via Python while keeping `_AIRBYTE_RAW_ID` / `_AIRBYTE_EXTRACTED_AT`.
  - *Future fix:* Network-isolate the agent container from source databases so only Airbyte can reach `elt-postgres`/`elt-mongodb`/`elt-localstack`/`elt-api`, split `AIRBYTE_ROLE` from `DBT_ROLE`, and/or validate each Airbyte connection's `sourceType`/host and Snowflake `QUERY_HISTORY`.

- **Mutable shared source containers across concurrent rollouts:** Source credentials in `config.yaml` have write privileges on the shared source containers, so a rogue rollout could mutate or drop source data for concurrent rollouts.
  - *Future fix:* Provision strictly read-only source credentials or run per-worker source containers.

- **Duplicate-pair XOR swap in `HASH_AGG`:** Even with row counts, Snowflake's XOR-based `HASH_AGG` cannot distinguish a table containing a pair of legitimate identical rows $(r, r)$ from one where those two rows were swapped for a pair of bogus identical rows $(r', r')$ (only applicable to tables that legitimately contain duplicate rows).
  - *Future fix:* Use an additive multiset hash or an in-warehouse `EXCEPT ALL` diff against a maintained `ELT_STAGE1_REF` database.

- **Memorization / hardcoded `VALUES` or dbt hooks during replay:** `/usr/local/bin/dbt run` executes arbitrary SQL/Jinja in `/workspace/elt`. While ground truth is hidden from the container, over repeated RL epochs on small tasks a model could memorize small output tables instead of computing them from raw tables.
  - *Future fix:* Enforce a static dbt/SQL check disallowing `on-run-start`/`run_query` hooks and requiring lineage from verified raw tables, plus evaluate on held-out `eval_tasks`.