# Zabbix Dashboard Research for Docker/Coolify Monitoring

**Date:** 2026-04-11
**Context:** Single VPS running Coolify with all services as Docker containers, monitored by Zabbix 7.4 with Agent 2.

---

## 1. Executive Summary

- **Trigger Overview widget is the single best widget for red/green per-container status.** It displays a grid of colored blocks (hosts as columns, triggers as rows) where green = OK and severity-colored = PROBLEM. This is the closest thing to a "red/green per service" indicator in native Zabbix.
- **The built-in "Global view" dashboard is a starting point, not a solution.** It ships with generic widgets (Problems by severity, Host availability, System info). You need a custom dashboard for per-container visibility.
- **Honeycomb widget (Zabbix 7.0+) provides a visual NOC-style overview** where each hexagonal cell represents a host or item, colored by status. Great for at-a-glance health but less useful for a single host with many containers.
- **Top Hosts widget is the workhorse for tabular container data** -- CPU, memory, status in a sortable table with color-coded bars and sparklines (7.2+).
- **Keep it to one dashboard page with 4-6 widgets.** Resist the temptation to add everything. A single-server setup needs: (1) overall problem summary, (2) per-container status grid, (3) resource usage table, and (4) a graph or two for trends.

---

## 2. Built-in Dashboards in Zabbix 7.4

### What ships out of the box

Zabbix 7.4 comes with a **"Global view"** default dashboard that includes:
- **Problems by severity** widget -- shows problem counts grouped by severity level (color-coded blocks)
- **Host availability** widget -- counts hosts as Available / Not available / Unknown per interface type
- **System information** widget -- Zabbix server stats
- Various other widgets depending on installation dataset

Source: [Zabbix 7.4 What's New](https://www.zabbix.com/documentation/current/en/manual/introduction/whatsnew)

### Which are useful for your setup

| Dashboard/Widget | Useful? | Why |
|---|---|---|
| Global view | Starting point only | Too generic for per-container monitoring |
| Host availability | Limited | Shows host-level availability (agent up/down), not container-level. You have one host, so it just shows 1 green dot |
| Problems by severity | YES | Good top-level "are there any problems?" indicator |

**Recommendation:** Do NOT try to customize Global view. Create a new custom dashboard called "Soul Journeys Services" instead. The Global view is meant as a starting point and gets overwritten on upgrades.

Source: [Zabbix Dashboards Documentation](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards)

---

## 3. Complete Widget List in Zabbix 7.4

For reference, here is the full list of available widgets:

| # | Widget | Added in | Relevance |
|---|--------|----------|-----------|
| 1 | Action log | -- | Low |
| 2 | Clock | -- | Low |
| 3 | Data overview | -- | **DEPRECATED** -- will be removed. Use Top hosts instead |
| 4 | Discovery status | -- | Low |
| 5 | Favorite graphs | -- | Low |
| 6 | Favorite maps | -- | Low |
| 7 | **Gauge** | **7.0** | Medium -- single item display (e.g., CPU overall) |
| 8 | Geomap | -- | Low (single server, no geo data needed) |
| 9 | **Graph** | -- | **HIGH** -- essential for time-series trends |
| 10 | Graph (classic) | -- | Medium |
| 11 | Graph prototype | -- | Low |
| 12 | **Honeycomb** | **7.0** | **HIGH** -- visual NOC-style overview of containers |
| 13 | **Host availability** | -- | Low for single host |
| 14 | **Host card** | **7.2** | **HIGH** -- comprehensive single-host info panel |
| 15 | Host navigator | 7.0 | Medium -- host selector for interactive dashboards |
| 16 | Item history | -- | Low |
| 17 | **Item card** | **7.4** | Medium -- deep dive into single item |
| 18 | Item navigator | 7.0 | Medium |
| 19 | Item value | -- | Medium -- display single metric prominently |
| 20 | Map | -- | Low |
| 21 | Map navigation tree | -- | Low |
| 22 | **Pie chart** | **7.0** | Medium -- good for resource distribution |
| 23 | **Problem hosts** | -- | **HIGH** -- shows problem count per host group |
| 24 | **Problems** | -- | **HIGH** -- live problem list with details |
| 25 | **Problems by severity** | -- | **HIGH** -- color-coded severity overview |
| 26 | SLA report | -- | Low for now |
| 27 | System information | -- | Low |
| 28 | **Top hosts** | -- | **HIGH** -- tabular view with bars, sparklines |
| 29 | **Top triggers** | -- | Medium -- which triggers fire most |
| 30 | **Trigger overview** | -- | **CRITICAL** -- the red/green grid you want |
| 31 | URL | -- | Medium -- embed external dashboards (Uptime Kuma?) |
| 32 | Web monitoring | -- | Medium -- if using Zabbix web scenarios |

Sources:
- [Zabbix 7.4 Dashboard Widgets](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets)
- [Zabbix 7.0 What's New](https://www.zabbix.com/whats_new_7_0)
- [Zabbix 7.2 What's New](https://www.zabbix.com/whats_new_7_2)
- [Zabbix 7.4 What's New](https://www.zabbix.com/whats_new_7_4)

---

## 4. Widget Types for Service Health (Red/Green per Container)

### Option A: Trigger Overview Widget (RECOMMENDED)

**What it does:** Displays a grid where rows = triggers, columns = hosts. Each cell is a colored block: green for OK, severity color (yellow/orange/red) for PROBLEM. Recent changes blink for 2 minutes.

**Why it is the best fit:**
- Directly maps to "red/green per container" -- each discovered container gets a "Container is not running" trigger from the Docker template
- Filter by tag or host group to show only container-status triggers
- Clicking a cell links to the problem details
- Works on a single host with multiple triggers (one per container)

**Configuration for your use case:**
- Host groups: your Docker host group
- Tag filter: filter to show only `docker.container_info` related triggers
- Show: "Any problems" or "Recent problems"

**Limitations:**
- With many triggers per host, the grid can get wide
- Cannot show non-trigger data (like CPU %) in the same widget

Source: [Trigger Overview Widget Documentation](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/trigger_overview)

### Option B: Honeycomb Widget

**What it does:** Displays hosts or items as hexagonal cells in a colored grid. Color intensity reflects the item value or problem severity.

**Why it is interesting:**
- Very visual, NOC-style appearance
- Each container could be a cell
- Color coding based on status item value
- Interactive -- clicking a cell can broadcast to other widgets (Graph widget shows that container's metrics)

**Configuration for your use case:**
- Host group: your Docker host
- Item pattern: container status items
- Color thresholds: 0 = red (stopped), 1 = green (running)

**Limitations:**
- Designed for multi-host environments; on a single host with discovered items it may look sparse
- Less precise than a table -- harder to read exact values

Source: [Honeycomb Widget Documentation](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/honeycomb)

### Option C: Top Hosts Widget

**What it does:** A customizable table showing hosts or items ranked by a metric. Supports columns with bars, sparklines (7.2+), and text values. Up to 100 entries.

**Why it is useful:**
- Best for showing container resource usage (CPU, memory) in a table
- Can include a status column alongside resource metrics
- Bars and color thresholds give quick visual feedback
- Sparkline mini-charts show trends inline (Zabbix 7.2+)

**Configuration for your use case:**
- Create columns: Container Name, Status, CPU %, Memory MB, Network I/O
- Set thresholds: CPU > 80% = orange, > 95% = red
- Sort by CPU descending to see resource hogs first

**Limitations:**
- Not strictly red/green -- it is more of a data table
- Requires careful column configuration

Source: [Top Hosts Widget Documentation](https://www.zabbix.com/documentation/current/en/manual/api/reference/dashboard/widget_fields/top_hosts)

### Option D: Problems Widget

**What it does:** Shows a live list of current problems with severity, host, trigger name, duration, and acknowledgment status.

**Why it is useful:**
- Immediate visibility of what is broken right now
- Can filter to show only Docker-related problems
- Shows problem age (how long has it been a problem?)

**Limitations:**
- Only shows problems -- if everything is green, the widget is empty (which is actually a good signal)
- No "all green" confirmation at a glance

Source: [Problems Widget Documentation](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/problems)

### Option E: Host Card Widget (Zabbix 7.2+)

**What it does:** Shows comprehensive info about a single host: availability, problem count by severity, inventory data, tags. Multi-column layout.

**Why it is useful:**
- Perfect as a header widget for your single-server dashboard
- Shows at a glance: is the host up? How many problems? What severity?
- Can include host description, tags, inventory fields

**Limitations:**
- Single host only -- does not show per-container breakdown
- More of a "header" than a primary monitoring widget

Source: [Host Card Widget Documentation](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/host_card)

### Comparison Table

| Widget | Red/Green per container? | Resource data? | At-a-glance? | Setup effort |
|--------|------------------------|----------------|--------------|-------------|
| Trigger Overview | YES - colored blocks | No | Excellent | Low |
| Honeycomb | YES - colored hexagons | Single metric | Good | Medium |
| Top Hosts | Partial - color thresholds | YES | Good | Medium |
| Problems | Only problems shown | No | Good (empty = good) | Low |
| Host Card | Host-level only | No | Excellent | Low |

---

## 5. Real-World Examples

### Example 1: Official Zabbix Blog -- Docker Container Monitoring

**Source:** [Docker Container Monitoring With Zabbix](https://blog.zabbix.com/docker-container-monitoring-with-zabbix/20175/)

**What it shows:** Step-by-step setup of Docker monitoring using Zabbix Agent 2 and the built-in "Docker by Zabbix agent 2" template. Demonstrates:
- Container discovery via `docker.containers.discovery[true]` (all containers) or `[false]` (running only)
- Auto-created items per container: state, CPU, memory, network I/O, block I/O
- Auto-created triggers: "Container stopped" triggers per discovered container
- Graphs showing container resource usage over time

**Layout described:** The article shows the Zabbix Monitoring > Latest data view filtered by Docker items, showing all discovered containers with their current metrics. Also shows auto-generated graphs for CPU and memory per container.

**Relevance:** This is your baseline -- the template you already have. The dashboard is the next step.

### Example 2: VirtualizationHowTo -- Why I Switched to Zabbix for Docker Monitoring

**Source:** [Why I Switched to Zabbix for Monitoring My Docker Containers](https://www.virtualizationhowto.com/2025/11/why-i-switched-to-zabbix-for-monitoring-my-docker-containers/)

**What it shows:** A home lab user monitoring Docker containers with Zabbix, showing:
- Custom dashboard creation via Dashboards > hamburger menu > Create new
- Graph widgets showing CPU utilization across all containers
- Memory usage comparison widgets
- Network traffic over time

**Layout described:** The author describes building a custom dashboard by adding Graph widgets for each metric category (CPU, memory, network). The approach is manual but effective for a small number of containers.

**Relevance:** Very similar setup to yours -- single server, home/small-business use case.

### Example 3: Zabbix Summit 2022 -- Single Pane of Glass for MSPs

**Source:** [Zabbix Dashboards: A Single Pane of Glass for MSP Environments (PDF)](https://assets.zabbix.com/files/events/2022/zabbix_summit_2022/Karlis_Salins_Zabbix_dashboards_A_single_pane_of_glass_for_MSP_environments.pdf)

**What it shows:** A presentation by Karlis Salins (Zabbix Technical Support Engineer) covering:
- Dashboard design principles for MSP environments
- Using Top Hosts widget for at-a-glance server status
- Item Value widgets for key metrics
- Geomap for distributed infrastructure
- Multi-page dashboards with different detail levels

**Layout described:** The recommended pattern is a hierarchical approach: Page 1 = high-level overview (problems, host availability), Page 2 = detailed metrics, Page 3 = specific service details.

**Relevance:** The principles apply even for a single server. The "Page 1 = overview, Page 2 = details" pattern works well.

### Example 4: Medium/DevSecOps -- Zabbix Dashboard Best Practices

**Source:** [Best Practices for Zabbix Dashboard Designs](https://medium.com/devsecops-community/best-practices-for-zabbix-dashboard-designs-enhancing-monitoring-visibility-and-efficiency-506a2d34e577)

**What it shows:** General best practices:
- Define monitoring objectives before designing
- Use appropriate widget variety (maps, graphs, HTML blocks)
- Optimize widget sizing: dashboard is 72 columns wide, max 64 rows at 70px each
- Configure refresh rates appropriately (10-30 seconds for real-time)
- Integrate alerts prominently on dashboards
- Use role-based dashboards for different audiences

**Relevance:** Good principles for widget sizing and layout. The 72-column grid means you can do a clean 2-column (36+36) or 3-column (24+24+24) layout.

### Example 5: Zabbix Forum -- Docker Container Resource Widget

**Source:** [Create widget showing resource intensive docker containers across multiple hosts](https://www.zabbix.com/forum/zabbix-help/465622-create-widget-showing-resource-intensive-docker-containers-across-multiple-hosts)

**What it shows:** A community discussion about creating a dashboard widget that displays the top 10 CPU/memory-intensive Docker containers. The recommended approach:
- Use the "Top hosts" widget type (or "Data overview" in older versions)
- Select host group containing Docker hosts
- Specify Docker-specific item keys for CPU and memory
- Configure to show top N containers by resource usage

**Relevance:** Directly applicable. Shows how to use Top Hosts widget for Docker container resource monitoring.

---

## 6. Docker Template Item Keys Reference

The "Docker by Zabbix agent 2" template creates the following items per discovered container (via low-level discovery with `docker.containers.discovery`):

| Item Key Pattern | Description | Use for Dashboard? |
|---|---|---|
| `docker.container_info["{#NAME}",State.Status]` | Container state (running, stopped, paused, etc.) | YES -- primary status indicator |
| `docker.container_info["{#NAME}",State.Error]` | Container error message | YES -- in problems |
| `docker.container_info["{#NAME}",State.Health.Status]` | Health check status (healthy, unhealthy) | YES -- if containers have healthchecks |
| `docker.container_stats.cpu_usage["{#NAME}"]` | CPU usage percentage | YES -- resource table |
| `docker.container_stats.memory.usage["{#NAME}"]` | Memory usage bytes | YES -- resource table |
| `docker.container_stats.online_cpus["{#NAME}"]` | Online CPUs | Low priority |
| `docker.container_stats.net.*["{#NAME}"]` | Network stats (tx/rx bytes, packets, errors) | Medium -- graphs |
| `docker.container_stats.blkio.*["{#NAME}"]` | Block I/O stats | Low priority |

**Triggers created per container:**
- Container is not running (based on State.Status)
- Container health check failed (if healthcheck is configured)

Sources:
- [Docker by Zabbix Agent 2 Template (GitHub)](https://github.com/zabbix/zabbix/blob/master/templates/app/docker/template_app_docker.yaml)
- [Zabbix Docker Integration Page](https://www.zabbix.com/integrations/docker)

---

## 7. Coolify-Specific Monitoring Notes

### No existing Zabbix-Coolify integration found

There are **no community-published Zabbix templates or dashboards specifically for Coolify** as of April 2026. This is likely because:
- Coolify is relatively new and its user base skews toward simpler monitoring (Uptime Kuma, Netdata)
- Most Coolify users rely on Coolify's built-in monitoring (disk usage, container status, backup alerts)
- Coolify recommends Grafana or Netdata for detailed monitoring

### How Coolify users typically monitor

1. **Coolify built-in dashboard** -- container status, deployment logs, disk usage alerts
2. **Uptime Kuma** -- deployed alongside, for HTTP endpoint monitoring
3. **Netdata** -- deployed via Docker Compose, for system-level metrics
4. **Grafana + Prometheus** -- more advanced stacks

### What you already have (your custom template)

Your `zabbix-template-coolify-api.yaml` covers the Coolify API side. Combined with the Docker template, you have:
- Container-level monitoring (Docker template)
- Coolify API health (custom template)
- Linux host metrics (Linux template)

Sources:
- [Coolify Monitoring Docs](https://coolify.io/docs/knowledge-base/monitoring)
- [Coolify Dashboard Docs](https://coolify.io/docs/services/dashboard)
- [Coolify + Netdata Tutorial](https://peturgeorgievv.com/blog/how-to-deploy-monitoring-on-your-vps-with-coolify-and-netdata)

---

## 8. Recommended Dashboard Layout for Soul Journeys

### Dashboard Name: "Soul Journeys Services"

**Single page, 6 widgets.** The 72-column grid is divided into a 2-column layout with a full-width header and footer.

### Text-Art Mockup

```
+============================================================+
| ROW 1: Host Card (full width, 72 cols, 2 rows)             |
| [VPS hostname] | Agent: OK | Problems: 0/0/0/0/0           |
| Tags: coolify, docker, soul-journeys                        |
+============================+===============================+
| ROW 2-4: Trigger Overview  | ROW 2-4: Problems by Severity |
| (36 cols, 5 rows)          | (36 cols, 5 rows)             |
|                            |                               |
| Containers:    Status      |  [Not classified] [Info]      |
| n8n            [GREEN]     |  [Warning] [Average]          |
| mautic         [GREEN]     |  [High] [Disaster]            |
| zabbix-server  [GREEN]     |  (colored blocks with counts) |
| zabbix-web     [GREEN]     |                               |
| uptime-kuma    [GREEN]     |                               |
| website        [GREEN]     |                               |
| shop           [GREEN]     |                               |
| postgres       [GREEN]     |                               |
+============================+===============================+
| ROW 5-8: Top Hosts - Container Resources                    |
| (full width, 72 cols, 6 rows)                               |
|                                                             |
| Container Name | Status | CPU %  [====    ] | Mem MB  [==] |
| n8n            | Running| 12.3%  [===     ] | 256     [==] |
| mautic         | Running| 8.1%   [==      ] | 512     [===]|
| shop           | Running| 3.2%   [=       ] | 128     [=  ]|
| ...            | ...    | ...               | ...          |
+============================================================+
| ROW 9-12: Graph - CPU & Memory Trends (72 cols, 6 rows)    |
|                                                             |
|  CPU %    ^                                                 |
|  30% |    /\    ___/\                                       |
|  20% |   /  \__/    \___                                    |
|  10% |__/                \___                               |
|       +-----|-----|-----|----> time                          |
|       -4h   -3h   -2h   -1h   now                          |
+============================+===============================+
| ROW 13: Problems List      | ROW 13: URL Widget            |
| (36 cols, 4 rows)          | (36 cols, 4 rows)             |
|                            |                               |
| (empty = all good!)        | Embed Uptime Kuma status page |
| or:                        | or Coolify dashboard           |
| [HIGH] n8n container down  |                               |
|   Duration: 5m             |                               |
+============================+===============================+
```

### Widget Specifications

#### Widget 1: Host Card (header)
- **Type:** Host card
- **Position:** Row 0, Col 0, Width 72, Height 2
- **Config:** Select your VPS host. Show: availability, problems by severity, tags
- **Purpose:** Instant "is the server up?" check

#### Widget 2: Trigger Overview (container status grid)
- **Type:** Trigger overview
- **Position:** Row 2, Col 0, Width 36, Height 5
- **Config:**
  - Host group: your Docker host group
  - Show triggers: "Any problems" (shows all triggers, green or problem)
  - Tag filter: consider filtering to only `docker.container` related triggers
- **Purpose:** THE red/green grid showing each container's status

#### Widget 3: Problems by Severity (summary)
- **Type:** Problems by severity
- **Position:** Row 2, Col 36, Width 36, Height 5
- **Config:** Host group: your Docker host group + Linux host group
- **Purpose:** Quick severity breakdown -- how bad are current issues?

#### Widget 4: Top Hosts (resource table)
- **Type:** Top hosts
- **Position:** Row 7, Col 0, Width 72, Height 6
- **Config:**
  - Host group: Docker host group
  - Columns: Container Name (text), Status (text), CPU % (bar + sparkline), Memory (bar + sparkline)
  - Order by: CPU % descending
  - Thresholds: CPU > 80% = orange, > 95% = red; Memory > 80% of limit = orange
- **Purpose:** Resource usage overview with visual indicators

#### Widget 5: Graph (trends)
- **Type:** Graph
- **Position:** Row 13, Col 0, Width 72, Height 6
- **Config:** Show stacked area chart of CPU usage for all containers over last 4 hours
- **Purpose:** Spot trends, correlate spikes across containers

#### Widget 6: Problems List (detail)
- **Type:** Problems
- **Position:** Row 19, Col 0, Width 36, Height 4
- **Config:** Show recent problems, filter to Docker + Linux host groups, limit to 10
- **Purpose:** Detailed problem list when something goes wrong

#### Widget 7 (optional): URL Widget (Uptime Kuma)
- **Type:** URL
- **Position:** Row 19, Col 36, Width 36, Height 4
- **Config:** Embed your Uptime Kuma status page URL
- **Purpose:** External HTTP monitoring alongside Zabbix infrastructure monitoring

### Alternative: Honeycomb Layout

If you prefer a more visual NOC-style dashboard, replace the Trigger Overview (Widget 2) with:

- **Honeycomb widget** showing all container status items
- Configure color thresholds: green for running, red for stopped
- Each hexagon = one container

This looks impressive on a wall-mounted display but is less information-dense than the trigger overview grid.

---

## 9. Implementation Steps

1. **Verify Docker template is applied and discovering containers.** Go to Data Collection > Hosts > your host > Discovery rules. Confirm `docker.containers.discovery` has found all containers.

2. **Create a host group** called "Soul Journeys Docker" and add your host to it. This makes dashboard filtering cleaner.

3. **Create the dashboard:** Monitoring > Dashboards > hamburger menu (top right) > Create new. Name it "Soul Journeys Services".

4. **Add widgets one by one** following the specifications above. Start with Trigger Overview (most valuable) and Problems by Severity. Add others as needed.

5. **Set refresh rate:** 30 seconds is a good balance. Configure in dashboard properties.

6. **Set as default:** If desired, set this as your home dashboard under User Settings.

---

## 10. Open Questions for User

1. **Container inventory:** Exactly which containers should be explicitly listed? List all container names as they appear in Docker (use `docker ps --format '{{.Names}}'` to get exact names). The template discovers them automatically but you may want to filter the dashboard to specific ones.

2. **Uptime Kuma embedding:** Do you want to embed Uptime Kuma's status page inside the Zabbix dashboard (via URL widget)? If so, what is the status page URL? Note: this requires Uptime Kuma to be accessible from the Zabbix web frontend.

3. **Alerting scope:** Should the dashboard only show Docker container problems, or also Linux host problems (disk space, CPU, memory at the OS level)?

4. **Web monitoring:** Do you want Zabbix to also do HTTP checks on your services (website, shop, n8n, Mautic) in addition to Docker container status? This would catch cases where a container is "running" but the service inside is broken. If yes, we should set up Zabbix Web scenarios.

5. **Dashboard audience:** Is this dashboard just for you, or will it be shared/displayed on a screen? This affects layout density and refresh rate choices.

6. **Coolify API template:** Your custom Coolify API template -- what items/triggers does it provide? These should be integrated into the dashboard as well.

7. **Honeycomb vs Trigger Overview:** Do you prefer the hexagonal visual (Honeycomb) or the table grid (Trigger Overview) for the container status section? Both give red/green, but look and feel are different.

8. **Historical data:** How much historical data do you want visible on the dashboard graphs? Options: 1 hour, 4 hours, 24 hours, 7 days.

---

## 11. Sources Index

### Official Zabbix Documentation
- [Dashboard Widgets Overview (7.4)](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets)
- [Dashboards (7.4)](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards)
- [What's New in Zabbix 7.4](https://www.zabbix.com/documentation/current/en/manual/introduction/whatsnew)
- [What's New in Zabbix 7.4 (marketing page)](https://www.zabbix.com/whats_new_7_4)
- [What's New in Zabbix 7.2](https://www.zabbix.com/whats_new_7_2)
- [What's New in Zabbix 7.0](https://www.zabbix.com/whats_new_7_0)
- [Trigger Overview Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/trigger_overview)
- [Host Availability Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/host_availability)
- [Honeycomb Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/honeycomb)
- [Problems by Severity Widget](https://www.zabbix.com/documentation/current/en/manual/api/reference/dashboard/widget_fields/problems_severity)
- [Problem Hosts Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/problem_hosts)
- [Problems Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/problems)
- [Host Card Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/host_card)
- [Item Card Widget (7.4)](https://www.zabbix.com/documentation/current/en/manual/api/reference/dashboard/widget_fields/item_card)
- [Gauge Widget](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/gauge)
- [Data Overview Widget (DEPRECATED)](https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/dashboards/widgets/data_overview)
- [Docker Integration](https://www.zabbix.com/integrations/docker)
- [Container Monitoring](https://www.zabbix.com/container_monitoring)

### Zabbix Blog and Presentations
- [Docker Container Monitoring With Zabbix (Blog)](https://blog.zabbix.com/docker-container-monitoring-with-zabbix/20175/)
- [Zabbix 7.0 Everything You Need to Know (Blog)](https://blog.zabbix.com/zabbix-7-0-everything-you-need-to-know/28210/)
- [Zabbix 7.2 Features (Blog)](https://blog.zabbix.com/see-whats-possible-in-zabbix-7-2/29373/)
- [What's New in Zabbix 7.4 (Blog)](https://blog.zabbix.com/whats-new-in-zabbix-7-4/30597/)
- [Interactive Dashboard Creation (Blog)](https://blog.zabbix.com/interactive-dashboard-creation-for-large-organizations-and-msps/30132/)
- [Dashboards: Single Pane of Glass for MSP (Summit 2022 PDF)](https://assets.zabbix.com/files/events/2022/zabbix_summit_2022/Karlis_Salins_Zabbix_dashboards_A_single_pane_of_glass_for_MSP_environments.pdf)

### Community and Third-Party
- [Why I Switched to Zabbix for Docker Containers (VirtualizationHowTo)](https://www.virtualizationhowto.com/2025/11/why-i-switched-to-zabbix-for-monitoring-my-docker-containers/)
- [Best Practices for Zabbix Dashboard Designs (Medium)](https://medium.com/devsecops-community/best-practices-for-zabbix-dashboard-designs-enhancing-monitoring-visibility-and-efficiency-506a2d34e577)
- [Monitoring Docker with Zabbix (DevOps.dev)](https://blog.devops.dev/monitoring-docker-containers-with-zabbix-481d4471c5c4)
- [How to Monitor Docker Using Zabbix (DigitalOcean)](https://www.digitalocean.com/community/tutorials/how-to-monitor-docker-using-zabbix-on-ubuntu-20-04)
- [Docker Container Resource Widget (Zabbix Forum)](https://www.zabbix.com/forum/zabbix-help/465622-create-widget-showing-resource-intensive-docker-containers-across-multiple-hosts)
- [Docker Template Source (GitHub)](https://github.com/zabbix/zabbix/blob/master/templates/app/docker/template_app_docker.yaml)
- [Coolify Monitoring Docs](https://coolify.io/docs/knowledge-base/monitoring)
- [Coolify + Netdata Tutorial](https://peturgeorgievv.com/blog/how-to-deploy-monitoring-on-your-vps-with-coolify-and-netdata)
- [Coolify Container Resource Monitoring (GitHub)](https://github.com/haydnjm/coolify-container-resource-monitoring)
- [Zabbix Docker Monitoring (Grafana Dashboard)](https://grafana.com/grafana/dashboards/12737-docker/)
- [What's New in Zabbix 7.4 (Hawatel)](https://hawatel.com/en/blog/whats-new-in-zabbix-7-4/)
