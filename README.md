# Securing LKE Cluster Part 5: Using OPA Gatekeeper for Admissions Control

## Introduction

In this part of the **Securing LKE Cluster** series, we will explore how to enforce fine-grained policies in your Kubernetes (K8s) cluster using **OPA Gatekeeper**. OPA Gatekeeper is an **admission controller** that enables you to define, apply, and audit policies for Kubernetes resources, ensuring they meet security and operational standards before they are created. 

By the end of this guide, you'll understand:
- What **OPA Gatekeeper** is and how it works with **Open Policy Agent (OPA)**.
- How to deploy OPA Gatekeeper on your **Linode Kubernetes Engine (LKE)** cluster.
- How to create **constraint templates** and **constraints** to enforce policies.
- How OPA Gatekeeper can be used to enhance cluster security through **admissions control**.

---

## Step 1: Deploy an LKE Cluster and Install OPA Gatekeeper

Before we can use OPA Gatekeeper, ensure you have a running LKE cluster. If you need to set one up, follow these steps:

1. **Deploy an LKE cluster** using the Linode Cloud Manager or `linode-cli`.
2. **Download the kubeconfig file** to connect to your cluster:
    ```bash
    export KUBECONFIG=~/path-to-your-kubeconfig-file
    ```
3. **Verify your connection**:
    ```bash
    kubectl get nodes
    ```

Next, we’ll install OPA Gatekeeper.

1. **Install Gatekeeper** using Helm:
    ```bash
    helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
    helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace
    ```

2. **Verify the installation**:
    ```bash
    kubectl get pods -n gatekeeper-system
    ```
   Ensure all Gatekeeper pods are running.

---

## Step 2: Understanding OPA Gatekeeper Components

OPA Gatekeeper relies on the following components to work with your Kubernetes cluster:

1. **OPA (Open Policy Agent)**: A policy engine that evaluates policies defined in a high-level declarative language called **Rego**.
2. **Admission Controller**: Intercepts Kubernetes API requests and evaluates them against policies before allowing the resource to be created.
3. **Constraint Templates**: Reusable definitions that contain the Rego logic for specific policies.
4. **Constraints**: Specific rules that bind the template to Kubernetes resources and apply policy checks.

---

## Step 3: Creating a Policy with OPA Gatekeeper

To demonstrate Gatekeeper in action, we’ll create a policy to ensure that all Pods have specific labels for governance purposes. We will:
1. Define a **constraint template** that enforces label requirements.
2. Define a **constraint** that applies the label enforcement policy to all Pods in the "production" namespace.

### Define the Constraint Template

1. **Create a constraint template** (`k8srequiredlabels-template.yaml`):

    ```yaml
    apiVersion: templates.gatekeeper.sh/v1beta1
    kind: ConstraintTemplate
    metadata:
      name: k8srequiredlabels
    spec:
      crd:
        spec:
          names:
            kind: K8sRequiredLabels
      targets:
        - target: admission.k8s.gatekeeper.sh
          rego: |
            package k8srequiredlabels
            violation[{"msg": msg}] {
              required := {"app", "team"}
              provided := {label | input.review.object.metadata.labels[label]}
              missing := required - provided
              count(missing) > 0
              msg := sprintf("Missing required labels: %v", [missing])
            }
    ```

This should render properly when pasted into GitHub. Let me know if you need further adjustments!

2. **Apply the template**:
    ```bash
    kubectl apply -f k8srequiredlabels-template.yaml
    ```

This constraint template defines a rule that requires all Pods to have the labels "app" and "team".

### Define the Constraint

1. **Create a constraint** (`k8srequiredlabels-constraint.yaml`) that references the template:
    ```yaml
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sRequiredLabels
    metadata:
      name: pod-must-have-labels
    spec:
      match:
        kinds:
          - apiGroups: [""]
            kinds: ["Pod"]
        namespaces: ["production"]
      parameters:
        labels: ["app", "team"]
    ```

2. **Apply the constraint**:
    ```bash
    kubectl apply -f k8srequiredlabels-constraint.yaml
    ```

Now, any Pod created in the "production" namespace without the labels "app" and "team" will be rejected by the Gatekeeper admission controller.

---

## Step 4: Testing the Policy

Let’s verify that OPA Gatekeeper is enforcing the policy correctly by creating two Pods: one that violates the policy and one that complies.

1. **Create a Pod without labels** (`no-label-pod.yaml`):
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: no-label-pod
      namespace: production
    spec:
      containers:
      - name: nginx
        image: nginx
    ```

2. **Attempt to apply the Pod**:
    ```bash
    kubectl apply -f no-label-pod.yaml
    ```

This will result in an error:
```
Error: admission webhook "validation.gatekeeper.sh" denied the request: Missing required labels: {"app","team"}
```

This indicates that the policy is being enforced correctly.

3. **Create a compliant Pod** (`labeled-pod.yaml`):
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: labeled-pod
      namespace: production
      labels:
        app: my-app
        team: my-team
    spec:
      containers:
      - name: nginx
        image: nginx
    ```

4. **Apply the Pod**:
    ```bash
    kubectl apply -f labeled-pod.yaml
    ```

This Pod will be successfully created because it meets the policy’s requirements.

---

## Step 5: Auditing Existing Resources

OPA Gatekeeper doesn’t just prevent invalid resources from being created—it can also audit existing resources to identify violations.

1. **Run an audit**:
    ```bash
    kubectl get constraintpodstatus
    ```

This will provide a list of existing resources that violate the active policies, giving you insight into resources that may need to be updated for compliance.

---

## Step 6: Real-World Use Cases for OPA Gatekeeper

OPA Gatekeeper’s flexibility allows for various security and operational governance use cases, including:

- **Security**: Preventing privileged containers, ensuring non-root users, blocking access to specific registries.
- **Resource Management**: Enforcing CPU and memory limits for all Pods.
- **Compliance**: Enforcing regulatory or organizational policies across namespaces or clusters.


1. **Create a constraint template** for resource limits (`k8srequiredlimits-template.yaml`):

    ```yaml
    apiVersion: templates.gatekeeper.sh/v1beta1
    kind: ConstraintTemplate
    metadata:
      name: k8srequiredlimits
    spec:
      crd:
        spec:
          names:
            kind: K8sRequiredLimits
      targets:
        - target: admission.k8s.gatekeeper.sh
          rego: |
            package k8srequiredlimits
            violation[{"msg": msg}] {
              input.review.kind.kind == "Pod"
              container := input.review.object.spec.containers[_]
              not container.resources.limits.cpu
              not container.resources.limits.memory
              msg := "Pod must have CPU and memory limits set"
            }
    ```


2. **Create the corresponding constraint**:
    ```yaml
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sRequiredLimits
    metadata:
      name: require-limits-on-pods
    spec:
      match:
        kinds:
          - apiGroups: [""]
            kinds: ["Pod"]
    ```

With this, Pods missing CPU and memory limits will be rejected, ensuring better resource governance.

---

## Conclusion

By integrating **OPA Gatekeeper** into your LKE cluster, you’ve gained powerful control over Kubernetes admissions. Gatekeeper ensures that only resources complying with defined security, operational, or compliance policies are admitted into the cluster. By combining Gatekeeper’s policy enforcement capabilities with real-time audits, you can proactively secure your cluster, preventing unauthorized actions and ensuring governance across namespaces.

This guide concludes Part 4 of the **Securing LKE Cluster** series. In the next part, we’ll explore more advanced security configurations, continuing our journey toward a robust and secure Kubernetes environment.
