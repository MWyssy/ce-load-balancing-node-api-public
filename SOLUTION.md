## What endpoints and what [request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) does the API currently support?

> The API current supports the following endpoints with the associated request methods:
>
> - '/' - This is the root endpoint
>   - GET
> - '/learners'
>   - GET
>   - POST
> - 'health-check'
>   - GET

## Why do you think we use the port **3000** within the security group setup?

> We use the port 3000 because that is the port we assigned in index.js to the app.listen method. Therefore, the API is listening for any requests on port 3000, so we need to use this in the security group to ensure all our requests are picked up.

## When setting up the ALB you had to specify a health check endpoint - why is that?

> Because we need an end-point, to test whether the server is working, that doesn't interact with any other end-point. This is so we can test the server without downloading (potentiall) massive amounts of data that may be stored in other end-points.

## What is a load balancer and why might you use one?

> The load balancer connects to multiple servers and acts as a digital traffic warden to ensure that no one server is overloaded. It will split the traffic between however many servers are active.

## Explain, in your own words, what security groups are?

> A Security Group is like a bouncer that will only allow certain traffic to and from the VPC. The default is to block all traffic, but you can define the ports and protocols that you want to allow.

## Currently the application listens on port 3000, this isn't a standard HTTP port - what two ports would be better to use?

> 80 or 443

## When the API is deployed behind a load balancer, if you add multiple learners and then re-try **GET**ting the learners, sometimes it shows the learners you have added other times it doesn't. Why is that?

> There is a flaw in how the code is designed because each server will have it's own instance of 'learners', so when a new learner is 'pushed' to 'learners', it is only pushed to one of the servers. Then, when the 'GET' request is made, the loadbalancer may direct you to the other server which doesn't yet contain the new learner. I assume this is because it takes a bit of time for the servers to be brought up to date with each other.

## Which bits of the setup do you think you could automate and why?

> The entire server setup could be automated using a script (or terraform), as I can imagine it getting quite tedious to set up mutliple servers all the time. The entire server setup can be done through CLI, so it is possible to script this.

## Add an image to the repository that shows your browser hitting the API and listing learners. Link that image within your SOLUTION.md markdown document

> ![GET learners end point](/API-learners-screenshot.png)

## If you completed the extension exercise and used the AWS CLI to undertake the AWS actions, include a section called **AWS Commands** in your SOLUTION.md that outlines the commands you used

1.  Create a security group

```
aws ec2 create-security-group \
--group-name learners-api-sg \
--description "learners-api-sg" \
--vpc-id <vpc-id>
```

2. Set Security Group Permissions

```
aws ec2 authorize-security-group-ingress \
--group-id <security-group-id> \
--protocol tcp \
--port 3000 \
--cidr 0.0.0.0/0 \
&& aws ec2 authorize-security-group-ingress \
--group-id <security-group-id> \
--ip-permissions IpProtocol=tcp,FromPort=3000,ToPort=3000,Ipv6Ranges="[{CidrIpv6="::/0"}]" \
&& aws ec2 authorize-security-group-ingress \
--group-id <security-group-id> \
--protocol tcp \
--port 80 \
--cidr 0.0.0.0/0 \
&& aws ec2 authorize-security-group-ingress \
--group-id <security-group-id> \
--ip-permissions IpProtocol=tcp,FromPort=80,ToPort=80,Ipv6Ranges="[{CidrIpv6="::/0"}]" \
&& aws ec2 authorize-security-group-ingress \
--group-id <security-group-id> \
--protocol tcp \
--port 22 \
--cidr <local-ip>
```

3. Launch an Ec2 instance

```
aws ec2 run-instances \
--image-id ami-09744628bed84e434 \
--instance-type t2.micro \
--key-name <your-key-pair> \
--security-group-ids <security-group-id> \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value="api-server-001"}]' \
&& aws ec2 run-instances \
--image-id ami-09744628bed84e434 \
--instance-type t2.micro \
--key-name <your-key-pair> \
--security-group-ids <security-group-id> \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value="api-server-002"}]'
```

4. SSH into each instance and install the required packages

- 4.1 - SSH

```
ssh -i /path/key-pair-name.pem ubuntu@instance-public-dns-name
```

- 4.2 - install packages

```
sudo apt update && sudo apt install nodejs && sudo apt install npm && sudo npm install pm2 -g
```

- 4.3 - clone the API repo, enter the directory, and install the dependencies

```
git clone https://github.com/USER/example-repo.git && cd /path-to-directory && npm install
```

- 4.4 - start the server

```
pm2 start src/index.js && exit
```

Repeat for the second server

5. Setup the Target Group

- 5.1 - create the group

```
aws elbv2 create-target-group \
--name api-server-targetgroup \
--protocol HTTP \
--port 3000 \
--target-type instance \
--vpc-id vpc-0810289a43394ca8f \
--health-check-protocol HTTP \
--health-check-path /health-check
```

- 5.2 - register the instances to the target group

```
aws elbv2 register-targets \
--target-group-arn <target-group-arn> \
--targets Id=<target-1> Id=<target-2>
```

6. Setup the Load Balancer

- 6.1 - Create the Load Balancer

```
aws elbv2 create-load-balancer \
--name api-server-loadbalancer \
--subnets <subnet-1> <subnet-2> <subnet-3> \
--type application \
--scheme internet-facing \
--ip-address-type ipv4 \
--security-groups <security-group>
```

- 6.2 - Create a listener

```
aws elbv2 create-listener \
--load-balancer-arn <load-balancer-arn> \
--protocol HTTP \
--port 80
--default-actions Type=forward,TargetGroupArn=<target-group-arn>

```

7. Have a cup of tea, you deserve it!
