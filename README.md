---
copyright:
  years: 2017, 2018
lastupdated: "2018-08-13"
---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# MQ Managed File Transfer for Containers
{: #mft_intro}

#### What is IBM MQ Managed File Transfer(MFT) ?
For many organizations, the exchange of files between business systems remains a common and important integration methodology. Files are the simplest unit of data to exchange and often represent the lowest common denominator for an enterprise infrastructure.

Although the exchange of files is conceptually simple, doing so in the enterprise is a challenge to manage and audit. This difficulty is brought into clear focus when an organization needs to perform file transfer with a different business organization, perhaps using a different physical network, with different security requirements, and perhaps a different governance or regulatory framework.

IBM® MQ File Transfer Edition provides an enterprise-grade managed file transfer capability that is both robust and easy to use. MQ File Transfer Edition exploits the proven reliability and connectivity of MQ to transfer files across a wide range of platforms and networks. MQ File Transfer Edition takes advantage of existing MQ networks, and you can integrate it easily with existing file transfer systems.

You can find more information at [IBM Knowledge Centre](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.1.0/com.ibm.mq.pro.doc/wmqfte_intro.htm)

---

## Running MFT Agents in Containers
MQ Advanced supports running MFT Agents in docker containers and this guide will help you setup MFT for docker, run MFT agents in containers and also run a successful file transfer using  MFT agents running in containers.

## Prerequisites
{: #mft_containers_prereq}

* [Install Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
* Clone the repository **mqmft-container** into your computer:
    1. Open a new command-shell and navigate to home path
        ```
        On Linux: cd /home
        On Windows: cd %HOMEDRIVE%
        ```
    2. Clone the current repository into **mqmft-container** directory
        ```
        HTTPS: git clone https://github.com/darbhakiran/mqmft-container.git
        ```
* Download **9.1.0.0-IBM-MQFA-Redist-LinuxX64.tar.gz** from [IBM Fixcentral](https://www.ibm.com/support/fixcentral/) into mqmft-container/agent/
**Note: 9.1.0.0-IBM-MQFA-Redist-LinuxX64.tar.gz** has to be in same path as **Dockerfile-agent**.

{:shortdesc}

---
### MFT Container Customisations
{: #mft_containers_customizations}
We will add few customizations that would simplify the configuration of MFT for coordination queue manager and agent setup.
**Note:** These custimization are only needed to simplify the MFT configuration. The same steps, that are set of mqsc commands, can be executed on base mq image directly using *docker exec*.  

#### Understanding Customizations  
1. Configuring MFT Coordination Queue manager:
    1.1 Coordination manager requires set of system queues and topics to be created, this is one time activity.
    1.2 Agents will need a SVRCONN channel using which they can communication with MQ queue manager. This SVRCONN channel has to be setup with appropriate CHLAUTH/CONNAUTH rules. This guide creates a new channel **MFT_SVRCONN** and setup CHLAUTH/CONNAUTH rules such that any user with a valid password can connect to queue manager.
    1.3 **mft_setupCoordination.sh** available in **mqmft-container/server** directory is aimed at simplifying this configuration.
    * *mqmft-container/server/setupQMForMFTAgents.mqsc* : MQSC script file that has definations for new resources.
    
2. New Agent Setup:
Every MFT Agent needs set of system queues to be created on the coordination queue manager. 
    2.1 **mft_setupAgent.sh** available in **mqmft-container/server/** directory is aimed at simplifying this configuration
     - *mqmft-container/server/createAgent.mqsc*: MQSC file that has definitions for creating new queues for Agent.

    2.2 **mqmft-container/server/mft_removeAgent.sh** available in **server** directory is to remote the agent resuorces from the coordination manager. This is to be executed if an mft agent has to be deleted.
    - *deleteAgent.mqsc*: MQSC script file that has definitions for delete Agent's resouces on coordination queue manager.

All the cutomizations (.sh and .mqsc) files are copied into **/etc/mqm/mft/** path of the container.

---

### Creating Customized MQ Queue Manager for MFT in containers
{: #mft_containers_cqm_setup}

1. Open a command shell and navigate to path of **mqmft-container/server**. For example */home/mqmft-container/server*(on Linux) or *C:\\mqmft-container\\server*(on Windows).  
2. Run a docker command to make sure docker is setup and docker service is running
    ```
    docker version
    docker ps -a
    ```
    First command will print docker information and second command will list all the available containers in your docker environment.
3. Create a new customized mq image
    ```
    docker build -t mqadvmft  -f dockerfile-server
    ```
4. Once the docker build is successful, run a new container of it, which is queue manager in container. This queue manager is to be used as coordination queue manager.
    ```
    docker run --env LICENSE=accept --env MQ_QMGR_NAME=QM1 --publish 1414:1414 --publish 9443:9443 --detach --name=QM1 mqadvmft
    ```
5. Run the below command to list the docker containers, find newly created container **QM1** and make a note of its container-id.
    ```
    docker ps
    ```
6. Setup the **QM1** as coordination queue manager. Running the **mft_setupCoordination.sh** script will create required configuration.
    ```
    docker exec -ti <QM1-container-id> /etc/mqm/mft/mqft_setupCoordination.sh
    ```
    Successful run of this script completes queue manager configuration for MFT.  
    
---

### Agent setup for MFT Containers
{: #mft_containers_agent_setup}

Agent package contains a dockerfile-agent to build the MFT agent docker image. MFT Agent is setup and started as part of the **mqft.sh** script. This package assumes a single queue manager(**QM1**) as coordination queue manager, command queue manager and agent queue manager.
**Note:** As per your application architecture, you can consider to have separate queue managers for coordination, command and agents.

1. Copy the MFT redistributable package: **9.1.0.0-IBM-MQFA-Redist-LinuxX64.tar.gz** to the Agent directory. For example 
    ```
    On Linux: /home/mqmft-container/agent 
    On Windows: %HOMEDRIVE$\mqmft-container/agent 
    ```
2. Open a command shell and navigate to path of **mqmft-container/agent** repository. For example */home/mqmft-container/server*(on Linux) or *C:\\mqmft-container\\server*(on Windows).  

3. Build mft agent image.
    ```
    docker build -t mftagentredist -f dockerfile-agent
    ```
4. We will create two agents as part of this document to demonstrate mft agents in container. These agents are **AGENTSRC** and **AGENTDEST**. As a first step of agent configuration, we have to create their congfiguration on coordination queue manager.
    ```
    docker exec -ti <QM1-container-id> /etc/mqm/mft/mqft_setupAgent.sh AGENTSRC
    docker exec -ti <QM1-container-id> /etc/mqm/mft/mqft_setupAgent.sh AGENTDEST
    ```
    **Note:** 
    1. QM1-Container-id is the id of the queue manager container created in above section.
    2. **mqft_setupAgent.sh** script requires MFT agent name as input parameter
    
5. Once the docker-agent build is successful, run a new container of it, which is agent in container. 
    ```
    docker run --env MQ_QMGR_NAME=QM1  --env MQ_QMGR_HOST=<docker-host-ip> --env MQ_QMGR_PORT=1414 --env MQ_QMGR_CHL=MFT.SVRCONN --env MFT_AGENT_NAME=AGENTSRC -d --name=AGENTSRC mftagentredist
    
    docker run --env MQ_QMGR_NAME=QM1  --env MQ_QMGR_HOST=<docker-host-ip> --env MQ_QMGR_PORT=1414 --env MQ_QMGR_CHL=MFT.SVRCONN --env MFT_AGENT_NAME=AGENTDEST -d --name=AGENTDEST mftagentredist
    ```
    **Note:** 
    1. <docker-host-ip>: Is the IP Address of the docker host. This could be found out by running `docker inspect` command and look for ipv4address field.
    2. MQ_QMGR_NAME=QM1: Is the queue manager we created and configured as coordination queue manager in above section.
    3. mftagentredist: Is the docker image of mft redistributable agents.
    
6. Run the below command to list the docker containers, find newly created containers **AGENTSRC**, **AGENTDEST** and make a note of their container-ids.
    ```
    docker ps
    ```
7. Check if the mft agents accept commands and show output. For example, run following command on Agent containers
    ```
    docker exec -ti <AGENTSRC-container-id> fteListAgents
    docker exec -ti <AGENTDEST-container-id> fteListAgents
    ```
    If both the containers show the Agents information, that confirms Agent's running in containers and accepting commands.
8. To verify the Agent setup, use following command to list the container logs
    ```
    docker logs <AGENTSRC-container-id>
    docker logs <AGENTDEST-container-id>
    ```
---

### Create a File Transfer with MFT Agents in Containers with a host path mounted
{: #mft_containers_filetransfer_demo}

We will create a text file on **AGENTSRC** that will be transferred to **AGENTDEST**. We then transfer this file using the **fteCreateTransfer** command.

1. Open a command shell and run the docker command to list all the containers. Check that newly created containers(**QM1,AGENTSRC and AGENTDEST**) are running.
    ```
    docker ps
    ```
2. Create a text file on **AGENTSRC** for file transfer.
    ```
    docker exec -ti <AGENTSRC-container-id> bash -c "echo 'Hello World' >> /tmp/file.txt"
    ```
3. Check if the file is created on **AGENTSRC**
    ```
    docker exec -ti <AGENTSRC-container-id> bash -c "cat /tmp/file.txt"
    ```
4. Check that file doesn't exist on **AGENTDEST**.
    ```
    docker exec -ti <AGENTDEST-container-id> bash -c "cat /tmp/transfer.txt"
    ```
    **Note:** Since the transfer.txt file doesn't exist on **AGENTDEST**, above command may result in an error. This error can be ignored for now.
5. Run the file transfer command to transfer newly created file (**file.txt**) to **AGENTDEST**.
    ```
    docker exec -ti <AGENTSRC-container-id> fteCreateTransfer -p QM1 -sa AGENTSRC -sm QM1 -da AGENTDEST -dm QM1 -df /tmp/transfer.txt /tmp/file.txt
    ```
6. Wait for couple of seconds and check if file is transferred to **AGENTDEST**.
    ```
    docker exec -ti <AGENTDEST-container-id> bash -c "cat /tmp/transfer.txt"
    ```
    **Note:** If the output of above command is *Hello World*, that confirms file transfer is complete and successful.

---

### Create a file transfer using host mounted volume
{: #mft_containers_vol}

Docker provides a mechanism to mount a file system path onto the docker container. Basically, all the files available on the path are accessible via the mounted path in the docker container. This filesystem mount facility is very useful for MFT, as any changes to mounted path can be used to trigger a file transfer.

Below steps guide you to create a host path as mount on to the container and then access a file system of host via the mount to transfer files in it to a destination agent.  

##### Create new direcotories on host computer for mount
{: #mft_containers_vol_setup}

1. Open a new command-shell or you can use any existing to create a sourcefile diretory that can be mounted on source agent container and a destinationfile directory that can be mounted on a destination agent container.
    ```
    On Computer running Source Agent: mkdir -p /home/mqmft-container/sourcefiles
    On Computer running Destination Agent: mkdir -p /home/mqmft-container/destinationfiles
    ```
2. Create a file that can be transferred
    ```
    echo "Hello World" > /home/mqmft-container/sourcefiles/sourcefile.txt
    ```
3. Check the file created
    ```
    cat /home/mqmft-container/sourcefiles/sourcefile.txt
    ```
    - Output of above command should print contents of file(ex: Hellow World)
4. You may now close the command-shell or continue to use it for next section below.
#### Create new source and destination agents:  
1. Open a new command shell or use existing. Run following command to list running containers.
    ```
    docker ps
    ```
    - Make a note of QM1 container-id.
2. Navigate to path of **mqmft-container/agent** repository. For example */home/mqmft-container/agent*(on Linux) or *C:\\mqmft-container\\agent*(on Windows).  

3. We will create two agents as part of this excercise to demonstrate mft agents in container with a host path mounted. These agents are **AGENTSRCVOL** and **AGENTDESTVOL**. As a first step of agent configuration, we have to create their congfiguration on coordination queue manager.
    ```
    docker exec -ti <QM1-container-id> /etc/mqm/mft/mqft_setupAgent.sh AGENTSRCVOL
    docker exec -ti <QM1-container-id> /etc/mqm/mft/mqft_setupAgent.sh AGENTDESTVOL
    ```
    **Note:** 
    1. QM1-Container-id is the id of the queue manager container created in above section.
    2. **mqft_setupAgent.sh** script requires MFT agent name as input parameter
    
5. Run new containers of **AGENTSRCVOL** and **AGENTDESTVOL**  using mft docker image, which are agents in container. 
    ```
    docker run -v /home/mqmft-container/sourcefiles:/home/mft/hostSourceFiles --env MQ_QMGR_NAME=QM1  --env MQ_QMGR_HOST=<docker-host-ip> --env MQ_QMGR_PORT=1414 --env MQ_QMGR_CHL=MFT.SVRCONN --env MFT_AGENT_NAME=AGENTSRCVOL -d --name=AGENTSRCVOL mftagentredist
    
    docker run --v /home/mqmft-container/destinationfiles:/home/mft/containerDestinationFiles --env MQ_QMGR_NAME=QM1  --env MQ_QMGR_HOST=<docker-host-ip> --env MQ_QMGR_PORT=1414 --env MQ_QMGR_CHL=MFT.SVRCONN --env MFT_AGENT_NAME=AGENTDESTVOL -d --name=AGENTDESTVOL mftagentredist
    ```
    **Note:** 
    1. <docker-host-ip>: Is the IP Address of the docker host. This could be found out by running `docker inspect` command and look for IPAddress field.
    2. MQ_QMGR_NAME=QM1: Is the queue manager we created and configured as coordination queue manager in above section.
    3. mftagentredist: Is the docker image of mft redistributable agents.
    
6. Run the below command to list the docker containers, find newly created containers **AGENTSRCVOL** and **AGENTDESTVOL** and make a note of their container-ids.
    ```
    docker ps
    ```
7. Check if the mft agents accept commands and show output. For example, run following command on Agent containers
    ```
    docker exec -ti <AGENTSRCVOL-container-id> fteListAgents
    docker exec -ti <AGENTDESTVOL-container-id> fteListAgents
    ```
    If both the containers show the Agents information, that confirms Agent's running in containers and accepting commands.
8. To verify the Agent setup, use following command to list the container logs
    ```
    docker logs <AGENTSRCVOL-container-id>
    docker logs <AGENTDESTVOL-container-id>
    ```

#### Run a file transfer using mounted volume
{: #mft_containers_vol_demo}

We will now transfer the file **/home/sourcefiles/sourcefile.txt** to the destination agent and verify the if the file exists on destination agent system after file transfer.

1. Open a new command shell or use any existing and run the docker command to list all the containers. Check that containers (**QM1,AGENTSRCVOL and AGENTDESTVOL**) are running.
    ```
    docker ps
    ```  
    
2. Check that file we want to transfer doesn't exist on **AGENTDESTVOL** (on both host path and mounted path)
    ```
    On Host file-system path:
    docker exec -ti <AGENTDESTVOL-container-id> bash -c "cat /home/mqmft-container/destinationfiles/containertransfer.txt"
    
    On Container file-system path:
    docker exec -ti <AGENTDESTVOL-container-id> bash -c "cat /home/mft/containerDestinationFiles/containertransfer.txt"
    ```
    **Note:** Since the containertransfer.txt file doesn't exist on **AGENTDESTVOL**, above command may result in an error. This error can be ignored for now.  
    
5. Run the file transfer command to transfer file (**/home/sourcefiles/sourcefile.txt**) to **AGENTDESTVOL**.
    ```
    docker exec -ti <AGENTSRCVOL-container-id> fteCreateTransfer -p QM1 -sa AGENTSRCVOL -sm QM1 -da AGENTDESTVOL -dm QM1 -df /home/mft/containerDestinationFiles/containertransfer.txt /home/mft/hostSourceFiles/sourcefile.txt
    ```
6. Wait for couple of seconds and check if file is transferred to **AGENTDESTVOL**.
    ```
    docker exec -ti <AGENTDESTVOL-container-id> bash -c "cat /home/mqmft-container/destinationfiles/containertransfer.txt"
    ```
    **Note:** 
    1. If the output of above command is *Hello World*, that confirms file transfer is complete and successful.
    2. In the fteCreateTransfer command, we mentioned -df to mount path on container where-as after the file transfer was complete, we printed file contents from host path of that mount. This demonstrates that mounted volumes from host to container give a powerful mechanism to trigger file transfer using mft monitors.

---
### Conclusion
{: #mft_containers_conclusion}

As part of this document we have created a customized MQ image for MFT, started MFT agents in containers and demonstrated that file transfer runs between the agents in containers and transferred file exists in destination agent.
