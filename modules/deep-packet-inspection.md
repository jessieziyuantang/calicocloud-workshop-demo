# Security: Deep packet inspection 


**Goal:** Use DPI on select workloads to efficiently make use of cluster resources and minimize the impact of false positives. 

>For each deep packet inspection resource (DeepPacketInspection), CalicoCloud creates a live network monitor that inspects the header and payload information of packets that match the [Snort community rules](https://www.snort.org/downloads/#rule-downloads). Whenever malicious activities are suspected, an alert is automatically added to the Alerts page in the Calico Manager UI.


## Steps

 1. Configure deep packet inspection in your target workload, we will use `hipstershop/frontend` as example.

   ```bash
   kubectl apply -f demo/dpi/sample-dpi-frontend.yaml
   ```  
   

 2. Configure resource requirements in IntrusionDetection.

  > For a data transfer rate of 1GB/sec on workload endpoints being monitored, we recommend a minimum of 1 CPU and 1GB RAM.
   
   ```bash
   kubectl apply -f demo/dpi/resource-dpi.yaml
   ```

 3. Verify deep packet inspection is running and the daemonset of `tigera-dpi` is also running. 

   ```bash
   kubectl get deeppacketinspections -n hipstershop
   ```

   ```bash
   kubectl get pods -n tigera-dpi
   ```
   
   ```text
   NAME               READY   STATUS    RESTARTS   AGE
   tigera-dpi-66858   1/1     Running   0          3h58m
   tigera-dpi-x67sj   1/1     Running   0          3h58m
   ```

 4. Trigger a snort alert basing on existing alert rules, we will use rule [21562](https://www.snort.org/rule_docs/1-21562)    

   ```bash
   SVC_IP=$(kubectl -n hipstershop get svc frontend-external -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
  
   # use below command if you are using `EKS` cluster, as EKS is using hostname instead of ip for loadbalancer
   SVC_IP=$(kubectl -n hipstershop get svc frontend-external -ojsonpath='{.status.loadBalancer.ingress[0].hostname}')

   # For RKE/kubeadm clusters, your `frontend-external` lb will be in <pending> mode, you can add external ip or use ingress controller to expose this svc. 
   kubectl edit -n hipstershop svc frontend-external
   ```

   ```text
   spec:
     clusterIP: 10.43.111.144
     clusterIPs:
     - 10.43.111.144
     externalIPs:    ## Add this line for your service
     - 34.xxx.xxx.88 ## Add your node public ip which have frontend pod running as endpoint for this value

   ```
   
   ```text 
   NAME                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
   frontend-external   LoadBalancer   10.43.111.144   34.xxx.xxx.88   80:31209/TCP   5h52m
   ```

   > A workaround for DPI alert not able to generate in UI as in policy `allow-tigera.guardian-access` block dpi alert to ES. We will update this policy with `allow tcp` in ingress before curl. 

   ```bash
   #curl your loadbalancer with head and smk data from outside of cluster
   curl http://$SVC_IP:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'
   ```

   ```bash
   #curl your external ip with Nodeport and smk data from outside of cluster
   curl http://34.xxx.xxx.88:31209 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'
   ```

 5. Confirm the `Signature Triggered Alert` in manager UI and also in Kibana `ee_event`
    ![Signature Alert](../img/signature-alert.png)


    ![ee event log](../img/ee-event-log.png)




[Next -> Wireguard Encryption](../modules/encryption.md) 

[Menu](../README.md)
