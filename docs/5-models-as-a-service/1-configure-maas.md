# ⚙️ Configure Models-as-a-Service

This lab guides you through the initial configuration of Models-as-a-Service (MaaS) in Red Hat OpenShift AI. You will install the Red Hat Connectivity Link Operator, the Leader Worker Set Operator, and a PostgreSQL database, configure the platform infrastructure, set up TLS for secure communication, configure the database connection, and enable MaaS components in the DataScienceCluster.

## Install required dependencies

Install the Red Hat Connectivity Link Operator, the Leader Worker Set Operator, and a PostgreSQL database before you configure MaaS. OpenShift AI does not install these for you.

### Install the Red Hat Connectivity Link Operator

Models-as-a-Service (MaaS) in Red Hat OpenShift AI uses Red Hat Connectivity Link (Kuadrant) for authentication, authorization, and rate limiting. Install operator version 1.4.x manually in the `openshift-operators` namespace.

?> 📎 `openshift-operators` already has a global OperatorGroup. Do not create another OperatorGroup in that namespace.

1. Create the Subscription. The `stable` channel installs Connectivity Link. The next step checks that the installed version is 1.4.x:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhcl-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: rhcl-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

2. Wait until the ClusterServiceVersion reports `Succeeded`:

```bash
oc get csv -n openshift-operators -o custom-columns=NAME:.metadata.name,VERSION:.spec.version,PHASE:.status.phase | grep rhcl-operator
```

Expected output (the patch version can differ):

```text
rhcl-operator.v1.4.3                1.4.3     Succeeded
```

3. Confirm the operator pod is running:

```bash
oc get pods -n openshift-operators | grep kuadrant-operator
```

?> ✅ You should see a `kuadrant-operator-controller-manager` pod in the **Running** state. The operator also installs Authorino, Limitador, and the DNS operator. You create the Kuadrant custom resource later in this lab.

### Install the Leader Worker Set Operator

Models published to MaaS run as an `LLMInferenceService` with distributed inference (`llm-d`). llm-d uses a LeaderWorkerSet to run one model replica as a leader pod plus worker pods across GPUs or nodes. Install the Leader Worker Set Operator manually in the `openshift-lws-operator` namespace.

?> 📎 Install the **cert-manager Operator for Red Hat OpenShift** before this operator. Leader Worker Set uses cert-manager for its metrics certificates. This lab assumes cert-manager is already present from the earlier modules.

1. Create the namespace, OperatorGroup, and Subscription. Channel `stable-v1.0` selects Leader Worker Set Operator 1.0:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-lws-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: leader-worker-set
  namespace: openshift-lws-operator
spec:
  targetNamespaces:
  - openshift-lws-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: leader-worker-set
  namespace: openshift-lws-operator
spec:
  channel: stable-v1.0
  installPlanApproval: Automatic
  name: leader-worker-set
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

2. Wait until the ClusterServiceVersion reports `Succeeded`:

```bash
oc get csv -n openshift-lws-operator -o custom-columns=NAME:.metadata.name,VERSION:.spec.version,PHASE:.status.phase | grep leader-worker-set
```

Expected output (the patch version can differ):

```text
leader-worker-set.v1.0.1   1.0.1   Succeeded
```

3. Create the `LeaderWorkerSetOperator` custom resource. The operator does not start the controller until this resource exists:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operator.openshift.io/v1
kind: LeaderWorkerSetOperator
metadata:
  name: cluster
  namespace: openshift-lws-operator
spec:
  managementState: Managed
EOF
```

4. Confirm both deployments are available:

```bash
oc get deployments -n openshift-lws-operator
```

Expected output:

```text
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
lws-controller-manager   2/2     2            2           61s
openshift-lws-operator   1/1     1            1           4m
```

?> ✅ `openshift-lws-operator` is the operator pod. `lws-controller-manager` is the controller that reconciles `LeaderWorkerSet` resources for llm-d.

### Install PostgreSQL

Models-as-a-Service (MaaS) in Red Hat OpenShift AI requires a PostgreSQL database (version 14 or later) for API key lifecycle management.

?> 📎 OpenShift AI does not provide a PostgreSQL database. You must provision and manage your own PostgreSQL instance. The database can be deployed within OpenShift or as an external service.

For lab purposes, set up a temporary PostgreSQL database as outlined in the subsection below to complete the lab.

1. Create the PostgreSQL namespace and the namespace where the MaaS API will run:

```bash
# Project for the PostgreSQL database
oc new-project maas-postgres

