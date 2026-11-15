# Kubernetes UI manifests

This folder contains Kubernetes YAML manifests for the `dev` environment. Each manifest declares a resource and its desired configuration. `kubectl apply` creates or updates the resource, and Kubernetes works to maintain the declared state.

## How the resources work together

```text
Browser
  -> Ingress / AWS Application Load Balancer
  -> Service (port 80)
  -> UI Pod (container port 80)
```

The Deployment creates the UI Pod from its template. The Service selects the Pod using the Pod's `app: pharma-ui` label. The Ingress routes web requests to the Service. The namespace groups these resources. The ServiceAccount supplies a Kubernetes identity to the Pod, and the ConfigMap stores application settings.

## `namespaces.yaml`

Declares the `dev` namespace, a Kubernetes grouping for the UI resources. It is not an AWS network or a separate AWS account.

- `metadata.name: dev` is the namespace's name. The UI manifests set `namespace: dev` so Kubernetes creates their resources inside it.
- `metadata.labels.env: dev` is a Kubernetes label used to identify this namespace as development; tools can filter namespaces using this label.
- `metadata.labels.managed-by: terraform` is a Kubernetes label declaring the intended management owner. It does not run Terraform or prove that Terraform currently manages the namespace.

**Verify:** `kubectl get namespace dev --show-labels` shows the namespace and its labels. These are Kubernetes labels, not AWS resource tags. The AWS Console does not show the namespace as a standalone AWS resource; if available, inspect it under **EKS → Clusters → your cluster → Resources**. AWS resource tags on the EKS cluster are separate.

## `raw-manifests/dev/pharma-ui-deployment.yaml`

This manifest defines the Deployment object and a template for the UI Pods that it creates. Read each dotted field below as its full YAML path.

### Top-level resource fields and metadata

- `apiVersion: apps/v1` tells Kubernetes which API version defines this Deployment resource.
- `kind: Deployment` tells Kubernetes the resource type. A Deployment manages Pods through a ReplicaSet and tries to maintain the desired number of replicas.
- `metadata` contains identifying information and Kubernetes labels for this Deployment object. These labels are not AWS resource tags.
- `metadata.name: pharma-ui` is this Deployment's name. It is how you refer to it in commands such as `kubectl describe deployment pharma-ui -n dev`.
- `metadata.namespace: dev` puts the Deployment in the `dev` namespace. The namespace must already exist.
- `metadata.labels.app: pharma-ui` sets a label on the **Deployment object itself**. Labels are key-value markers that can help people and tools organize or filter Kubernetes objects. This label does not get copied to Pods, and the Service does not use it to find Pods. It is useful if a tool queries Deployments with `app=pharma-ui`; otherwise, it is just descriptive metadata.

### Deployment `spec`

- `spec` describes the desired behavior of the Deployment.
- `spec.replicas: 1` asks the Deployment to keep one Pod running. If a Pod fails or is deleted, the Deployment works to create a replacement.
- `spec.selector.matchLabels.app: pharma-ui` is the Deployment's Pod selector rule, not a label being set anywhere. It means “manage Pods whose labels include `app=pharma-ui`.” Kubernetes requires this selector to match the labels in `spec.template.metadata.labels`; Deployment selectors are not changed after creation.
- `spec.template` is the blueprint used to create each Pod.
- `spec.template.metadata.labels.app: pharma-ui` is where the Pod label is actually **declared**. Whenever the Deployment creates a Pod from this template, Kubernetes puts `app=pharma-ui` in that Pod's metadata. Both the Deployment selector above and the Service selector in `pharma-ui-service.yaml` look for this Pod label. This is separate from the top-level `metadata.labels.app`, even though the key/value are the same.
- `spec.template.spec` contains the settings for the Pod's containers, identity, security, and storage.
- `spec.template.spec.serviceAccountName: pharma-ui` assigns the `pharma-ui` ServiceAccount to the Pod. That ServiceAccount must exist in the `dev` namespace.
- `spec.template.spec.securityContext.runAsNonRoot: true` tells Kubernetes the Pod's containers must run as a non-root user. The image must be configured with a non-root user or the container may fail to start.

#### Where the selector's label comes from

The label is not created by `matchLabels`, and it is not looked up in AWS. It is declared in the Pod template's `metadata.labels`. The Deployment uses `selector.matchLabels` to find and manage Pods with that label; the Service independently uses its own selector to find the same Pods and send traffic to them.

