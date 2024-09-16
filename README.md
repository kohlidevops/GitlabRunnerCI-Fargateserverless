# Gitlab Runner CI on AWS Fargate Serverless

## Create a project in Gitlab

To create a project called sampletest in Gitlab and add one new file called .gitlab-ci.yml add the content using below link

```
https://gitlab.com/kohlidevops/sampletest/-/blob/main/.gitlab-ci.yml?ref_type=heads
```

## To create an IAM Role

To create an IAM Role called "EC2RoleForECS" and add below policy

![image](https://github.com/user-attachments/assets/1b08d4f5-fdee-4ee1-8397-eeec35cd0265)

## Launch EC2 instance

![image](https://github.com/user-attachments/assets/875bce59-b6ee-44c5-8a32-ebaa9d640226)

To launch ubuntu-24 instance with T3.medium and run below commands and add the IAM Role "EC2RoleForECS" which is created just now

```
sudo apt-get update -y
sudo apt-get upgrade -y
sudo mkdir -p /opt/gitlab-runner/{metadata,builds,cache}
curl -s "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install gitlab-runner
```

## Generate the registration token

Navigate to Gitlab > Project > sampletest > Settings > CICD

![image](https://github.com/user-attachments/assets/00c465aa-483b-4740-86a2-8212a81910c7)

Runners > Expand > New Project Runner

![image](https://github.com/user-attachments/assets/9d3c47ad-bee8-417b-98b9-337e9bd3eb47)

Tags - nodejs > create Runner // Note the Token

## To register the Gitlab Runner in EC2 instance

To register the gitlab-runner in ubuntu ec2 instance, you can run below command

```
sudo gitlab-runner register

Enter the GitLab instance URL (for example, https://gitlab.com/): https://gitlab.com
Enter the registration token: AAAAABBBBCCCCDDD <Add your token here>
Enter a name for the runner. This is stored only in the local config.toml file: aws-fargate-runner
Enter an executor: docker-autoscaler, instance, custom, shell, virtualbox, docker-windows, docker+machine, kubernetes, ssh, parallels, docker: custom
```

![image](https://github.com/user-attachments/assets/fa3c0e94-a37f-4c91-bfb8-6c0f7ae9f8b9)

## To edit the config.toml file

To ssh to the ubuntu instance and execute below command

```
sudo vim /etc/gitlab-runner/config.toml
```

Add the below content 

```
concurrent = 1
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "aws-fargate-runner"
  url = "https://gitlab.com"
  id = 41228032
  token = "AAAAABBBBCCCCCDDDDD"
  token_obtained_at = 2024-09-16T05:26:12Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "custom"
  builds_dir = "/opt/gitlab-runner/builds"
  cache_dir = "/opt/gitlab-runner/cache"
  [runners.custom]
    config_exec = "/opt/gitlab-runner/fargate"
    config_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "config"]
    prepare_exec = "/opt/gitlab-runner/fargate"
    prepare_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "prepare"]
    run_exec = "/opt/gitlab-runner/fargate"
    run_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "run"]
    cleanup_exec = "/opt/gitlab-runner/fargate"
    cleanup_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "cleanup"]
````

Make ensure the runners name, token and URL - Then save and exit the file //What you have created before

## To create a fargate.toml file

```
sudo vim /etc/gitlab-runner/fargate.toml
```

Then add the below content

```
LogLevel = "info"
LogFormat = "text"

[Fargate]
  Cluster = "test-cluster"
  Region = "ap-south-1"
  Subnet = "subnet-123456789"
  SecurityGroup = "sg-123456789"
  TaskDefinition = "test-task"
  EnablePublicIP = true

[TaskMetadata]
  Directory = "/opt/gitlab-runner/metadata"

[SSH]
  Username = "root"
  Port = 22
```

save and exit the file

#### Note the cluster name, region, task definition name - we have to use this while creating ECS cluster

## Install the fargate driver

To install the fargate driver on ubuntu ec2 instance using below commands

```
sudo curl -Lo /opt/gitlab-runner/fargate "https://gitlab-runner-custom-fargate-downloads.s3.amazonaws.com/latest/fargate-linux-amd64"
sudo chmod +x /opt/gitlab-runner/fargate
```

## Create an IAM Role

To create an IAM role called "ECSTaskExecutionRole" and below IAM policy

![image](https://github.com/user-attachments/assets/bc934374-0415-46d3-ad24-4653ac37a379)


## Create an ECS Fargate Cluster

Navigate to the AWS ECS Cluster > Create new cluster

![image](https://github.com/user-attachments/assets/35c7c21b-f47f-4850-bdb4-9a3bd34e6edb)

```
name - test-cluster
```

To view your cluster > update cluster

![image](https://github.com/user-attachments/assets/a2eaa418-4e5f-42e2-9562-6db2a0709b9a)

Add capacity provider

![image](https://github.com/user-attachments/assets/6520cc22-d597-4fba-891e-68ba3ad13a42)

Add Fargate > update

![image](https://github.com/user-attachments/assets/dd4830e9-1680-4f69-b915-8eee29173217)

## To create an ECS Taks definition

Navigate to again ECS > Task definition > create a new task definition

![image](https://github.com/user-attachments/assets/bcb7ad5c-79fb-4eeb-b29e-5f1ac8913222)

```
name - gitlab-runner-ci-nodejs
Task Role - ECSTaskExecutionRole
Task Execution Role - ECSTaskExecutionRole
Container name - ci-coordinator
Image URI - registry.gitlab.com/aws-fargate-driver-demo/docker-nodejs-gitlab-ci-fargate:latest
Essential container - Yes
Port mapping - // as per below image and rest of things leave as default
Then create a task definition
```

![image](https://github.com/user-attachments/assets/0ee32d6f-02a0-47ac-b1d8-f25aa459b879)

## Trigger the Gitlab CI Pipeline

Navigate to Gitlab repo > select > your project > open the gitlabci-yaml file

```
stages:
  - test

test_job:
  stage: test
  tags:
    #- gitlab-org
    #- fargate
    #- custom
    - nodejs
  script:
    - echo "This job runs on the custom executor."
```

Ensure nodejs tag added in tags section. Because we have created tag called nodejs while I created Gitlab project runner.

Select your project > Settings > CICD > Runner > Expand 

![image](https://github.com/user-attachments/assets/103555dc-3b1d-4e4f-881b-c37ebc3e0931)

Make ensure - Your nodejs runner should be Active "Green color"

now open your gitlabci yaml file edit and add something like below

```
stages:
  - test

test_job:
  stage: test
  tags:
    #- gitlab-org
    #- fargate
    #- custom
    - nodejs
  script:
    - echo "This job runs on the custom executor."
    - echo "New update"
```

Now save it - automatically pipeline will trigger and start the ECS task as per configured.

test has been started

![image](https://github.com/user-attachments/assets/76b621b7-f0a9-4ebe-855c-f642721a44d5)

ECS task is now running that is fargate serverless

![image](https://github.com/user-attachments/assets/172e72c5-7916-42d9-8ecf-eeb7a225c1d7)

once the task has been completed then fargate automatically removed from the ECS cluster

![image](https://github.com/user-attachments/assets/61570aeb-9a2a-40b3-8eef-5594e56808b3)

my gitlab job has been done

![image](https://github.com/user-attachments/assets/5c3ccb46-12ce-4968-9207-c27aae927908)





