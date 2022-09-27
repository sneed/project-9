## TOOLING WEBSITE DEPLOYMENT AUTOMATION WITH CONTINUOUS INTEGRATION. INTRODUCTION TO JENKINS

In previous Project 8 we introduced horizontal scalability concept, which allow us to add new Web Servers to our Tooling Website and you have successfully deployed a set up with 2 Web Servers and also a Load Balancer to distribute traffic between them. If it is just two or three servers – it is not a big deal to configure them manually. Imagine that you would need to repeat the same task over and over again adding dozens or even hundreds of servers.

DevOps is about Agility, and speedy release of software and web solutions. One of the ways to guarantee fast and repeatable deployments is **Automation** of routine tasks.

In this project we are going to start automating part of our routine tasks with a free and open source automation server – [Jenkins.](https://en.wikipedia.org/wiki/Jenkins_(software)) It is one of the most popular [CI/CD](https://en.wikipedia.org/wiki/CI/CD) tools, it was created by a former Sun Microsystems developer Kohsuke Kawaguchi and the project originally had a named "Hudson".

According to Circle CI, **Continuous integration (CI)** is a software development strategy that increases the speed of development while ensuring the quality of the code that teams deploy. Developers continually commit code in small increments (at least daily, or even several times a day), which is then automatically built and tested before it is merged with the shared repository.

In our project we are going to utilize Jenkins CI capabilities to make sure that every change made to the source code in GitHub <mark>https://github.com/<yourname>/tooling</mark> will be automatically be updated to the Tooling Website.

#### Side Self Study
Read about [Continuous Integration, Continuous Delivery and Continuous Deployment.](https://circleci.com/continuous-integration/)

#### Task
Enhance the architecture prepared in Project 8 by adding a Jenkins server, configure a job to automatically deploy source codes changes from Git to NFS server.

Here is how your updated architecture will look like upon competion of this project:

![Architecture](../images/Architecture.png)

Let us do it!


## INSTALL AND CONFIGURE JENKINS SERVER

### Step 1 – Install Jenkins server

1. Create an AWS EC2 server based on Ubuntu Server 20.04 LTS and name it "Jenkins"

2. Install JDK (since Jenkins is a Java-based application)
```
sudo apt update
sudo apt install default-jdk-headless
```
3. Install Jenkins
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
/etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt-get install jenkins
```

Make sure Jenkins is up and running

`sudo systemctl status jenkins`

![Jenkins up and running](../images/jenkins-running.png) 

4. By default Jenkins server uses TCP port 8080 – open it by creating a new Inbound Rule in your EC2 Security Group

![TCP port 8080](../images/jenkins-tcp8080.png)

5. Perform initial Jenkins setup.
   From your browser access <mark>http://<Jenkins-Server-Public-IP-Address-or-Public-DNS-Name>:8080</mark>

You will be prompted to provide a default admin password

![Prompted default password](../images/prompted-defaultpassword.png)

Retrieve it from your server:

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`

![Jenkins password](../images/Jenkins-password.png)

Then you will be asked which plugins to install – choose suggested plugins.

![Plugin to install](../images/unlock-jenkins.png)

Once plugins installation is done – create an admin user and you will get your Jenkins server address.

The installation is completed!

![Installation completed](../images/Jenkins-ready.png)


### Step 2 – Configure Jenkins to retrieve source codes from GitHub using Webhooks

1. In this part, you will learn how to configure a simple Jenkins job/project (these two terms can be used interchangeably). This job will will be triggered by GitHub webhooks and will execute a ‘build’ task to retrieve codes from GitHub and store it locally on Jenkins server.

Enable webhooks in your GitHub repository settings

![Enable webhooks](../images/add-webhook.png)


2. Go to Jenkins web console, click "New Item" and create a "Freestyle project"

![alt text](image.jpg)

To connect your GitHub repository, you will need to provide its URL, you can copy from the repository itself

![alt text](image.jpg)

In configuration of your Jenkins freestyle project choose Git repository, provide there the link to your Tooling GitHub repository and credentials (user/password) so Jenkins could access files in the repository.

![alt text](image.jpg)


Save the configuration and let us try to run the build. For now we can only do it manually.
Click "Build Now" button, if you have configured everything correctly, the build will be successfull and you will see it under <mark>#1</mark>

![Build Now](../images/build-now.png)

You can open the build and check in "Console Output" if it has run successfully.

If so – congratulations! You have just made your very first Jenkins build!

But this build does not produce anything and it runs only when we trigger it manually. Let us fix it.

![Console Output](../images/console-output.png)

3. Click "Configure" your job/project and add these two configurations
Configure triggering the job from GitHub webhook:

![Configure webhook](../images/configure-hooktrigger.png)

Configure "Post-build Actions" to archive all the files – files resulted from a build are called "artifacts".

![Post build actions](../images/postbuild-actions.png)

Now, go ahead and make some change in any file in your GitHub repository (e.g. README.MD file) and push the changes to the master branch.

You will see that a new build has been launched automatically (by webhook) and you can see its results – artifacts, saved on Jenkins server.

![alt text](../)


You have now configured an automated Jenkins job that receives files from GitHub by webhook trigger (this method is considered as ‘push’ because the changes are being ‘pushed’ and files transfer is initiated by GitHub). There are also other methods: trigger one job (downstreadm) from another (upstream), poll GitHub periodically and others.

By default, the artifacts are stored on Jenkins server locally


`ls /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/`


## CONFIGURE JENKINS TO COPY FILES TO NFS SERVER VIA SSH

### Step 3 – Configure Jenkins to copy files to NFS server via SSH
Now we have our artifacts saved locally on Jenkins server, the next step is to copy them to our NFS server to <mark>/mnt/apps</mark> directory.

Jenkins is a highly extendable application and there are 1400+ plugins available. We will need a plugin that is called "Publish Over SSH".

1. Install "Publish Over SSH" plugin.
On main dashboard select "Manage Jenkins" and choose "Manage Plugins" menu item.

On "Available" tab search for ["Publish Over SSH"](https://plugins.jenkins.io/publish-over-ssh/) plugin and install it


![alt text](image.jpg)

2. Configure the job/project to copy artifacts over to NFS server.
On main dashboard select "Manage Jenkins" and choose "Configure System" menu item.

Scroll down to Publish over SSH plugin configuration section and configure it to be able to connect to your NFS server:

1. Provide a private key (content of .pem file that you use to connect to NFS server via SSH/Putty)
2. Arbitrary name
3. Hostname – can be private IP address of your NFS server
4. Username – ec2-user (since NFS server is based on EC2 with RHEL 8)
5. Remote directory – /mnt/apps since our Web Servers use it as a mounting point to retrieve files from the NFS server

Test the configuration and make sure the connection returns Success. Remember, that TCP port 22 on NFS server must be open to receive SSH connections.

![alt text](image.jpg)

Save the configuration, open your Jenkins job/project configuration page and add another one "Post-build Action"

![alt text](image.jpg)


Configure it to send all files produced by the build into our previously define remote directory. In our case we want to copy all files and directories – so we use <mark>**</mark>.
If you want to apply some particular pattern to define which files to send – [use this syntax.](https://ant.apache.org/manual/dirtasks.html#patterns)

![alt text](image.jpg)


Save this configuration and go ahead, change something in <amrk>README.MD</mark> file in your GitHub Tooling repository.

Webhook will trigger a new job and in the "Console Output" of the job you will find something like this:

SSH: Transferred 25 file(s)
Finished: SUCCESS
To make sure that the files in <mark>/mnt/apps</mark> have been updated – connect via SSH/Putty to your NFS server and check README.MD file

`cat /mnt/apps/README.md`


If you see the changes you had previously made in your GitHub – the job works as expected.

Congratulations!
You have just implemented your first Continous Integration solution using Jenkins CI. Watch out for advanced CI configurations in upcoming projects.