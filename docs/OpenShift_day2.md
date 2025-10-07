---
hide:
  - toc
---

# Day 2 Tasks for OpenShift

## Current OpenShift Cluster

- OCP Version

### Walkthruough of Current Architecture and Install Process

- Hypervisor if Virtual Machines
- Installation Method (IPI or UPI)
- Storage Classes
    - RWX
    - RWO
    - ObjectStorage

    **Example Storage Classes**

    ```oc describe sc```

      | Storage Class Name          | Storage Type  | Provisioner                           | Description
      |-----------------------------|---------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------|
      | ocs-storagecluster-ceph-rbd | RWO, RWX      | openshift-storage.rbd.csi.ceph.com    | Provides RWO Filesystem volumes, and RWO and RWX Block volumes,reclaimspace.csiaddons.openshift.io/schedule=@weekly |
      | ocs-storagecluster-ceph-rgw | Object Bucket | openshift-storage.ceph.rook.io/bucket | Provides Object Bucket Claims (OBCs)                                                                                |
      | ocs-storagecluster-cephfs   | RWO, RWX      | openshift-storage.cephfs.csi.ceph.com | Provides RWO and RWX Filesystem volumes                                                                             |
      | openshift-storage.noobaa.io | Object Bucket | openshift-storage.noobaa.io/obc       | Provides Object Bucket Claims (OBCs)                                                                                |
      | thin-csi                    |               | csi.vsphere.vmware.com                | Default VMWare stroage class with platform enabled, access to the datastore                                         |

- Ingress Load Balancer Type? Depends on Installation Method
- DHCP or DHCP Reservations for control plane and worker nodes
- [IBM Software Entitlement Key](https://myibm.ibm.com/products-services/containerlibrary)
- [Redhat Pull Secret](https://console.redhat.com/openshift/install/pull-secret) 
- [Local Image Repository Status](./config-image-registry.md){target="_blank"}
- [Update Global Pull Secret](./update-global-pull-secret.md){target="_blank"}


### Identity Providers for OpenShift

- [HTPassword Identity Provider](./config-identity-provider.md)
- [Authentication Providers with AD](https://rhthsa.github.io/openshift-demo/infrastructure-authentication-providers.html)
- [Adding Active Directory OAUTH Provider](https://myopenshiftblog.com/adding-active-directory-oauth-provider/)
- [configuring Active Directory in RHOS](https://myopenshiftblog.com/adding-active-directory-oauth-provider/)
- [Authentication Providers with AD](https://rhthsa.github.io/openshift-demo/infrastructure-authentication-providers.html)

### Adding Roles and Role Bindings for Users

[Authentication and authorization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index)

### Monitoring and Alerting

- [Observability Overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/observability_overview/index)
- Get a health check of our current clusters, and setup, to determine action items
    - [Gathering Baseline Openshift Usage Information from Metrics](https://myopenshiftblog.com/gathering-baseline-openshift-usage-information-from-metrics/)
    - [Openshift Monitoring, Logging, Observability, and Troubleshooting (Part 1)](https://myopenshiftblog.com/openshift-observability/)

### Content Creators For Some Links

# [Keith Calligan - Red Hat - Senior Solutions Architect](https://myopenshiftblog.com/about-2-2-2-2-2-3/)

### Additional Links

- [Red Hat Developer e-books](https://developers.redhat.com/e-books)
- [Ask an OpenShift Admin (Ep 26): Day 2 Operations, part 1](https://www.youtube.com/watch?v=AM9mggoOu_E)
- [Ask an OpenShift Admin (Ep 26): Day 2 Operations, part 2](https://www.youtube.com/watch?v=k0vZQHWwxe8)
- [Understanding the OCP Install Process (Part 1)](https://myopenshiftblog.com/understanding-the-ocp-install-processes-part-1/)
- [Understanding the OCP Install Process (Part 2)](https://myopenshiftblog.com/understanding-the-ocp-install-processes-part-2/)
