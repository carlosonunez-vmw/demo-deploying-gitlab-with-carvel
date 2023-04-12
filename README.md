## Deploying GitLab with Tekton and VMware Application Catalog

This demo shows you how you can use Tekton to deploy a GitLab
infrastructure entirely within Kubernetes.

### Watch the Demo


[![Watch the video](https://img.youtube.com/vi/1RzjV71VKO8/default.jpg)](https://youtu.be/1RzjV71VKO8)

#### Tech You'll Use

- Tanzu Kubernetes Grid
- Tekton
- Helm
- [`kapp`](https://carvel.io/kapp)

### Caveats

- We assume that you have a TKG-flavored Kubernetes cluster running and
  accessible within your default Kubeconfig. You'll need to set that up if you
  do not.
- We're not going to go super deep into configuring GitLab in this demo.
  It _IS_ possible to configure it with Kubernetes, though!

### Demo

#### Configure /etc/hosts (if needed)

If your Kubernetes cluster's ingress controller creates IPs that are not tied
to DNS, create a static entry in your `/etc/hosts` file:

```sh
sudo sh -c 'cat [IP_ADDRESS] cluster.example.local >> /etc/hosts'
```

To get your IP address, run:

```sh
kubectl -n tanzu-system-ingress get service envoy
```


#### Install `kapp`

First, install the `kapp` client into your machine system-wide:

```sh
sudo sh -c 'curl -L https://carvel.dev/install.sh | bash'
```

#### Install Tekton

Next, use `kapp` to package and install Tekton into your cluster.

First, create the Tekton namespace:

```sh
kubectl create ns tekton
```

Next, add the Tekton Helm chart repo:

```sh
helm repo add cdf https://cdfoundation.github.io/tekton-helm-chart
```

Next, use `helm template` to render Kubernetes manifests from the Tekton
Helm chart for `kapp` to consume:

```sh
helm template cdfoundation/tekton-pipelines -n tekton |
    kapp deploy -n tekton -a tekton -f - -c -y
```

Finally, run `kapp list -A` to see how `kapp` is tracking this app.

### Deploy the GitLab pipeline

Create a namespace for our pipelines:

```sh
kubectl create ns pipelines
```

Our pipeline has a step that retrieves a kubeconfig from an external source.

However, we're not really doing this. Instead, we're retrieving the kubeconfig
from within the cluster through a configuration file stored in a `ConfigMap`.

It doesn't currently exist. Let's create it.

```sh
kubectl create configmap kubeconfig -n pipelines --from-file=$HOME/.kube/config
```

Use `kapp` to deploy our GitLab pipeline. This allows us to track the GitLab
pipeline as an app and declaratively update it whenever we want to make changes
to it, like supporting new parameters:

```sh
kapp deploy -n pipelines -a gitlab-pipeline -f ./gitlab_pipeline.yaml - \
    -c -y
```

### Trigger a new GitLab Pipeline

Let's introduce some new tools into the fold:

- `ytt`: Like Helm's templating engine, but uses Starlark, so immediately
  more powerful

Use `ytt` to render our GitLab Pipeline run.

```sh
ytt -v team_name=foo \
    -v cluster_name=example_cluster \
    -v pipeline_name=gitlab-pipeline \
    -v values=[] \
    -v nonce=$(date +%s) -f ./trigger.yaml | kubectl apply -f -
```

### Watch it go

Use `kubectl` to see the status of our pipelines:

```sh
kubectl get pr -n pipelines
```

You'll get this error:

```text
NAMESPACE   NAME                             SUCCEEDED   REASON             STARTTIME   COMPLETIONTIME
pipelines   gitlab-pipeline-run-1681264653   False       ParameterMissing   2m22s       2m22s
```

Let's dig into why:

```sh
kubectl describe pr gitlab-pipeline-run-$NONCE
```

This will generate:

```text
  Warning  Failed         2m41s  PipelineRun  PipelineRun pipelines parameters is missing some parameters required by Pipeline gitlab-pipeline-run-1681264653's parameters: PipelineRun missing parameters: [num_linux_runners num_windows_runners users_to_create]
  Warning  InternalError  2m40s  PipelineRun  1 error occurred:
           * PipelineRun missing parameters: [num_linux_runners num_windows_runners users_to_create]
```

So it looks like we're missing some variables.

Let's mark those as default.

1. Open `gitlab_pipeline.yaml`
2. Set `users_to_create`, `num_linux_runners` and `num_windows_runners` to zero
   by adding this underneath each key:

```yaml
default: "0"
```

Also, set `users_to_create` to an empty array:

```yaml
default: []
```

3. Re-deploy the `gitlab-pipeline` kapp:

```sh
kapp deploy -n pipelines -a gitlab-pipeline -f ./gitlab_pipeline.yaml - \
    -c -y
```

Notice how `kapp` handles updating the resource in place. Also notice the diff
that `kapp` produces.

#### Let's try again

Let's check on our pipelines now:

```sh
kubectl get pr -n pipelines
```

We should see them in a `Running` state:

```sh
NAMESPACE   NAME                             SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
pipelines   gitlab-pipeline-run-1681266871   Unknown     Running   109s
```

Every pipeline task gets executed as a `TaskRun`, which is run inside of a
`Pod`. Let's check that out:

```sh
NAME                                                              READY   STATUS     RESTARTS   AGE
gitlab-pipeline-run-1681266871-get-cluster-kubeconfig-tm4-q2m9t   0/1     Init:0/2   0          2m38s
```

We can also find the `Pod` name from the YAML representation of this pipeline:

```sh
kubectl get pr -A -o yaml
```

Which yields:

```text
# lots of text
    taskRuns:
      gitlab-pipeline-run-1681266871-get-cluster-kubeconfig-tm4gq:
```

#### Let's Visualize

We've been working with Tekton through `kubectl` so far. This is fun and very
automation friendly, but inconvenient for quick tasks.

Let's install the Tekton Dashboard and Tekton CLI.

First, the Dashboard:

```sh
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml
```

Next, the CLI, `tkn`

```sh
choco install tektoncd-cli --confirm # Windows
brew install tektoncd-cli # Mac
```

#### Wait a while

Installing GitLab takes a lot of time! Go for a cup of coffee, or a run, or
both!

When the pipeline finishes, GitLab will be available at
`https://gitlab.cluster.local`.