```yaml
# Deployment excerpt:
spec:
  selector:
    matchLabels:
      app: pharma-ui
  template:
    metadata:
      labels:
        app: pharma-ui

# Separate Service excerpt:
spec:
  selector:
    app: pharma-ui
```

In the Deployment excerpt, `template.metadata.labels` **sets** the label on new Pods; `selector.matchLabels` **matches** it so the Deployment knows which Pods belong to it. In the Service excerpt, `selector` independently **matches** that same Pod label so the Service knows where to send traffic.

The sequence is: `kubectl apply` stores the Deployment's desired configuration; the Deployment controller creates Pods from `template`; each created Pod receives the template label; the Deployment selector recognizes and manages matching Pods; and the Service selector finds matching, ready Pods to route traffic to. Kubernetes requires the Deployment selector and template labels to match, so an inconsistent Deployment is rejected. If the Pod lacks the label required by the Service selector, the Service has no matching Pod endpoints.

The top-level Deployment label and Pod-template label are both written in YAML because they belong to two different Kubernetes objects. The Deployment label is optional for this setup and helps identify/filter the Deployment itself. The Pod label is essential here because both the Deployment and Service select Pods using it.

### Container fields

- `spec.template.spec.containers` is the list of containers to run in each Pod. This manifest defines one.
- `containers[].name: pharma-ui` names the container within the Pod.
- `containers[].image` gives the image to run. Use the exact registry, repository, and tag currently written in this manifest. The tag can change when a new UI image is released; the cluster must be able to pull the selected image.
- `containers[].securityContext.readOnlyRootFilesystem: true` makes the container's root filesystem read-only. The writable paths required by NGINX are provided by the mounted temporary volumes below.
- `containers[].ports[].containerPort: 80` documents the port on which the container is expected to listen. It does not expose the port outside the Pod; the Service and Ingress provide the network path.
- `containers[].resources.requests.cpu: 50m` requests 50 millicores (0.05 CPU), used when Kubernetes schedules the Pod.
- `containers[].resources.requests.memory: 64Mi` requests 64 mebibytes of memory for scheduling.
- `containers[].resources.limits.cpu: 200m` caps CPU use at 200 millicores (0.2 CPU).
- `containers[].resources.limits.memory: 128Mi` caps memory use at 128 mebibytes. Exceeding the memory limit can cause the container to be terminated.

### Health probes

- `containers[].livenessProbe.httpGet.path: /` and `port: 80` make Kubernetes send an HTTP GET request to the container's `/` path on port 80 to check whether it is alive.
- `containers[].livenessProbe.initialDelaySeconds: 10` waits 10 seconds after container startup before liveness checking begins.
- `containers[].livenessProbe.periodSeconds: 15` sets the interval between liveness checks. Repeated failures cause Kubernetes to restart the container.
- `containers[].readinessProbe.httpGet.path: /` and `port: 80` check whether the application is ready to receive traffic.
- `containers[].readinessProbe.initialDelaySeconds: 5` waits 5 seconds after container startup before readiness checking begins.
- `containers[].readinessProbe.periodSeconds: 10` sets the interval between readiness checks. A Pod that is not ready is excluded from Service traffic; readiness failure alone does not restart the container.

### Writable temporary volumes

- `containers[].volumeMounts` connects named Pod volumes to paths inside the container.
- `volumeMounts[].name: tmp` with `mountPath: /tmp` provides writable temporary storage at `/tmp`.
- `volumeMounts[].name: nginx-cache` with `mountPath: /var/cache/nginx` provides writable NGINX cache storage.
- `volumeMounts[].name: nginx-run` with `mountPath: /var/run` provides writable runtime storage.
- `spec.template.spec.volumes` declares the volumes available to the Pod. The names must match the names in `volumeMounts`.
- Each `volumes[].emptyDir: {}` creates an initially empty temporary directory for that Pod. It lasts for the Pod's lifetime; its contents are lost when the Pod is removed and recreated.

### How to inspect this Deployment

```bash
kubectl get deployment pharma-ui -n dev --show-labels
kubectl describe deployment pharma-ui -n dev
kubectl get pods -n dev -l app=pharma-ui --show-labels
kubectl describe pods -n dev -l app=pharma-ui
```

The first command shows labels on the Deployment object. The Pod commands show labels set by `spec.template.metadata.labels`. The `-l app=pharma-ui` filter finds Pods with that label. `describe` also shows replica status, events, probes, and container configuration.

