## Deploying GitLab with Tekton and VMware Application Catalog

This demo shows you how you can use Tekton to deploy a hardened GitLab infrastructure entirely within Kubernetes.

### Tech You'll Use

- Tanzu Kubernetes Grid
- Tekton
- Helm
- [`kapp`](https://carvel.io/kapp)

### Caveats

- We assume that you have a TKG--setlavored Kubernetes cluster running and
  accessible within your default Kubeconfig. You'll need to set that up if you
  do not.

### Demo

#### Install `kapp`

First, install the `kapp` client into your machine system-wide:

```sh
sudo sh -c 'curl -L https://carvel.dev/install.sh | bash'
```

Then use `kapp` to install the `kapp-controller` into your cluster. (The
`kapp-controller` is used by `kapp` to deploy applications and manage them.)

```sh
kapp deploy -a kc -f https://github.com/vmware-tanzu/carvel-kapp-controller/releases/latest/download/release.yml -y
```

Finally, observe that `kapp` can keep track of your applications by running
`kapp list -A`.

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
