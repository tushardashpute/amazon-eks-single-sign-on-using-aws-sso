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
      --group system:masters \
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
            - system:masters
            rolearn: arn:aws:iam::176886134554:role/AWSReservedSSO_ReadOnlyAccess_c77f42e70907aa5d
            username: cluster-view-only
          - groups:
            - system:masters
            rolearn: arn:aws:iam::176886134554:role/AWSReservedSSO_AdministratorAccess_026e0779fc59ced0
            username: cluster-admin
        mapUsers: |
          []
      kind: ConfigMap


