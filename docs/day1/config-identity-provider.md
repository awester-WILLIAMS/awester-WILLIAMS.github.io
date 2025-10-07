---
Title: Configure the Identity Provider
hide:
    - toc
---

# Configure the Identity Provider

To enable the clients developers to log into the cluster and meet the security requirements, an Identity Provider (IdP) must be configured. The primary purpose of an IdP is to provide a centralized authentication mechanism for users accessing the cluster. This ensures that only authorized individuals can gain access to the cluster.

Although OpenShift supports many IdP's, for this example, we will use the HTPasswd provider, which is a popular and widely-supported option. The HTPasswd provider uses a password file to store user credentials, making it easy to manage access to the cluster.

It is also a best practice to delete the built-in `kubeadmin` user after configuring the IdP. This ensures that the only users able to log into the cluster are those who have been explicitly authorized by the administrators.

1. Ensure the `htpasswd` command is available on the bastion host.

    ```sh
    sudo dnf install -y httpd-tools
    ```

2. Create the htpasswd file with user `admin` and password `OCP4all!`.
    
    ```sh
    htpasswd -Bbc /tmp/htpasswd admin OCP4all!
    ```

3. Add the clients developers.
    
    ```sh
    for dev in abbott ben webb
    do
        htpasswd -b /tmp/htpasswd ${dev} OCP4all!
    done
    ```

4. Create a Secret in the `openshift-config` namespace with data from the htpasswd file, name it `localusers`.
    
    ```sh
    oc -n openshift-config create secret generic localusers \
    --from-file htpasswd=/tmp/htpasswd
    ```

5. Export the OAuth resource.
    
    ```sh
    oc get oauth cluster -o yaml > /tmp/oauth.yaml
    ```

6. Edit `/tmp/oauth.yaml`, under `spec:` add the HTPasswd identity provider.
    
    ```yaml title="/tmp/oauth.yaml"
    #...
    spec:
      identityProviders:
        - htpasswd:
            fileData:
              name: localusers
          name: localusers
          type: HTPasswd
          mappingMethod: claim
    ```
    
7. Update the OAuth resource.
    
    ```sh
    oc replace -f /tmp/oauth.yaml
    ```

8. Monitor the Pods in namespace `openshift-authentication`, they will restart, the rollout is not instant, so please be patient.

    ```sh
    watch oc -n openshift-authentication get pods
    ```

9.  Assign cluster-admin privileges to user admin.
    
    ```sh
    oc adm policy add-cluster-role-to-user cluster-admin admin
    ```

10. Ensure user admin can log in.
    
    ```sh
    oc login --insecure-skip-tls-verify=true -u admin
    ```

11. Verify user admin has cluster role cluster-admin, only users with this role are authorized to use get nodes.
    
    ```sh
    oc auth can-i get nodes
    ```

    ```{.text .no-copy title="Example Output"}
    Warning: resource 'nodes' is not namespace scoped

    yes
    ```

12. Log in as one of the developers and verify the use of get nodes.
    
    ```sh
    oc login --insecure-skip-tls-verify=true -u ben
    ```
    
    ```sh
    oc auth can-i get nodes
    ```

    ```{.text .no-copy title="Example Output"}
    Warning: resource 'nodes' is not namespace scoped

    no
    ```

13. List the users.
    
    ```sh
    oc get users
    ```

    ```{.text .no-copy title="Example Output"}
    NAME    UID                                    FULL NAME   IDENTITIES
    admin   a7cfbf3a-0892-40e0-9dcb-7ba37ecc1824               localusers:admin
    ben     0e3884e4-529b-4b5f-a7e6-2e333796d03a               localusers:ben
    ```

!!! note
    Only users who have logged in are listed when running the `oc get users` command.

!!! note
    In a normal environment the default admin account would be deleted. For this environment please **DO NOT** delete this account. These are the steps to delete the 'kubeadmin' user in a normal environment.

1. Switch back to user that has cluster-admin role. The user 'admin' is a user with cluster admin role in this example.
    
    ```sh
    oc login --insecure-skip-tls-verify=true -u admin
    ```

2. Delete user `kubeadmin`.
    
    ```sh
    oc -n kube-system delete secret kubeadmin
    ```
