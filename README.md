# EDAwithMCP

Show Event-Driven Ansible and the Ansible Automation Platform Model Context Protocol (MCP) server working on the same Satellite `httpd` failure.

Event-Driven Ansible notices that Satellite is unreachable and launches a read-only diagnose job. It does not restart `httpd`. The diagnose output shows a broken `httpd.conf`, so a restart would fail the same way. MCP is then used to launch a restore job that puts the backup back and starts `httpd` only after `httpd -t` passes.

## Lab

| Item | Value |
|---|---|
| Platform | Ansible Automation Platform 2.7 containerized |
| Controller | `https://dhcp230-245.awxlab.pnq2.redhat.com` |
| Satellite | `smajumda-rhsat.syslab.pnq2.redhat.com` |
| Organization | Default |
| Inventory | `satellite` (one ungrouped host) |
| Machine credential | `satellite cred` (SSH as `root`) |
| Git repo | `https://github.com/sohamsat/EDAwithMCP.git`, branch `main` |

## Repository files

| File | Purpose |
|---|---|
| `break_httpd_conf.yml` | Back up `httpd.conf` with a date suffix, insert an invalid directive, restart `httpd` so the service goes down |
| `diagnose_httpd.yml` | Read-only: `systemctl is-active httpd`, `httpd -t`, last 40 lines of the error log |
| `restore_httpd_conf.yml` | Restore the newest `httpd.conf.YYYY-MM-DD` backup, run `httpd -t`, start `httpd` only if the test passes |
| `rulebooks/satellite_httpd_down.yml` | Poll Satellite every 30 seconds and launch `satellite-httpd-diagnose` when the connection fails |

The rule fires only when `event.url_check.error_msg` is set (connection failure). A redirect or login page is not treated as an outage. The rule is throttled to once every 5 minutes.

## Controller objects

1. Project **satellite** (id 7): Git project for this repo, branch `main`, clean on update, update on launch.
2. Job template **satellite-break** (id 8): `break_httpd_conf.yml`, inventory `satellite`, credential `satellite cred`.
3. Job template **satellite-httpd-diagnose** (id 9): `diagnose_httpd.yml`, same inventory and credential.
4. Job template **satellite-httpd-restore** (id 10): `restore_httpd_conf.yml`, same inventory and credential.

The playbooks use `hosts: all` because the Satellite host is not in a group named `satellite`.

## Break `httpd`

Launched **satellite-break**. Job **4** succeeded as an Ansible run and left the service down.

- Backup: `/etc/httpd/conf/httpd.conf.2026-09-25`
- `httpd` state: `failed`
- `httpd -t`: syntax error on line 54, `InvalidDirectiveThisBreaksHttpd`

A reload would have left the running process up. The playbook restarts `httpd` so the bad config is loaded and the service fails.

## Event-Driven Ansible

Created in **Automation Decisions**, separate from the Controller project:

1. Decision environment: `registry.redhat.io/ansible-automation-platform-27/de-supported-rhel9:latest`
2. Credential **eda-controller**, type **Red Hat Ansible Automation Platform**
   - Host: `https://dhcp230-245.awxlab.pnq2.redhat.com/api/controller`
   - OAuth token: a Controller token that can launch the diagnose template
   - Username and password left blank
   - Verify SSL unchecked (lab CA)
3. EDA project **eda project**: same Git repo and `main`
4. Rulebook activation **eda activation**
   - Rulebook: `satellite_httpd_down.yml`
   - Credential: `eda-controller`
   - Decision environment: the supported image above
   - Event streams: empty (the URL check is inside the rulebook)
   - Restart policy: Always
   - Log level: Info
   - Activation enabled

The first activations exited 1. `ansible-rulebook` called `/api/v2/config/` and got 404. On this platform the Controller API is `/api/controller/v2/`. The credential host has to include `/api/controller`. After that change, the activation was restarted (disable and enable) and stayed running.

Fire count went to 1. That is not an email or Slack message. The action launched Controller job **8**, **satellite-httpd-diagnose**, which reported:

- `httpd is-active: failed`
- `httpd -t` failed on `InvalidDirectiveThisBreaksHttpd`

Slack and email were not configured. Internal Red Hat Slack does not allow this user to install an app, and this lab has no SMTP relay for the AAP host.

## Same example with an event stream

This lab did not use an event stream. The activation **Event streams** field was left empty because `ansible.eda.url_check` polls Satellite from inside the rulebook.

An event stream is a webhook on Event-Driven Ansible. Something else posts JSON when `httpd` is down. Event streams replace a webhook source in the activation. They do not replace `url_check`, so the rulebook source has to change:

```yaml
---
- name: Satellite httpd health
  hosts: all
  sources:
    - name: satellite_httpd
      ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000
  rules:
    - name: Satellite httpd is unreachable
      condition: event.payload.status == "down"
      throttle:
        once_within: 5 minutes
        group_by_attributes:
          - event.payload.host
      action:
        run_job_template:
          name: satellite-httpd-diagnose
          organization: Default
```

Setup:

1. **Automation Decisions → Event Streams**: create a stream and enable **Forward events to rulebook activation**. Copy the URL the UI shows.
2. On **eda activation**, use the gear beside **Event streams**. Map rulebook source `satellite_httpd` to that event stream. The activation then receives events on the EDA webhook URL. It does not listen on port 5000 inside the decision-environment pod.
3. When `httpd` is down, a sender that can see the service posts to that URL:

```bash
curl -k -H 'Content-Type: application/json' \
  -d '{"host":"smajumda-rhsat.syslab.pnq2.redhat.com","service":"httpd","status":"down"}' \
  https://<event-stream-url>
```

`event.payload.status == "down"` matches that body. The throttle still allows one diagnose job per host every 5 minutes. The event stream only receives the post. A cron on Satellite, or a manual `curl`, still has to decide that `httpd` failed.

## Restore with MCP

The diagnose output is the decision point. The follow-up is the restore template, not a restart.

MCP is connected to `https://dhcp230-245.awxlab.pnq2.redhat.com:8448/mcp` with the same Controller token. Cursor cannot verify the lab CA, so the client is a local `mcp-remote` process with TLS verification disabled.

Launched **satellite-httpd-restore**. Job **23** succeeded:

- Restored from `/etc/httpd/conf/httpd.conf.2026-09-25`
- `httpd -t` passed
- `httpd is-active: active`
