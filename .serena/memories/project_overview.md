# Project Overview

## Purpose
The **apiextensions-apiserver** is a Kubernetes API Extensions API Server that provides the implementation for **CustomResourceDefinitions (CRDs)**. It is included as a delegated API server inside the main `kube-apiserver`.

## Key Characteristics
- **Read-only staging repository**: This is automatically published from `k8s.io/kubernetes/staging/src/k8s.io/apiextensions-apiserver`
- **No direct contributions**: All pull requests and issues should be made to the main Kubernetes repository at https://github.com/kubernetes/kubernetes
- **Implements Third Party Resources**: Based on the Kubernetes design proposal for third-party resources
- **Version compatibility**: HEAD of this repo matches HEAD of k8s.io/apiserver, k8s.io/apimachinery, and k8s.io/client-go

## Main Components
- **API Server Core**: Handles CRUD operations for custom resources
- **Discovery Controller**: Manages API discovery for CRDs
- **Schema Validation**: JSON schema validation, CEL expressions, and OpenAPI schema handling
- **Code Generation**: Uses Kubernetes code-generator for client libraries, deepcopy functions, and OpenAPI specs

## Entry Point
- `main.go`: Simple binary that starts the API server using a Cobra-based command structure
- The server creates a delegated API server that integrates with the main kube-apiserver
