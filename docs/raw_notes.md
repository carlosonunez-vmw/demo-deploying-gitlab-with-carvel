### Tech

- Container Platform: TKG
- Infrastructure Provisioning:
    - Tekton
        - Tekton is a Kubernetes-native build executor
        - Developers can use it for continuous delivery
        - We can use it for self-service provisioning, ideally as a backend to something nicer like vRealize
        - TAP uses Tekton for Supply Chains underneath the hood
    - kubectl
        - THE Kubernetes command line client
- App Provisioning:
    - Helm
        - Like Ansible, but for apps in Kubernetes
        - Makes it really easy to package stuff into Kubernetes as a single unit
    - kapp
        - Like rpm, but for apps in Kubernetes
        - Makes it easy to install stuff into Kubernetes from 
    - Avi Operator for Kubernetes (AKO)
        - Deploys resources into your Avi LB
        - Avi is like haproxy or nginx, but can do way, way more
        - Ingress inside of Kubernetes clusters will flow through VIPs created by AKO

### Workflow

- Tekton
    - Input: JSON or YAML file with some configuration information, like:
        - Location (i.e. which Kubernetes cluster)
        - Number of Linux runners
        - Number of Windows runners
        - Users to create
    - Work: Pipeline
        - Parses and validates your YAML file
        - Locates kubeconfig for Kubernetes cluster
        - Creates a values file for ytt
        - Feeds that values file into kapp
- kapp
    - Input: Values file and kubeconfig
    - Work:
        - Fetches Gitlab Helm chart
        - Merges Helm chart and values file to install app into target Kubernetes cluster and namespace
- AKO
    - Input: new Ingress created by Gitlab Helm chart
    - Work:
        - Creates a Virtual Service to front the Ingress
        - Creates a VIP
