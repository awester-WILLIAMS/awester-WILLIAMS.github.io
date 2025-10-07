# Roles and Role Bindings

[Authentication and authorization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index)

## Overview

In OpenShift Container Platform, Role-Based Access Control (RBAC) objects determine what actions users can perform on which resources. Roles and RoleBindings are key components of this access control system.

## Roles

A Role contains rules that represent a set of permissions. Roles define what actions can be performed on which resources.

### Types of Roles

1. **Role**: Namespace-scoped roles that define permissions within a specific namespace.
2. **ClusterRole**: Cluster-wide roles that define permissions across the entire cluster.

### Default Cluster Roles

OpenShift comes with several pre-defined ClusterRoles:

- **admin**: Full control within a namespace
- **basic-user**: Basic user access
- **cluster-admin**: Superuser access to manage any resource in the cluster
- **edit**: Modify resources in a namespace
- **view**: Read-only access to most objects in a namespace

## Role Bindings

RoleBindings grant the permissions defined in a role to users or groups. A RoleBinding maps users and groups to roles.

### Types of Role Bindings

1. **RoleBinding**: Grants permissions within a specific namespace
2. **ClusterRoleBinding**: Grants cluster-wide permissions

## Managing Roles and Role Bindings

### Viewing Roles and Role Bindings

```bash
# List roles in a namespace
oc get roles -n <namespace>

# List cluster roles
oc get clusterroles

# List role bindings in a namespace
oc get rolebindings -n <namespace>

# List cluster role bindings
oc get clusterrolebindings
```

### Creating Roles and Role Bindings

**Creating a Role**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**Creating a RoleBinding**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### CLI Commands for RBAC

```bash
# Add a role to a user
oc adm policy add-role-to-user <role> <username> -n <namespace>

# Add a cluster role to a user
oc adm policy add-cluster-role-to-user <clusterrole> <username>

# Remove a role from a user
oc adm policy remove-role-from-user <role> <username> -n <namespace>

# Add a role to a group
oc adm policy add-role-to-group <role> <groupname> -n <namespace>
```

## Step-by-Step Example: Setting up Developer Access

This example walks through creating a developer user with access to the mkdocs namespace.

### 1. Create the Namespace

First, create the mkdocs namespace if it doesn't already exist:

```bash
oc new-project mkdocs
```

### 2. Create a User (htpasswd provider)

If using htpasswd as your identity provider:

```bash
# Create or update htpasswd file with a new user
htpasswd -c -B -b /path/to/htpasswd.file developer password123

# Update the OAuth configuration to use this file
oc create secret generic htpasswd-secret --from-file=htpasswd=/path/to/htpasswd.file -n openshift-config

# Apply the OAuth configuration
oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
EOF
```

### 3. Create a Developer Group

Create a group for developers:

```bash
oc adm groups new devs
```

### 4. Add the User to the Group

Add the developer user to the devs group:

```bash
oc adm groups add-users devs developer
```

### 5. Create a Custom Role for MkDocs Access

Create a role that provides full access to resources in the mkdocs namespace:

```bash
cat <<EOF | oc create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mkdocs-admin
  namespace: mkdocs
rules:
- apiGroups: [""]  # Core API group
  resources: ["pods", "services", "endpoints", "persistentvolumeclaims", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["route.openshift.io"]
  resources: ["routes"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
```

### 6. Bind the Role to the Developer Group

Create a RoleBinding to assign the mkdocs-admin role to the devs group:

```bash
cat <<EOF | oc create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devs-mkdocs-admin
  namespace: mkdocs
subjects:
- kind: Group
  name: devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: mkdocs-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 7. Verify the Access

You can verify the setup with:

```bash
# Check that the group exists with the user as a member
oc get group devs

# Verify that the role exists
oc get role mkdocs-admin -n mkdocs -o yaml

# Verify the role binding
oc get rolebinding devs-mkdocs-admin -n mkdocs -o yaml
```

### 8. Test the Access

Have the developer log in and verify they can access the mkdocs namespace:

```bash
# Login as developer
oc login -u developer

# Verify access to mkdocs namespace
oc project mkdocs
oc get pods
oc get deployments
```

## Best Practices

- Follow the principle of least privilege
- Use namespace-scoped roles when possible instead of cluster roles
- Regularly audit role bindings to ensure appropriate access
- Consider using groups to manage permissions for multiple users
- Document your RBAC strategy and keep it updated

## Troubleshooting Access Issues

If a user is unable to perform an action:

1. Check if they have the necessary role: `oc get rolebindings -n <namespace>`
2. Check what permissions their roles grant: `oc describe role <rolename> -n <namespace>`
3. Look at authorization failures in the audit logs