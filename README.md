## Highly Available Kubernetes Cluster Creation using Kubeadm

Hardware: 5 Ubuntu 20.04 nodes for the kubernetes cluster(3 for master, 2 for worker), 1 Ubuntu 20.04 node for Loadbalancer and Keepalived.

Steps: 
1. Deploy a Loadbalancer and Keepalived

    If you have a single loadbalancer you can omit Keeplived installation. But if you have multiple loadbalancer it's necessary to have a VIP which you will get from keeplived.

    Here I have only a single Loadbalancer and I'm using HAproxy for it. It is recommended that you should keep your Loadbalancer server separate from your kubernetes nodes.

    Install haproxy software in the instance
    ```
    sudo apt install --no-install-recommends software-properties-common

    sudo add-apt-repository ppa:vbernat/haproxy-2.4 -y
    ```

    ```
    sudo apt install haproxy=2.4.\*
    ```

    After installing HAproxy now its time to update the configuration. For a highly available cluster you need to have a minimum 3 nodes for master and considering this let's apply the below haproxy configuration
    ```
    # /etc/haproxy/haproxy.cfg
    #---------------------------------------------------------------------
    # Global settings
    #---------------------------------------------------------------------
    global
        log stdout format raw local0
        daemon

    #---------------------------------------------------------------------
    # common defaults that all the 'listen' and 'backend' sections will
    # use if not designated in their block
    #---------------------------------------------------------------------
    defaults
        mode                    http
        log                     global
        option                  httplog
        option                  dontlognull
        option http-server-close
        option forwardfor       except 127.0.0.0/8
        option                  redispatch
        retries                 1
        timeout http-request    10s
        timeout queue           20s
        timeout connect         5s
        timeout client          35s
        timeout server          35s
        timeout http-keep-alive 10s
        timeout check           10s

    #---------------------------------------------------------------------
    # apiserver frontend which proxys to the control plane nodes
    #---------------------------------------------------------------------
    frontend apiserver
        bind *:${APISERVER_DEST_PORT}
        mode tcp
        option tcplog
        default_backend apiserverbackend

    #---------------------------------------------------------------------
    # round robin balancing for apiserver
    #---------------------------------------------------------------------
    backend apiserverbackend
        option httpchk

        http-check connect ssl
        http-check send meth GET uri /healthz
        http-check expect status 200

        mode tcp
        balance     roundrobin
        
        server ${HOST1_ID} ${HOST1_ADDRESS}:${APISERVER_SRC_PORT} check verify none
        server ${HOST2_ID} ${HOST2_ADDRESS}:${APISERVER_SRC_PORT} check verify none
        server ${HOST3_ID} ${HOST3_ADDRESS}:${APISERVER_SRC_PORT} check verify none
        # [...]
        
    ```

    Here you need to replace the below variables with your own values
    - APISERVER_DEST_PORT: The port on which the loadbalancer will listen for incoming requests. This should be the same as the port you will use to access the kubernetes API server.
    - APISERVER_SRC_PORT: The port on which the kubernetes API server is running. This is usually 6443.
    - HOST1_ID: The ID of the first kubernetes master node. This can be any unique identifier for the node.
    - HOST1_ADDRESS: The IP address of the first kubernetes master node.
    - HOST2_ID: The ID of the second kubernetes master node. This can be any unique identifier for the node.
    - HOST2_ADDRESS: The IP address of the second kubernetes master node.
    - HOST3_ID: The ID of the third kubernetes master node. This can be any unique identifier for the node.
    - HOST3_ADDRESS: The IP address of the third kubernetes master node.

    After updating the configuration, restart the haproxy service
    ```
    sudo systemctl restart haproxy
    ```
    You can check the status of the haproxy service using the below command
    ```
    sudo systemctl status haproxy
    ```
    You can also check the haproxy logs using the below command
    ```
    sudo tail -f /var/log/haproxy.log
    ```
    You can also check the haproxy stats page using the below command
    ```
    sudo haproxy -c -f /etc/haproxy/haproxy.cfg
    ```

2. Install the dependencies and Configure the first master node
    Install the required packages from the script provided in all the kubernetes nodes. In the repository you will find a script called `install.sh` which will install the required packages for your kubernetes cluster. You can run the script using the below command
    ```
    sudo bash install.sh
    ```

    After instaling the required packages, you need to configure the kubernetes first master node. You can do this by running the below command
    ```
    kubeadm init --control-plane-endpoint ${LOADBALANCER_DNS}:${APISERVER_DEST_PORT} --upload-certs
    ```
    Here you need to replace the below variables with your own values
    - LOADBALANCER_DNS: The DNS name of the loadbalancer. This should be the same as the DNS name you will use to access the kubernetes API server.
    - APISERVER_DEST_PORT: The port on which the loadbalancer will listen for incoming requests. This should be the same as the port you will use to access the kubernetes API server.

    After running the above command, you will see a message like below
    ```
    You can now join any number of control-plane node by running the following command on each as a root:
    kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
    ```
3. Now one by one coyp the node joining command apply on the rest of the nodes.

4. If you complete the above tasks go the nodes that will run as worker and run the worker joining command that you got from the first master node.

5. Finally you successfully created a highly available kubernetes cluster using kubeadm.
6. You can check the status of the kubernetes cluster using the below command
    ```
    kubectl get nodes
    ```
    You should see all the nodes in the kubernetes cluster with the status `Ready`.
    You can also check the status of the kubernetes cluster using the below command
    ```
    kubectl get cs
    ```
    You should see all the components in the kubernetes cluster with the status `Healthy`.