**Where to verify in AWS:** The Deployment and Pod labels are Kubernetes metadata, not AWS resource tags. They do not appear as tags on EC2 instances or the EKS cluster. If your EKS console exposes Kubernetes resources, look under **EKS → Clusters → your cluster → Resources**; otherwise, use the `kubectl` commands above. The image is stored in AWS ECR: open **ECR → Repositories → pharma-ui → Images** in the region/account from the image URL and check that the tag currently specified in `containers[].image` exists. EC2 instances are worker nodes that run Pods, not the Deployment itself.

## `raw-manifests/dev/pharma-ui-service.yaml`

Declares an internal network endpoint that sends traffic to matching UI Pods.

- `apiVersion: v1` selects the core Kubernetes API.
- `kind: Service` declares a Service.
- `metadata.name: pharma-ui` is the Service's name; the Ingress refers to this name.
- `metadata.namespace: dev` places the Service in the same namespace as the Deployment and Pods.
- `spec.type: ClusterIP` creates a stable address reachable inside the cluster, not directly from the public internet.
- `spec.ports` lists the Service's network ports.
- `ports.port: 80` is the port clients use on the Service.
- `ports.targetPort: 80` is the destination port on each selected Pod.
- `ports.protocol: TCP` specifies the transport protocol.
- `spec.selector.app: pharma-ui` is a selection rule, not a label being set on the Service. It selects Pods whose labels include `app: pharma-ui`. It matches `spec.template.metadata.labels` in the Deployment, not the Deployment's own top-level metadata labels.

**Verify:** Use `kubectl describe service pharma-ui -n dev` to check the selector and ports. Use `kubectl get pods -n dev --show-labels` to confirm selected Pod labels, and `kubectl get endpointslice -n dev -l kubernetes.io/service-name=pharma-ui` to check that ready Pod IPs are endpoints.

**Where to verify in AWS:** A `ClusterIP` Service is a Kubernetes object; it does not create its own AWS load balancer or appear as a standalone resource in EC2. You can inspect it in the EKS cluster's **Resources** view if available, but `kubectl` is the reliable place to check its selector and endpoints. The ALB requested by the Ingress is a separate AWS resource.

## `raw-manifests/dev/pharma-ui-ingress.yaml`

Declares HTTP routing from an external entry point to the Service. The AWS Load Balancer Controller interprets this Ingress and, if configured correctly, provisions an AWS Application Load Balancer.

### Resource identity and annotations

- `apiVersion: networking.k8s.io/v1` selects the current stable Ingress API.
- `kind: Ingress` declares HTTP/HTTPS routing rules.
- `metadata.name: pharma-ui` is the Ingress's name.
- `metadata.namespace: dev` places it in the `dev` namespace. The backend Service must be in the same namespace.
- `metadata.annotations.alb.ingress.kubernetes.io/scheme: internet-facing` requests an internet-facing AWS load balancer.
- `metadata.annotations.alb.ingress.kubernetes.io/target-type: ip` requests that the load balancer target Pod IP addresses directly.
- `metadata.annotations.alb.ingress.kubernetes.io/group.name: pharma-dev` associates this Ingress with the AWS Load Balancer Controller group named `pharma-dev`; Ingresses in the same group may share a load balancer.

Annotations are controller-specific settings. The cluster needs a compatible AWS Load Balancer Controller, AWS permissions, and suitable network configuration for these settings to take effect. An Ingress manifest describes routing but does not, by itself, guarantee a working public address.

### Routing rule

- `spec.ingressClassName: alb` asks the controller for the IngressClass named `alb`.
- `spec.rules` contains the routing rules. No `host` is specified, so the path rule is not limited to one domain name.
- `http.paths` lists HTTP path routes.
- `path: /` matches requests starting at the site root.
- `pathType: Prefix` matches the path as a prefix, including `/` and paths beneath it.
- `backend.service.name: pharma-ui` sends matching requests to the Service named `pharma-ui`.
- `backend.service.port.number: 80` selects port 80 on that Service.

If provisioning succeeds, `kubectl get ingress -n dev` should show an address. The browser-facing URL also depends on DNS and, if HTTPS is required, TLS configuration, neither of which is defined in this Ingress file.

