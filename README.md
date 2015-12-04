Blue-Green Deployment Workflow To Docker Swarm with Jenkins
===========================================================

The idea behind this article is to explore ways to deploy releases to [Docker Swarm](https://docs.docker.com/swarm/) without downtime. We'll use *blue-green* process. More info about the process and one possible implementation can be found in the [Blue-Green Deployment, Automation and Self-Healing Procedure](http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-docker-compose-and-consul-part-34-blue-green-deployment-automation-and-self-healing-procedure/) article. One of the downsides of the process we used in that article is Ansible itself. While it is probably the best tool for provisioning and orchestration, it had some downsides when we tried to use it as the tool to deploy containers especially if the process is complex. It lacked some constructs common in most programming languages. This time we'll try to implement the same process but using the [Jenkins Workflow Plugin] and a bit of Groovy scripts.

Let's see it in action first and then discuss how it was done.

Setting Up Docker Swarm Cluster and Jenkins
-------------------------------------------

We'll start by creating the cluster we'll need for running Docker Swarm. We'll use VirtualBox and Vagrant to create the VMs and then provision them with Ansible. I won't go into details how Ansible playbooks we'll run work since that information can be found in other articles in this blog. The setup will be similar to the one described in the [Deploying Containers with Docker Swarm and Docker Networking](http://technologyconversations.com/2015/11/25/deploying-containers-with-docker-swarm-and-docker-networking/) article published recently. The major difference will be the addition of Jenkins.

I'll assume that you already have [VirtualBox](https://www.virtualbox.org/), [Vagrant](https://www.vagrantup.com/) and [Git](https://git-scm.com/) installed. If you are a Windows user, please follow the instructions described in [Running Linux VMs on Windows](http://technologyconversations.com/2015/11/24/running-linux-vms-on-windows/) before diving into those described below.

Let's get going! We can create the servers by running the following command.

```bash
git clone https://github.com/vfarcic/blue-green-docker-jenkins.git

cd blue-green-docker-jenkins

vagrant up swarm-master swarm-node-1 swarm-node-2
```

Next, we should provision servers with Docker Swarm, Docker Compose, Consul, Consul Template and Registrator. If you are new to Docker Swarm, you might benefit from reading the [Docker Clustering Tools Compared: Kubernetes vs Docker Swarm](http://technologyconversations.com/2015/11/04/docker-clustering-tools-compared-kubernetes-vs-docker-swarm/) article. Information about Consul and the other tools we'll use for service discover can be found in the [Service Discovery: Zookeeper vs etcd vs Consul](http://technologyconversations.com/2015/09/08/service-discovery-zookeeper-vs-etcd-vs-consul/) article.

It might take some time to create all three servers especially if this is the first time you're using Vagrant with Ubuntu. Once they are up and running, we can use Ansible to configure the cluster. Since the creation of VMs included installation of Ansible in the *swarm-master* node, we can use it to run the [swarm.yml](https://github.com/vfarcic/blue-green-docker-jenkins/blob/master/ansible/swarm.yml) playbook.

```bash
vagrant ssh swarm-master

ansible-playbook /vagrant/ansible/swarm.yml \
    -i /vagrant/ansible/hosts/prod
```

Finally, the only thing left is to set up Jenkins. We'll continue using Ansible for this task. It will create few directories, run the Jenkins container, fiddle with few configurations, install plugins we'll need and, finally, create the job that will do the actual deployment. Since the focus of this article is to experiment with Jenkins Workflow in the context of blue-green deployment to the Swarm cluster, I'll skip explanations how the [jenkins.yml](https://github.com/vfarcic/blue-green-docker-jenkins/blob/master/ansible/jenkins.yml) playbook works. Feel free to take a look at the code or to post a comment and, if there is enough interest, I'll write a separate article about setting up Jenkins with Docker and Ansible.

```bash
ansible-playbook /vagrant/ansible/jenkins.yml \
    -i /vagrant/ansible/hosts/prod
```

Exploring The Jenkins Workflow
==============================

Let's start the build. Since the first run last a bit longer than consecutive runs, we'll us that time to discuss the solution. Please open [http://10.100.192.200:8080/job/books-ms/build?delay=0sec](http://10.100.192.200:8080/job/books-ms/build?delay=0sec) and click the *Build* button. You can monitor progress by opening the [http://10.100.192.200:8080/job/books-ms/lastBuild/console](http://10.100.192.200:8080/job/books-ms/lastBuild/console) page.

![Workflow job build screen](img/build.png)

While the job is running, let's go through the process and the Groovy script [service-flow.groovy](https://github.com/vfarcic/blue-green-docker-jenkins/blob/master/ansible/roles/jenkins/templates/service-flow.groovy) that accomplishes it. Please note that, for simplicity, I skipped steps that build containers, run unit and functional tests and push container to the registry. You're advised to put them inside the workflow we are about to explore. In this case, the Groovy script assumes that those steps were already performed in a separate process and that the container we are about to deploy is validated.

We should start by provisioning the cluster. Even though we already setup Swarm and service discovery tools, it is always a good idea to make sure that everything is still running as expected. Besides the cluster we'll need load balancer as well (in this case [nginx](http://nginx.org/)). If everything is properly set, provisioning will take only few seconds. On the other hand, if some process stopped or, as in case of nginx, if was never even running, our provisioning stage will correct that.

Steps:

* Provision Docker Swarm cluster
* Provision load balancer (nginx)

```groovy
node("cd") {
    stage "> Provisioning"
    if (provision.toBoolean()) {
        sh "ansible-playbook /vagrant/ansible/swarm.yml \
            -i /vagrant/ansible/hosts/prod"
        sh "ansible-playbook /vagrant/ansible/nginx.yml \
            -i /vagrant/ansible/hosts/prod --extra-vars \
            \"proxy_host=swarm-master\""
    }
...
```

We started by declaring that the steps will run inside the Jenkins node called or labeled *cd* (short for continuous deployment). For simplicity, in this case *cd* is one of the labels of the *swarm-master* node. In production, you should run as much as possible in dedicated servers and not in production. Next is the *stage* declaration that serves multiple purposes. It marks a group of steps, allows us to constrain concurrency and, if you choose to use [CloudBees Jenkins Enterprise Edition](https://www.cloudbees.com/products/cloudbees-jenkins-platform/enterprise-edition), provides visualization, ability to restart from selected stage and few other features. Below the stage, you'll notice that the commands are inside an `if` statement. The value of the *provision* variable comes from the checkbox that you've see in the *build* screen (more about those parameters later on). The "meat" of this snippet are two `sh` statement. It is one of the step types provided by the Workflow plugin and, as you might have guessed, it runs any shell command we specify. In this case, we're using it to run Ansible playbooks that will take care of provisioning. Please note that we did not install nginx when we set up the servers. This script will do that for us.

With provisioning out of the way, we should start the deployment. Since we are deploying already built docker containers, all we need is a *docker-compose.yml* file that is best kept in the same repository as the service code. We can get it by cloning the repository with a simple *git* step.

```groovy
    git url: "https://github.com/${repo}.git"
```

Please note the usage of the `${repo}`. This is another case of us utilizing parameters that can be specified in the build screen.

Before we start the actual deployment, we need to discover how many instances should be run.

![Workflow console screen](img/console.png)

TODO: Mention CloudBees Jenkins Platform Enterprise Edition




