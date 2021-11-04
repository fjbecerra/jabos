# jabos

**Just Another Boring Ops System**
Jabos attempts to be a fully automated K8s GitOps framework.

**This is WIP** - any comments, requests or issues would be welcome! please use [this link](https://github.com/srfrnk/jabos/issues)

## TL;DR - What does that mean?

### What you need to do?

1. Installing Jabos into your K8s cluster using
1. Setting up the K8s objects for your
   - Git Repository
   - Docker images
   - Manifest folder

### What happens next?

1. Any new commits would be picked up from Git automatically
1. Docker images would get build from new commits and pushed automatically
1. New manifest versions would be deployed automatically

## Goals

- Automate all steps to deploy from Git repositories into a K8s cluster
- No GUI requiring manual human intervention
- Git as a single source of truth
- Pull only model from within runtime environments.
- Isolation of build environment from runtime environment
- Idempotent builds
- Preview environment + Pre-deploy integration testing

## Guidelines

- Minimal set of tools/technologies as pre-requisites/installs
- Stay as tech-stack agnostic as possible
- Minimal steps to install or setup development environment. Automated as possible.

## Installation

1. Make sure Metacontroller is installed on your cluster. Find instructions [here](https://metacontroller.github.io/metacontroller/guide/install.html)
1. Create a namespace for jabos to use. E.g. `kubectl create namespace jabos`. Use the same namespace with the next command.
1. Run `kubectl apply -n <NAMESPACE> -f https://github.com/srfrnk/jabos/releases/latest/download/jabos-manifests.yaml`

## Usage

Jabos uses [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) in order for users to define a codebase and how to build and deploy that.

### GitRepository

`GitRepository` objects define a git codebase and how to pull from it.

#### `GitRepository` Properties

- `apiVersion`: Must be "jabos.io/v1"
- `kind`: Must be "GitRepository"
- `metadata`
  - `name`: Name for the object (As with any K8s object)
  - `namespace`: Namespace for the object (As with any K8s object)
- `spec`
  - `url`: The URL to use to access the git repository.
  - `branch`: The name of the branch to watch and pull from.
  - `ssh`: Credentials for ssh access
    - `secret`: Name of secret to use. **Must be in the same namespace**
    - `passphrase`: Name of the key inside the secret to use for ssh passphrase. Default: "passphrase"
    - `key`: Name of the key inside the secret to use for ssh key. Default: "key"

E.g.:

```yaml
apiVersion: jabos.io/v1
kind: GitRepository
metadata:
  name: first-repo
  namespace: example-env
spec:
  url: https://github.com/srfrnk/jabos-examples.git
  branch: main
```

### DockerImage

`DockerImage` objects define images, how to build and push them.

#### `DockerImage` Properties

- `apiVersion`: Must be "jabos.io/v1"
- `kind`: Must be "DockerImage"
- `metadata`
  - `name`: Name for the object (As with any K8s object)
  - `namespace`: Namespace for the object (As with any K8s object)
- `spec`
  - `gitRepository`: The name of a `GitRepository` object which defines the codebase for the image. **Must be in the same namespace**.
  - `contextPath`: The path **relative to the git repository root** where the context for the docker build is found. Default: "." (Repository root)
  - `dockerFile`: The name of the Dockerfile to use for the docker build. Default: "Dockerfile"
  - `imageName`: The full image name excluding the tag part. Usually this will be prefixed with the registry address (e.g. `registry.kube-system:80/example-image`)
  - `insecureRegistry`: "True" to use insecure registries with docker push. Default: False
  - `dockerConfig`: Optional docker config yaml file to use. This allows configuring `Kaniko` access to external registries if required. **Be careful not to include any secret information as this value is logged.**

E.g.:

```yaml
apiVersion: jabos.io/v1
kind: DockerImage
metadata:
  name: first-image
  namespace: example-env
spec:
  gitRepository: first-repo
  contextPath: simple-runtime/images
  imageName: registry.kube-system:80/example-image
```

### JsonnetManifest

`JsonnetManifest` objects define a folder with [jsonnet](https://jsonnet.org/) based manifests to deploy.

#### `JsonnetManifest` Properties

- `apiVersion`: Must be "jabos.io/v1"
- `kind`: Must be "JsonnetManifest"
- `metadata`
  - `name`: Name for the object (As with any K8s object)
  - `namespace`: Namespace for the object (As with any K8s object)
- `spec`
  - `gitRepository`: The name of a `GitRepository` object which defines the codebase for the manifests. **Must be in the same namespace**.
  - `path`: The folder path **relative to the git repository root** where the jsonnet manifests are found. Default: "." (Repository root)
  - `commitTLAKey`: The name of a [Top-Level Argument (TLA)](https://jsonnet.org/ref/language.html#top-level-arguments-tlas) to use for injecting the latest git commit hash. The commit hash is used to tag generated images for example. See usage example below. Default: "latestCommitHash"
  - `targetNamespace`: The namespace into which deployment should be made. **Important Note: Due to a limitation with [Metacontroller](https://github.com/metacontroller/metacontroller) this has to be the same namespace as this object is put into.**

E.g.:

```yaml
apiVersion: jabos.io/v1
kind: JsonnetManifest
metadata:
  name: simple-jsonnet-manifest
  namespace: example-env
spec:
  gitRepository: first-repo
  path: simple-runtime/manifests
  targetNamespace: example-env
  commitTLAKey: latestCommitId
```

`example.jsonnet`:

```jsonnet
function(latestCommitId) {
  apiVersion: 'apps/v1',
  kind: 'Deployment',
  metadata: {
    name: 'test-deployment',
    labels: {
      app: 'test-deployment',
    },
  },
  spec: {
    replicas: 1,
    selector: {
      matchLabels: {
        app: 'test-deployment',
      },
    },
    template: {
      metadata: {
        labels: {
          app: 'test-deployment',
        },
      },
      spec: {
        containers: [
          {
            name: 'test-deployment',
            image: 'registry.kube-system:80/example-image:' + latestCommitId,
          },
        ],
      },
    },
  },
}
```

## Github Repository Authentication

### Using SSH Keys

1. [Create an SSH key and add it to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) - **Optionally skip the "adding it to the ssh-agent" section.**
1. Create a secret with the passphrase and key created in the previous step. (i.e. `kubectl create secret generic -n example-env first-repo-private --from-file=passphrase=./build/passphrase --from-file=key=./build/key`)
1. Deploy a `GitRepository` with the `ssh` configuration to point to the secret created in the previous step. E.g.:

```yaml
apiVersion: jabos.io/v1
kind: GitRepository
metadata:
  name: first-repo-private
  namespace: example-env
spec:
  url: git@github.com:srfrnk/jabos-examples-private.git
  branch: main
  ssh:
    secret: first-repo-private
```

### Using Deploy Keys (**Not yet supported**)

1. Follow instruction [here](https://docs.github.com/en/developers/overview/managing-deploy-keys#setup-2)

### Metrics

All metrics are exported into `Prometheus` using the `ServiceMonitor` API by `kube-prometheus-stack`.
To otherwise configure `Prometheus` to collect the metrics you need to point it to 'OPERATOR_POD_IP:3000/metrics'.

All metrics exported are prefixed with `jabos_operator_`.
Numerous metrics are exported most of them describe `nodsjs` and `expresjs` operations.

Important metrics for the operation of Jabos are:

```yaml
# HELP jabos_operator_latest_commit_changed new "latest commit" detected for git repository
# TYPE jabos_operator_latest_commit_changed counter

# HELP jabos_operator_docker_image_build_trigger new build triggered for a docker image
# TYPE jabos_operator_docker_image_build_trigger counter

# HELP jabos_operator_jsonnet_manifests_build_trigger new build triggered for jsonnet manifests
# TYPE jabos_operator_jsonnet_manifests_build_trigger counter

# HELP jabos_operator_git_repository_updater_start GitRepositoryUpdater start
# TYPE jabos_operator_git_repository_updater_start counter

# HELP jabos_operator_git_repository_updater_end GitRepositoryUpdater end
# TYPE jabos_operator_git_repository_updater_end counter

# HELP jabos_operator_git_repository_updater_duration GitRepositoryUpdater duration
# TYPE jabos_operator_git_repository_updater_duration gauge

# HELP jabos_operator_docker_image_builder_start DockerImageBuilder start
# TYPE jabos_operator_docker_image_builder_start counter

# HELP jabos_operator_docker_image_builder_end DockerImageBuilder end
# TYPE jabos_operator_docker_image_builder_end counter

# HELP jabos_operator_docker_image_builder_duration DockerImageBuilder duration
# TYPE jabos_operator_docker_image_builder_duration gauge

# HELP jabos_operator_jsonnet_manifests_builder_start JsonnetManifestsBuilder start
# TYPE jabos_operator_jsonnet_manifests_builder_start counter

# HELP jabos_operator_jsonnet_manifests_builder_end JsonnetManifestsBuilder end
# TYPE jabos_operator_jsonnet_manifests_builder_end counter

# HELP jabos_operator_jsonnet_manifests_builder_duration JsonnetManifestsBuilder duration
# TYPE jabos_operator_jsonnet_manifests_builder_duration gauge
```

## Development

### Prerequisites

1. `make` installed (Depending on your OS - start [here](https://www.gnu.org/software/make/))
1. `docker` installed (To install wee [here](https://www.docker.com/get-started))
1. `minikube` installed (To install minikube see [this](https://minikube.sigs.k8s.io/docs/start/))
1. `NodeJS` installed (To install NodeJS see [this](https://nodejs.org))
1. `Typescript` development tools installed `npm install -g ts-node typescript '@types/node'`
1. `GNU Parallel` installed for [your OS](https://www.gnu.org/software/parallel/). For Debian based you can use `sudo apt-get install parallel`.

### Environment Setup

1. Clone repo: `git clone git@github.com:srfrnk/jabos.git` (or using HTTPS/GitHub CLI - see instructions [here](https://github.com/srfrnk/jabos))
1. CD into folder
1. Start a minikube cluster `minikube start`
1. Run `make setup` once
1. Run `make build` after each code change
1. Run terminal with `kubectl port-forward -n efk svc/efk-kibana 5601` then open [kibana](http://localhost:5601/app/discover)
1. To deploy examples
   1. Locally clone [jabos-examples repo](https://github.com/srfrnk/jabos-examples)
   1. Follow instructions from the README file in the cloned folder
   1. Locally clone [jabos-examples-private repo](https://github.com/srfrnk/jabos-examples-private)
   1. Follow instructions from the README file in the cloned folder
   1. Back within the `jabos` folder (from second step) run `make deploy-examples`
   1. The examples would be deployed into namespace `example-env`

## Credits

- Jabos uses [the kaniko project](https://github.com/GoogleContainerTools/kaniko) to build docker images inside the kubernetes cluster.
- Jabos uses [yq](https://github.com/mikefarah/yq) to parse and update yaml and json data.
- Jabos uses [jsonnet](github.com/google/go-jsonnet) to process jsonnet templates and create K8s manifests.
- Jabos uses [minikube](https://github.com/kubernetes/minikube) for local development
- Jabos uses [metacontroller](https://github.com/metacontroller/metacontroller) to control K8s operators.
- Jabos uses [efk-stack-helm](https://github.com/srfrnk/efk-stack-helm) for local centralized logging.
- Jabos uses [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) for local monitoring and alerting.
- Jabos uses [GNU Parallel](https://www.gnu.org/software/parallel/) for local port-forwarding to multiple services
- Jabos uses [expressjs](https://github.com/expressjs/express) as the web server to run the operator
- Jabos uses [express-prometheus-middleware](https://github.com/joao-fontenele/express-prometheus-middleware) to export basic metrics to prometheus
