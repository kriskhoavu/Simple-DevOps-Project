# Deploy on a docker container using Ansible
# *Jenkins Job name:* `Deploy_on_Container_using_ansible`

### Pre-requisites

1. Jenkins server 
1. Docker-host server 
1. Ansible server
1. `Dockerfile` under *`/opt/docker`* on Ansible server **[Get Help Here]()**
   ```sh 
   # Pull tomcat latest image from dockerhub 
   From tomcat

   # To be resolved issue with latest tomcat
   RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps

   # copy war file on to container 
   COPY ./webapp.war /usr/local/tomcat/webapps
1. Create `create-docker-image.yml` unser *`/opt/docker`* on Ansible server **[Get Help Here]()**
   ```sh
   ---
   - hosts: ansible
     #ansadmin doesn't need root access to create an image
     become: true 

     tasks:
       - name: create docker image
         command: docker build -t regapp:latest .
         args:
           chdir: /opt/docker

       - name: create tag to push image onto dockerhub
         command: docker tag regapp:latest krisvu/regapp:latest

       - name : push docker image
         command: docker push krisvu/regapp:latest
   ```
1. Create `create-docker-container.yml` under *`/opt/docker`* on Ansible server **[Get Help Here]()**
   ```ssh
   ---
   - hosts: docker
     become: ture

     tasks:
       - name: stop existing container
         command: docker stop regapp-server
         ignore_errors: yes

       - name: remove the container
         command: docker rm regapp-server
         ignore_errors: yes

       - name: remove image
         command: docker rmi krisvu/regapp:latest   
         ignore_errors: yes

       - name: create container
         command: docker run -d --name regapp-server -p 8080:8080 krisvu/regapp:latest
   ```

### Integration between Ansible-control-node and Jenkins

Install "publish Over SSH"
 - `Manage Jenkins` > `Manage Plugins` > `Available` > `Publish over SSH`

Enable connection between Ansible-control-node and Jenkins

- `Manage Jenkins` > `Configure System` > `Publish Over SSH` > `SSH Servers` 

	- SSH Servers:
                - Name: `ansible-server`
		- Hostname:`<ServerIP>`
		- username: `ansadmin`
               
       -  `Advanced` > chose `Use password authentication, or use a different key`
		 - password: `*******`
 
### Steps to create "Deploy_on_Container_using_ansible" Jenkin job
#### From Jenkins home page select "New Item"
   - Enter an item name: `Deploy_on_Container_using_ansible`
     - Copy from: `Deploy_on_Container`
     
   - *Source Code Management:*
      - Repository: `https://github.com/yankils/hello-world.git`
      - Branches to build : `*/master`  
   - *Poll SCM* :      - `* * * *`

   - *Build:*
     - Root POM:`pom.xml`
     - Goals and options: `clean install package`

 - *Post-build Actions*
   - Send build artifacts over SSH
     - *SSH Publishers*
      - SSH Server Name: `ansible-server`
       - `Transfers` >  `Transfer set`
            - Source files: `webapp/target/*.war`
	       - Remove prefix: `webapp/target`
	       - Remote directory: `//opt//docker`
	       - Exec command: 
                ```sh 
                ansible-playbook -i /opt/docker/hosts /opt/docker/create-docker-image.yml;
                sleep 10;
                ansible-playbook -i /opt/docker/hosts /opt/docker/create-docker-container.yml;
                ```

Save and run the job now.