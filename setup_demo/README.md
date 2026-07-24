# AAP Patching — Configuration as Code

Config-as-code for the workflow in [`sonstar2/aap_patching`](https://github.com/sonstar2/aap_patching),
structured the same way as [`aap_provisioning/setup_demo`](https://github.com/sonstar2/aap_provisioning/blob/main/setup_demo/setup.yml):
a thin `setup.yml` loads everything under `configs/` and hands it to
`infra.aap_configuration.dispatch`, which fans each variable out to the matching role
(`controller_projects` → projects role, `controller_templates` → job_templates role, etc.).

## Layout

```
.
├── requirements.yml
├── setup.yml
└── configs/
    ├── controller_projects.yml               # points at the aap_patching repo
    ├── controller_credentials.yml             # ServiceNow / AWS / Slack creds + notification template
    ├── controller_inventories.yml             # inventory + AWS EC2 dynamic source (the "Inventory Sync" step)
    ├── controller_job_templates.yml           # one job template per playbook in aap_patching
    └── controller_workflow_job_templates.yml  # the 10-step workflow itself
```

## Run it

```bash
ansible-galaxy collection install -r requirements.yml
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=changeme
export CONTROLLER_VERIFY_SSL=false
export SNOW_HOST=... SNOW_USERNAME=... SNOW_PASSWORD=...
export AWS_ACCESS_KEY=... AWS_SECRET_KEY=...
export SLACK_TOKEN=...

ansible-playbook setup.yml
```

Everything is idempotent — re-running `setup.yml` reconciles Controller to match the YAML,
same as the provisioning repo's `setup.yml`.

## Workflow node design

The 10 requested stages map to job templates 1:1, except **Inventory Sync**, which is a
workflow node pointing directly at the `AWS EC2 - Patching Targets` inventory source
rather than at a job template (AAP workflows support inventory-sync nodes natively).

The trickier part is step 9, **Restore Snapshots if failed**. Looking at
`snapshot_restore.yml` in the source repo, the playbook doesn't rely on the workflow's
success/failure edge alone — it filters hosts itself:

```yaml
when:
  - patch_progress[inventory_hostname] == 'fail'
  - patch_stage[inventory_hostname] == 'post_patch_tasks' or
    patch_stage[inventory_hostname] == 'post_app_tasks' or
    patch_stage[inventory_hostname] == 'apply_patching'
```

`patch_progress` and `patch_stage` are per-host dicts propagated between jobs via
`ansible.builtin.set_stats` (see `apply_patching.yml`). That means a job template can
report overall "successful" to Controller while still containing individual failed
hosts. So the workflow can't just use a `failure_nodes` edge into the restore step —
it needs to reach it on **both** outcomes and let the playbook's own `patch_progress`
check decide who actually gets rolled back. That's why the workflow config wires:

- `apply_patching` → `failure_nodes: [restore_snapshot]` (whole-job failure)
- `post_patching_tasks` → `failure_nodes: [restore_snapshot]` (whole-job failure)
- `post_app_tasks` → `always_nodes: [restore_snapshot]` (catches per-host failures
  even when the job itself reports success)

`generate_report` is likewise wired as `always_nodes` off of `restore_snapshot`, so the
report always gets produced whether or not anything needed restoring.

An 11th, optional node (`close_change_request` → `snow_close_cr.yml`) is included
because `apply_patching.yml` sets a `cr_close_status` stat specifically for this purpose.
Delete that node plus its job template in `controller_job_templates.yml` if you only
want the 10 stages you listed.

## Adjust before use

- `controller_projects.yml` — set `scm_credential` if `aap_patching` is private.
- `controller_job_templates.yml` — the individual playbooks (`pre_patch_tasks.yml`,
  `apply_patching.yml`, etc.) expect roles from `demo.patching`, `demo.cloud`, and
  `demo.process` collections/roles per the source repo; make sure those are resolvable
  from your Execution Environment or add a `requirements.yml` inside the `aap_patching`
  project itself.
- `controller_workflow_job_templates.yml` — the survey currently asks for
  `change_owner` and `change_environment` since `snow_create_cr_wait.yml` reads both;
  extend the survey if other playbooks need additional launch-time input.