## Bootstrapping an Amazon EC2 to run a Python Web App

**Introduction**

Manually setting up and configuring the packages required to run a Python web app using Nginx and uWSGI on a server can be time consuming — and it's tough to accomplish without any errors. But why do that hard work when you can automate it? Using AWS CDK, we can set up user data scripts and an infrastructure to preconfigure an EC2 instance - which in turn will turn a manual, time-intensive process into a snap. In this tutorial, we will be using a combination of bash scripts and AWS CodeDeploy to install and configure Nginx and uWSGI, set up a systemd service for uWSGI, and copy our application using CDK. Then, we are going to deploy our Python-based web application from a GitHub repository.

To deploy this web application I used **AWS CDK** in **AWS Cloud9** to create and deploy the underlying infrastructure. The infrastructure consists of an EC2 instance, a VPC, CI/CD pipeline, and accompanying resources like Security Groups and IAM Permissions. 

Using the `ec2-cdk directory`, all code was written into `lib/ec2-cdk-stack.ts` for the creation of the resource stack. 
A resource stack is a set of cloud infrastructure resources (in your particular case, they will be all AWS resources) that will be provisioned into a specific account. The account and Region where these resources are provisioned can be configured in the stack

In the resource stack, I create the following resources:

  * IAM roles: This role will be assigned to the EC2 instance to allow it to call other AWS services.
  * EC2 instance: The virtual machine you will use to host your web application.
  * Security group: The virtual firewall to allow inbound requests to your web application.
  * Secrets manager secret: This is a place where you will store your Github Token that we will use to authenticate the pipeline to it.
  * CI/CD Pipeline: This pipeline will consist of AWS CodePipeline, AWS CodeBuild, and AWS CodeDeploy.

**Final Resource Stack**
    
