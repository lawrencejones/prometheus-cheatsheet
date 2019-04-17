# prometheus-cheatsheet

## stolon-pgbouncer

Cluster identifier is labelled with `cluster_name` and `store_prefix`, with the
intention to join this metric against others.

```promql
stolon_pgbouncer_cluster_identifier
```

| Labels | Value |
|--------|-------|
| `stolon_pgbouncer_cluster_identifier{app="stolon-main-pgbouncer",cluster_name="lab-production/main",pod_name="stolon-main-pgbouncer-backend-default-5bd9967fd9-xhntn",store_prefix="stolon/cluster"}` | 85473287811880 |
| `stolon_pgbouncer_cluster_identifier{app="stolon-main-pgbouncer",cluster_name="lab-production/main",pod_name="stolon-main-pgbouncer-backend-default-5bd9967fd9-nd2mk",store_prefix="stolon/cluster"}` | 85473287811880 |
| `stolon_pgbouncer_cluster_identifier{app="stolon-main-pgbouncer",cluster_name="lab-production/main",pod_name="stolon-main-pgbouncer-backend-default-5bd9967fd9-8mgkq",store_prefix="stolon/cluster"}` | 85473287811880 |

It's possible to filter other metrics by matching on
`stolon_pgbouncer_cluster_identifier` and using the `and` binary operator,
however this will drop the additional labels.

```promql
stolon_pgbouncer_store_poll_interval and ignoring(cluster_name, store_prefix) stolon_pgbouncer_cluster_identifier
```

To keep the labels, we need to build a query with our desired labels as the left
operand. If we set `stolon_pgbouncer_cluster_identifier` to 1, we can multiply
that against any of our goal values and keep the `cluster_name` and
`store_prefix` labels.

```promql
(0 * stolon_pgbouncer_cluster_identifier{cluster_name="lab-production/main"} + 1) * ignoring(cluster_name, store_prefix) stolon_pgbouncer_store_poll_interval
```

### Joining twice (`group_left`)

We have three metrics that we want to combine into a single vector containing
all target labels:

- `stolon_pgbouncer_cluster_identifier{cluster_name,store_prefix}`
- `stolon_pgbouncer_host_hash{keeper}`
- `stolon_pgbouncer_last_reload_seconds{}`

Our goal is to create a single vector with values for the time since last
reload, including the `cluster_name`, `store_prefix` and `keeper` labels, so we
can display a table of stolon-pgbouncer's and where they're pointed.

We want to carry our keeper label onto our cluster identifier, then carry that
result onto the last reload seconds.

```promql
time() - stolon_pgbouncer_last_reload_seconds * on(pod_name) group_left(cluster_name, store_prefix, keeper) (
  1 + 0 * stolon_pgbouncer_cluster_identifier{cluster_name="lab-production/main"} * ignoring(cluster_name, store_prefix, keeper) group_left(keeper) stolon_pgbouncer_host_hash
)
```

The inner subquery sets our cluster identifier to have a value of 0, exploiting
the multiplication property to create a vector with our desired labels
(`cluster_name`, `store_prefix`,`keeper`) and set the value to 0, then adding
the scalar 1 to make it the multiplication identity. This vector of series with
all our desired labels and values 1 can be multiplied against our target value,
using `group_left` again to bring across the labels we want from our previous
join.