# Project where the MaaS API runs
oc new-project redhat-ai-gateway-infra
```

2. Create the database credential secret and the connection secret that the MaaS controller reads:

```bash
# Database credentials
oc create secret generic postgres-creds \
  -n maas-postgres \
  --from-literal=POSTGRES_USER=maas \
  --from-literal=POSTGRES_DB=maas \
  --from-literal=POSTGRES_PASSWORD=maas123

# Connection URL for MaaS services
oc create secret generic maas-db-config \
  -n redhat-ai-gateway-infra \
  --from-literal=DB_CONNECTION_URL="postgresql://maas:maas123@postgres.maas-postgres.svc:5432/maas?sslmode=disable"
```

3. Create the data volume:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: maas-postgres
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

4. Deploy PostgreSQL 16:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: maas-postgres
  labels:
    app: postgres
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: registry.redhat.io/rhel9/postgresql-16:latest
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRESQL_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-creds
                  key: POSTGRES_USER
            - name: POSTGRESQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-creds
                  key: POSTGRES_PASSWORD
            - name: POSTGRESQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: postgres-creds
                  key: POSTGRES_DB
            - name: PGDATA
              value: /var/lib/pgsql/data/userdata
          readinessProbe:
            exec:
              command:
                - /usr/libexec/check-container
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U "\$POSTGRESQL_USER" -d "\$POSTGRESQL_DATABASE"
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgsql/data
              subPath: userdata
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
EOF
```

5. Expose PostgreSQL inside the cluster:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: maas-postgres
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
EOF
```

6. Confirm that PostgreSQL is accepting connections:

```bash
# Check all components
oc get deployment,pod,svc,pvc -n maas-postgres -l app=postgres

# Test database connection
oc exec -n maas-postgres deployment/postgres -- \
  psql -U maas -d maas -c "SELECT version();"
```

?> ✅ The query prints a PostgreSQL 16 version string.

## Verify prerequisites

Before configuring MaaS, verify that the following key CRDs have been registered:

```bash
# RHOAI
oc get crd datascienceclusters.datasciencecluster.opendatahub.io

# Connectivity Link
oc get crd kuadrants.kuadrant.io

# cert-manager
oc get crd certificates.cert-manager.io

# Leader Worker Set
oc get crd leaderworkersetoperators.operator.openshift.io
```

?> 📎 The Red Hat Connectivity Link Operator and the Leader Worker Set Operator are installed in the previous section. If any other required operator dependency is missing, install it via the **Software Catalog**:

- **cert-manager Operator for Red Hat OpenShift**
- **Red Hat OpenShift AI**

### Enable User Workload Monitoring

MaaS stays `Degraded` until user workload monitoring is enabled. That is how token rate-limit metrics are collected.

1. Check whether the monitoring ConfigMap already exists:

```bash
oc get configmap cluster-monitoring-config -n openshift-monitoring
```

If it exists, edit `data.config.yaml` and set `enableUserWorkload: true`. Do not replace the ConfigMap, because it may already contain other monitoring settings.

If it does not exist, create it:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

2. Confirm the setting:

```bash
oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml | grep enableUserWorkload
```

Expected output:

```text
  enableUserWorkload: true
```

3. Confirm that the user workload monitoring pods are running. Every pod should report all of its containers ready:

```bash
oc get pods -n openshift-user-workload-monitoring
```

Expected output:

```text
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-5f9c78bb5d-w95vs   2/2     Running   0          18s
prometheus-user-workload-0             6/6     Running   0          15s
thanos-ruler-user-workload-0           4/4     Running   0          15s
```

?> ✅ You should see **prometheus-user-workload** and **thanos-ruler** pods running.

### Deploy Kuadrant

Apply this section only when Red Hat Connectivity Link is not already configured for llm-d.

1. Create the Kuadrant namespace:

```bash
oc new-project kuadrant-system
```

2. Create the `Kuadrant` resource in that namespace:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
spec:
  observability:
    enable: true
EOF
```

3. Configure TLS for Authorino so the gateway can authenticate to it.

   a. Annotate the Authorino service. The service CA generates a certificate and stores it in `authorino-server-cert`:

```bash
oc annotate svc/authorino-authorino-authorization \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert \
  -n kuadrant-system
```

?> 📎 The service CA operator signs this certificate with the cluster service CA.

   Confirm that the secret exists:

```bash
oc get secret authorino-server-cert -n kuadrant-system
```

   Expected output:

```text
NAME                     TYPE                DATA   AGE
authorino-server-cert    kubernetes.io/tls   2      30s
```

   b. Enable TLS on the Authorino listener:

```bash
oc patch authorino authorino -n kuadrant-system --type=merge --patch '{
  "spec": {
    "listener": {
      "tls": {
        "enabled": true,
        "certSecretRef": {
          "name": "authorino-server-cert"
        }
      }
    }
  }
}'
```

   Confirm that the listener reports TLS as enabled. The command prints `true`:

```bash
oc get authorino authorino -n kuadrant-system -o jsonpath='{.spec.listener.tls.enabled}{"\n"}'
```

   c. Point the Authorino deployment at the cluster service CA bundle:

```bash
oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt
```

?> 📎 OpenShift mounts the service CA bundle at that path. You do not create the file yourself.

   Wait until the deployment finishes rolling out:

```bash
oc rollout status deployment/authorino -n kuadrant-system
```

## Configure Gateway Infrastructure

MaaS requires Gateway API resources for routing and securing model endpoints.

### Verify GatewayClass

1. Verify that the `openshift-default` GatewayClass exists:

```bash
oc get gatewayclass openshift-default
```

Expected output:

```text
NAME               CONTROLLER                                    ACCEPTED   AGE
openshift-default  openshift.io/gateway-controller               True       10d
```

If the command reports `NotFound`, create the class named `openshift-default`:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1
EOF
```

### Create the MaaS Gateway

Leave the existing `GatewayConfig` named `default-gateway` unchanged. It publishes the OpenShift AI data-science host. The MaaS hostname is a separate `Gateway` and `Route`.

1. Store the cluster application domain:

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "maas.${CLUSTER_DOMAIN}"
```

2. Create the gateway parameters ConfigMap. The annotation asks the service CA for a certificate named `maas-default-gateway-service-tls`, and the Service stays `ClusterIP` because the OpenShift router publishes it:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: ConfigMap
metadata:
  name: maas-default-gateway-config
  namespace: openshift-ingress
  labels:
    platform.opendatahub.io/part-of: gatewayconfig
data:
  service: |
    metadata:
      annotations:
        service.beta.openshift.io/serving-cert-secret-name: "maas-default-gateway-service-tls"
    spec:
      type: ClusterIP
EOF
```

3. Create the `Gateway` in `openshift-ingress`. The listener hostname is `maas.<cluster-domain>`:

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
cat << EOF | oc apply -f-
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"
    security.opendatahub.io/authorino-tls-bootstrap: "true"
spec:
  gatewayClassName: openshift-default
  infrastructure:
    parametersRef:
      group: ""
      kind: ConfigMap
      name: maas-default-gateway-config
  listeners:
  - name: https
    hostname: maas.${CLUSTER_DOMAIN}
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
      - group: ""
        kind: Secret
        name: maas-default-gateway-service-tls
EOF
```

?> 📎 **Annotations:**

- `opendatahub.io/managed: "false"` keeps the OpenShift AI model controller from replacing MaaS authorization policies.
- `security.opendatahub.io/authorino-tls-bootstrap: "true"` turns on TLS between the gateway and Authorino.

4. Wait until the gateway controller creates the Service and the serving certificate:

```bash
oc wait --for=condition=Programmed gateway/maas-default-gateway -n openshift-ingress --timeout=180s
oc get secret maas-default-gateway-service-tls -n openshift-ingress
```

5. Create the `Route` only after that Service exists. Use `passthrough` so the client's TLS server name reaches the gateway. The gateway listener matches `maas.<cluster-domain>`. A `reencrypt` route opens a new TLS session from the router, the gateway finds no filter chain for that handshake, and clients receive `Application is not available`.

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
cat << EOF | oc apply -f-
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
spec:
  host: maas.${CLUSTER_DOMAIN}
  port:
    targetPort: 443
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: Redirect
  to:
    kind: Service
    name: maas-default-gateway-openshift-default
    weight: 100
  wildcardPolicy: None
EOF
```

6. Verify the gateway, its Service, and the Route:

```bash
oc get gateway maas-default-gateway -n openshift-ingress
oc get svc maas-default-gateway-openshift-default -n openshift-ingress
oc get route maas-default-gateway -n openshift-ingress
```

Expected output:

```text
NAME                   CLASS               ADDRESS                                                                      PROGRAMMED   AGE
maas-default-gateway   openshift-default   maas-default-gateway-openshift-default.openshift-ingress.svc.cluster.local   True         44m
NAME                                     TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)             AGE
maas-default-gateway-openshift-default   ClusterIP   172.231.114.126   <none>        15021/TCP,443/TCP   44m
NAME                   HOST/PORT                                        PATH   SERVICES                                 PORT   TERMINATION          WILDCARD
maas-default-gateway   maas.apps.<cluster-domain>                            maas-default-gateway-openshift-default   443    passthrough/Redirect None
```

## Enable MaaS in DataScienceCluster

Enable MaaS components in your OpenShift AI deployment by updating the DataScienceCluster configuration.

### Update DataScienceCluster

1. Navigate to the OpenShift console and go to **Ecosystem** → **Installed Operators** → **Red Hat OpenShift AI** → **Data Science Cluster**.
2. Click on your DSC instance (typically `default-dsc`).
3. Select the **YAML** tab.
4. Locate the `spec.components` section and ensure the following components are configured:

```yaml
spec:
  components:
    aigateway:
      managementState: Managed
      modelsAsAService:
        managementState: Managed
```

?> 📎 **Important:** The MaaS controller will automatically discover the `maas-db-config` secret in the infrastructure namespace (`redhat-ai-gateway-infra`). You don't need to reference it in the DataScienceCluster configuration.

5. Click **Save**.
6. Wait for the DataScienceCluster to reconcile. Monitor the status:

```bash
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}{"\n"}'
```

?> ✅ The status should be **Ready** after reconciliation completes.

## Verify MaaS Deployment

After enabling MaaS in the DataScienceCluster, verify that all MaaS components are deployed and running.

### Verify MaaS Namespace

1. Verify that the `models-as-a-service` namespace was created:

```bash
oc get namespace models-as-a-service
```

Expected output:

```text
NAME                   STATUS   AGE
models-as-a-service    Active   2m
```

### Verify MaaS API Deployment

1. Verify that the MaaS API deployment is running:

```bash
oc get deployment maas-api -n redhat-ai-gateway-infra
```

Expected output:

```text
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
maas-api   1/1     1            1           3m
```

2. Check the MaaS API pod logs to verify successful database connection:

```bash
oc logs -n redhat-ai-gateway-infra -l app.kubernetes.io/name=maas-api --tail=50
```

?> ✅ Look for log entries indicating successful database connection and API server startup, such as "Connected to PostgreSQL database" or "Starting HTTP server on port 8080".

### Verify MaaS Controller

1. Verify that the MaaS controller is running:

```bash
oc get pods -n redhat-ods-applications -l app=maas-controller
```

Expected output:

```text
NAME                              READY   STATUS    RESTARTS   AGE
maas-controller-7b8f5c9d8-xk4pl   1/1     Running   0          4m
```

### Dashboard configuration for Models-as-a-Service

To access Models-as-a-Service (MaaS) features in the OpenShift AI dashboard, you must enable the required dashboard configuration flags in the OdhDashboardConfig custom resource.

1. Enable the MaaS and observability pages in the dashboard configuration:

```bash
oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type=merge --patch '{
  "spec": {
    "dashboardConfig": {
      "modelAsService": true,
      "observabilityDashboard": true
    }
  }
}'
```

### Verify Models-as-a-Service deployment

After you deploy Models-as-a-Service, you can run a series of checks to confirm that the required custom resources, monitoring components, and tenant configuration are in place.

1. Verify that the MaaS component is enabled in the DataScienceCluster custom resource:

```bash
oc get dsc default-dsc -o jsonpath='{.status.conditions[?(@.type=="ModelsAsAServiceReady")]}' | jq
```

Expected output shows the ModelsAsAServiceReady condition with status: "True":

```yaml
{
  "lastTransitionTime": "2026-11-24T14:22:59Z",
  "message": "modelsAsAService is Managed and deployments are available",
  "reason": "Ready",
  "status": "True",
  "type": "ModelsAsAServiceReady"
}
```

2. Verify that the Models-as-a-Service (MaaS) custom resource definitions (CRDs) are installed:

```bash
oc get crd | grep -E 'maas.opendatahub.io|aitenants'
```

Expected output:

```text
aitenants.maas.opendatahub.io                                                      2026-11-24T14:22:06Z
configs.maas.opendatahub.io                                                        2026-11-24T14:22:06Z
externalmodels.maas.opendatahub.io                                                 2026-11-24T14:22:06Z
maasauthpolicies.maas.opendatahub.io                                               2026-11-24T14:22:06Z
maasmodelrefs.maas.opendatahub.io                                                  2026-11-24T14:22:07Z
maassubscriptions.maas.opendatahub.io                                              2026-11-24T14:22:07Z
maastenantconfigs.maas.opendatahub.io                                              2026-11-24T14:22:07Z
tenants.maas.opendatahub.io                                                        2026-11-24T14:22:07Z
```

### Verify Default Tenant Configuration

1. Verify that the default tenant configuration was created:

```bash
oc get aitenant models-as-a-service -n ai-tenants
```

Expected output:

```text
NAME                  READY   TENANT NAMESPACE      GATEWAY                AGE
models-as-a-service   True    models-as-a-service   maas-default-gateway   71m
```

**Congratulations!** You have successfully configured Models-as-a-Service infrastructure. In the next section, you will deploy models with MaaS governance.

## Objects created by this procedure

The lab creates the objects in the first table. Operators and controllers then create the objects in the second table. The MaaS hostname `maas.apps.<cluster-domain>` is the `Route` and the `Gateway` listener. The existing `GatewayConfig` `default-gateway` continues to publish the OpenShift AI data-science host.

### Objects you apply

| Kind | Name | Namespace | Purpose |
| --- | --- | --- | --- |
| Subscription | `rhcl-operator` | `openshift-operators` | Installs Red Hat Connectivity Link 1.4.x |
| Namespace | `openshift-lws-operator` | — | Operator namespace for Leader Worker Set |
| OperatorGroup | `leader-worker-set` | `openshift-lws-operator` | Scopes the Leader Worker Set Operator |
| Subscription | `leader-worker-set` | `openshift-lws-operator` | Installs Leader Worker Set Operator 1.0 |
| LeaderWorkerSetOperator | `cluster` | `openshift-lws-operator` | Starts the LeaderWorkerSet controller |
| Namespace | `maas-postgres` | — | PostgreSQL project |
| Namespace | `redhat-ai-gateway-infra` | — | Namespace where the MaaS API runs |
| Secret | `postgres-creds` | `maas-postgres` | Database user, password, and database name |
| Secret | `maas-db-config` | `redhat-ai-gateway-infra` | Connection URL discovered by the MaaS controller |
| PersistentVolumeClaim | `postgres-data` | `maas-postgres` | PostgreSQL data volume |
| Deployment | `postgres` | `maas-postgres` | PostgreSQL 16 database |
| Service | `postgres` | `maas-postgres` | In-cluster database endpoint on port 5432 |
| ConfigMap | `cluster-monitoring-config` | `openshift-monitoring` | Enables User Workload Monitoring |
| Namespace | `kuadrant-system` | — | Connectivity Link instance namespace |
| Kuadrant | `kuadrant` | `kuadrant-system` | Enables Authorino, Limitador, and observability |
| ConfigMap | `maas-default-gateway-config` | `openshift-ingress` | Forces the gateway Service to `ClusterIP` and requests its serving certificate |
| Gateway | `maas-default-gateway` | `openshift-ingress` | MaaS HTTPS listener |
| Route | `maas-default-gateway` | `openshift-ingress` | Publishes `maas.<cluster-domain>` with TLS passthrough |
| DataScienceCluster | `default-dsc` | — | Sets `aigateway.modelsAsAService` to `Managed` |

Create `kuadrant-system` and the `Kuadrant` resource only when Connectivity Link is not already configured for llm-d. Create a `GatewayClass` only when `openshift-default` is missing.

### Objects created by the controllers

| Kind | Name | Namespace | Created by |
| --- | --- | --- | --- |
| Deployments | `kuadrant-operator-controller-manager`, Authorino, Limitador, and DNS operators | `openshift-operators` | Connectivity Link Operator |
| Deployments | `authorino`, `limitador` | `kuadrant-system` | `Kuadrant` resource |
| Secret | `authorino-server-cert` | `kuadrant-system` | service-ca, after the Authorino Service annotation |
| Deployments | `openshift-lws-operator`, `lws-controller-manager` | `openshift-lws-operator` | Leader Worker Set Operator |
| Pods | `prometheus-user-workload`, `thanos-ruler` | `openshift-user-workload-monitoring` | Cluster monitoring |
| Service | `maas-default-gateway-openshift-default` | `openshift-ingress` | Gateway controller, from the `Gateway` and ConfigMap |
| Secret | `maas-default-gateway-service-tls` | `openshift-ingress` | service-ca, from the serving-cert annotation in the ConfigMap |
| Config | `default` | cluster-scoped | MaaS controller |
| Namespace | `ai-tenants` | — | MaaS controller |
| AITenant | `models-as-a-service` | `ai-tenants` | MaaS controller; references `maas-default-gateway` |
| Namespace | `models-as-a-service` | — | MaaS controller; tenant namespace |
| Deployment and Service | `maas-api` | `redhat-ai-gateway-infra` | MaaS controller |
| Deployment | `maas-controller` | `redhat-ods-applications` | OpenShift AI operator |
