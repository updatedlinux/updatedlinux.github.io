# [Upgrade and Migration to another GitLab server](https://updatedlinux.github.io)

 * Objetive: Update gitlab to the latest version available and then migrate it to a new server. (Method of installation of packages)

 * Scope: Physical and Virtual Servers of OpenStack, AWS or VMWare with Centos/Rhel 8.

 ## Let's start step by step

In our environment Gitlab is deployed under version 11.3, Take into account that gitlab does not allow updating to higher versions immediately, you must perform a progressive update.

For this LAB the latest version is: gitlab-ce-13.7.1-ce.0.el7

This whole environment is based on CentOS 7 and CentOS 8.

Let's start updating from version 11.3 to 11.8
```bash
  yum install gitlab-ce-11.8.0-ce.0.el7
  gitlab-ctl restart
```

We will continue making **progressive version** jumps until we reach the last one. 

Continue with:
```bash
yum install gitlab-ce-12.0.0-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-12.2.5-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-12.2.5-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-12.9.5-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-12.10.14-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-13.0.0-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-13.2.10-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-13.4.3-ce.0.el7
gitlab-ctl restart
yum install gitlab-ce-13.7.1-ce.0.el7
gitlab-ctl restart
```

Now we are going to create the backup to migrate to another server

run the following commands: 

```bash
mkdir gitlab-old
cp -R /etc/gitlab/* ~/gitlab-old
cd gitlab-old/
cp /var/opt/gitlab/backups/XXXXXXXXXXX_XXXX_XX_XX_XX.X.X_gitlab_backup.tar . 
cd ..
tar -cvzf gitlabold.tar.gz gitlab-old/
```
 
At this point we must move the tar.gz that we just created, the way I leave it to you, you can use SCP if you have that permission allowed between servers (I do not recommend this in a productive environment) or directly extract the tar.gz and upload it to the new server. It is up to you to choose the way.

Remember: **Great power comes with great responsibility. =)** 

Now we go to the new server (CentOS 8), where we will install Gitlab in its latest version in a simple way.

```bash
dnf -y install curl policycoreutils openssh-server openssh-clients
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
dnf install gitlab-ce -y
```

Now we are going to deploy the backup made on the old server (CentOS 7)
Remember that previously we compressed everything necessary in tar.gz format, I am going to decompress it in my workspace and then place all the data in the correct folders.

```bash
tar -xvzf gitlabold.tar.gz 
```
now we begin to locate the files:
```bash
cp -R gitlab-old/* /etc/gitlab
gitlab-ctl reconfigure
```

Take into consideration that you must have all the precise information from the /etc/gitlab.rb file, for example if you have authentication by ldap, that the new server can have communication with the AD server, if you enter git by IP and not by FQDN, you should change the IP to the current server.

I am telling you all this because if it has errors, the reconfigure will fail.

## Deploy Backup

Now we are going to deploy the Backup

```bash
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
```
Copy backup file to /var/opt/gitlab/backups, then change ownership and permission to git user
```bash
cp gitlab-old/XXXXXXXXXX_gitlab_backup.tar /var/opt/gitlab/backups
chown git:git /var/opt/gitlab/backups/XXXXXXXXXX_gitlab_backup.tar
```
Run the GitLab restore process

```bash
gitlab-rake gitlab:backup:restore BACKUP=XXXXXXXXX
```
I know if you got to this point, it is because **you have common sense**, but it is worth clarifying that XXXXX is the same file that you generated with gitlab-rake, let's see Git will create it with a UUID that will be different from this one recipe, remember: **common sense**.

```bash
cd /var/opt/gitlab/backups/
gitlab-rake gitlab:backup:restore BACKUP=XXXXXXXXX
gitlab-ctl start
gitlab-rake gitlab:check SANITIZE=true
```
That is all. -

Easy? comment on my linkedIn


- **By Jonathan Alexander Melendez Duran**
- **Caracas, Venezuela**
- **soyjonnymelendez AT gmail DOT com**
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)**




