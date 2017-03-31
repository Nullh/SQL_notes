# Installing SQL Server for Windows on Docker for Windows

* Install Docker CE for Windows
* When set up, right click the notification area icon and click 'Switch to Windows containers'
* start a powershell Window
* **docker pull microsoft/mssql-server-windows**
* **docker run -d -p 1433:1433 -e sa_password=\<PASSWORD> -e ACCEPT_EULA=Y microsoft/mssql-server-windows**
* Find IP for server by running **docker network ls** and check each entry in the list using **docker network inspect <NETWORK ID>** until you find your docker container
* Connect to the IP with management studio
* ???
* Profit!

### OR

* **docker run -d -p 1433:1433 --ip \<ip address> -e sa_password=\<PASSWORD> -e ACCEPT_EULA=Y microsoft/mssql-server-windows**
* ...which will start the container with the specified ip

####NOTE: None of these require quotes around paths etc.

# Starting a Docker Container at Interactive Prompt

* **docker run --ip \<ip address> --name \<name> -it <image name> \<cmd or powershell>**
* To make this continue to run in the background so you can attach to it, add the **-d** parm

# Connecting to a Running Container

* **docker exec -it \<container name or id> \<cmd or powershell>**

# Creating a Container With Attached Drives

* **docker run --ip \<ip address> --name \<name> -c \<host folder>:\<container folder> -it -d <image name> \<cmd or powershell>**
