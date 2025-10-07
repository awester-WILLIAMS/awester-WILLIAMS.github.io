# Monitoring and Alerting

- [Observability Overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/observability_overview/index)
- Get a health check of our current clusters, and setup, to determine action items
    - [Gathering Baseline Openshift Usage Information from Metrics](https://myopenshiftblog.com/gathering-baseline-openshift-usage-information-from-metrics/)
    - [Openshift Monitoring, Logging, Observability, and Troubleshooting (Part 1)](https://myopenshiftblog.com/openshift-observability/)

## OpenShift Monitoring Stack

OpenShift Container Platform includes a pre-configured, pre-installed, and self-updating monitoring stack that provides monitoring for core platform components.

### Key Components

- **Prometheus**: Time-series database and monitoring system
- **Alertmanager**: Handles alerts and routes them to receivers
- **Grafana**: Visualization and dashboarding tool
- **Thanos Querier**: Long-term storage and high-availability for metrics
- **Node Exporter**: Collects hardware and OS metrics
- **kube-state-metrics**: Generates metrics about Kubernetes objects

## Monitoring Types

OpenShift provides two types of monitoring:

1. **Cluster Monitoring**: Monitors OpenShift cluster components (enabled by default)
2. **User Workload Monitoring**: Monitors user-defined projects (requires explicit enablement)

## Accessing Monitoring Tools

```bash
# Access routes for monitoring tools
oc get routes -n openshift-monitoring
```

### Key Interfaces

- **Prometheus UI**: Access metrics and run PromQL queries
- **Alertmanager UI**: View and manage active alerts
- **Grafana**: View pre-built dashboards (restricted to cluster admins)

## Configuring Alerts

Alerts in OpenShift are defined using Prometheus AlertRules.

### Default Alerting Rules

OpenShift comes with predefined alerts for:
- Cluster availability issues
- Resource constraints (CPU, memory, storage)
- Control plane component issues
- Authentication and certificate problems

### Custom AlertRules Example

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-alert
  namespace: ns1
spec:
  groups:
  - name: example
    rules:
    - alert: HighRequestLatency
      expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: High request latency
```

## User Workload Monitoring

To enable monitoring for your applications:

1. Enable user workload monitoring in the cluster monitoring ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

2. Create a ServiceMonitor or PodMonitor resource:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  namespace: myapp
spec:
  endpoints:
  - interval: 30s
    port: web
    scheme: http
  selector:
    matchLabels:
      app: example-app
```

## Alerting Configuration

### Alert Routing

Configure AlertManager to route alerts to external systems:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-main
  namespace: openshift-monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
    route:
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: default
      routes:
      - match:
          severity: critical
        receiver: pagerduty
    receivers:
    - name: default
    - name: pagerduty
      pagerduty_configs:
      - service_key: <pagerduty_service_key>
```

### Receivers

AlertManager supports various receiver types:
- Email
- PagerDuty
- Slack/Teams
- Webhook
- OpsGenie
- VictorOps

## Best Practices

- Start with default alerts before adding custom ones
- Set appropriate thresholds to avoid alert fatigue
- Implement proper alert routing and escalation policies
- Use alert grouping and silencing for maintenance periods
- Define clear runbooks for each alert type
- Regularly review and tune alert thresholds

## Troubleshooting Monitoring Issues

```bash
# Check monitoring operator status
oc -n openshift-monitoring get pods

# View Prometheus logs
oc -n openshift-monitoring logs prometheus-k8s-0

# Check AlertManager status
oc -n openshift-monitoring logs alertmanager-main-0

# View current alerts
oc -n openshift-monitoring port-forward svc/alertmanager-main 9093
# Then access http://localhost:9093 in your browser
```
