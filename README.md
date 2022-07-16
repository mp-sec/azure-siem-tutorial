# Azure Siem Home Lab Tutorial 

# Forewords 
The goal of this lab is to utilize Azure Virtual Machine as a honeypot, and Azure Sentinel, and Azure Log Analytics Workspace to collect log information that will be used to create a heatmap that will display the geolocation of where potential attackers are. This is a way to get SIEM exposure and understanding without paying for software. The end result of this lab will look like this in Azure Sentinel:  
![image](https://user-images.githubusercontent.com/99374038/179326634-e7a2f931-76dd-4fdf-9ac2-3fc2b441d7a3.png)  
This is a rough diagram that shows what will be happening:  
![image](https://user-images.githubusercontent.com/99374038/179328292-76b58a48-d691-4266-b5eb-77793fa50be9.png)  
You will be using the Azure environment to create a cloud-based virtual machine. This machine will run a script that collects failed login attempts, gets the IP from the audit log, feeds that into a website via API key to get location data, and then writes all of this information into a log file. This log file will be given to Log Analytics Workspace to be trained with, to then will automate the data parsing for you. The parsed data will be displayed via Sentinel over a map that shows where the login attempt was made from geographically. The more connection attempts from a location, the greater the indication on the map is for that region. 

There are some things to know before starting this lab:
- An Azure account and credit card are required. This lab utilizes the $200 worth of free credits an Azure account is given to avoid being charged, and the labs resources will be deleted afterwards to prevent any charges, so you don't have to worry about paying anything. 
- The PowerShell script and the idea for this lab are not mine. The source of this project, and the script file can be found on Josh Madakor's GitHub page (https://github.com/joshmadakor1/Sentinel-Lab). 

# Getting Azure Credits 
When you go to *portal.azure.com*, you will be see an offer that looks like this:  
![Untitled](https://user-images.githubusercontent.com/99374038/179326990-c9e4ae50-4cc2-4b72-82d0-dbea8d3c86b5.png)  
The one you will want is the Azure free trial. When you click on that, you will be offered to start the free trial:  
![Untitled](https://user-images.githubusercontent.com/99374038/179327048-f57c142a-c917-48e9-aa4f-b5a0c6b80216.png)   
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
Next is to name the VM and specify which regional data center it will be running from. The name can be whatever you want, but the region should be one that is accessible to you. While doing this lab, I did find that some regions would give me issues. If that occurs, then just choose a similar one. For me, US West 2 and US West 3 worked:  
![image](https://user-images.githubusercontent.com/99374038/179133703-d3a093ad-f507-4923-b1ff-7dd03548606d.png)  
Under the administrator account section, give a name and password to the VM that you will remember. When you use Remote Desktop to get access to the VM, you will need these credentials. I suggest putting them in a Notepad file. Do not use simple passwords, because this VM is a honeypot, you can expect thousands of potential connections that may try and use common passwords from files like RockYou.txt, or brute force, dictionary, or rainbow table attacks. Choose something unique to prevent this. Before moving on to the disk settings, click the checkbox for the multi-tenant licensing agreement to agree to it:  
![image](https://user-images.githubusercontent.com/99374038/179134179-f648e625-e58d-426c-ba39-d2f6f53387b9.png)  
In the disk settings, you can click next. There should be no changes required. 

## Networking 
In here we will be altering the firewall settings. We want this VM to be a honeypot, so it has to be vulnerable and exposed to the internet. To do this, we will need to allow any and all inbound connections. To start, we will change the NIC network security group settings to Advanced, and then click to create a new set of configured policies:  
![Untitled](https://user-images.githubusercontent.com/99374038/179134656-b5d785a0-68cd-4ff1-85d5-fe80f72b7afc.png)  
There should be one pre-made inbound policy already in place. This needs to be removed, which can be done by using the ellipses on the far-right side of it:  
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
The Log Analytics Workspace can be found using the search bar. This resource is used to ingest log data on the VM and inside of Event Viewer on the VM. Later on, we will train the Log Analytics Workspace to parse log data correctly for us, which will automate a lot of mundane work moving forward. The log data will ultimately be used by Sentinel later for the heat map. For now, however, we will focus on creating the resource. In the middle of the screen is a button for creating a Log Analytics Workspace. Clicking on that will bring us to a brief settings page. On this page, we will change the resource group to be the one we made prior. Also, a name will be given to the instance, and a region selected. I suggest using the same region as before or one that works:  
![Untitled](https://user-images.githubusercontent.com/99374038/179136735-225f9e26-d896-406c-b45b-fe0abba39a27.png)  
There is nothing more required for this, so click Ok, then the review + create button, then create. 

# Security Center 
Just as before, this resource can be found by searching for it. The goal with it is for the Security Center to gather logs from the virtual machine, and give them to the Log Analytics Workspace for us. Under the Management section, there is Pricing & settings. Clicking that will display the created Log Analytics Workspace, which we also want to click:  
![Untitled](https://user-images.githubusercontent.com/99374038/179137523-07520369-d52d-4690-be7b-f05dcb47cee2.png)  

## Azure Defender plans
We want to enable Azure Defender, and disable the SQL servers on machines:  
![Untitled](https://user-images.githubusercontent.com/99374038/179137970-8cb30049-7c4d-42cb-976e-17b5694842ef.png)  
The SQL server is not needed, and will eat up the credits, so it's better to just turn it off. With that, the save button on the top left can be clicked.

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
Sentinel will be used for visualizing the geolocational data from the Log Analytics Workspace. It can be found by searching for Azure Sentinel in the search bar. In the middle of the page will be a button for creating an Azure Sentinel. On the following page is the Log Analytics workspace that was made, which will be clicked, and then the add button pressed to start the process of connecting the two together. In the meantime, whilst the connection is taking place, we can get information from the VM on Azure for us to connect to it with. 

# Remotely Logging Into the VM 
Return to the virtual machines' settings page, and click on the VM that was created earlier. In here, you can find a number of property values of the VM, but the one we want is the IP address of the machine:  
![Untitled](https://user-images.githubusercontent.com/99374038/179311470-aec2dbe0-516a-4f4c-9fb3-30b4aadd5254.png)  
With the IP address, you can open up the Remote Desktop Connection application:  
![image](https://user-images.githubusercontent.com/99374038/179311591-95c101a5-b668-48aa-af97-859e1e728fbf.png)  
You can change the display settings so that the resolution is lower than your monitors resolution to stop the VM from taking up all of your screen real estate, but if you don't mind that, then you can enter the IP address of the machine into the Computer field, and press the connect button:  
![Untitled](https://user-images.githubusercontent.com/99374038/179311802-cc34ea1e-e659-46e0-be78-6bb640f83538.png)  
You will get a new sub window that will have blue text saying "More choices," which, when clicked, will allow you to use a different account, which is what we need to connect to the VM. Both the username and password credentials for logging into the VM are what you had set when first creating the virtual machine, so enter those values. I suggest entering the login information incorrectly at least once so you can see the failed login within the Event Viewer application inside of the VM. Event Viewer, and the failed audits for wrong username or bad passwords is what the heat map will be generated from, so this gives you a pre-emptive chance to see what those audit entries will look like. Either way, you will be given a certificate warning that you can agree to. At this point, the remainder of logging into the VM is clicking through settings, and waiting until you are presented the desktop. The settings are optional, you can opt in or not, it won't affect the lab. When you see the desktop, you should note the blue banner at the top of the screen that has the IP address of the VM on it. If you want to ensure you are using your VM and not your host machine, hover your mouse over the top middle of the screen to ensure the banner appears. 

# Finalizing the VM 
Inside of the VM we will be altering the Windows Defender settings, using Event Viewer, and creating a log file, but this will be done by the PowerShell script found in this repository. 

## Disabling the Firewall
We want the VM to be as easily discoverable as possible, so the built-in protection that the VM offers needs to be disabled. You want to search for the Windows Defender Firewall:  
![image](https://user-images.githubusercontent.com/99374038/179314152-8af0b836-61b4-4d7d-a5ac-0f149f90256a.png)  
There is text that can be clicked for viewing the Windows Defender Firewall properties, which we will need:  
![Untitled](https://user-images.githubusercontent.com/99374038/179314252-2bf84b9c-170b-4b05-8b3b-f856b08f2439.png)  
In here, the Firewall State needs to be turned off for the Domain Profile, Private Profile, and Public Profile:  
![Untitled](https://user-images.githubusercontent.com/99374038/179314414-e264fe1e-05fb-4509-9769-bfd2e5fd8227.png)  
To test that the settings have left the VM sufficiently vulnerable, you can ping the VM from your host machine. If the pings are going through, then the VM is made vulnerable:  
![Untitled](https://user-images.githubusercontent.com/99374038/179314792-55ca12d2-9a2c-4db4-b44c-9a1b0dbc9fe1.png)  
The earlier pings failed because the firewall was blocking them, but they began to go through after being disabled. You can use the *-t* option at the end of your ping to keep the ping running perpetually. 

## Reviewing Audit Logs in Event Viewer
Search for the Event Viewer application in Windows:  
![image](https://user-images.githubusercontent.com/99374038/179313011-ae24f618-339b-4620-928c-e60d41016478.png)  
Inside of Event Viewer, you want to click on Windows Logs, and then Security to find the category of logs we want. The ones we are interested in are the ones defined with the ID of 4625:  
![Untitled](https://user-images.githubusercontent.com/99374038/179313141-16283d6a-13e0-492b-b7da-d7d4deb324f2.png)  
These logs are for failed audits that will contain an IP address in them, found below the list of audit logs in the Event Viewer window. These IP addresses can be fed to a site like https://ipgeolocation.io/, which we will be using to get the geolocational data of the IP, such as the longitude, latitude, and country of the connecting IP. These pieces of data will be used for the visualization by Sentinel after being parsed by a trained Log Analytics Workspace. 

## Getting the PowerShell Script on the VM 
You can use Edge to browse to this repository and download the script file. Alternatively, if there is a protection warning that comes up that blocks the download, you can click on the script file to see the code, and copy it. If you copied the code, you can open up PowerShell ISE, and paste the code in there:  
![image](https://user-images.githubusercontent.com/99374038/179315189-afbc77f7-e857-4a27-8e27-19e5cdb9f319.png)  
I suggest saving the pasted code as a *.ps1* file on the desktop so it is always readily available if needed. 

## Getting Your API Key for Geolocation
The API key that is in the file is the one that I had, so you will want your own. To get that, you will navigate to https://ipgeolocation.io/, and sign up for an account. After signing up, you can verify your account via email and log in to see your API key:  
![Untitled](https://user-images.githubusercontent.com/99374038/179315944-473b0b50-53c5-4fbc-9d20-a10b76e051ab.png)  
You should see a value for Consumed API Requests out of 1000, as well. This tracks the total daily number of requests made, which is capped off at 1000. You can get more, but it will cost you some money to do so. It's up to your whether or not that is worth it. Either way, with your own API key copied, you should change the API key value at the top of the PowerShell script to your own, and then save the file. 

## Running the Script
The script will go into Event Viewer and grab the IP address listed for any audit entry that has an ID of 4625. This IP address is then fed into https://ipgeolocation.io/ to get the geolocation of the IP, which is then written into a log file. This log filed is located in C:\ProgramData\, and has the name failed_rdp.log. This means the absolute path of the log is C:\ProgramData\failed_rdp.log. When the script is run, if you had intentionally failed to log in with Remote Desktop Connection, you should see some purple text at the bottom of PowerShell ISE. These lines contain the data that will be found inside of the log file, which are successfully found logs with an ID of 4625. You can check this by leaving the script running in the VM, and then failing to log into the VM again from your host machine. A new entry should appear at the bottom in the terminal. If you navigate to and open the log file itself, you should see more entries than there were at the bottom of PowerShell ISE. All preceding entries in the log are samples created automatically. These samples can be identified by their destination host value being listed as sampledata. You want to keep these because they will be used to train the Log Analytics Workspace to automate the data parsing. To avoid the sample data from affecting the heat map, they will be excluded, which will happen later. I recommend not closing the VM or stopping the script. Both are required for later parts of this lab. 

# Creating Custom Logs
Back on the host machine, you want to navigate back to the Log Analytics Workspace. Inside the created workspace, there is a setting for Custom Logs:  
![Untitled](https://user-images.githubusercontent.com/99374038/179317640-1fd131d1-8c50-4dfd-bdd0-b95733d82d78.png)  
There should be a button in the middle of the screen for adding a custom log, which you will need to click. 

## Sample
You will be prompted to give a sample log file, but the log file exists inside of the virtual machine. To get that data, you will need to go to C:\ProgramData\failed_rdp.log on your VM, copy the contents of the log file, and then paste it into a Notepad file in your host machine with the extension *.log*. This local log file can be given to fed in as the sample log:  
![image](https://user-images.githubusercontent.com/99374038/179317928-a5da88b2-9b3f-4576-9657-2e6a701b6f0e.png)  

## Record Delimiter 
Clicking next will present you with a preview of the log files contents:  
![image](https://user-images.githubusercontent.com/99374038/179317994-e97475c3-c9d0-4fbb-82b0-3e201d93784b.png)  
It should look the same, but it's always good to give things a once over to ensure everything is in working order. Once you're are done reviewing, click next. 

## Collection Paths 
This is where we tell the custom log the location of the log file inside of the VM. We will want to select the file system type, which is Windows, and the path of the log file, which is C:\ProgramData\failed_rdp.log:  
![Untitled](https://user-images.githubusercontent.com/99374038/179318633-b1d512a0-0d24-44a4-a8c2-98825b37692b.png)  
Give the path a look over to make sure it is correct or else the log file will not be located properly, and then click next. 

## Details
You can give the custom log whatever name you want to, and the details are optional. Click next to go to the review + create page, and then, finally, click create. The custom log is created.  

You should see a screen that has the name of your custom log. It will take time for Log Analytics Workspace to sync with the VM are start gathering the log data from the VM. Under the General settings, you can find an option for Logs:  
![Untitled](https://user-images.githubusercontent.com/99374038/179319026-aaefd952-490c-4455-b009-19a13017bf78.png)  
You can run a query here that will be the name of the custom log you gave earlier. It is likely that the return value for the query will be nothing:  
![image](https://user-images.githubusercontent.com/99374038/179319174-b5d9d9b9-7da3-4e84-8ee5-f6a4572586ef.png)  
Instead, you can run the query *SecurityEvent* to see all of the security events that were logged inside of Event Viewer to see if some progress has been made:  
![image](https://user-images.githubusercontent.com/99374038/179319315-d99794c9-2d80-46d2-8cf0-7ab3aa009090.png)  
This confirms that the VM can be seen and looked at by Log Analytics Workspace, but it still needs time to sync up. 

# Training Log Analytics Workspace 
After time has passed, and running a query on the custom log returns all of the VMs log data correctly, we can begin training. Clicking on the right arrow next to the timestamp of a log will expand the details of it. The first thing listed is an ellipsis that gives the option to extract fields from the raw data:  
![image](https://user-images.githubusercontent.com/99374038/179320118-5c14db0d-267d-4fbe-b163-18ffc425616c.png)  
To do the training, you need to highlight the value for field, such as the value that comes after the colon for latitude:  
![image](https://user-images.githubusercontent.com/99374038/179320425-759dd6da-6c25-4ba6-8ad6-5f99e1e82a64.png)  
A sub window will appear where you will name the custom field, and select the data type of the entry:  
![image](https://user-images.githubusercontent.com/99374038/179320510-99fd694c-f3fb-4043-80f0-561a59e9face.png)  
The sample data and real data that was imported to the custom log will now try and use a search to find all of the latitude data for you:  
![image](https://user-images.githubusercontent.com/99374038/179320610-a1b12db7-d0e9-462d-a4af-9b31c0da478c.png)  
In my instance, the data was correctly identified for me in all log entries, so the save extraction button can be pressed; however, clicking to extract data from the same log entry, the search results for longitude for me had incorrect entries:  
![Untitled](https://user-images.githubusercontent.com/99374038/179321016-fbfd0667-0bcc-4e16-aa46-e84046e5b065.png)    
When this happens, they need to be fixed manually. This is done by clicking on the icon to the right of the search result, and selecting to modify the highlight:  
![image](https://user-images.githubusercontent.com/99374038/179321132-e52218f8-4724-446a-85f9-d54d1869a83f.png)  
A new section will be opened on the screen where you can highlight the correct value, and then re-enter the name and data type for it:  
![image](https://user-images.githubusercontent.com/99374038/179321192-6dd53f43-8232-4845-b9f6-1d0f2c92be33.png)  
Clicking the Ok button should fix that entry, and hopefully others, but depending on the number of mismatched fields, you will need to continue checking each entry to ensure the correct data is being parsed for the custom field.

This is the repetitive part, but you will need to go through each field and do this same training process. You can keep using the same log entry over and over for each custom field you make, but you need to ensure the data is correctly parsed or else this will negatively affect Sentinels performance. If you don't want to parse each field, you can, alternatively, only parse the longitude, latitude, country, and label, but I believe it's best to be thorough, even if it isn't always required. For optimal results, go through each field. Once that is done, you have successfully trained Log Analytics Workspace to parse the formatting structure of the log file inside of the VM, that it will automatically read for you. It will take some time for Log Analytics Workspace to fill in the fields for you, so you'll need to wait. Before the custom fields are populated, you can test the accuracy of the training by intentionally failing to log in via Remote Desktop Connection to the VM. This will create a new log entry which can be used to ensure the segments of data are placed under the correct custom field. If something is wrong, you can use the new log entry for further training. If everything is fine, we can move on to setting up Sentinel. 

# Azure Sentinel 
Sentinel can be found with the search bar, and then the workspace that was made can be clicked. In here, under Threat management, you can click on Workbooks, and then add a new one:  
![Untitled](https://user-images.githubusercontent.com/99374038/179322049-edb7b6c9-0921-4f89-86bc-2a6dde070135.png)  
In the top left is an edit button. Using that will let us remove the default widgets that are in there. They can be removed by clicking the ellipses and clicking Remove on all of them:  
![Untitled](https://user-images.githubusercontent.com/99374038/179324809-839b374f-4998-4a2e-ba19-9a18645c0f17.png)  
Now we can add our own query:  
![Untitled](https://user-images.githubusercontent.com/99374038/179324909-c0c1f7a9-eea9-4be7-8df5-51c440f4bce6.png)  
The query to use is: 
> FAILED_RDP_WITH_GEO_CL | summarize event_count=count() by sourcehost_CF, latitude_CF, longitude_CF, country_CF, label_CF, destinationhost_CF  
> | where destinationhost_CF != "samplehost"  
> | where sourcehost_CF != ""  

This query will summarize and get a count of based on the multiple fields, and ignore any sample data, and empty results. For this query, change any of the custom field names or the name of the custom log to match what you had used. Also, I recommend leaving the last WHERE clause in because it filters out any empty entries that may otherwise appear. When the query is returning only good results, you can change the visualization format to map view, and, optionally, change the map size to full for a better visualization experience:  
![Untitled](https://user-images.githubusercontent.com/99374038/179325245-7126e426-591d-474b-9b2d-fb6ec9e9633e.png)  
On the far-right side of the screen, there will be map settings. You can choose to get the displays on the map by changing the location info usage. You can either use the latitude and longitude or the country. Sometimes one works and the other doesn't, I don't know why, but you can try switching to the other one if you have some oddities:  
![Untitled](https://user-images.githubusercontent.com/99374038/179325365-c7a47d03-5280-4798-bcdc-64c9cc0fb6bc.png)  
There is also a metrics section for the map settings where you will want to change the metric label to use the label custom field you made earlier, and the metric value should be the event count:  
![Untitled](https://user-images.githubusercontent.com/99374038/179325866-b1416ce5-03e2-4dca-bf57-3776af630637.png)  
You can now apply the map settings. If you intentionally failed to login, you will see a location marked on the map relative to where the geolocation for the IP is. The API is fairly accurate, so the location on the map should be pretty close. There will also be a country and IP listed below the map. At the very bottom of the map settings is a button named Save and Close. Once you have the map settings right, you can click this button to close the settings. You can now save the settings for the workbook, give the workbook a title, and then select a location:  
![Untitled](https://user-images.githubusercontent.com/99374038/179326022-1ed61de9-5c16-47c8-98c8-c8bc52712bba.png)  
With that, you can click the Done Editing button. I suggest you set the map to auto refresh. This will keep the map relatively up to date without your intervention:  
![Untitled](https://user-images.githubusercontent.com/99374038/179326397-e3d07a01-c130-4396-bdb7-c636da1885c3.png)  
For the refreshing to gather new data, the PowerShell script inside the VM, and the VM itself, need to be running. 

# Heat Map
Over time, new entries will begin appearing on the heat map. This is because the honeypot has been found. For me, it looked like this:  
![image](https://user-images.githubusercontent.com/99374038/179326614-bd8466a9-5e6f-4f66-b222-c50eb0065226.png)  
Over time, the heat map will begin displaying the country of the connecting IPs, and be updated for you. This may be limited by the number of free API requests that you have, but if you run it over a couple of days, the number of instances on the map will grow. 

With that, you have completed the lab. 

# Deleting Resource Groups
Once you are satisfied with the end results, you should delete the resources built to avoid being billed for services. If the resource group is not deleted, it will eat up the free credits, and you will be charged. To prevent this, you can search for resource groups, and click on the one that was made for this lab. In the Overview settings, click on the created resource group. There is a button for deleting the resource group. When pressed, a side panel will come up saying that you need to enter the name of the resource group to delete it:  
![Untitled](https://user-images.githubusercontent.com/99374038/179326879-ce21abfd-ce22-47db-baec-bf153f271d23.png)  
Make sure you do this before the free credits you were given are all used up. 