```Typescript
import * as cdk from 'aws-cdk-lib';
import { readFileSync } from 'fs';
import { Construct } from 'constructs';

import { Vpc, SubnetType, Peer, Port, AmazonLinuxGeneration, 
  AmazonLinuxCpuType, Instance, SecurityGroup, AmazonLinuxImage,
  InstanceClass, InstanceSize, InstanceType
} from 'aws-cdk-lib/aws-ec2';

import { Role, ServicePrincipal, ManagedPolicy } from 'aws-cdk-lib/aws-iam';
import { Pipeline, Artifact } from 'aws-cdk-lib/aws-codepipeline';
import { GitHubSourceAction, CodeBuildAction, CodeDeployServerDeployAction } from 'aws-cdk-lib/aws-codepipeline-actions';
import { PipelineProject, LinuxBuildImage } from 'aws-cdk-lib/aws-codebuild';
import { ServerDeploymentGroup, ServerApplication, InstanceTagSet } from 'aws-cdk-lib/aws-codedeploy';
import { SecretValue } from 'aws-cdk-lib';

export class PythonEc2BlogpostStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    // IAM
    // Policy for CodeDeploy bucket access
    // Role that will be attached to the EC2 instance so it can be 
    // managed by AWS SSM
    const webServerRole = new Role(this, "ec2Role", {
      assumedBy: new ServicePrincipal("ec2.amazonaws.com"),
    });

    // IAM policy attachment to allow access to
    webServerRole.addManagedPolicy(
      ManagedPolicy.fromAwsManagedPolicyName("AmazonSSMManagedInstanceCore")
    );
    
    webServerRole.addManagedPolicy(
      ManagedPolicy.fromAwsManagedPolicyName("service-role/AmazonEC2RoleforAWSCodeDeploy")
    );

    // VPC
    // This VPC has 3 public subnets, and that's it
    const vpc = new Vpc(this, 'main_vpc',{
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: 'pub01',
          subnetType: SubnetType.PUBLIC,
        },
        {
          cidrMask: 24,
          name: 'pub02',
          subnetType: SubnetType.PUBLIC,
        },
        {
          cidrMask: 24,
          name: 'pub03',
          subnetType: SubnetType.PUBLIC,
        }
      ]
    });

    // Security Groups
    // This SG will only allow HTTP traffic to the Web server
    const webSg = new SecurityGroup(this, 'web_sg',{
      vpc,
      description: "Allows Inbound HTTP traffic to the web server.",
      allowAllOutbound: true,
    });
    
    webSg.addIngressRule(
      Peer.anyIpv4(),
      Port.tcp(80)
    );
    
    // EC2 Instance
    // This is the Python Web server that we will be using
    // Get the latest AmazonLinux 2 AMI for the given region
    const ami = new AmazonLinuxImage({
      generation: AmazonLinuxGeneration.AMAZON_LINUX_2,
      cpuType: AmazonLinuxCpuType.X86_64,
    });

    // The actual Web EC2 Instance for the web server
    const webServer = new Instance(this, 'web_server',{
      vpc,
      instanceType: InstanceType.of(
        InstanceClass.T3,
        InstanceSize.MICRO,
      ),
      machineImage: ami,
      securityGroup: webSg,
      role: webServerRole,
    });

    // User data - used for bootstrapping
    const webSGUserData = readFileSync('./assets/configure_amz_linux_sample_app.sh','utf-8');
    webServer.addUserData(webSGUserData);
    // Tag the instance
    cdk.Tags.of(webServer).add('application-name','python-web')
    cdk.Tags.of(webServer).add('stage','prod')
    
    // Pipeline stuff
    // CodePipeline
    const pipeline = new Pipeline(this, 'python_web_pipeline', {
      pipelineName: 'python-webApp',
      crossAccountKeys: false, // solves the encrypted bucket issue
    });

    // STAGES
    // Source Stage
    const sourceStage = pipeline.addStage({
      stageName: 'Source',
    });
    
    // Build Stage
    const buildStage = pipeline.addStage({
      stageName: 'Build',
    });
    
    // Deploy Stage
    const deployStage = pipeline.addStage({
      stageName: 'Deploy',
    });

    // Add some action
    // Source action
    const sourceOutput = new Artifact();
    const githubSourceAction = new GitHubSourceAction({
      actionName: 'GithubSource',
      oauthToken: SecretValue.secretsManager('github-oauth-token'), // SET UP BEFORE
      owner: 'ianmnguy', 
      repo: 'sample-python-web-app',
      branch: 'main',
      output: sourceOutput,
    });

    sourceStage.addAction(githubSourceAction);

    // Build Action
    const pythonTestProject = new PipelineProject(this, 'pythonTestProject', {
      environment: {
        buildImage: LinuxBuildImage.AMAZON_LINUX_2_3
      }
    });
    
    const pythonTestOutput = new Artifact();
    const pythonTestAction = new CodeBuildAction({
      actionName: 'TestPython',
      project: pythonTestProject,
      input: sourceOutput,
      outputs: [pythonTestOutput]
    });

    buildStage.addAction(pythonTestAction);

    // Deploy Actions
    const pythonDeployApplication = new ServerApplication(this,"python_deploy_application", {
      applicationName: 'python-webApp'
    });

    // Deployment group
    const pythonServerDeploymentGroup = new ServerDeploymentGroup(this,'PythonAppDeployGroup', {
      application: pythonDeployApplication,
      deploymentGroupName: 'PythonAppDeploymentGroup',
      installAgent: true,
      ec2InstanceTags: new InstanceTagSet(
      {
        'application-name': ['python-web'],
        'stage':['prod', 'stage']
      })
    });

    // Deployment action
    const pythonDeployAction = new CodeDeployServerDeployAction({
      actionName: 'PythonAppDeployment',
      input: sourceOutput,
      deploymentGroup: pythonServerDeploymentGroup,
    });

    deployStage.addAction(pythonDeployAction);

    // Output the public IP address of the EC2 instance
    new cdk.CfnOutput(this, "IP Address", {
      value: webServer.instancePublicIp,
    });
  }
}


```
Here is the user data bash script attaached to the EC2 Instance. This code sits in a file named `configure_amz_linux_sample_app.sh` in the `assets` directory in the root of the CDK Application.

```Typescript
#!/bin/bash -xe
# Install OS packages
yum update -y
yum groupinstall -y "Development Tools"
amazon-linux-extras install -y nginx1
yum install -y nginx python3 python3-pip python3-devel ruby wget
pip3 install pipenv wheel
pip3 install uwsgi

# Code Deploy Agent
cd /home/ec2-user
wget https://aws-codedeploy-us-west-2.s3.us-west-2.amazonaws.com/latest/install
chmod +x ./install
./install auto
```
**Connection to Github**

