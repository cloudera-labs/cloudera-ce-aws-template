<!-- Copyright 2026 Cloudera, Inc.

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

         https://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License. -->

# Ranger Policies

Once a [cluster](clusters.md) is running, the `ranger-policies.yml` playbook grants the `cdp-admins` and `cdp-users` groups access to the cluster's services through Apache Ranger. It is optional and is not part of the core deployment stages — run it on demand after the cluster playbook.

One playbook covers every cluster topology. It asks Cloudera Manager which services the cluster has and only applies the policies for those services, so a smaller cluster such as the Ozone cluster needs no different playbook and no extra configuration.

## Prerequisites

- A cluster playbook (`ozone-cluster.yml`, `streaming-cluster.yml`, or `full-cluster.yml`) has completed and the cluster is started. Ranger Admin and the Knox gateway must be reachable.
- The FreeIPA groups `cdp-admins` and `cdp-users` exist. `services.yml` creates them and Ranger UserSync imports them into Ranger.

## Run

```bash
ansible-navigator run playbooks/ranger-policies.yml -e @config.yml
```

To deploy a cluster and apply its policies in one command, add the playbook after the cluster playbook:

```bash
ansible-navigator run playbooks/infrastructure.yml playbooks/services.yml playbooks/cms.yml playbooks/ozone-cluster.yml playbooks/ranger-policies.yml -e @config.yml
```

The playbook calls the Cloudera Manager API and the Ranger Admin REST API, which it reaches through Knox at `https://knox.<gateway_ip>.<public_domain>/gateway/cdp-proxy-api/ranger`. It authenticates to Ranger as `admin` with `common_password`.

## Playbook Steps

1. **Service Discovery** — lists the services of the base cluster in Cloudera Manager and prints which policy sections apply
2. **Policy Sections** — applies every section whose service is deployed, and skips the rest

## Policy Sections

| Section | Applied when the cluster has | Ranger service | Policies |
|---------|------------------------------|----------------|----------|
| Kafka | Kafka | `cm_kafka` | `cdp-admins` can publish, consume, and manage all topics, consumer groups, transactional IDs, the cluster, and delegation tokens, and publish to `ATLAS_HOOK`. The `admin` user gets cluster-level access, and `cdp-users` can describe the cluster and use idempotent producers. |
| SSB Kafka | Kafka and SQL Stream Builder | `cm_kafka` | The `ssb` user can consume and describe all topics and consumer groups. Creates the `SSB - Internal topics` and `SSB - Internal consumer groups` policies (`__ssb*`) for `ssb` and `cdp-users`. |
| NiFi and EFM | NiFi | `<cluster_name>_nifi` | Creates the Data Access, Provenance, System, Site-To-Site, Input ports, and Output ports policies. `cdp-admins` (plus `cdp-users` and the `nifi` group where needed) get access to Flow, Controller, Policies, Tenants, Restricted Components, and the Root Process Group. |
| NiFi Registry | NiFi Registry | `<cluster_name>_nifiregistry` | `cdp-admins` gets access to Actuator, Buckets, Tenants, Policies, Swagger, and Proxies. |
| Kudu | Kudu | `cm_kudu` | The `kudu` and `ssb` users and `cdp-admins` get full access to all databases, tables, and columns. |

`<cluster_name>` is the name of the Cloudera Manager cluster that hosts NiFi (for example `full-base-cluster`). The playbook reads it from Cloudera Manager, so it is correct for whichever topology you deployed.

### Sections Applied per Cluster

| Cluster playbook | Kafka | SSB Kafka | NiFi and EFM | NiFi Registry | Kudu |
|------------------|:-----:|:---------:|:------------:|:-------------:|:----:|
| `ozone-cluster.yml` | Yes | — | — | — | — |
| `streaming-cluster.yml` | Yes | Yes | Yes | Yes | Yes |
| `full-cluster.yml` | Yes | Yes | Yes | Yes | Yes |

At the start of a run the playbook prints the service types it found and whether each section applies. Tasks in a section that does not apply are reported as `skipping` — this is expected, not a failure.

!!! tip
    Policies are created when missing and merged into existing policies otherwise, so the playbook is safe to re-run, for example after adding services to the cluster in Cloudera Manager.

