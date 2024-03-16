# amazon-eks-single-sign-on-using-aws-sso
amazon-eks-single-sign-on-using-aws-sso

https://aws.amazon.com/blogs/containers/a-quick-path-to-amazon-eks-single-sign-on-using-aws-sso/

Assumption:
-------------------

You have AD enabled and SSO for AWS is integrated. If not Please [follow link.](https://github.com/tushardashpute/sso_eks_authentication)


Lets create two AD users eksadmin and eksreadonly.
------------------------------------------------
eksadmin
![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/ef187892-5990-4c30-9b1c-2051844c736a)

eksreadonly
![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/e15ec34f-fff3-4481-9af6-f7e1e7a82439)

Now add these users to AWS SSO app:

Gotot aure directory --> enterprize app --> select your app

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/dd26f19d-8f49-4b05-bff4-d7fa2e8b4be3)

We have added both eksadmin and eksreadonly users to app.

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/76819857-9b14-4e8a-87d7-bf4ba690e26f)

Let's verify users in AWS console as well:

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/84f3c96f-3492-45c9-a2fc-6c52989a37e9)

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/27308f9d-f6a4-4e75-b9ef-5630957b53cc)

Kubernetes RBAC and IAM federation
----------------------------------

Create aws-auth identity mapping for EKSClusterAdminAccess role.

aws-auth cm before:

      # kubectl get cm aws-auth -n kube-system -o yaml
      apiVersion: v1
      data:
        mapRoles: |
          - groups:
            - system:bootstrappers
            - system:nodes
            rolearn: arn:aws:iam::176886134554:role/eksctl-eksdemo-nodegroup-eksdemo-n-NodeInstanceRole-GH9DKQBuxdha
            username: system:node:{{EC2PrivateDNSName}}
      kind: ConfigMap
      metadata:
        creationTimestamp: "2024-03-16T05:56:45Z"
        name: aws-auth
        namespace: kube-system
        resourceVersion: "1416"
        uid: 210a92e9-2b43-4395-b002-69a9d7549d78

create Identity mapping:

      eksctl create iamidentitymapping \
      --cluster eksdemo  \
      --arn arn:aws:iam::176886134554:role/AWSReservedSSO_ReadOnlyAccess_c77f42e70907aa5d  \
      --username cluster-view-only  \
      --group system:reader \
      --region us-east-1

   
      eksctl create iamidentitymapping \
       --cluster eksdemo \
       --arn arn:aws:iam::176886134554:role/AWSReservedSSO_AdministratorAccess_026e0779fc59ced0 \
       --username cluster-admin \
       --group system:masters \
       --region=us-east-1	

kubectl get cm aws-auth -n kube-system -o yaml

      apiVersion: v1
      data:
        mapRoles: |
          - groups:
            - system:bootstrappers
            - system:nodes
            rolearn: arn:aws:iam::176886134554:role/eksctl-eksdemo-nodegroup-eksdemo-n-NodeInstanceRole-GH9DKQBuxdha
            username: system:node:{{EC2PrivateDNSName}}
          - groups:
            - system:reader
            rolearn: arn:aws:iam::176886134554:role/AWSReservedSSO_ReadOnlyAccess_c77f42e70907aa5d
            username: cluster-view-only
          - groups:
            - system:masters
            rolearn: arn:aws:iam::176886134554:role/AWSReservedSSO_AdministratorAccess_026e0779fc59ced0
            username: cluster-admin
        mapUsers: |
          []
      kind: ConfigMap

Now generate the sso profile for admin user first:

             C:\Users\tushar dashpute\.aws> **aws configure sso**
            SSO session name (Recommended): eksadmin
            SSO start URL [None]: https://d-9067fba349.awsapps.com/start#
            SSO region [None]: us-east-1
            SSO registration scopes [sso:account:access]: eksadmin
            Attempting to automatically open the SSO authorization page in your default browser.
            If the browser does not open or you wish to use a different device to authorize this request, open the following URL:
            
            https://device.sso.us-east-1.amazonaws.com/
            
            Then enter the code:
            
            KDPM-SDTF
            The only AWS account available to you is: 176886134554
            Using the account ID 176886134554
            The only role available to you is: AdministratorAccess
            Using the role name "AdministratorAccess"
            CLI default client Region [us-east-1]:
            CLI default output format [json]:
            CLI profile name [AdministratorAccess-176886134554]:
            
            To use this profile, specify the profile name using --profile, as shown:
            
            aws s3 ls --profile AdministratorAccess-176886134554

Now verify the AWS profile:

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/ff228567-b8bc-4d7d-8439-0e51c04f7403)

Generate the kubeconfig with this admin sso role:

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/81181f64-817c-4b5d-827a-543b148e88a3)

Let's try to create a sample deployment with this user and then delete it:

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/fe758207-7319-493a-880c-3c721bd116e1)

If you look at the kubeconfig you can see user authentication is done using the AWS_PROFILE  AdministratorAccess-176886134554

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/2257a7bb-4270-4a18-945a-ad15e875dc72)

Verify access with view-only profile:
-------------------------------------
Now generate the sso profile for viewonly user first:

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/a49ca35e-1ef1-4a77-bec0-8d01f975ddc8)

We got the error for deployment creation as user is not having permission to create deployment.

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/6a561fb8-2aa0-426a-a757-f93090b6f6f0)

If you look at the kubeconfig you can see user authentication is done using the AWS_PROFILE ReadOnlyAccess-176886134554

![image](https://github.com/tushardashpute/amazon-eks-single-sign-on-using-aws-sso/assets/74225291/ef1f84a5-e2b9-4cdb-a2ef-0c73488d7dda)


