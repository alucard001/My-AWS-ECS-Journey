# My Journey to use GPU in AWS ECS

## 4 steps processes to upload your code to AWS
- To login to AWS ECR:
  - `aws ecr get-login-password --region ap-east-1 | docker login -u AWS --password-stdin (your_aws_id).dkr.ecr.ap-east-1.amazonaws.com`
- To build image for AWS ECR:
  - `docker build -t medicalreportocr-gpu .`
- To tag the image to AWS ECR:
  - `docker tag medicalreportocr-gpu:latest (your_aws_id).dkr.ecr.ap-east-1.amazonaws.com/medicalreportocr-gpu:latest`
- To push the image to AWS ECR:
  - `docker push (your_aws_id).dkr.ecr.ap-east-1.amazonaws.com/medicalreportocr-gpu:latest`

## Before you start

This is a long story.  And there are a lot of nuances and difficulties when using this AWS ECS (ECS for short) thing.  And I don't want to keep it "short and sweet", because I think it is my responsibilites to put everything I encountered, the problems that I met during this process.

This project, from having a running code in Python local development, to ECS "production" (or first GPU version), already tooks more than 1/2 years.

While I don't want to keep it short and sweet, there are somethings that I would not mention:

- Background: The reason why this project is developed.  Unless this is related to the technical side of the project, I tried not to mention here.
- Some AWS terms, like S3 or ECR/ECS.  Look it up for yourself if you don't know what it is.

Let's begin!

