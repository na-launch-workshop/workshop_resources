# Workshop Runtimes Helm Chart

A Helm chart for provisioning OpenShift resources needed for workshop environments. This chart deploys all required runtime resources into `<user>-devspaces` namespaces, bypassing typical access restrictions.

## Overview

While primarily designed for DevSpaces workshop environments, this chart can provision any cluster resources needed for workshop participants. Resources are deployed into user-specific namespaces following the pattern `<user>-devspaces`.

## What's Included

This chart provisions the following resources:

- **MinIO**: S3-compatible object storage (API on port 9000, console on 9001)
- **Kafka**: Strimzi-managed Kafka cluster with auto-topic creation
- **Kafka Topics**: Pre-configured topics for workshop exercises
- **Kafka UI**: Web-based Kafka management interface
- **KEDA**: Kubernetes Event-Driven Autoscaling for Kafka topics
- **Tekton Pipelines**: CI/CD pipelines for Spring Boot applications
  - Maven build and package
  - Container image build
  - Application deployment
- **Tekton Triggers**: Automated pipeline triggers for git pushes
- **Routes**: OpenShift routes for API and console access
- **RBAC**: Necessary service accounts and role bindings

## Usage

### Installation

#### Option 1: Using the Published Helm Repository (Recommended)

First, add the workshop Helm repository to your OpenShift cluster:

```bash
oc apply -f https://raw.githubusercontent.com/na-launch-workshop/platform-workshop-runtimes/master/install.yaml
```

Then install the chart to a user's namespace:

```bash
helm install <release-name> workshop-charts/workshop-runtimes -n <user>-devspaces
```

For example:
```bash
helm install runtimes workshop-charts/workshop-runtimes -n alice-devspaces
```

#### Option 2: Install Directly from Source

Clone the repository and deploy the chart:

```bash
git clone https://github.com/na-launch-workshop/platform-workshop-runtimes.git
cd platform-workshop-runtimes
helm install <release-name> . -n <user>-devspaces
```

### Upgrade

Update an existing deployment:

```bash
# From the published repository
helm upgrade <release-name> workshop-charts/workshop-runtimes -n <user>-devspaces

# Or from local source
helm upgrade <release-name> . -n <user>-devspaces
```

### Uninstall

Remove all provisioned resources :

```bash
helm uninstall <release-name> -n <user>-devspaces
```

## Configuration

Edit `values.yaml` to customize the deployment:

```yaml
replicaCount: 1

image:
  repository: quay.io/minio/minio
  tag: latest
  pullPolicy: IfNotPresent

env:
  rootUser: admin
  rootPassword: admin123

service:
  type: ClusterIP
  ports:
    api: 9000
    console: 9001
```

### Key Configuration Options

- **MinIO Credentials**: Change `env.rootUser` and `env.rootPassword`
- **Image Version**: Update `image.tag` for specific MinIO versions
- **Resource Limits**: Add resource requests/limits in `resources`
- **Node Placement**: Configure `nodeSelector`, `tolerations`, `affinity`

## Adding New Resources

To add new workshop resources:

1. Create a new template file in `templates/` (e.g., `templates/myresource.yaml`)
2. Use Helm templating for namespace injection: `namespace: {{ .Release.Namespace }}`
3. Add configuration values to `values.yaml` if needed
4. Deploy with `helm upgrade`

## Tekton Pipelines

The chart includes a complete CI/CD stack for Spring Boot applications:

### Pipeline: springboot-build-deploy

1. **maven-package**: Clones git repo and runs Maven build
2. **build-image**: Creates container image using Buildah
3. **deploy-app**: Deploys to OpenShift with the image

### Triggers

Automated triggers respond to git webhooks and start pipelines automatically:
- `workshop-springboot-hello_by_lang` - Standard Spring Boot app
- `workshop-springboot-using_camel` - Camel integration app

## Architecture

```
<user>-devspaces namespace
├── MinIO (Object Storage)
│   ├── API Service (:9000)
│   └── Console Service (:9001)
├── Kafka Cluster
│   ├── Kafka Broker (:9092)
│   ├── Zookeeper
│   └── Topics (auto-created)
├── Kafka UI (Management)
├── KEDA ScaledObjects
└── Tekton CI/CD
    ├── Pipeline Definitions
    ├── Task Definitions
    ├── Triggers (EventListener)
    └── PersistentVolumeClaims
```

## Publishing

This chart is automatically published to GitHub Pages on every push to the `master` branch via GitHub Actions. The workflow:

1. Packages the Helm chart
2. Generates the Helm repository index
3. Publishes to `https://na-launch-workshop.github.io/platform-workshop-runtimes`

Users can then install via the HelmChartRepository CR in `install.yaml`.

## Development

### Chart Structure

```
.
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── install.yaml            # HelmChartRepository CR for OpenShift
├── templates/              # Kubernetes resource templates
│   ├── deployment.yaml     # MinIO deployment
│   ├── service.yaml        # MinIO services
│   ├── route-*.yaml        # OpenShift routes
│   ├── kafka.yaml          # Kafka cluster
│   ├── kafka-topics.yaml   # Topic definitions
│   ├── kafka-ui.yaml       # Kafka UI
│   ├── keda-*.yaml         # KEDA autoscaling
│   └── tekton-*.yaml       # CI/CD pipelines
└── .github/
    └── workflows/
        └── helm-release.yaml  # Auto-publish workflow
```

### Testing Changes

1. Make changes to templates or values
2. Validate with `helm template . --debug`
3. Test deployment in a dev namespace
4. Verify resources are created: `oc get all -n <namespace>`
5. Push to `master` to trigger automatic publishing to the Helm repository

## Common Tasks

### Check Deployed Resources

```bash
helm list -n <user>-devspaces
oc get all -n <user>-devspaces
```

### View MinIO Console

```bash
oc get route console -n <user>-devspaces
```

### Monitor Tekton Pipelines

```bash
tkn pipeline list -n <user>-devspaces
tkn pipelinerun logs -f -n <user>-devspaces
```

### Access Kafka

```bash
oc get kafka -n <user>-devspaces
oc get kafkatopic -n <user>-devspaces
```

## Troubleshooting

### Resources Not Creating

Check Helm deployment status:
```bash
helm status <release-name> -n <user>-devspaces
```

### Pipeline Failures

View pipeline run logs:
```bash
tkn pipelinerun ls -n <user>-devspaces
tkn pipelinerun logs <pipelinerun-name> -n <user>-devspaces
```

### Kafka Issues

Check Kafka operator logs:
```bash
oc logs -l name=strimzi-cluster-operator -n openshift-operators
```

## Notes

- Resources use ephemeral storage by default (suitable for workshops)
- Kafka topics are auto-created when accessed
- The chart assumes Strimzi Kafka operator is installed cluster-wide
- Tekton operator must be installed for pipeline resources
- KEDA operator required for autoscaling features

## License

This chart is for workshop/educational use.
