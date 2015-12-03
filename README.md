Blue-Green Deployment Workflow To Docker Swarm with Jenkins
===========================================================

The idea behind this article is to explore ways to deploy releases to [Docker Swarm](https://docs.docker.com/swarm/) without downtime. We'll use *blue-green* process. More info about the process and one possible implementation can be found in the [Blue-Green Deployment, Automation and Self-Healing Procedure](http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-docker-compose-and-consul-part-34-blue-green-deployment-automation-and-self-healing-procedure/) article. One of the downsides of the process we used in that article is Ansible itself. While it is probably the best tool for provisioning and orchestration, it had some downsides when we tried to use it as the tool to deploy containers especially if the process is complex. It lacked some constructs common in most programming languages. This time we'll try to implement the same process but using the [Jenkins Workflow Plugin] and a bit of Groovy scripts.

Let's see it in action first and then discuss how it was done.

Setting Up the Docker Swarm Cluster
-----------------------------------

We'll start by creating the cluster we'll need for running Docker Swarm. We'll use VirtualBox and Vagrant to create the VMs and then provision them with Ansible. I won't go into details how Ansible playbooks we'll run work since that information can be found in other articles in this blog. The setup will be similar to the one described in the [Deploying Containers with Docker Swarm and Docker Networking](http://technologyconversations.com/2015/11/25/deploying-containers-with-docker-swarm-and-docker-networking/) article published recently. The major difference will be the addition of Jenkins.

I'll assume that you already have [VirtualBox](https://www.virtualbox.org/), [Vagrant](https://www.vagrantup.com/) and [Git](https://git-scm.com/) installed. If you are a Windows user, please follow the instructions described in [Running Linux VMs on Windows](http://technologyconversations.com/2015/11/24/running-linux-vms-on-windows/) before diving into those described below.

Let's get going! We can create the servers by running the following command.

```bash
git clone https://github.com/vfarcic/blue-green-docker-jenkins.git

cd blue-green-docker-jenkins

vagrant up swarm-master swarm-node-1 swarm-node-2
```

Next, we should provision servers with Docker Swarm, Docker Compose, Consul, Consul Template and Registrator. If you are new to Docker Swarm, you might benefit from reading the [Docker Clustering Tools Compared: Kubernetes vs Docker Swarm](http://technologyconversations.com/2015/11/04/docker-clustering-tools-compared-kubernetes-vs-docker-swarm/) article. Information about Consul and the other tools we'll use for service discover can be found in the [Service Discovery: Zookeeper vs etcd vs Consul](http://technologyconversations.com/2015/09/08/service-discovery-zookeeper-vs-etcd-vs-consul/) article.

It might take some time to create all three servers especially if this is the first time you're using Vagrant with Ubuntu. Once they are up and running, we can use Ansible to configure the cluster. Since the creation of VMs included installation of Ansible in the *swarm-master* node, we can use it to run the [swarm.yml](TODO) playbook.

```bash
vagrant ssh swarm-master

ansible-playbook /vagrant/ansible/swarm.yml \
    -i /vagrant/ansible/hosts/prod
```

Finally, the only thing left is to set up Jenkins. We'll continue using Ansible for this task. It will create few directories, run the Jenkins container, fiddle with few configurations, install plugins we'll need and, finally, create the job that will do the actual deployment. Since the focus of this article is to experiment with Jenkins Workflow in the context of blue-green deployment to the Swarm cluster, I'll skip explanations how the [jenkins.yml](TODO) playbook works. Feel free to take a look at the code or to post a comment and, if there is enough interest, I'll write a separate article about setting up Jenkins with Docker and Ansible.

```bash
ansible-playbook /vagrant/ansible/jenkins.yml \
    -i /vagrant/ansible/hosts/prod
```

Exploring The Jenkins Workflow
==============================

Let's start the build. Since the first run last a bit longer than consecutive runs, we'll us that time to discuss the solution. Please open [http://10.100.192.200:8080/job/books-ms/build?delay=0sec](http://10.100.192.200:8080/job/books-ms/build?delay=0sec), deselect the *build* checkbox and click the *Build* button. You can monitor progress by opening the [http://10.100.192.200:8080/job/books-ms/lastBuild/console](http://10.100.192.200:8080/job/books-ms/lastBuild/console) page.