To connect the Sample Application to our CDK I configured a Github Token to be used by the CI/CD Pipeline.
The token has two purposes:
*  To provide authentication to stage, commit, and push code from local repo to the GitHub repo. You may also use SSH keys for this.
*  To connect GitHub to CodePipeline, so whenever new code is committed to GitHub repo it automatically triggers pipeline execution.
The token should have the scopes **repo** (to read the repository) and **admin:repo_hook** (if you plan to use webhooks, true by default) as shown in the image below.

![Alt Text](https://community.aws/_next/image?url=https%3A%2F%2Fcommunity.aws%2Fraw-post-images%2Ftutorials%2Fusing-ec2-userdata-to-bootstrap-python-web-app%2Fimages%2FGitHub-repo-hooks.png&w=828&q=75)

Now, for AWS CodePipeline to read from this GitHub repo, I had to configure the GitHub personal access token just created. This token should be stored as a plaintext secret (not a JSON secret) in AWS Secrets Manager under the exact name `github-oauth-token`. I used the following AWS Secrets command and description:
```
aws secretsmanager create-secret \ 
  --name github-oauth-token \ 
  --description "Github access token for cdk" \ 
  --secret-string My_GITHUB_ACCESS_TOKEN \ 
  --region us-east-1
```

**Additional Files for Testing and Deploying**

To properly test and deploy the application, I needed to add some additional content to the sample repository forked. These files are used by the CodeBuild and CodeDeploy services. On top of that, I wrote a simple Python unit test. 
All of these Test files are located in the repository under the `Tests` directory.

To create the tests, in the root directory of the sample application create a `tests` directory, and add the `test_sample.py` file to it.
This test will run the Flask application and see if it returns a `200` HTTP status code

On top of this file, just for posterity, let's create a `__init__.py` file in the same directory.

Next I was ready to create the `buildspec.yml file`. This file is used by CodeDeploy as an instruction set of what it needs to do to build your code. In our case, we are instructing it on how to run the tests.

Finally, let's add some much needed files for CodeDeploy. Similarly to CodeBuild, CodeDeploy takes a file called `appspec.yml` as an instruction set on how to deploy your application to its final destination. On top of that file, we will be adding a few shell scripts to configure and launch the application on the server. This is needed as we need to create a specific `nginx` website, and do some service restarts. 
Here I am involving 4 different scripts in different phases of the deployment. This is required to properly set up the EC2 instance before and after code deployment. These scripts should sit in a directory called scripts in the root of the sample application (located in the repository above as well)
After these files are added and saved I made sure to add, commit, and push the changes to the sample code to this repository. 

**Bootstrap CDK**
Before deploying the CDK app, I had to configure CDK on the account I was deploying to. Edit the `bin/ec2-cdk.ts` and uncomment line 14:
```
env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION },
```
This will use the account ID and Region configured in the AWS CLI.
I also needed to bootstrap CDK in my account. This created the required infrastructure for CDK to manage infrastructure in my account.
```
cdk bootstrap

#output
⏳  Bootstrapping environment aws://AWS_ID/<us-east-1>...
✅  Environment aws://AWS_ID/<us-east-1> bootstrapped
Deploying the stack
```
Once verifying that the appropriate account was configured, I was ready to deploy using the following command
```
cdk deploy
```
I was presented with the following output and confirmation screen. Because there are security implications for our stack, I saw a summary of these and had to confirm them before deployment proceeds. This will always be shown if you are creating, modifying, or deleting any IAM policy, role, group, or user, and when you change any firewall rules.

![Alt Text](https://community.aws/_next/image?url=https%3A%2F%2Fcommunity.aws%2Fraw-post-images%2Ftutorials%2Fusing-ec2-userdata-to-bootstrap-python-web-app%2Fimages%2Fdeployment.png&w=1080&q=75)

Confirming that I wanted to commit these changes, I entered `y` to continue deployment and create the resources in the specified account. Below are the results listed in the CLI and the output defined in the CDK. 

![Alt Text](https://github.com/ianmnguy/AWS-Project-Portfolio/blob/main/sample-python-web-app-main/sample-python-web-app-main/Python-Web-App.JPG?raw=true)

Heading over to my Console, in the AWS CodePipeline console and finding my `python-webapp` I now see the Source, Build, and Deploy that initiated from CDK:




