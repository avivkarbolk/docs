# Alerts Documentation

## Summary

Kubecost alerts allow teams to receive updates on real-time Kubernetes spend. They are configurable via the Kubecost UI or via Helm values. This resource gives an overview of how to configure Kubecost email and Slack alerts using [Kubecost helm chart values](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml).
  

As of v1.72.0, Kubecost supports four types of notifications:

 1. [Recurring update](#type-recurring-update) - sends an email and/or Slack alert with cluster spend across all or a set of namespaces, with costs broken down by namespace

 2. [Budget](#type-budget) -- sends an email and/or Slack alert when spend crosses a defined threshold

 3. [Spend Change](#type-spend-change) -- sends an email and/or Slack alert reporting unexpected spend increases relative to moving averages
 
 4. [Health Diagnostic](#type-health-diagnostic) -- used for production monitoring for the health of Kubecost itself
  
## Configuring Alerts in Helm

*Note: `values.yaml` is a source of truth. Alerts set through `values.yaml` will continually overwrite any manual alert settings set through the Kubecost UI.* 

### Global Alert Parameters  
The alert settings, under `global.notifications.alertConfigs` in `cost-analyzer/values.yaml`, accept four global fields:

* `enabled` determines whether Kubecost will schedule and display the configured alerts in Notifications, default set to `false`     
* `frontendUrl` optional, your cost analyzer front end URL used for linkbacks in alert bodies
* `globalSlackWebhookUrl` optional, a global Slack webhook used for alerts, enabled by default if provided
* `globalAlertEmails` a global list of emails for alerts

#### Example Helm values.yaml

```
notifications:
	alertConfigs:
		enabled: true # the settings under alertConfigs are never scheduled unless enabled is set to true
		frontendUrl: http://localhost:9090  # optional, used for linkbacks
		globalSlackWebhookUrl: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX  # optional, used for Slack alerts
		globalAlertEmails:
			- recipient@example.com
			- additionalRecipient@example.com
		alerts:
		# list of individual alerts
		...
```

### Configuring Each Alert Type

In addition to `globalSlackWebhookUrl` and `globalAlertEmails` fields, every alert allows optional individual `ownerContact` (a list of email addresses) and `slackWebhookUrl` (if different from `globalSlackWebhookUrl`) fields. Alerts will default to the global Slack and email settings if these optional fields are not supplied.

#### Type: Recurring Update

Required parameters (all namespaces):

- `type: recurringUpdate`
- `aggregation: namespace`
- `filter: '*'`
- `window: 7d`

Required parameters (by individual namespace):

- `type: recurringUpdate`
- `aggregation: namespace`
- `filter: <value>` -- configurable, accepts a single namespace name (comma separated values unsupported)
- `window: 7d`

#### Example Helm values.yaml

```
			# Recurring weekly namespace update on all namespaces
			- type: recurringUpdate
				window: weekly  # or 7d
				aggregation: namespace
				filter: '*'
			# Recurring weekly namespace update on kubecost namespace
			- type: recurringUpdate
				window: weekly  # or 7d
				aggregation: namespace
				filter: kubecost
				ownerContact: # optional, overrides globalAlertEmails default
					- owner@example.com
					- owner2@example.com
				slackWebhookUrl: https://hooks.slack.com/services/<different-from-global> # optional, overrides globalSlackWebhookUrl default
```

#### Type: Budget

Define spend budgets and alert on budget overruns.

Required parameters:

- `type: budget`
- `threshold: <amount>` -- cost threshold in configured currency units
- `aggregation: <agg-parameter>` -- configurable, accepts all aggregations supported by the [aggregated cost model API](https://github.com/kubecost/docs/blob/2ea9021e8530369d53184ea5382b2e4c080bb426/allocation-api.md#aggregated-cost-model-api)
- `filter: <value>` -- configurable, accepts a single filter value (comma separated values unsupported)
- `window: <N>d` or `<M>h` -- configurable, (1 ≤ N ≤ 7, 1 ≤ M ≤ 24)

#### Example Helm values.yaml

```
			# Daily namespace budget alert on namespace `kubecost`
			- type: budget
				threshold: 50
				window: daily # or 1d
				aggregation: namespace
				filter: kubecost
			# 3d cluster budget alert on cluster `cluster-one`
			- type: budget
				threshold: 600
				window: 3d
				aggregation: cluster
				filter: cluster-one
```

#### Type: Spend Change

Detect unexpected spend increases/decreases relative to historical moving averages.

Required parameters:

- `type: spendChange`
- `relativeThreshold: <N>` -- configurable, N ≥ -1
- `aggregation: <agg-value>` -- configurable, accepts all aggregations supported by the [aggregated cost model API](https://github.com/kubecost/docs/blob/2ea9021e8530369d53184ea5382b2e4c080bb426/allocation-api.md#aggregated-cost-model-api)
- `filter: <value>` -- configurable, **accepts comma separated values**
- `window: <N>d` or `<M>h` -- configurable, (1 ≤ N ≤ 7, 1 ≤ M ≤ 24)
- `baselineWindow: <N>d` -- configurable, N ≥ 1

### Example Helm values.yaml

```
			# Daily spend change alert on the 
			- type: spendChange
			  relativeThreshold: 0.20  # change relative to baseline average cost. Must be greater than -1 (can be negative).
			  window: 1d                # accepts ‘d’, ‘h’
			  baselineWindow: 30d       # previous window, offset by window
			  aggregation: namespace
			  filter: kubecost, default # accepts csv
```

#### Type: Health Diagnostic

Enabling diagnostic alerts in Kubecost sends a Slack message (email coming soon) when an event impacts product uptime. This feature can be enabled in seconds from a values file. The following health events are detected:

* Prometheus is unreachable
* Kubecost metrics missing over last 5 minutes
* More coming soon.

To enable, provide a `globalSlackWebhookUrl` value and set this [flag](https://github.com/kubecost/cost-analyzer-helm-chart/blob/31dc60d2c539720f2b2a72c8e22b2f6b866580bd/cost-analyzer/values.yaml#L31) to `true`

## Alerts Scheduler

All times in UTC.

The back end scheduler runs if `alertConfigs.enabled` is set to `true`, and alerts at regular times of day. Alert send times are determined by parsing the supplied `window` parameter. If `alertConfigs.enabled` is `false`, alerts are still configurable via the Kubecost UI Notifications tab, and will send alerts through the front end cron job scheduler.

Alert diagnostics with next and last scheduled run times are only available for alerts configured via Helm, and are viewed via `<your-kubecost-url>/model/getCustomAlertDiagnostics`.

Supported: `weekly` and `daily` special cases, `<N>d`, `<M>h` (1 ≤ N ≤ 7, 1 ≤ M ≤ 24)
Currently Unsupported: time zone adjustments, windows greater than `7d`, windows less than `1h`

### Scheduler Behavior

An `<N>d` alert sends at 00:00 UTC N day(s) from now, i.e., N days from now rounded down to midnight. 

For example, a `5d` alert scheduled on Monday will send on Saturday at 00:00, and subsequently the next Thursday at 00:00

An `<N>h` alert sends at the earliest time of day after now that is a multiple of N.

For example, a `6h` alert scheduled at any time between 12pm and 6pm will send next at 6pm and subsequently at 12am the next day.

If 24 is not divisible by the hourly window, schedule at next multiple of `<N>h` after now, starting from current day at 00:00.

For example, a `7h` alert scheduled at 22:00 checks 00:00, 7:00, 14:00, and 21:00, before arriving at a next send time of 4:00 tomorrow.

## Troubleshooting

Review these steps to verify alerts are being passed to the Kubecost application correctly.

First, ensure that the Helm values are successfully read into the configmap:

-   Confirm that the `global.notifications.alertConfigs.enabled` field is set to `true`
-   Run `helm template ./cost-analyzer -n kubecost > test-alert-config.yaml`
-   Open `test-alert-config.yaml`
-   Find the section starting with `# Source: cost-analyzer/templates/cost-analyzer-alerts-configmap.yaml`
-   Ensure that the Helm values are successfully read into the configmap under the `data` field.
-   Example:

```
# Source: cost-analyzer/templates/cost-analyzer-alerts-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
name: alert-configs
labels:
	app.kubernetes.io/name: cost-analyzer
	helm.sh/chart: cost-analyzer-1.68.1
	app.kubernetes.io/instance: RELEASE-NAME
	app.kubernetes.io/managed-by: Helm
app: cost-analyzer
data:
	alerts.json: <all-fields-and-values-under-alertConfigs-as-json-string>
```

-   Ensure that the json string is successfully mapped to the appropriate configs

Next, confirm that Kubecost product has received configuration data:

- Go to `<your-kubecost-url>/notify.html` in the Kubecost UI to view configured email and Slack settings, weekly updates, namespace updates, cluster budget, and namespace budget alerts.

Additionally, confirm that the alerts scheduler has properly parsed and scheduled a next run for each custom alert by visiting `<your-kubecost-url>/model/getCustomAlertDiagnostics` to view individual alert parameters as well as next and last scheduled run times for individual alerts.

- Confirm that `nextRun` has been updated from "0001-01-01T00:00:00Z"
- Settings that do not appear in the Notifications page (for example, `spendChange` alerts), but are visible from the `/model/getCustomAlertDiagnostics` endpoint are scheduled to send.

If `nextRun` fails to update, or alerts are not sending at the `nextRun` time, check pod logs by running `kubectl logs $(kubectl get pods -n kubecost | awk '{print $1}' | grep "^kubecost-cost-analyzer.\{16\}") -n kubecost -c cost-model > kubecost-logs.txt`

- Common causes of misconfiguration include the following:
	- Not setting `.Values.global.notifications.alertConfigs.enabled` to `true` -- Kubecost will not read the resulting configmap
	- unsupported csv filters -- `spendChange` alerts accept `filter` as comma-separated values; other alert types do not.
	- unsupported alert type -- all alert type names are in camelCase -- check spelling and capitalization for all alert parameters
	- unsupported aggregation parameters -- see the [aggregated cost model API](https://github.com/kubecost/docs/blob/2ea9021e8530369d53184ea5382b2e4c080bb426/allocation-api.md#aggregated-cost-model-api) for details
