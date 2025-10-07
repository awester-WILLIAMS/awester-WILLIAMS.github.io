---
hide:
  - toc
---

# LDAP Configuration 

- [Authentication Providers with AD](https://rhthsa.github.io/openshift-demo/infrastructure-authentication-providers.html)
- [Adding Active Directory OAUTH Provider](https://myopenshiftblog.com/adding-active-directory-oauth-provider/)
- [Configuring Active Directory in RHOS](https://myopenshiftblog.com/adding-active-directory-oauth-provider/)

## Overview

LDAP (Lightweight Directory Access Protocol) integration with OpenShift allows organizations to:
- Authenticate cluster users against an existing directory service
- Implement Single Sign-On (SSO) for cluster access
- Centralize user management across the enterprise
- Map LDAP groups to OpenShift roles for authorization

## LDAP Integration Methods

OpenShift supports two primary methods for LDAP integration:

1. **Direct LDAP Integration**: Configure OpenShift to communicate directly with LDAP
2. **LDAP through an identity provider**: Use a third-party identity provider that connects to LDAP

## Configuring Direct LDAP Integration

### Prerequisites

- Network connectivity between OpenShift and the LDAP server
- A service account in LDAP with read access to user and group information
- TLS certificates if using LDAPS (recommended)

### Configuration Process

1. Create a ConfigMap with the CA certificate (for LDAPS):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ldap-ca
  namespace: openshift-config
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDCzCCAfOgAwIBAgIQVfgGT1XBOt6p7kiFWFN6YDANBgkqhkiG9w0BAQsFADAd
    ...
    -----END CERTIFICATE-----
```

2. Edit the OAuth configuration:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ldap-provider
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - sAMAccountName
      bindDN: "CN=service-account,OU=Service Accounts,DC=example,DC=com"
      bindPassword:
        name: ldap-secret
      ca:
        name: ldap-ca
      insecure: false
      url: "ldaps://ldap.example.com/DC=example,DC=com?sAMAccountName?sub?(objectClass=person)"
```

3. Create a Secret with the bind password:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ldap-secret
  namespace: openshift-config
type: Opaque
stringData:
  bindPassword: "password123"
```

## Group Synchronization

OpenShift can synchronize LDAP groups to manage role-based access control:

1. Create a sync configuration file:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldaps://ldap.example.com
bindDN: CN=service-account,OU=Service Accounts,DC=example,DC=com
bindPassword: password123
insecure: false
ca: /path/to/ca.crt
rfc2307:
  groupsQuery:
    baseDN: "OU=Groups,DC=example,DC=com"
    scope: sub
    derefAliases: never
    filter: "(objectClass=group)"
  groupUIDAttribute: dn
  groupNameAttributes: [ cn ]
  groupMembershipAttributes: [ member ]
  usersQuery:
    baseDN: "OU=Users,DC=example,DC=com"
    scope: sub
    derefAliases: never
    filter: "(objectClass=person)"
  userUIDAttribute: dn
  userNameAttributes: [ sAMAccountName ]
```

2. Run the sync command:

```bash
oc adm groups sync --sync-config=ldap-sync-config.yaml --confirm
```

## Automating Group Synchronization

To automate LDAP group synchronization, create a CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ldap-group-sync
  namespace: openshift-authentication
spec:
  schedule: "0 */12 * * *"  # Every 12 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: ldap-group-sync
            image: registry.redhat.io/openshift4/ose-cli:latest
            command:
            - /bin/bash
            - -c
            - |
              oc adm groups sync --sync-config=/etc/config/ldap-sync-config.yaml --confirm
            volumeMounts:
            - name: config
              mountPath: /etc/config
            - name: kubeconfig
              mountPath: /kubeconfig
          volumes:
          - name: config
            configMap:
              name: ldap-sync-config
          - name: kubeconfig
            secret:
              secretName: ldap-sync-kubeconfig
          restartPolicy: OnFailure
```

## Troubleshooting LDAP Integration

### Common Issues and Solutions

1. **Connection Issues**:
   - Verify network connectivity: `nc -zv ldap-server.example.com 636`
   - Check firewall rules allow LDAP traffic

2. **Authentication Failures**:
   - Verify bindDN and bindPassword
   - Check service account permissions in LDAP

3. **Group Synchronization Issues**:
   - Verify LDAP query filters match your directory structure
   - Check group membership attributes (member vs memberOf)

4. **Certificate Issues**:
   - Ensure CA certificates are correctly formatted and trusted
   - Verify certificate expiration dates

### Debugging Tips

View OAuth pod logs:

```bash
oc logs -f $(oc get pods -n openshift-authentication -o name | grep oauth)
```

Test LDAP connection from within the cluster:

```bash
oc debug deployment/oauth-openshift -n openshift-authentication
# Inside the debug pod
ldapsearch -H ldaps://ldap.example.com -D "CN=service-account,OU=Service Accounts,DC=example,DC=com" \
  -w password123 -b "DC=example,DC=com" -s sub "(objectClass=person)"
```

## Best Practices

1. Always use LDAPS (LDAP over SSL/TLS) for secure communication
2. Create a dedicated service account in LDAP with minimal permissions
3. Implement regular group synchronization to keep permissions current
4. Use a specific filter to limit the scope of users and groups
5. Monitor authentication logs for suspicious activity
6. Document your LDAP configuration for future reference
7. Test authentication changes in a non-production environment first

