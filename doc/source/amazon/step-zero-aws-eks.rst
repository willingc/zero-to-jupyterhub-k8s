.. _amazon-aws-eks:

Step Zero: Amazon Web Services (AWS) - Elastic Container with Kubernetes (EKS)
------------------------------------------------------------------------------

AWS recently released native support for Kubernetes with its
[EKS](https://aws.amazon.com/eks/) offering. 

.. note::

   EKS is only available in US West (Oregon) (us-west-2) and US East 
   (N. Virginia) (us-east-1).

This guide uses AWS to set up a cluster. It mirrors the steps found at
`Getting Started with Amazon EKS`_ and fills in some details that are absent
in the Amazon documentation but helpful for the user.

Procedure
~~~~~~~~~

1. Create an **IAM Role** for **EKS Service Role**.

   It should have the following policies:

   * `AmazonEKSClusterPolicy`
   * `AmazonEKSServicePolicy`
   
   From the user interface, select EKS as the service, then follow the default
   steps.
   
2. Create a **VPC** if you don't already have one. 

   This step has a lot of variability so specific settings are left to the
   user. One deployment example can be found at 
   `Getting Started with Amazon EKS`_, under 
   *Create your Amazon EKS Cluster VPC*.
   
3. Create a **Security Group** for the **EKS Control Plane** to use.
    
   You do not need to set any permissions on this. The following steps will 
   automatically define access control between the EKS Control Plane and the 
   individual nodes.

4. Create your EKS cluster (using the Amazon user interface).
 
   Use the **IAM Role** in step 1 and **Security Group** defined in step 3. 
   The cluster name is going to be used throughout the rest of this section.
   We'll use ``Z2JHKubernetesCluster`` as an example cluster name.
    
5. Install **kubectl** and **heptio-authenticator-aws**.

   Refer to `Getting Started with Amazon EKS`_ under 
   *Configure kubectl for Amazon EKS*.

6. Configure **kubeconfig**. 

   Also see `Getting Started with Amazon EKS`_ under 
   *Step 2: Configure kubectl for Amazon EKS*.

   From the user interface on AWS you can retrieve the ``endpoint-url`` and
   ``base64-encoded-ca-cert``. ``cluster-name`` is the name given in step 4.
   If you are using **profiles** in your AWS configuration, you can uncomment
   the ``env`` block and specify your profile as ``aws-profile``.

   .. code-block:: yaml
    
      apiVersion: v1
      clusters:
      - cluster:
        server: <endpoint-url>
        certificate-authority-data: <base64-encoded-ca-cert>
        name: kubernetes
        contexts:
        - context:
      	  cluster: kubernetes
      	  user: aws
      	  name: aws
      	  current-context: aws
      	  kind: Config
      	  preferences: {}
      	  users:
    	    - name: aws
    	      user:
    	      exec:
    	      apiVersion: client.authentication.k8s.io/v1alpha1
    	      command: heptio-authenticator-aws
    	      args:
              - "token"
                - "-i"
                  - "<cluster-name>"
    		        # - "-r"
    		        # - "<role-arn>"
    		        # env:
    		        # - name: AWS_PROFILE
    		        #   value: "<aws-profile>"


7. Verify ``kubectl`` works.

   .. code-block:: bash

      kubectl get svc    

   This should return ``kubernetes`` and ``ClusterIP``.
    
8. Create the nodes using **CloudFormation**.

    See `Getting Started with Amazon EKS`_ under
    *Step 3: Launch and Configure Amazon EKS Worker Nodes*.

    .. warning::

       If you are endeavoring to deploy on a private network, the 
       cloudformation template creates a public IP for each worker node though
       there is no route to get there if you specified only private subnets. 
       Regardless, if you wish to correct this, you can edit the 
       cloudformation template by changing 
       ``Resources.NodeLaunchConfig.Properties.AssociatePublicIpAddress`` from
       ``true`` to ``false``.
    
9. Create an AWS authentication **ConfigMap**.

   This is necessary for the workers to find the master plane.
   Download ``aws-auth-cm.yaml`` file.

   .. code-block:: bash

       curl -O https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/aws-auth-cm.yaml

   or copy it:

   .. code-block:: yaml

       apiVersion: v1
       kind: ConfigMap
       metadata:
       name: aws-auth
       namespace: kube-system
       data:
       mapRoles: |
       - rolearn: <ARN of instance role (not instance profile)>
         username: system:node:{{EC2PrivateDNSName}}
         groups:
         - system:bootstrappers
           - system:nodes


   To find the ARN of the instance role, you can pull up any node created in 
   Step 8. The nodes will be of the format ``<Cluster Name>-<NodeName>-Node``, 
   for example ``Z2JHKubernetesCluster-Worker-Node``. Click on the IAM Role
   for that node, you should see a ``Role ARN`` and ``Instance Profile ARN``s.
   Use the ``Role ARN`` in the above yaml file.

Then run:

    .. code-block:: bash

       kubectl apply -f aws-auth-cm.yaml


10. Preparing authenticator for Helm.

    .. note::

       There might be a better way to configure this. If you find a better
       way, please file an issue. Thanks.

    Since the described helm deployment in the next section uses RBAC, a
    ``system:anonymous`` user must be given access to administer the cluster. 
    This can be done by the following command:

    .. code-block:: bash

       kubectl create clusterrolebinding cluster-system-anonymous \
               --clusterrole=cluster-admin \
               --user=system:anonymous


.. _Getting Started with Amazon EKS: https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html

