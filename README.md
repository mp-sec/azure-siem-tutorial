# Azure Siem Home Lab Tutorial 

# Forewords 
The goal of this lab is to utilize Azure Virual Machine as a honeypot, and Azure Sentinel, and Azure Log Analytics Workspace to collect log information that will be used to create a heatmap that will display the geolocation of where potential attackers are. This is a way to get SIEM exposure and understanding without paying for software. The end result of this lab will look like this in Azure Sentinel:  
[SCREENSHOT HERE]  
There are some things to know before starting this lab:
- An Azure account and credit card are required. This lab utilizes the $200 worth of free credits an Azure account is given to avoid being charged, and the labs resources will be deleted afterwards to prevent any charges, so you don't have to worry about paying anything. 
- The PowerShell script and the idea for this lab are not mine. The source of this project, and the script file can be found on Josh Madakor's GitHub page (https://github.com/joshmadakor1/Sentinel-Lab). 

# Getting Azure Credits 
When you go to *portal.azure.com*, you will be see an offer that looks like this:  
![image](https://user-images.githubusercontent.com/99374038/179122286-01340e73-1cba-4718-9810-302413d0c030.png)  
The one you will want is the Azure free trial. When you click on that, you will be offered to start the free trial:  
![image](https://user-images.githubusercontent.com/99374038/179122353-3e6203d1-b10f-4fb8-bb0f-3a41f74ee1c7.png)  
From here, you will need to enter your information, and then your credit card information. You will see some messages along the way that will let you know that you are getting the credits, so keep an eye out for those. One such message would be this one:  
![Untitled](https://user-images.githubusercontent.com/99374038/179122601-e1b77525-d845-4825-95a7-f4bfdb9ec136.png)  
Once you have everything set up, the lab can be started. 

# Creating and Configuring the Virtual Machine 
The VM takes time to be fully created, so it's best to start here. In the search bar, search for virtual machine to see it come up:  
![image](https://user-images.githubusercontent.com/99374038/179123610-d757a2f0-fc5d-4725-9504-40b50396f007.png)  
In the top left part of the settings page there is an option to create a virtual machine:  
![Untitled](https://user-images.githubusercontent.com/99374038/179123693-e560078d-ff19-487d-aa93-999f8d18442e.png) 

## Basics
You will now need to give the appropriate settings to the VM, but first you will need to make a resource group. This is a grouping of all of the Azure tools and software under one umbrella. This means that deleting the resource group will delete the VM, the log analytics workspace, Sentinel, and all of their configurations. This will be done at the end to avoid any billing, but that will come in time. You'll need to click to create a new resource group, and then give it a name:  
![Untitled](https://user-images.githubusercontent.com/99374038/179133419-a4c24600-b44c-451a-8255-d7e4401976cd.png)  
Next is to name the VM and specify which regional data centre it will be running from. The name can be whatever you want, but the region should be one that is accessible to you. While doing this lab, I did find that some regions would give me issues. If that occurs, then just choose a similar one. For me, US West 2 and US West 3 worked:  
![image](https://user-images.githubusercontent.com/99374038/179133703-d3a093ad-f507-4923-b1ff-7dd03548606d.png)  
Under the administator account section, give a name and password to the VM that you will remember. When you use Remote Desktop to get access to the VM, you will need these credentials. I suggest putting them in a Notepad file. Before moving on to the disk settings, click the checkbox for the multi-tenant licensing agreement:  
![image](https://user-images.githubusercontent.com/99374038/179134179-f648e625-e58d-426c-ba39-d2f6f53387b9.png)  
In the disk settings, you can click next. There should be no changes required. 

## Networking 
In here we will be altering the firewall settings. We want this VM to be a honeypot, so it has to be vulnerable and exposed to the internet. To do this, we will need to allow any and all inbound connections. To start, we will change the NIC network security group settings to Advanced, and then click to create a new set of configured policies:  
![Untitled](https://user-images.githubusercontent.com/99374038/179134656-b5d785a0-68cd-4ff1-85d5-fe80f72b7afc.png)  
There should be one pre-made inbound polciy already in place. This needs to be removed, which can be done by using the ellipses on the far right side of it:  
![Untitled](https://user-images.githubusercontent.com/99374038/179134965-f92769d0-2871-42e0-8dd7-617ec01ebd97.png)  
With the policy gone, we can make our own inbound rule:  
![image](https://user-images.githubusercontent.com/99374038/179135167-b670ab58-0457-4274-97f1-4b4c3521bf3f.png)  
The intent of the firewall is to let any inbound connection in. The source port will be changed to an asterisk to allow any port, same for the destination port, and the priority will be lowered. You can name the policy whatever you want, and, optionally, give it a description, but this is all that is required to allow all inbound connections:  
![Untitled](https://user-images.githubusercontent.com/99374038/179135291-97529851-977b-416f-b40e-17dbeb208564.png)  
With that, you can click on Ok to set the new policy. No further changes to the VM are required, so you can click the review + create button. A screen with all of the settings will be displayed to you that you can look over to ensure everything is correct. Afterwards, you can click on the create button. This begins the creation process for the VM; however, it will take time. You will see a screen like this for some time:  
![image](https://user-images.githubusercontent.com/99374038/179135789-2e46c54e-051d-42bb-a487-34e131359f33.png)  
Over time all of these resources will be have the status of created, which is when the VM is ready for use:  
![image](https://user-images.githubusercontent.com/99374038/179136081-5b8c0c94-8192-460d-a1a2-c252288ced51.png)  
While the honeypot is being created, we will move on and continue working on other parts of the lab. 

# Creating the Log Analytics Workspace 
The Log Analytics Workspace can be found using the search bar. This resource is used to ingest log data on the VM and inside of Event Viewer on the VM. Later on we will train the Log Analytics Workspace to parse log data correctly for us, which will automate a lot of mundane work moving forward. The log data will ultimately be used by Sentinel later for the heat map. For now, however, we will focus on creating the resource. In the middle of the screen is a button for creating a Log Analytics Workspace. Clicking on that will bring us to a brief settings page. On this page, we will change the resource group to be the one we made prior. Also, a name will be given to the instance, and a region selected. I suggest using the same region as before or one that works:  
![Untitled](https://user-images.githubusercontent.com/99374038/179136735-225f9e26-d896-406c-b45b-fe0abba39a27.png)  
There is nothing more required for this, so click Ok, then the review + create button, then create. 

# Security Center 
Just as before, this resource can be found by searching for it. The goal with it is for the Security Center to gather logs from the virtual machine, and give them to the Log Analytics Workspace for us. Under the Management section, there is Pricing & settings. Clicking that will display the created Log Analytics Workspace, which we also want to click:  
![Untitled](https://user-images.githubusercontent.com/99374038/179137523-07520369-d52d-4690-be7b-f05dcb47cee2.png)  

## Azure Defender plans
We want to enable Azure Defender, and disable the SQL servers on  machines:  
![Untitled](https://user-images.githubusercontent.com/99374038/179137970-8cb30049-7c4d-42cb-976e-17b5694842ef.png)  
The SQL server is not needed, and will eat up the credits, so it's better to just turn it off. With that, the save button on the top left can be clicked, and then the 

## Data Collection 
The Data Collection settings tab can be viewed. In here, we can set Security Center to store data on all security events that happen in the VM:  
![Untitled](https://user-images.githubusercontent.com/99374038/179138779-f9101ff6-3dab-4b9e-8e03-1df78b8c6f7c.png)  
All of the Security Center settings have been completed, so you can save the settings with the save button, and then return to the Log Analytics Workspace. 

# Connecting the Log Analytics Workspace and VM 
Under the Workspace Data Sources setting section, there is one named Virtual machine. Clicking on that will list all VMs that exist. The one we created should be displayed, so click on it:  
![Untitled](https://user-images.githubusercontent.com/99374038/179140574-4801959d-02d1-43f8-abe3-5ecd3e575189.png)  
There is little to connecting the two resources. There is a status indicator that states that the VM is not connected. To remedy this, click the Connect button:  
![Untitled](https://user-images.githubusercontent.com/99374038/179141428-4b547b2a-dd5a-4c14-9a2d-9fb5486b61df.png)  
The status will change to connecting. This will take a bit of time to complete, so it's more efficient to move on while it works in the background:  
![Untitled](https://user-images.githubusercontent.com/99374038/179141645-e66642a8-58fe-423d-9353-fe736fe239e7.png)  

# Azure Sentinel 
Sentinel will be used for visualizing the geolocational data from the Log Analytics Workspace. It can be found by searching for Azure Sentinel in the search bar. In the middle of the page will be a button for creating an Azure Sentinel. In the page is the Log Analytics workspace that was made, which will be clicked. 

# Deleting Resource Groups
If the resource group is not deleted, it will eat up the free credits and you will be charged. To prevent this you can search for resource groups, and click on the one that was made for this lab. 
