---
Title: Configure Internal Image Registry Storage
hide:
    - toc
---

# Configure Internal Image Registry Storage

On platforms that do not provide shareable object storage, vSphere without VSAN for example, the OpenShift image registry operator bootstraps itself as `Removed`. This allows the installer to complete on these platform types.

For this project the client's developers will leverage source-to-image (S2I). One of the things that S2I does is push container images to the internal image registry. To enable this, we'll need to configure the internal image registry with storage. Let's configure the internal image registry so it uses NooBaa, an object storage type provided by OpenShift Data Foundation.

1. Validate that the `managementState` of the image registry operator is `Removed`.
    
    ```sh
    oc get config.image/cluster -ojsonpath='{.spec.managementState}{"\n"}'
    ```

    ```{.text .no-copy title="Example Output"}
    Removed
    ```

2. Create an object bucket claim (OBC) named image-registry in the openshift-image-registry namespace.
    
    ```sh
    oc apply -f - <<EOF
    apiVersion: objectbucket.io/v1alpha1
    kind: ObjectBucketClaim
    metadata:
      name: image-registry
      namespace: openshift-image-registry
    spec:
      storageClassName: openshift-storage.noobaa.io
      generateBucketName: image-registry
    EOF
    ```

3. Save the name of the OBC in a shell variable.
    
    ```sh
    export bucket_name=$(oc -n openshift-image-registry get obc image-registry -o jsonpath='{.spec.bucketName}')
    ```

4. Save the credentials in shell variables.
    
    ```sh
    export AWS_ACCESS_KEY_ID=$(oc -n openshift-image-registry get secret image-registry -o yaml | grep -w "AWS_ACCESS_KEY_ID:" | head -n1 | awk '{print $2}' | base64 --decode)
    export AWS_SECRET_ACCESS_KEY=$(oc -n openshift-image-registry get secret image-registry -o yaml | grep -w "AWS_SECRET_ACCESS_KEY:" | head -n1 | awk '{print $2}' | base64 --decode)
    ```

5. Create the secret named `image-registry-private-configuration-user`.
    
    ```sh
    oc -n openshift-image-registry create secret generic image-registry-private-configuration-user \
      --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=${AWS_ACCESS_KEY_ID} \
      --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=${AWS_SECRET_ACCESS_KEY}
    ```

6. Save the s3 route's hostname into a shell variable.
    
    ```sh
    export s3_hostname=$(oc -n openshift-storage get route s3 -o=jsonpath='{.spec.host}')
    ```

7. Copy the Ingress CA bundle into a ConfigMap named `image-registry-s3-bundle`.
    
    ```sh
    oc -n openshift-ingress extract secret/router-certs-default --confirm
    oc -n openshift-config create configmap image-registry-s3-bundle --from-file=ca-bundle.crt=./tls.crt
    ```
    
8. Patch the `config.imageregistry/cluster`.
    
    ```sh
    oc patch config.image/cluster -p '{"spec":{"managementState":"Managed","replicas":2,"storage":{"managementState":"Unmanaged","s3":{"bucket":'\"${bucket_name}\"',"region":"us-east-1","regionEndpoint":'\"https://${s3_hostname}\"',"virtualHostedStyle":false,"encrypt":true,"trustedCA":{"name":"image-registry-s3-bundle"}}}}}' --type=merge
    ```

9. Check the image registry Pods.
    
    ```sh
    oc -n openshift-image-registry get pods -l docker-registry=default
    ```

    ```{.text .no-copy title="Example Output"}
    NAME                              READY   STATUS    RESTARTS   AGE
    image-registry-7b555754d6-5cjqg   1/1     Running   0          2m15s
    image-registry-7b555754d6-k8hn7   1/1     Running   0          2m15s
    ```
    
10. Validate that S2I successfully pushes container images to the internal image registry.
    
    Ensure the `git` command is available on the bastion host.
    
    ```sh
    sudo yum install -y git-core
    ```

    Create a project.
    
    ```sh
    oc new-project validate-s2i
    ```
    
    Use S2I to create a `hello-world` application.
    
    ```sh
    oc new-app \
        --name hello-world \
        https://github.com/RedHatTraining/DO280-apps \
        --context-dir hello-world-nginx
    ```
    
    Follow the build logs.
    
    ```sh
    oc logs -f buildconfig/hello-world
    ```

    ```{.text .no-copy title="Example Output"}
    #...
    Writing manifest to image destination
    Successfully pushed image-registry.openshift-image-registry.svc:5000/validate-s2i/hello-world@sha256:a6cabaa667cd38a5d90220faab76881b0cf24232709fefdc309bee9b31492cd4
    Push successful
    ```

    Clean up.
    
    ```sh
    oc delete project validate-s2i
    ```

Congratulations the deployment is complete! You have created a platform for the client project team to work on, the application modernization journey can start.
