# 🤖 OpenShift AI Operator & Components

## Introduction to Red Hat OpenShift AI

Red Hat OpenShift AI is a flexible, scalable, hybrid-cloud AI platform that provides a unified environment for the entire lifecycle of  **predictive, generative, and agentic AI** . It bridges the gap between data science and IT operations by integrating enterprise-grade open-source technologies into a secure framework for distributed inference, governed Model-as-a-Service, and automated AI monitoring. It simplifies the path from development to production by providing distributed inference and secure, centralized model access.

## Architecture: The Meta-Operator Model

The platform utilizes a Meta-Operator architecture. The main Operator does not just manage one service; it orchestrates a suite of specialized controllers and sub-operators that together form the AI platform.

### Core Custom Resources (CRs)

Managing the platform's state is achieved through two primary Custom Resources:

1. DSCInitialization (DSCI): This resource sets the global foundation. It configures cluster-wide settings, including:

* Namespace management for applications and monitoring.
* The observability stack (standardized data ingestion and metrics).
* Trusted Certificate Authority (CA) bundles for secure communication.

2. DataScienceCluster (DSC): This resource defines the functional "profile" of your installation. It is used to enable or disable specific service layers (Dashboard, Workbenches, Pipelines, etc.) by defining their desired state.

### Component Management States

Each component within the DataScienceCluster resource can be assigned a managementState:

* Managed: The Operator actively installs, maintains, and updates the component.
* Removed: The Operator ensures the component is not present on the cluster.
* Unmanaged: The Operator ignores the component, allowing for manual specialized configurations or integration with external controllers.

### Default System Namespaces

Upon installation, the Operator manages resources across several key projects:

* redhat-ods-operator: The control plane for the Meta-Operator.
* redhat-ods-applications: The service layer containing the Dashboard and system components.
* redhat-ods-monitoring: The dedicated observability and metrics stack.
* rhods-notebooks: The default location for user workbench deployments.

## Lab Prerequisites

To successfully complete this lab and install the foundational AI infrastructure, ensure your environment meets the following requirements:

### 1. Cluster Requirements

* Platform: Red Hat OpenShift Container Platform on a supported environment.
* Minimum Capacity: * At least 2 worker nodes.
* 8 CPUs and 32 GiB RAM per node.
* Storage: A default StorageClass must be configured with support for dynamic provisioning.

### 2. Access & Permissions

* Administrative Access: You must have cluster-admin privileges on the OpenShift cluster.
* Identity Provider: A functional Identity Provider must be configured in OpenShift to manage user authentication for the AI dashboard.

### 3. Tooling & Network

* CLI: The OpenShift CLI (oc) must be installed on your local workstation.
* Connectivity: Access to the Red Hat container registry and Quay.io.
* DNS: Functional DNS resolution for the cluster's API and application routes.
