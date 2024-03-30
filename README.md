# Saltstack-Basics
SaltStack is an open-source software for configuration management, remote execution, and orchestration of infrastructure. It allows system administrators to manage and automate the configuration and deployment of servers and network devices, as well as the execution of commands and scripts across large numbers of systems.

* SaltStack provides a central repository for storing configuration files and execution modules that can be used to manage and automate the configuration of systems.
* The Salt Master distributes these files to the Salt Minions (servers or devices being managed) and ensures that they are configured correctly.
* SaltStack also allows administrators to execute commands and scripts across many systems simultaneously, which can save time and reduce errors.
* A practical analogy to understand SaltStack is to think of it as a chef who manages a large kitchen with many cooks and many dishes to prepare.
* Just like a chef who needs to ensure that each dish is prepared according to the recipe and delivered to the customer on time, system administrators need to ensure that each system in their infrastructure is configured 
   correctly, running smoothly, and delivering services to end-users efficiently.
* SaltStack helps system administrators to maintain consistency, reduce errors, and save time, just like a chef who manages a large kitchen with many cooks and many dishes to prepare.

## Setting up a Salt environment

Setting up a Salt environment involves creating a working Salt infrastructure that includes a Salt Master server and one or more Salt Minion servers, and configuring them to work together to manage your IT infrastructure.
****************
We'll set up a basic Salt Master - Minion using Docker containers. We'll create 3 containers: one for the Salt master server and another two for the Salt minion servers.
1. Install Docker on Ubuntu distro
   ```
      sudo apt update
      sudo apt install docker.io -y
      sudo systemctl start docker
      sudo docker --version
   ```
2. Create configuration files for the Salt master and Salt minion servers. These files define settings such as the IP addresses or hostnames of the Master and Minion servers, the communication protocols to use, the location of SSL certificates, and other relevant parameters.
   ```mkdir -p workspace/saltcfg```
   
   ```cd workspace/saltcfg```
   
   ```vim master-config.conf```
   
   Add below line into the **master-config.conf**
   
   ``interface: 0.0.0.0``

    ---------
   The interface directive is used to specify the network interface the Salt Master should bind to. By setting it to 0.0.0.0, we're telling the Salt Master to bind to all available network interfaces. This allows the Salt Master to be accessible from any IP address, including the Docker containers.

   -------
   Ensure you are in the ```workspace/saltcfg``` Directory and create config file for minions.
   
   ```vim minion-config.conf```

   Add below line to minion-config.conf file

   ```master: salt-master```

   ----
   The master directive is used to specify the hostname or IP address of the Salt Master server. In our case, we set it to salt-master, which will be the service name of the docker container which will be created in next steps. Docker's internal DNS resolver will automatically resolve this service name to the IP address of the Salt Master container. This way, the Salt Minion knows where to find the Salt Master.
     
   ------

3. Create Docker images for Salt Master and Minions. First, create a Dockerfile for the Salt Master:
   
   ```vim Dockerfile.master```
   Copy below code snippet.

   ```
   FROM ubuntu:18.04
   
   RUN apt-get update && \
       apt-get install -y salt-master nano
   
   COPY master-config.conf /etc/salt/master
   
   EXPOSE 4505 4506
   
   CMD ["salt-master", "-l", "info"]
   ```

   Now, create a Dockerfile for the Salt Minions:

   ```vim Dockerfile.minion```
   Copy below code snippet.

   ```
   FROM ubuntu:18.04
   
   RUN apt-get update && \
       apt-get install -y salt-minion
   
   COPY minion-config.conf /etc/salt/minion
   
   CMD ["salt-minion", "-l", "info"]
   ```
   The /etc/salt/ directory contains the Salt configuration files, including the master and minion configuration files.

4. Build Docker images. In the ~/workspace/saltcfg directory containing the Dockerfile.master and Dockerfile.minion, run these commands:
   ```
     sudo docker build -t salt-master -f Dockerfile.master .
   ```

   ```
     sudo docker build -t salt-minion -f Dockerfile.minion .
   ```
   
5. Create a custom Docker network to allow the Salt Master and Minions to communicate with each other.
   
    ```
     sudo docker network create salt-network
    ```

6. Run Salt Master container
   
   ```   
    sudo docker run -d --name salt-master --network salt-network -p 4505:4505 -p 4506:4506 salt-master
   ```
   
7. Run a few Salt Minion containers:

   ```
     sudo docker run -d --name salt-minion-1 --network salt-network salt-minion
   ```

   ```
     sudo docker run -d --name salt-minion-2 --network salt-network salt-minion
   ```
8. When a new Salt Minion is set up and attempts to connect to the Salt Master, the Master will receive a request to accept the Minion's public key. Run the below command to list the public keys of all the Salt Minions that 
  have requested to connect to the Salt Master.

   ```
      sudo docker exec -it salt-master salt-key -L
   ```
   
9. Accept all pending public keys for Salt Minions that have requested to connect to the Salt Master, without any manual confirmation prompts using below command:

    ```
       sudo docker exec -it salt-master salt-key -A -y
    ```
    -----------
   Accepting a Minion's key means that the Master server is acknowledging that it trusts the Minion and will allow it to communicate with the Master. This process involves verifying the Minion's identity and ensuring that 
   it is authorized to connect to the Master. Once the key is accepted, the Master will add the Minion to its list of authorized clients and grant it access to the Salt infrastructure.
   
10. Test your Salt Master and Minions setup by running a simple command from the Salt Master.
    ```
      sudo docker exec -it salt-master salt '*' test.ping
    ```

    **Reference**
    [Saltstak](https://docs.saltproject.io/en/getstarted/system/index.html)

    ## Remote Execution

       Remote execution is a core feature of Salt that allows you to execute commands on multiple systems concurrently using the Salt Master. Salt uses a high-speed messaging system to communicate between the Master and its 
       Minions. You can use Salt to execute shell commands, manage files, or execute custom Salt modules on Minion nodes. This feature allows Salt administrators to manage large fleets of systems in a centralized manner 
       and perform various administrative tasks, such as software updates, system monitoring, and troubleshooting.

       Salt uses a publish-subscribe model where Minions listen for commands from the Master and execute them. Commands are sent securely over an encrypted channel, and results are returned to the Master.

    **To check the current date and time on all Minion nodes, run the following command from the Salt Master:**

    Salt lets you remotely execute shell commands across multiple systems using ```cmd.run.```
    
    ```sudo docker exec -it salt-master salt '*' cmd.run 'date'```

    **Here's a breakdown of what each part of the command does:**

   * salt '*': This part of the command specifies that the command should be run on all connected Salt Minions, using the "" target specifier.
   * cmd.run 'date': This part of the command specifies the command that should be executed on the Salt Minions. In this case, the "date" command is being executed, which will return the current date and time on each Minion.
   * The output of this command will be the current date and time on each connected Salt Minion, which can be useful for quickly checking the system clock across multiple systems.


   **To run a command on a specific minion in SaltStack, you can use Salt's targeting system to specify which minions should execute the command.**

For example, if you want to run a date command only on a minion with id ```08549666c3a1```, then the command would look like this:- ```salt-master salt '08549666c3a1' cmd.run 'date'```.

Read more about the targeting system here:- [Saltstack](https://docs.saltproject.io/en/getstarted/fundamentals/targeting.html)
    
    

   
    
    
   



   


   
   
   