## Building a docker image for AWS ECS using GPU
- First, everytime you tested your code in development, you need to follow [this 4 steps processes](#4-steps-processes-to-upload-your-code-to-aws) here to upload (or called `push`) to AWS ECR
- Please note the following:
  - When you are building image for GPU instance, **use correct paddlepaddle images**
    - I spent 3 - 4 days to do build-push process and keep getting unknown errors, and at last I found that I am using a CPU image for GPU instance...
  - I ran those 4 commands within VS code (just in case you don't know: it is powershell)
  - Until recently I found that, everytime after building an image and push to ECR, my notebook computer suddenly run slow.
    - It turns out that in Windows docker, after you build your image, there is a process called `docker-index.exe` executed, and it tooks 4 GB - 10 GB RAM
      - Also sometimes it runs more than 15 mins
    - At the time of this writing, after I build the image, and when my computer performance is down, I will open task manager and manually *kill/terminate/end* `docker-index.exe`.
      - It goes well after doing it.

## The architecture (Introducing Celery and Celery Flower)
- At first, when we first did POC on this project, all we did was to create an API and uploaded files via HTTP POST
- Which means, everytime when we called APIs, the OCR process will run right away, which causes the following problems:
  - The OCR process starts immediately, and it takes times to do it.
    - If there are a lot of people calling this, they may not get fast response, or even get no response because the server already hang-up.
  - This is especially true when you are doing OCR via CPU machine.
    - At the time of this writing, scanning 4 images will take 5 mins (around 200 kb each) and 7 mins for a 4 pages (in 1 file) PDF
- This is clearly unacceptable, when it needs to become a production-level service.
- Therefore, we need a way to make this OCR process "run faster".
  - Our way to do it is to move the process running from "foreground" to "background"

### How to solve this problem?

Here is what I did:

- When user uploads file through API, we save it to S3, and kick start the OCR process
- However, instead of doing it right away, we save it to "a queue"
- And there is another program, called "broker", which will read this "queue", get the task, and run it
- While this broker is doing it on the background, it also returns the id of the task (`task_id`)
- Therefore, instead of waiting for 5 - 7 mins (or longer depends on the uploaded files), we get a response immediately.
- Once the task on the backend is completed, the result will be saved as a `json` file and upload to s3
- After that, the whole reuslt will be sent to another API for further processing
  - This is our custom API which will further process our return

So how to do all these?

### Introducing Celery (and Flower)
- [Celery](https://docs.celeryq.dev/en/stable/index.html), according to the official site, is:
  - a simple, flexible, and reliable distributed system (in Python) to process vast amounts of messages, while providing operations with the tools required to maintain such a system.
- Actually, before knowing that there is a more elegant and simple way, called message queue (RabbitMQ or AWS SQS), the only way I can think of in solving this problem is by using Celery
- Celery works well with Redis and AWS SQS
  - Unfortunately, if I know the power of SQS/MessageQueue, I think I will save at least half of the month on this project

### How Celery fits into our requiement
- Remember the term "broker" in our [How to solve this problem?](#how-to-solve-this-problem) section?  Celery is the "broker".
- Celery will main its own message queue using Redis, and run the task accordingly
  - Redis is **not the only way** to maintain a queue.  [Other options](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/index.html) included RabbitMQ and SQS can also do that.
- But having a broker is not enough, we need to manage this with a system/tool, or called "backend" in Celery term.
- In Celery, you can run this backend by
  - Redis
  - RabbitMQ
  - SQLAlchemy (a Python package)
- However, this backend is used and maintained by Celery, in other words, there is no UI/admin for this "backend"

### Flower - web-based tool to manage Celery
- To manage such "backend", we need a UI,
- Here comes [Flower](https://flower.readthedocs.io/en/latest/index.html).  According to official site:
  - Flower is a web based tool for monitoring and administrating Celery clusters
- This service uses port `5555` to receive traffic

This is how all the "mess" come into this projects...

*side note*: At this point, you may already note/find/discover certain things/ways that you think "I can do better".  I agree, and actually welcome.  If you think you can put it in a much better way, go do it.  No hard-feeling.  You have my permission.  :D

## How it all comes together
- At this point, in terms of architecture, you have the following
  - Redis
  - ECS (Within single docker container)
    - supervisord, it runs
      - Celery
      - Celery Flower
      - Python3.7
        - Nginx
        - Gunicorn
- Here are the difficulties/problems I encounter when making the above architecture works

### Redis - It is not what you think
- Redis is a fast K/V store DB.
- However, connect to Redis got me a lot of trouble.
  - i.e. the problem is not "using" it, but just "connecting to" it.
- In AWS, you have 2 services available when you are using Redis
  - MemoryDB (new, recommended)
  - ElastiCache (existing).  It supports
    - Redis
    - Memcache
- And, there are 2 "modes" when running redis
  - Standalone / Single server
  - Cluster
- At first, this is almost a no-brianer to use MemoryDB cluster to create your Redis service
- However, when I deploy the code to ECS, I cannot get my code to work, because of connection problem
  - The error is: `<your_redis_cluster_endpoint>`: name or service not known
- To not mention what I did to search and solve this problem, I found that MemoryDB may be too new to this project, so I switch to ElastiCache
- But the problem persists even when I switch to ElastiCache+Redis.
- Please note the following features/characteristics about AWS ElastiCache/MemoryDB
  - It cannot be connected from external network
    - which means, even if you have the DNS name of Redis configuration endpoint, you cannot connect to it because of AWS security measures
      - No, you cannot do it even in redis-cli
  - You can only connect it **within** AWS network
    - For example, if you setup a EC2 instance, you can connect to your redis via it, either programmatically (JS, Python, PHP etc) or `redis-cli`.
- So the connection problem is **NOT** networking problem.

#### Cluster vs Standalone
- When we want to use Redis as cache, there are two ways to create such service: Cluster and standalone
- In simple terms:
  - Cluster: A bunch (usually 3) of redis servers connected each other, and accept request from a single endpoint
  - Standalone: Single (1) server to handle all requests
- The connection problem is about the cluster, or precisly, how cluster works when you connect.
- Simply put:
  - When you are connecting to Redis cluster, you need to use different Python package(!)
    - If you are connecting to a standalone redis, you use `import redis`
      - When connect, you use: `new redis.redis()`
    - But if you are connecting to a cluster redis, you use `from rediscluster import RedisCluster`
      - When connect, you use: `new RedisCluster()`
  - References
    - https://aws.amazon.com/blogs/database/work-with-cluster-mode-on-amazon-elasticache-for-redis/
    - https://redis-py-cluster.readthedocs.io/en/stable/client.html
- However, even when you are able to conect to a cluster, another problem arise
  - You got "MOVED" error
    - `MOVED <some_bytes> <your_redis_server>:6379`
  - (Another) Simply put:
    - The key you saved to Redis is in different shard when you fetch it
    - Remember, this is **cluster**!
  - References
    - (Same as above) https://aws.amazon.com/blogs/database/work-with-cluster-mode-on-amazon-elasticache-for-redis/
    - https://blog.csdn.net/liu0808/article/details/80098568
- You'll never know how depress I am when I encouter this situation...
- In order to solve this, you are either
  1. Moderately rewrite the connection **AND** host mapping, or
  1. Use standalone mode
- I choose the later one
- When I made my decision, everything works well, and fast again.  Almost everything works, smooth and fast.

#### Notes on Standalone Redis
- You may argue/disagree that it may not be as good as cluster mode.  I agree.
- However, if we really need to find a reason to use standalone mode, here are why:
  1. There is a auto-failover option in AWS redis, so even if it really "fails", it will be recovered, although it would have "downtime"
  2. I setup this redis service for OCR purpose **ONLY**.  A redis service supports 100K READ/s and 10K write/s.  I personally don't think that this service will have such a high traffic loading.
  3. By the way, the instance type I setup is `cache.t3.medium` (3GB Memory, 5GB network performance), Redis v7.0, 2 replicas.  In other words, at this point, if the traffic volumn is so high that an instance type like this cannot support, we can simply increase the instance type.
- In any case, it works~

### ECS - Wait, you need to wait
- Once you got the Redis to work, another main bottleneck for me is ECS

#### Debugging in ECS
- One of the common characteristics of a programmer is to find out: how everything works inside, live!
- Therefore in a closed environment like ECS, you would want to save/write all the debug info you can in your code, this is what I did.
- However in this case, even when I put a lot of debug info, it still cannot compare to the experience to debug a running, live container, which would save a lot of time.
- Now, when you are thinking: you cannot go inside a container to see how everything works: **No, it is not.  You can `/bin/bash` to a running container in ECS**.
- You did that by SSH to your container instance (i.e. an EC2 instance running your docker containers) and run `docker` commands, like `docker exec -it <your_container_id> /bin/bash`.
- How to connect to your container instance? By SSH of course
  - Remember you have a `.ppk` file downloaded/saved when you create your ECS service (more on this later)?  That `.ppk` key allows you to connect to your container instance.
  - You never know how I feel relief and happy when I found that out, especially when I thought I already lost that key.
- When you connect to your container instance, you can immediately run `docker ps -a` to show all running containers, using your ECR images.
- When you are there, you can go into your container and do whatever debug you want.

#### Access your ECS container via `aws ecs execute-command`
- This is actually what I intent to do before I know that I can SSH to container instance via my `.ppk` key
- Simply put: you can connect to a running container inside your container instance right from your VS code/Terminal using `aws ecs`
- Reference:
  - https://aws.amazon.com/blogs/containers/new-using-amazon-ecs-exec-access-your-containers-fargate-ec2/
  - https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html
- However, in order to do it, you need to
  - Download an installer: AWS SSM agent (Google yourself)
  - Create you ECS service with `--enable-execute-command`
    - This option is **NOT** available in ECS creation UI, as of this writing, you can only run it in command line
- I haven't completed the whole process to connect my container via `aws ecs`.  But for your reference, I put the `create-service` command here:

```
# Remove all \ linebreak if you run it in Windows Powershell
aws ecs create-service \
  --cluster Medical-Report-OCR \
  --service-name Medical-Report-OCR-Service-CLI \
  --enable-execute-command \
  --task-definition Medical-Report-OCR-GPU-Task:31 \
  --launch-type EC2 \
  --scheduling-strategy DAEMON \
  --load-balancers '[{ \"targetGroupArn\":\"xxx\",\"containerName\":\"Medical-Report-OCR-GPU\",\"containerPort\":8080}, {\"targetGroupArn\":\"xxx\", \"containerName\":\"Medical-Report-OCR-GPU\", \"containerPort\":5555 }]'
```

As you can see:
- You must define: `--enable-execute-command`
- `--load-balancers`: This is a tough one: you need to manually quote the double-qoute(") to make it in JSON format
  - Reference: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/register-multiple-targetgroups.html

**Read all the necessary documentation to create and run ECS (and related service) in command-line.  It is required.**

#### Some words on Load balancer health checking
- Why such topic?  Does it related to this project?  Yes and in a big way.
- When you setup ECS and use the [above method](#debugging-in-ecs) to debug, you may enconter a situation like this:
  - While you are debugging, your container is removed automatically
- The reason why is that your container cannot meet the health-check requirement defined in AWS Loadbalancer you used in ECS
- In other words, when your container did not meet loadbalancer health check requirement, your container will be "drained", disconnected and removed, all automatically.
- And when the health check fails several times, ECS will not create new task for you until you fix the health check issue.
- This is actually reasonable: who will keep a container/application which "is not working"?
- However, if you are debugging in a container after health check fails, you will be "kick out" from the container, all changes and debug work you did in that container, were lost.
- While in best practice, you should never do a live debug in production environment like this.  But what if...
- So when you are absolutely necessary to debug in ECS container, you are either:
  - Doing it before all health-check fails, or
  - Extend your health check to the longest possible (as of this writing)
    - You did it by going to: EC2 > Target groups > (your target group) > Health checks > Edit
    - No, you cannot disable health check

#### Advice when you update your load balancer target group
- Sometimes you may update your port mapping on ECS target group
  - e.g. create/remove groups
- After updating, you have to go to EC2 > Auto Scaling groups, select the group which was affected, and update the load balancing config (scroll down to see the Load balancing section after selecting the group) accordingly
- Or you may encouter error like: some target groups are not mapped to load balancer, even if in EC2 > target groups, all groups are already mapped appropriately.

## Get back to basic: About ECS
- Let's get back to the whole ECS thing: What is it and how it works?
- Instead of giving your official definition of ECS (which you can look up yourself), I would like to tell you how each components work each other.
- The "structure" of a ECS looks like this
  - ECS is a **"Cluster"**
  - which run one or many **"Services"**
  - and services run **"Task"** based on custom-predefined **"Task definitions"**
  - A **"Task"** will take one or many instances (i.e. EC2 or external computer) resource to complete
  - All running inside an **"Account"**
- You will see what I learnt/found when running ECS

### Cluster and Services in ECS, ONLY for this project
- A Cluster runs one or more Services
- In our case, since we are using GPU container instance, the launch type is `EC2`, not `Fargate`
- `EC2` is "custom capacity", which means: you manage your own computer
  - Unlike `Fargate`, which will managed for you.
- Since you are using `EC2` launch type, it means, you have a EC2 instance, which is created by ECS.
- Application type is "Service", see the description in the UI, it said clearly.
- Task definition
  - Think of "Task definition" as a "template", which contains a set of insturction on how a task should be configured.
- Service type: **DAEMON**
  - As the name implied: Only 1 container will be created to serve all requests, which is different from Fargate.

### Task/Task definition in ECS, ONLY for this project
- As said before: Task definition is a kind of "template" on how a task should behave
- It is versioning manually.  i.e. When you first create one, it is `task_definition:1`, if you update it, it becomes `task_definition:2`, and so on.
- So when you create/update your Service, you can choose which version of task definition to run
- In the definition:
  - You told ECS which ECR image to use, including the version
    - i.e. If you are defining your ECR images as: `some_image_name:latest`, every time you update you image, you don't need to update your task definition.
  - Port mapping
    - Similar to AWS security group
    - Host: it is your container instance
    - Container: your container, no question.
    - Remember, every ECS comes with a load balaner, and target group
      - **Caution:** in ECS, your target group seems port-mapping to **container**, not **container instance**
      - Working configs,
        - Task definition:
          - Host:80 -> Container:8080
          - Host:5555 -> Container:5555
        - EC2 - Load balancer Target groups
          - LB:80 -> Forward to: TG: 8080
          - LB:5555 -> Forward to: TG: 5555
      - I tried but not work
        - Task definition:
          - Host:8080 -> Container:8080
        - EC2 - Load balancer Target groups
          - LB:80 -> Forward to: TG: 8080
        - or
        - Task definition:
          - Host:80 -> Container:8080
        - EC2 - Load balancer Target groups
          - LB:80 -> Forward to: TG: 80
      - Container:8080 is a port defined in `Dockerfile`: `expose 5555 8080`

#### Some words on environment variable(s) in ECS task definition:
- In the program, you will see that there are two folders: `conf` and `env`
- `conf` is a config directory which stores all the config files of different service (For example, like nginx, supervisor etc)
  - This directory will be packed into docker image
- `env` is another directory which stores all environment variable
  - This directory will **NOT** be packed into docker image
  - Rather, the file is **manually** uploaded to S3
- In task definition, I set the above resource location to `production.env`
- Therefore, everytime when we need to add/update an environment variable, instead of rebuilding the whole docker image,  you can simply replace the `production.env` file in S3
- Actually, also in task definition, you can define individual environment variable (e.g. `foo=bar`), which will **override** the same variable in `.env` file
- However, because I would like to put all environment variables in one place for simple management, I put it in `.env`.
- You can see a list of defined **and loaded** environment variables in the following ways:
  1. Through `\health` API
  2. If you go inside the running container (i.e. `docker exec -it <container_id> /bin/bash`), you can run `printenv` (this is a linux function) to get a list of loaded variables.

#### Wait? Why Wait!?
- I think this is the most time-consuming part of ECS
- When you did the following
  - Update your ECR image
  - Update your Services
  - Update your Task definition
- You need to "Update your service" by going to (as of this writing)
  - ECS > Clusters >(Your cluster) > Services > (Your Service name), at the top right hand corner, click "Update service"
  - Click "Update"
- About "Force new deployment" when you update service:
  - I found that I do not need to tick "Force new deployment" and the task would still use the updated config
  - As far as I know, checking this box will force yor current running task to stop and recreate immediately
  - However, even if you don't do that, if you simply click "Update", after 3 - 5 mins, you should expect that the service will recreate a task for you
- What "Update your service" do
  - It goes into your container instance, stop current running container, and recreate one
- Do I need to wait 5 mins?  I want to make it faster!
  - No.  Actually after updating service, I tried to speed up the process by stop and remove current running container, the service will not create a new task for me.
  - No.  Even after remove the task in admin UI, stop and remove the container in instance, service will still NOT create a new task for me.
  - No.  Keep running "Update Service" multiple times would not create a new task for me
  - If you choose "Run Task" in task definition, chances are you will get an error that said something like: Not enough CPU/GPU/Memory resources for your task.  It is because you are competing instance resources with your ECS service.
- At the time of this writing, all I can do to speed up the process is to click "Update Service", probably 2 - 3 times, and keep waiting until it starts its process to create a new task for you.
  - i.e. You can only wait, until...

## How to save money for your company by turning off your service

### How to turn off Redis server
- Redis service cannot be paused/disable.  You can either create one or delete it
  - It has backup option, I suggest you to use it.
    - Because when you need it again, you don't need to re-create one.
    - Which saves some time.

### How to stop container instance
- You can't.  Yes, you cannot "turn off" that instance
- Because ECS will be so smart to terminate your instance and re-create one
- So currently what I did is to control the number of instance created (or capacity, for short)
- Go to EC2 > Auto Scaling groups, select your group
  - In the first section - Group details, you can edit your desired, min and max capacity.
  - Simply change the value from 1 to 0 for all values, and update it.
- It will automatically turn off all instances.
