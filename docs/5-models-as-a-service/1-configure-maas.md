# ⚙️ Configure Models-as-a-Service

This lab guides you through the initial configuration of Models-as-a-Service (MaaS) in Red Hat OpenShift AI. You will configure the platform infrastructure, set up TLS for secure communication, configure the database connection, and enable MaaS components in the DataScienceCluster.

## Verify  Prerequisites

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

?> 📎 If the required operator dependencies are missing, install the following via the **Software Catalog**:

- **Red Hat Connectivity Link**
- **cert-manager Operator for Red Hat OpenShift**
- **Leader Worker Set Operator (LWS) Operator**
- **Red Hat OpenShift AI**

### Enabling User Workload Monitoring

User Workload Monitoring is mandatory for MaaS to collect and expose token rate-limiting metrics. If User Workload Monitoring is disabled, the MaaS deployment will show a `Degraded` status.

To verify and enable User Workload Monitoring, edit the cluster monitoring configuration map in the `openshift-monitoring` namespace:

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

#### Verify User Workload Monitoring

MaaS requires User Workload Monitoring to collect and expose usage metrics.

1. Verify that User Workload Monitoring is enabled:

```bash
oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml | grep enableUserWorkload
```

Expected output:

```text
  enableUserWorkload: true
```

2. Verify that user workload monitoring pods are running:

```bash
oc get pods -n openshift-user-workload-monitoring
```

?> ✅ You should see **prometheus-user-workload** and **thanos-ruler** pods running.

### PostgreSQL Deployment for MaaS

MaaS requires a PostgreSQL database (version 14 or later) for API key lifecycle management.

?> 📎 OpenShift AI does not provide a PostgreSQL database. You must provision and manage your own PostgreSQL instance. The database can be deployed within OpenShift or as an external service. 

For lab purposes, set up a temporary PostgreSQL database as outlined in the subsection below to complete the lab.

---

#### 1. Create namespace for PostgreSQL

```bash
# namespace/project creation for postgresql
oc new-project maas-postgres

# namespace for ai-gatewey (later on will be used by maas api)
oc new-project redhat-ai-gateway-infra
```

---

#### 2. Create Secrets

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

---

#### 3. Create Storage

```yaml
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
```

---

#### 4. Deploy PostgreSQL

```yaml
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
```

---

#### 5. Create Service

```yaml
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
```

---

#### 6. Verify Deployment

```bash
# Check all components
oc get deployment,pod,svc,pvc -n maas-postgres -l app=postgres

# Test database connection
oc exec -n maas-postgres deployment/postgres -- \
  psql -U maas -d maas -c "SELECT version();"
```

---

### Deploy the Kuadrant
Apply this section if the Red Hat Connectivity Link (Kuadrant) has not ben set for llm-d yet.


#### 1. Create namespace for Kuadrant

```yaml
oc new-project kuadrant-system
```

#### 2. Deploy the Kuadrant Custom Resource
Create the root Kuadrant Custom Resource in the kuadrant-system namespace
```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
spec:
  observability:
    enable: true
```

#### 3. Configure TLS for Authorino
To enable secure authentication and authorization for model endpoints, configure TLS communication between Authorino and the MaaS API service.

1. Annotate the Authorino service to generate an OpenShift serving certificate secret

```bash
oc annotate svc/authorino-authorino-authorization \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert \
  -n kuadrant-system
```

?> 📎 The service-ca-operator in OpenShift generates a TLS certificate signed by the cluster service CA and stores it in the authorino-server-cert secret.

Verify that the certificate secret was created:
```bash
oc get secret authorino-server-cert -n kuadrant-system
```
Expected output:

```text
NAME                     TYPE                DATA   AGE
authorino-server-cert    kubernetes.io/tls   2      30s
```

2. Enable TLS on Authorino's listener:

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

Verify that TLS is enabled in the Authorino CR:

```bash
oc get authorino authorino -n kuadrant-system -o jsonpath='{.spec.listener.tls.enabled}'
```

3. Configure the Authorino deployment with environment variables for TLS certificate validation:

```bash
oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt
```

?> 📎 The cluster CA bundle is automatically populated by the service-ca-operator in OpenShift.

Wait for the Authorino deployment to roll out:

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

If it does not exist, create a GatewayClass as shown below (**Import YAML):**   

```bash
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: "openshift.io/gateway-controller/v1"
```

### Create the MaaS Gateway

1. Create the `maas-default-gateway` in the `openshift-ingress` namespace using the YAML below (Import YAML). Before applying it, replace  with your cluster's application domain and certificate-name

You can retrieve the cluster domain by running:
```bash
oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}'
```

You can retrieve the certificate-name by running:
```bash
oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}'
```

```yaml
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
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.<CLUSTER_DOMAIN>"
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.<CLUSTER_DOMAIN>"
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: <CERT_NAME>
```

?> 📎 **Annotations explained:**

- `opendatahub.io/managed: "false"` - Prevents the ODH Model Controller from overriding MaaS-managed authorization policies
- `security.opendatahub.io/authorino-tls-bootstrap: "true"` - Enables TLS communication between the Gateway and Authorino

2. Verify that the Gateway was created:

```bash
oc get gateway maas-default-gateway -n openshift-ingress
```

Expected output:

```text
NAME                   CLASS              ADDRESS   PROGRAMMED   AGE
maas-default-gateway   openshift-default            True         1m
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

!> 🔧 **Important:** If you are upgrading from OpenShift AI 3.3 or 3.4 where MaaS was enabled under `spec.components.kserve.modelsAsService`, you must migrate to the new location under `spec.components.aigateway.modelsAsAService`.

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
oc get deployment maas-api -n models-as-a-service
```

Expected output:

```text
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
maas-api   1/1     1            1           3m
```

2. Check the MaaS API pod logs to verify successful database connection:

```bash
oc logs -n models-as-a-service -l app=maas-api --tail=50
```

?> ✅ Look for log entries indicating successful database connection and API server startup, such as "Connected to PostgreSQL database" or "Starting HTTP server on port 8080".

### Verify MaaS Controller

1. Verify that the MaaS controller is running:

```bash
oc get pods -n redhat-ods-applications -l control-plane=controller-manager,app.kubernetes.io/name=maas-controller
```

Expected output:

```text
NAME                              READY   STATUS    RESTARTS   AGE
maas-controller-7b8f5c9d8-xk4pl   1/1     Running   0          4m
```

### Verify Dashboard Configuration

1. Verify that the MaaS dashboard configuration was created:

```bash
oc get odhdashboardconfig default-odh-dashboard-config -n redhat-ods-applications -o jsonpath='{.spec.dashboardConfig.modelsAsAService}' | jq '.'
```

Expected output should show MaaS configuration including the gateway reference:

```json
{
  "enabled": true,
  "gateway": {
    "name": "maas-default-gateway",
    "namespace": "openshift-ingress"
  }
}
```

### Verify Default Tenant Configuration

1. Verify that the default tenant configuration was created:

```bash
oc get tenants.maas.opendatahub.io default-tenant -n models-as-a-service
```

Expected output:

```text
NAME             AGE
default-tenant   5m
```

2. View the tenant configuration:

```bash
oc get tenants.maas.opendatahub.io default-tenant -n models-as-a-service -o yaml
```

?> 📎 The tenant configuration includes gateway references, API key expiration settings, and telemetry options.


**Congratulations!** You have successfully configured Models-as-a-Service infrastructure. In the next section, you will deploy models with MaaS governance.
