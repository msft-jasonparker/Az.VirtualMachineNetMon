# Microsoft Network Monitor for Azure Virtual Machines

## Overview

Deploying the 'in-guest' application, Microsoft Network Monitor, is not top of mind when deploying virtual machines into Azure.  However, there are many situation, especially for support cases where having Microsoft Network Monitor installed is imperative to the troubleshooting process.  This repository will provide all the necessary components to install (deploy) and remotely configure Microsoft Network Monitor 3.4 for any number of virtual machines running in Azure.

## Prerequisites

- Contributor RBAC rights to the Resource Group or Subscription in Azure
- Administrative rights on the target virtual machines
- PowerShell Modules Installed
  - Az
  - xPSDesiredStateConfiguration
- Ability to open PowerShell as Administrator
- Azure Storage Account (DSC Archive)
- SMB File share (XML and EXE storage) *Optional*

## Install / Deploy the application

1. Review the WindowsNetMon.ps1 Desired State Configuration (DSC) script and update the *SourcePath* with the path where the Microsoft Network Monitor 3.4 executable is located

    > Note: Be sure to download Microsoft Network Monitor 3.4 from [here](https://www.microsoft.com/en-us/download/details.aspx?id=4865)

    ````PowerShell
    File DownloadPackage
    {
        Ensure = "Present"
        SourcePath = "\\###SOURCE-PATH###\NM34_x64.exe"
        DestinationPath = "C:\Temp\packages\NM34_x64.exe"
        Type = "File"
        Force = $true
        DependsOn = "[File]CreatePackageDirectory"
    }
    ````

2. Create the DSC Archive zip file

    ````PowerShell
    Publish-AzVMDscConfiguration .\WindowsNetMon.ps1 -OutputArchivePath .\WindowsNetMon.zip -Force
    ````

3. Upload the DSC archive zip file to an Azure storage account and create a Shared Access Signature (SAS) uri.

    > Note: If not created, create a storage account and a subsequent container.

    - Navigate to the storage account and container, then click 'Upload'

        ![upload](_media/sa_upload.png)

    - After the upload, select the file and then select 'Generate SAS'

        ![blah](_media/sa_zip_sas.png)

    - Provide an expiration date for the SAS key, then click 'Generate SAS token and URL'

        ![img](_media/sa_zip_generate.png)

    - Once generated, click the 'copy' icon to copy the URL (save it for your records)

        ![img](_media/sa_zip_generate2.png)

4. Open PowerShell as Administrator and navigate to the directory where this repository was cloned or extracted and run the deployment script

    ````PowerShell
    .\New-AzVMNetMonDSCDeployment.ps1 -ResourceGroupName RG1 -DscConfigUri "https://mylong-storage-account-sas-url" -TemplateFilePath "D:\_Scripts\GitHubRepos\Az.VirtualMachineNetMon" -Verbose
    ````

    > Note: Using -Verbose allows the script to display the ongoing status of the deployment job

5. The script will do some verification steps then prompt to select the Azure subscription (should be where the Resource Group is). Select the appropriate subscription and the script will continue.

6. The script will get a list of virtual machines from the Resource Group Name and will call the `New-AzResourceGroupDeployment` cmdlet and attempt to deploy the WindowsNetMon.json template to the VMs found in the Resource Group.

7. Once complete, review the deployment from PowerShell or the Azure portal (Resource Group -> Deployment blade)

## Configure Network Monitor Scheduled Task

This portion of the solution creates a scheduled task on each of the virtual machines where Network Monitor was installed. This will create a direction on each virtual machine (*C:\Windows\Utilities\NetMon*) where the XML for the task is copied and it is also the location of the capture files. The scheduled task XML is configured to begin a capture at user logon and will not create another instance if it is already running.  The capture is configured to create 250MB files.

> Note: It may be necessary to purge the capture files from the directory, which is not currently part of this solution

1. From the previous PowerShell console execute the configuration script

    ````PowerShell
    .\New-AzVMNetMonScheduledTask.ps1 -ResourceGroupName RG1 -LocalPath "\\server\share" -domainSuffix "contoso.com"
    ````

    > Note: the -LocalPath parameter is the location of the XML configuration file

2. Similarly to the installation script, it will prompt the user to select their Azure subscription.

3. The virtual machines will be collected based on the Resource Group Name (checks the Powerstate for '*vm running*')

4. The script will check for the remote directory structure, if not found, it will be created.

5. The XML file is copied to the virtual machine

6. The script will use PS Remoting and invoke the SchTasks.exe command on the remote computer and create the Scheduled Task

7. Log on to the VM and check the Scheduled Tasks under \Microsoft\Windows\NetTrace