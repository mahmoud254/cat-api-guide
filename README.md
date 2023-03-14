# 1. Deploying Atlas MongoDB

1. Deploying the database is very simple, I choose shared becuase it's free
![Alt text](./docs/mongo_1.png?raw=true "Architecture")

2. Choose the region, not that for the ips that can access the database, make sure
to add the the public ip of the nat gateway, that way only resources in the private
subents can access the mongo database.
![Alt text](./docs/mongo_2.png?raw=true "Architecture")

3. After the database is deployed, you can connect to it
![Alt text](./docs/mongo_3.png?raw=true "Architecture")


# 2. Creating pipeline for push events to deploy the app (there's another section for running the unit tests on PR)

1. We first need to create a secret in secrets manager that will hold the env variables for our backend app.
![Alt text](./docs/secret_new.png?raw=true "Architecture")

2. We now need to create the role that will be assummed by the pipeline to run the kubectl commands, the policy
that will be attached to the role can be seen in 'docs/EKS_ROLE_CODEBUILD_POLICY.json'. Make sure to replace the 'SECRET_ARN' 
with the secret created in the previous step. We will be using aws CodeBuild for CICD, next are the steps for it.
![Alt text](./docs/cicdRole.png?raw=true "Architecture")

4. Note how in the image below we didn't specify a source branch, you could
but a better approch is leave it and select which branch is built in the buildspec.yaml file
![Alt text](./docs/push_1.png?raw=true "Architecture")

4. We select the webhook to and we trigger buids based on push
![Alt text](./docs/push_2.png?raw=true "Architecture")

5. In the Environment secton there are some notes:
- we chose that image (aws/codbuild/amzonlinux2_x86_64-standard:2.0) because chhosing 
  aarch64 has a terminal that causes issues with some commands, like kubectl commands.
- We Enable Privileged to be able to use docker commands in the buildspec.yaml file
![Alt text](./docs/push_3.png?raw=true "Architecture")

6. We create a new role, and specify the bath of the build spec file
![Alt text](./docs/push_4.png?raw=true "Architecture")

7. In the additional configuration section we select our vpc and the private subnets as well as the vpc default security group
![Alt text](./docs/push_5.png?raw=true "Architecture") 

8. We need to add the env variables that will be used in the build pipeline (buildspec.yaml)
- AWS_REGION  --> the region where the EKS cluster is located
- REPOSITORY  --> cat images backend ecr uri
- EKS_KUBECTL_ROLE_ARN --> the arn of role created in step 2
- EKS_CLUSTER_NAME  --> the name of the eks cluster
![Alt text](./docs/pipeline_env.png?raw=true "Architecture") 

9. Now we can click the Create button. 

10. We need to update the role created by code build to have ecr access, we do that by attaching ECR full access policy to the
   role, I know fine grain access is the best and we could do that, but we will be accessing ECR alot to read/and write images
   so that's fine.
![Alt text](./docs/push_6.png?raw=true "Architecture")

11. We now need to update the trust relation of the role created in step 2 to allow it to be assummed by
the cod build role. You can find the whole json in the file 'docs/EKS_ROLE_CODEBUILD_POLICY_TRUST_RELATIONS.json"
Make sure to replace ACCOUNT_ID by the account id and CODEBUILD_ROLE with the code build role name (the role created in step 6)
![Alt text](./docs/trust_relation.png?raw=true "Architecture")

12. We need to update the aws-auth config map to allow the role created in step 2 to access the EKS cluster. we run:
- kubectl edit cm -n kube-system aws-auth
- then we add:
```bash
- rolearn: arn:aws:iam::ACCOUNT_ID:role/EksCICDDemoKubectlRole
  username: EksCICDDemoKubectlRole
  groups:
  - system:masters
```
![Alt text](./docs/aws_auth.png?raw=true "Architecture")

13. Now we can run the build and deploy the app to the EKS cluster
![Alt text](./docs/pipeline_finished.png?raw=true "Architecture")

# 3. Creating pipeline for PR to tun unit tests
The steps are mostly the same, I will add images for the differences:

1. Note how we changed the event type to only be PULL_REQUEST_CREATED
![Alt text](./docs/pr_1.png?raw=true "Architecture")

2. Note how we changed the image to be (aws/codbuild/amzonlinux2_x86_64-standard:4.0) because this image 
   has node js version 16.
![Alt text](./docs/pr_2.png?raw=true "Architecture")

3. We also add the backend applicatin env vars, (Mongoose connection, redis, port,...) because those 
   needs to be exposed while testing the application.
![Alt text](./docs/pr_3.png?raw=true "Architecture")

4. We also edit the buildspec file location to the buildspec for the tests
![Alt text](./docs/pr_4.png?raw=true "Architecture")

5. That's it. We dont't need to edit roles or create new role/
![Alt text](./docs/pr_5.png?raw=true "Architecture")

# 4. Seeing the application in action