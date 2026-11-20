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

# Pause and Resume

Stop the deployment when you are not using it to avoid paying for idle compute, then bring it back later with your data and configuration intact.

Unlike [Tear Down](teardown.md), pausing does not destroy anything. The EC2 instances are stopped, not terminated.

## Pause

```bash
ansible-navigator run playbooks/pause.yml -e @config.yml
```

The playbook runs in two steps:

1. **Stop the cluster services cleanly.** Through Cloudera Manager, it stops the base clusters, then the Cloudera Management Service.
2. **Stop the instances.** It stops every EC2 instance that matches the `asset_tags` for this deployment.

Services are stopped before the hosts are powered off so that they shut down gracefully instead of being killed mid-operation.

Pausing takes a few minutes: roughly 1 minute to stop the cluster services, then about 3 minutes to stop the instances.

!!! note
    Run pause while the deployment is running. The first step connects to Cloudera Manager, so the playbook fails if the instances are already stopped.

## Resume

```bash
ansible-navigator run playbooks/resume.yml -e @config.yml
```

The playbook runs in two steps:

1. **Start the instances.** It starts every EC2 instance that matches the `asset_tags` for this deployment.
2. **Start the cluster services.** It waits for the Cloudera Manager API to respond, starts the Cloudera Management Service, starts the base clusters, and then waits for both to report healthy.

The services do not start on their own when the instances come back, which is why the playbook starts them explicitly.

Resuming takes roughly 10 minutes on a typical cluster, and longer for larger ones. Each wait (Cloudera Manager API, Cloudera Management Service health, cluster health) polls for up to about 15 minutes before failing.

