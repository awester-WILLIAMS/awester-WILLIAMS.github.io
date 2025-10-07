---
Title: Add IBM Software Entitlement Key
hide:
    - toc
---

# Add IBM Software Entitlement Key

This procedure will update the global pull secret for the OpenShift cluster to allow for IBM Software Titles to be installed on the cluster. By updating the global pull secret IBM Software can be installed in all namespaces in the OpenShift cluster.

1. Retrieve IBM Software Entitlement Key

    [IBM Software Entitlement Key](https://myibm.ibm.com/products-services/containerlibrary)

2. Save the contents of the key in an environment variable

    ```bash
    export IBM_KEY='**********************************'
    ```

3. Verify the entitlement key

    ```bash
    podman login cp.icr.io --username cp --password $IBM_KEY
    ```

    ```{.text .no-copy title="Example Output"}
    Login Succeeded!
    ```

4. Extract the current global pull secret.

    ```bash
    oc extract secret/pull-secret -n openshift-config --keys=.dockerconfigjson --to=. --confirm
    ```

    !!! Note
        To display the current contents of the file .dockerconfigjson use ```python3 -m json.tool .dockerconfigjson```

5. Update the JSON file ```.dockerconfigjson``` with the icr.io credentials used in Step 3.

    ```bash
    oc registry login --registry="cp.icr.io" --auth-basic="cp:$IBM_KEY" --to=.dockerconfigjson
    ```

    ```{.text .no-copy title="Example Output"}
    Saved credentials for cp.icr.io into .dockerconfigjson
    ```

6. Update the global pull secret to used the new contents of the ```.dockerconfigjson``` file.  This will cause all nodes in the cluster to reload configuation details to update this secret.  It could take time for all nodes to get the updated configuration.

    ```bash
    oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson
    ```

    ```{.text .no-copy title="Example Output"}
    secret/pull-secret data updated
    ```