**Verify in AWS:** In **EC2 → Load Balancers**, find the Application Load Balancer created by the AWS Load Balancer Controller and compare its DNS name with the Ingress address from `kubectl get ingress -n dev`. Inspect its listeners and rules. In **EC2 → Target Groups**, inspect the target group and confirm the registered Pod IP targets are healthy. The annotation values are Kubernetes annotations interpreted by the controller; they are not necessarily displayed as identical AWS resource tags. Check the Ingress events with `kubectl describe ingress pharma-ui -n dev` if no ALB/address appears.

## `raw-manifests/dev/pharma-ui-serviceaccount.yaml`

Creates the Kubernetes identity assigned to the UI Pod.

- `apiVersion: v1` selects the core Kubernetes API.
- `kind: ServiceAccount` declares a ServiceAccount.
- `metadata.name: pharma-ui` names the account.
- `metadata.namespace: dev` places it in the Pod's namespace.

The Deployment selects it through `spec.template.spec.serviceAccountName`. Creating a ServiceAccount alone does not grant additional Kubernetes API permissions; those require RBAC configuration. Access to AWS services requires separate AWS identity and permission configuration.

**Verify:** Run `kubectl get serviceaccount pharma-ui -n dev -o yaml` and inspect the Deployment's `serviceAccountName`. The ServiceAccount is a Kubernetes identity, not an IAM role or user. This manifest alone creates no AWS IAM resource and does not give the Pod AWS permissions. If AWS access is configured separately through IAM Roles for Service Accounts or EKS Pod Identity, verify that separate association and IAM role in the EKS/IAM console.

## `raw-manifests/dev/pharma-ui-configmap.yaml`

Stores non-secret application configuration as key-value data.

- `apiVersion: v1` selects the core Kubernetes API.
- `kind: ConfigMap` declares a ConfigMap.
- `metadata.name: pharma-ui` names it.
- `metadata.namespace: dev` places it in the `dev` namespace.
- `data.API_BASE_URL: /api` defines the API base path.
- `data.AUTH_BASE_URL: /api/auth` defines the authentication API base path.
- `data.ENV: dev` identifies the configured environment.

A ConfigMap only stores the values; a Pod must explicitly consume them, for example with `envFrom`, individual `env` entries, or a mounted volume. The current Deployment does not reference this ConfigMap, so these settings are not currently supplied to the container by these manifests. Do not store passwords, tokens, or other secrets in a ConfigMap.

**Verify:** Run `kubectl get configmap pharma-ui -n dev -o yaml` to see the stored keys and values. This ConfigMap is a Kubernetes object, not an AWS Systems Manager Parameter Store parameter or Secrets Manager secret; it has no standalone AWS Console resource. It may be visible in the EKS cluster's **Resources** view if that view is available.

## Apply the manifests

From the `gitops` directory, create the namespace and apply the `dev` UI resources:

```bash
kubectl apply -f k8s/namespaces.yaml
kubectl apply -f k8s/raw-manifests/dev/
```

`kubectl apply` can be run again after manifest changes; it creates missing resources and updates existing ones to match the manifests.

## Check status

```bash
kubectl get pods,services,ingress,serviceaccounts -n dev
```

To investigate a Pod that is not running or ready:

```bash
kubectl describe pods -n dev
kubectl logs -n dev deployment/pharma-ui
```

To inspect Ingress provisioning:

```bash
kubectl describe ingress pharma-ui -n dev
```

If the UI Pod is healthy but the Ingress has no address, check the AWS Load Balancer Controller and the cluster's AWS/network configuration. If the Pod is not healthy, inspect its events and logs for image-pull, startup, or health-check errors.

## End-to-end verification

The Kubernetes resources in these files are not all AWS resources. Kubernetes labels such as `app: pharma-ui` and namespace labels are not the same thing as AWS tags. Use `kubectl` to verify Kubernetes configuration; use the AWS Console for the AWS resources that the configuration references or causes a controller to create. In the AWS Console, select the correct account and region (`us-east-1` for the ECR image shown here).

```bash
kubectl config current-context
kubectl get pods,services,ingress,serviceaccounts -n dev
kubectl get events -n dev --sort-by=.metadata.creationTimestamp
kubectl logs -n dev deployment/pharma-ui
```

The Deployment should become ready (`1/1`), and the Service should have ready Pod endpoints. If the Ingress receives an ALB DNS address, try `http://<alb-dns-name>/` in a browser. This Ingress does not configure a custom hostname or HTTPS/TLS. If there is no address or the site is unavailable, inspect Ingress events, ALB listeners/rules and target health, then the Pod's readiness and logs.
