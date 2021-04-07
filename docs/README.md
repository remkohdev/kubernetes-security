# Kubernetes Security

## Kubernetes Networking

1. **Kubernetes Networking 101** (60 mins), you will use different ways to control traffic on a Kubernetes cluster with Service types, Ingress, Network Policy and Calico. Start [here](https://ibm.github.io/kubernetes-networking/services/).
2. **Kubernetes Network Security using a Virtual Private Cloud (VPC)** (90 mins), you will deploy a guestbook application to a Kubernetes cluster in a Virtual Private Cloud (VPC) Gen2, you will create the VPC, add a subnet, attach a public gateway, and update a security group with rules to allow inbound traffic to the guestbook application. Start [here](https://ibm.github.io/kubernetes-networking/vpcgen2/).
3. **[Istio](https://ibm.github.io/istio101/)**, use Istio to manage network traffic, load balance across microservices, enforce access policies, verify service identity, and more.

## Configuration Management

1. [Application Configuration for Kubernetes](config/README.md)
    1. [Lab0. Setup](kubeconfig/0_setup.md)
    1. [Lab1. Container Configuration](kubeconfig/1_container_config.md)
    1. [Lab2. Using Environment Variables in Pod Config](kubeconfig/2_config_using_env.md)
    1. [Lab3. Store Key-Value Pairs using ConfigMap](kubeconfig/3_config_using_configmap.md)
    1. [Lab4. Store Sensitive Data using Secrets](kubeconfig/4_config_using_secrets.md)
    1. [Lab5. Pull an Image from a Private Registry](kubeconfig/5_config_private_registry.md)
1. Key Management Services (KMS)
    1. [Lab1. Encrypt Secrets using a Cloud-Managed Vault Service (IBM Secrets Manager)](vault/1_ibm_secrets_manager.md)
    1. Lab 2. Using Vault on OpenShift
        1. [Lab2. Setup Internal Vault with Vault Agent Injector on OpenShift](2_setup_internal_vault.md)
        1. [Lab3. Access Internal Vault using Vault Agent Injector](3_access_internal_vault_using_injector_annotations.md)
