# Running CMK at OpenShift Container Platform (OCP) 4.1

## Preparation
- Clone the upstream repo
    ```
    git clone https://github.com/intel/CPU-Manager-for-Kubernetes.git
    ```
- Modify all Pods definition to use local containr registry
    ```
    sed -r -i.bak "s|^(\s*)image:.*|    image: bastion.shift.zone:5000/cmk/cmk:v1.3.1|"  `find ./resources/pods -name "*.yaml"`

- Modify all Pods definitions with a namespace
```
sed -r -i 's|^(.*)namespace.*|  namespace: intel-cmk|' `find ./resources/pods -name "*.yaml"`
```

- Update CMK Cluster Init Pod configuration as per [upstream github](https://github.com/intel/CPU-Manager-for-Kubernetes/blob/master/docs/operator.md#prepare-cmk-nodes-by-running-cmk-cluster-init) documentation

    - To prepare CMK for all nodes:
        ```
        sed -r -i 's|^(.*)/cmk/cmk.py.*|      \- "/cmk/cmk.py cluster-init --all-hosts --saname=cmk-serviceaccount --namespace=intel-cmk --cmk-img=bastion.shift.zone:5000/cmk/cmk:v1.3.1"|' ./resources/pods/cmk-cluster-init-pod.yaml
        ```

    - To prepare CMK for a specific list of nodes:
        ```
        sed -r -i 's|^(.*)/cmk/cmk.py.*|      \- "/cmk/cmk.py cluster-init --host-list=node1,node2,node3 --cmk-cmd-list=init,discover --saname=cmk-serviceaccount --namespace=intel-cmk --cmk-img=bastion.shift.zone:5000/cmk/cmk:v1.3.1"|' ./resources/pods/cmk-cluster-init-pod.yaml
        ```

    - To prepre CMK for a specific node:
        ```
        sed -r -i 's|^(.*)/cmk/cmk.py.*|      \- "/cmk/cmk.py cluster-init --host-list=worker-2.ocp4poc.lab.shift.zone --cmk-cmd-list=init,discover --saname=cmk-serviceaccount --namespace=intel-cmk --cmk-img=bastion.shift.zone:5000/cmk/cmk:v1.3.1"|' ./resources/pods/cmk-cluster-init-pod.yaml
        ```

- Build CMK container and tag with the local registry
    ```
    podman build -t bastion.shift.zone:5000/cmk/cmk:v1.3.1 -f Dockerfile
    ```

- Upload image to local registry
    ```
    podman push bastion.shift.zone:5000/cmk/cmk:v1.3.1
    ```


## Create Namespace, Service Account and Deploy

- Create namespace
    ```
    oc new-project intel-cmk
    ```

- Create ServiceAccount and assign required RBACs and Privileges
    ```
    oc create serviceaccount cmk-serviceaccount -n intel-cmk
    ```

-  Update and apply RBACs
    ```
    sed -r -i.bak "s|^(.*)namespace.*|  namespace: intel-cmk|g" ./resources/authorization/cmk-rbac-rules.yaml
    ```

- Apply RBACs, Roles and SCCs
    ```
    oc create -f ./resources/authorization/cmk-rbac-rules.yaml
 
    oc adm policy add-role-to-user admin system:serviceaccount:intel-cmk:cmk-serviceaccount -n intel-cmk

    oc adm policy add-cluster-role-to-user view system:serviceaccount:intel-cmk:cmk-serviceaccount -n intel-cmk

    oc adm policy add-scc-to-user privileged -n intel-cmk -z cmk-serviceaccount
    ```



- Deploy Init Pod and wait for it to complete
    ```
    oc create -f ./resources/pods/cmk-cluster-init-pod.yaml -n intel-cmk

    oc get pods -w
    ```
## Test running the CMK Isolate Hello World Pod

- Add `hostPath` volumes to the pod
    ```
    echo '
    volumes:
    - hostPath:
        # Change this to modify the CMK installation dir in the host file system.
        path: "/opt/bin"
        name: cmk-install-dir
    - hostPath:
        # Change this to modify the CMK config dir in the host file system.
        path: "/etc/cmk"
        name: cmk-conf-dir
    ' >> ./resources/pods/cmk-isolate-pod.yaml
    ```
- Deploy the Hello World CMK Pod
    NOTE: ONLY AFTER a sucessful deployment
    ```
    oc create -f ./resources/pods/cmk-isolate-pod.yaml
    ```

## Uninstall and remove RBAC, Privs and ServiceAccount

- Delete Init Pod
    ```
    oc delete -f ./resources/pods/cmk-cluster-init-pod.yaml  -n intel-cmk
    oc delete all --all -n intel-cmk
    ```

- Remove ServiceAccount, RBAC and Privileges
    ```
    oc delete serviceaccount cmk-serviceaccount -n intel-cmk

    oc delete -f ./resources/authorization/cmk-rbac-rules.yaml

    oc adm policy remove-role-from-user admin system:serviceaccount:intel-cmk:cmk-serviceaccount -n intel-cmk

    oc adm policy remove-cluster-role-from-user view system:serviceaccount:intel-cmk:cmk-serviceaccount -n intel-cmk

    oc adm policy remove-scc-from-user privileged -n intel-cmk -z cmk-serviceaccount
    ```
