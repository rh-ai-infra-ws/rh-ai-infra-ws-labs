# 🔐 Models-as-a-Service

## Introduction to Models-as-a-Service

Models-as-a-Service (MaaS) in Red Hat OpenShift AI provides subscription-based governance for large language model (LLM) serving across your organization. This platform helps you manage resource consumption and governance challenges when serving models to a large user base. MaaS acts as a governance layer between users and model serving infrastructure, enabling you to enforce centralized policies without modifying the underlying model serving components.

With MaaS, administrators can expose models through managed API endpoints with enforced consumption policies for different user groups, delivering AI models as shared resources with appropriate access levels and usage tracking.

## Key Capabilities

**Subscription-based Quota Management** 🎫

Define multiple subscriptions that grant specific groups quota for models with configurable token limits. Users can belong to multiple subscriptions, with priority levels determining which subscription is used when multiple options are available. This replaces the tier-based model used in earlier versions with enhanced flexibility managed through custom resources.

**Self-service API Key Management** 🔑

Users can create, list, and revoke their own API keys for model access. Administrators can also provision and manage API keys on behalf of users. API keys can be permanent or configured with custom expiration times, and individual keys can be revoked without affecting other keys for the same user.

**OpenAI API Compatibility** 🔌

Call models using the standard OpenAI `/v1/chat/completions` endpoint with the model name in the request body. This enables drop-in compatibility with OpenAI-compatible SDKs and clients such as the Python openai library, LangChain, and LlamaIndex without requiring custom URL paths. Legacy path-based routing is also supported for backwards compatibility.

**Usage Tracking and Observability** 📊

Monitor subscription-level token consumption, request counts, and rate-limit violations through the MaaS observability dashboard. Track consumption metrics for cost allocation and billing. Export usage data in CSV format for cost attribution and showback reporting to finance teams. The MaaS observability dashboard is a Technology Preview feature in Red Hat OpenShift AI.

**External Models** ☁️

Route inference requests to models hosted by external cloud providers such as AWS Bedrock, Azure OpenAI, or Google Vertex AI through the same MaaS gateway used for locally deployed models. External models is a Technology Preview feature in Red Hat OpenShift AI.

**Multi-tenancy** 🏢

Provision isolated tenants for Models-as-a-Service where each tenant receives a dedicated gateway, identity realm, namespace, and API infrastructure. Multi-tenancy is a Technology Preview feature in Red Hat OpenShift AI.

## When to Use MaaS

The following table summarizes when MaaS is the right choice and when standard model serving is sufficient:

| **Use MaaS** | **Use Standard Model Serving** |
| --- | --- |
| Centralized governance across multiple teams or projects is required | Deploying models for single-team or single-user use cases |
| Token limit enforcement and usage tracking for cost control is needed | Prototyping or developing models in a single-user environment |
| You prefer declarative configuration management via GitOps | Simplified deployment is preferred over centralized control |
| Self-service API key management for users is required | Direct access to model endpoints is sufficient |
| Multi-provider external model routing is needed | Only local models are being served |

## Architecture Overview

MaaS uses Kubernetes custom resources for declarative configuration management, enabling integration with GitOps workflows and version control. The platform operates through several key components:

- **AI Gateway**: Routes traffic and enforces policies for model endpoints
- **Authorino**: Handles authentication and authorization with API key validation
- **MaaS API**: Manages subscriptions, authorization policies, and API key lifecycle
- **PostgreSQL Database**: Stores API key metadata and user associations
- **Kuadrant**: Provides rate limiting and policy enforcement capabilities

## Administrator Responsibilities

MaaS administration is divided into initial configuration and ongoing management:

### Initial Configuration

**Cluster Administrators:**

- Enable MaaS in the OpenShift AI operator
- Configure the underlying cluster infrastructure to support model serving
- Install required operators and dependencies
- Configure gateway and TLS infrastructure

**OpenShift AI Administrators:**

- Define the initial governance structure by creating subscriptions and authorization policies
- Assign users to groups
- Configure model quota and token limits for each subscription
- Validate that users can successfully access models through MaaS

### Ongoing Operations

**Cluster Administrators:**

- Scale MaaS components to handle increased load
- Apply software updates
- Troubleshoot infrastructure performance issues

**OpenShift AI Administrators:**

- Monitor usage metrics to track costs
- Adjust subscription configurations and user group assignments
- Modify token limits based on demand patterns
- Manage API keys for external consumers or assist users with key lifecycle management
- Troubleshoot authentication and authorization issues

## Prerequisites

This module builds on a cluster with OpenShift AI already configured. Before proceeding, ensure the following requirements are met:

### Platform Requirements

* Red Hat OpenShift 4.19.9 or later
* Red Hat OpenShift AI 3.5 or later installed and configured
* OpenShift CLI (`oc`) installed on your local workstation
* Cluster administrator access to install Operators and create cluster-scoped resources
* A functional ingress controller with valid TLS certificates for external access

### Operator Requirements

* Red Hat Connectivity Link Operator version 1.4.x installed in the `openshift-operators` namespace
* Distributed inference with llm-d enabled, including authentication for LLM Inference Service
* User Workload Monitoring enabled on your OpenShift cluster (required for metrics collection)

### Infrastructure Requirements

* A PostgreSQL database instance (version 14 or later) reachable from the OpenShift cluster network
  - This database is required for API key lifecycle management
  - OpenShift AI does not provide a PostgreSQL database - you must provision your own instance
* Gateway API infrastructure configured with:
  - GatewayClass resource (or using the pre-existing `openshift-default` GatewayClass)
  - Gateway resource named `maas-default-gateway` in the `openshift-ingress` namespace with required annotations
* TLS certificates configured for secure communication between components

> The required components for this module assume that modules 1-3 have been completed, particularly the distributed inference setup from Module 3.

## Lab Structure

This module consists of five hands-on sections that will guide you through deploying and managing Models-as-a-Service:

1. **Configure Models-as-a-Service** - Platform setup, gateway configuration, TLS setup, and component enablement
2. **Deploy Models with MaaS** - Creating model references, configuring inference services, and verifying deployments
3. **Managing Subscriptions** - Understanding and configuring subscription-based access control
4. **Authorization & API Keys** - Managing authorization policies and API key lifecycle
5. **Observability & Monitoring** - Enabling observability, monitoring usage, and troubleshooting issues
