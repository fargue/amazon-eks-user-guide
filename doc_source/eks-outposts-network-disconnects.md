# Preparing for network disconnects<a name="eks-outposts-network-disconnects"></a>

You can continue to use your local Amazon EKS cluster on an Outpost if your local network has lost connectivity with the AWS Cloud\. This topic covers some of the important considerations to prepare for your local cluster to be in a disconnected state

**Considerations**
+ Local clusters enable stability and continued operations during temporary, unplanned network disconnects\. AWS Outposts remains a fully connected offering that acts as an extension of the AWS Cloud in your data center\. In the event of network disconnects between your Outpost and AWS Cloud, you should attempt to restore your connection following the [AWS Outposts rack network troubleshooting checklist](https://docs.aws.amazon.com/outposts/latest/userguide/network-troubleshoot.html) in the AWS Outposts User Guide\. For information on how to troubleshoot issues with local clusters, see [Troubleshooting local clusters for Amazon EKS on AWS Outposts](eks-outposts-troubleshooting.md)\.
+ Outposts emit a `ConnectedStatus` metric that you can use to monitor the connectivity state of your Outpost\. For more information, see [Outposts Metrics](https://docs.aws.amazon.com/outposts/latest/userguide/outposts-cloudwatch-metrics.html#outposts-metrics) in the AWS Outposts User Guide\.
+ Local clusters use IAM as the default authentication mechanism using the [AWS Identity and Access Management authenticator for Kubernetes](https://github.com/kubernetes-sigs/aws-iam-authenticator)\. Because IAM isn't available during network disconnects, local clusters support an alternative authentication mechanism using `x.509` certificates that you can use to connect to your cluster during network disconnects\. For information on how to obtain and use an `x.509` certificate for your cluster, see [Authenticating to your local cluster during a network disconnect](#outposts-network-disconnects-authentication)\.
+ If you are unable to access Route 53 during network disconnects, consider using local DNS servers in your on\- premises environment\. The Kubernetes control plane instances use static IP addresses\. You can configure the hosts that you use to connect to your cluster with the endpoint hostname and IP addresses as an alternative to using local DNS servers\. For more information, see [DNS](https://docs.aws.amazon.com/outposts/latest/userguide/how-outposts-works.html#dns) in the AWS Outposts User Guide\.
+ If you anticipate increases in application traffic during network disconnects, you can provision spare compute capacity in your cluster when connected to the cloud\. Amazon EC2 instances are included in the price of AWS Outposts, so running spare instances doesn't impact your AWS usage cost\.
+ During network disconnects, to enable create, update, and scale operations for workloads, your application’s container images must be accessible over the local network and your cluster must have enough capacity\. Local clusters don't host a container registry for you\. Container images are cached on the nodes if the pods have previously run on those nodes\. If you typically pull your application’s container images from Amazon ECR in the cloud, consider running a local cache or registry if you require create, update, and scale operations for workload resources during network disconnects\.
+ Local clusters use Amazon EBS as the default storage class for persistent volumes and the Amazon EBS CSI driver to manage the lifecycle of Amazon EBS persistent volumes\. During network disconnects, pods backed by Amazon EBS can't be created, updated, or scaled because these operations require calls to the Amazon EBS API in the cloud\. If you're deploying stateful workloads on local clusters and require create, update, or scale operations during network disconnects, consider using an alternative storage mechanism\.
+ The Amazon EKS control plane logs are cached locally on the Kubernetes control plane instances during network disconnects\. Upon reconnect, the logs are sent to CloudWatch Logs in the parent AWS Region\. You can use your choice of tools such as Prometheus, Grafana, and Amazon EKS partner solutions to monitor the cluster locally using the Kubernetes API server’s metrics endpoint or using Fluent Bit for logs\.
+ If you are using the AWS Load Balancer Controller on Outposts for application traffic, existing pods fronted by the AWS Load Balancer Controller continue to receive traffic during network disconnects\. New pods created during network disconnects don't receive traffic until the Outpost is reconnected to the AWS Cloud\. Consider setting the replica count for your applications while connected to the AWS Cloud to accommodate your scaling needs during network disconnects\.
+ The Amazon VPC CNI plugin for Kubernetes default to [secondary IP mode](https://aws.github.io/aws-eks-best-practices/networking/vpc-cni/#overview)\. It is configured with `WARM_ENI_TARGET`=`1`, which allows the plugin to keep "a full elastic network interface" of available IP addresses available\. Consider changing `WARM_ENI_TARGET`, `WARM_IP_TARGET`, and `MINIMUM_IP_TARGET` values according to your scaling needs during a disconnected state\. For more information, see the [https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md) file for the plugin on GitHub\. For a list of the maximum number of pods supported by each instance type, see the [https://github.com/aws/amazon-vpc-cni-k8s/blob/master/misc/eni-max-pods.txt](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/misc/eni-max-pods.txt) file on GitHub\.

## Authenticating to your local cluster during a network disconnect<a name="outposts-network-disconnects-authentication"></a>

AWS Identity and Access Management isn't available during network disconnects\. You aren't able to authenticate to your local cluster using IAM credentials while disconnected\. You can connect to your cluster over your local network using `x509` certificates when disconnected\. You need to download and store a client `X509` certificate to use during disconnects\. In this topic you learn how to create and use the certificate to authenticate to your cluster when it's in a disconnected state\.

1. Create a certificate signing request\.

   1. Generate a certificate signing request\.

      ```
      openssl req -new -newkey rsa:4096 -nodes -days 365 \
          -keyout admin.key -out admin.csr -subj "/CN=admin"
      ```

   1. Create a certificate signing request in Kubernetes\.

      ```
      BASE64_CSR=$(cat admin.csr | base64 -w 0)
      cat << EOF > admin-csr.yaml
      apiVersion: certificates.k8s.io/v1
      kind: CertificateSigningRequest
      metadata:
        name: admin-csr
      spec:
        signerName: kubernetes.io/kube-apiserver-client
        request: ${BASE64_CSR}
        usages:
        - client auth
      EOF
      ```

1. Create a certificate signing request using `kubectl`\.

   ```
   kubectl create -f admin-csr.yaml
   ```

1. Check the status of the certificate signing request\.

   ```
   kubectl get csr admin-csr
   ```

   The example output is as follows\.

   ```
   NAME       AGE   REQUESTOR                       CONDITION
   admin-csr  11m   kubernetes-admin                Pending
   ```

   Kubernetes has created the certificate signing request\.

1. Approve the certificate signing request\.

   ```
   kubectl certificate approve admin-csr
   ```

1. Recheck the certificate signing request status for approved\.

   ```
   kubectl get csr admin-csr
   ```

   The example output is as follows\.

   ```
   NAME       AGE   REQUESTOR                     CONDITION
   admin-csr  11m   kubernetes-admin              Approved
   ```

1. Retrieve and verify the certificate\.

   1. Retrieve the certificate\.

      ```
      kubectl get csr admin-csr -o jsonpath='{.status.certificate}' | base64 --decode > admin.crt
      ```

   1. Verify the certificate\.

      ```
      cat admin.crt
      ```

1. Create a cluster role binding for an `admin` user\.

   ```
   kubectl create clusterrolebinding admin --clusterrole=cluster-admin \
       --user=admin --group=system:masters
   ```

1. Generate a user\-scoped kubeconfig for a disconnected state\. 

   You can generate a `kubeconfig` file using the downloaded `admin` certificates\. Replace *my\-cluster* and *apiserver\-endpoint* in the following commands\.

   ```
   aws eks describe-cluster --name my-cluster \
       --query "cluster.certificateAuthority" \
       --output text | base64 --decode > ca.crt
   ```

   ```
   kubectl config --kubeconfig admin.kubeconfig set-cluster my-cluster \
       --certificate-authority=ca.crt --server apiserver-endpoint --embed-certs
   ```

   ```
   kubectl config --kubeconfig admin.kubeconfig set-credentials admin \
       --client-certificate=admin.crt --client-key=admin.key --embed-certs
   ```

   ```
   kubectl config --kubeconfig admin.kubeconfig set-context admin@my-cluster \
       --cluster my-cluster --user admin
   ```

   ```
   kubectl config --kubeconfig admin.kubeconfig use-context admin@my-cluster
   ```

1. View your `kubeconfig` file\.

   ```
   kubectl get nodes --kubeconfig admin.kubeconfig
   ```

1. If you have services already in production on your Outpost, then skip this step\. If Amazon EKS is the only service running on your Outpost and the Outpost isn't currently in production, then you can simulate a network disconnect\. Before you go into production with your local cluster, you can simulate a disconnect to make sure that you can still access your cluster when it's in a disconnected state\. 

   1. Apply firewall rules on the networking devices that connect your Outpost to the AWS Region\. This disconnects the service link of the Outpost\. You aren't able to create any new instances\. Currently running instances lose connectivity to the AWS Region and the internet\.

   1. You can test the connection to your local cluster while disconnected using the `x509` certificate\. Make sure to change your `kubeconfig` to the `admin.kubeconfig` you created in a previous step\. Replace *my\-cluster* with the name of your local cluster\.

      ```
      kubectl config use-context admin@my-cluster --kubeconfig admin.kubeconfig
      ```

   If you notice any issues with your local clusters while they're in a disconnected state, then we recommend opening a support ticket\.