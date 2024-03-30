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

--------

## Salt Execution Modules
While shelling out using cmd.run is certainly useful, the real power comes when you add Salt execution functions. Salt execution modules are the core components of the SaltStack configuration management system. These modules provide a wide range of functions that can be used to manage and configure systems remotely. They are Python files that define functions that can be called using the Salt command-line interface (CLI) or through Salt's Remote Execution system.

**The Ad hoc commands executed from the command line against one or more managed systems. Useful for:**

* Real-time monitoring, status, and inventory
* One-off commands and scripts 
* Deploying critical updates

**Some common built-in execution modules include:**

* pkg: Manages software packages
* file: Manages files and directories
* service: Manages system services
* user: Manages user accounts
* network: Manages network settings

*****

The Salt community has put tremendous effort into creating hundreds of functions that simplify most management tasks. Even better, the same function can be used consistently across all supported platforms. Resist the urge to shell out. Learn the ways of the Salt execution functions.

We can list all the available execution modules on all connected Minions using the following ```salt '*' sys.list_modules``` command.

For instance, if you want to install the Apache web server on a remote system, you can run the following command:
```
sudo docker exec -it salt-master salt '*' pkg.install apache2
```

The command ```salt-master salt '*' pkg.install apache2 ``` will perform the following actions in detail:

* ```salt-master``` refers to the Salt master, which is the central point of control for managing all connected minions.
* ```salt '*' ``` specifies that the ```pkg.install apache2``` command should be executed on all minions connected to the Salt master, denoted by the wildcard character *.
* ```pkg.install``` is a Salt execution module function that installs packages on remote systems.
* ```apache2``` is the name of the package that will be installed on all the connected minions.

  The command will trigger the following steps on the connected minions:

1. The Salt master will send the pkg.install command with the apache2 package name to all the connected minions.
2. Each minion will receive the command and execute it.
3. The minion will check if the apache2 package is already installed on the system.
4. If the package is not already installed, the minion will install it using the package manager specified for that operating system. For example, on Ubuntu or Debian-based systems, it will use apt-get to install the apache2 package.
5. If the package is already installed, the minion will take no action and report back to the Salt master that the installation is already up-to-date.
6. Once the installation is complete, the minion will report back to the Salt master that the installation was successful.

   In summary, this command will use SaltStack to automate the installation of the apache2 package on all connected minions, making it easy to manage software packages across a large number of systems with a single command.

   You can find more salt execution modules and it’s functions in the official Salt documentation:
   [Execution moudules] https://docs.saltproject.io/en/latest/ref/modules/all/index.html

*******

## Local Execution

Local execution in Salt refers to running Salt commands directly on the Minion without involving the Master. This is particularly useful when you want to execute ad-hoc commands, troubleshoot issues on a specific Minion, or when the Master is unavailable.

To execute a command on a specific Minion using local execution, we use the ```salt-call``` command followed by the module and function to call.

**To get the disk usage information of ```salt-minion-1```, run:**

``` sudo docker exec -it salt-minion-1 salt-call disk.usage ```

The ```salt-call disk.usage``` command is used to retrieve disk usage information on the local system. Here's what this command does in more detail:

* ```salt-call``` is a command-line tool used to execute Salt "execution modules" locally on a minion.
* ```disk.usage``` is the name of a Salt execution module that retrieves disk usage information for the current system.
* When you execute ```salt-call disk.usage```, Salt will run the ```disk.usage``` module on the local system and return the results.

We can browse through all the available execution functions related to disk management on all the Salt Minions that are connected to the Salt Master using ```salt '*' sys.list_functions disk ```command. 
To view functions for other modules, you can replace disk with the given module name in the above command.

  



    
    

   
    
    
   



   


   
   
   






