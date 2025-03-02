---
title: "Configure SharePoint Server to search for archived Skype for Business data"
ms.reviewer: 
ms.author: serdars
author: SerdarSoysal
manager: serdars
ms.date: 12/20/2018
audience: ITPro
ms.topic: quickstart
ms.prod: skype-for-business-itpro
f1.keywords:
- NOCSH
ms.localizationpriority: medium
ms.collection: IT_Skype16
ms.assetid: 17f49365-8778-4962-a41b-f96faf6902f1
description: "Summary: Configure SharePoint Server to search for data archived by Exchange Server and Skype for Business Server."
---

# Configure SharePoint Server to search for archived Skype for Business data
 
**Summary:** Configure SharePoint Server to search for data archived by Exchange Server 2016 or Exchange Server 2013 and Skype for Business Server.
  
One of the major advantages to storing instant messaging and Web conferencing transcripts in Exchange Server instead of Skype for Business Server is that storing data in the same location allows administrators to use a single tool to search for archived Exchange data and/or archived Skype for Business Server data. Because all the data is stored in the same place (Exchange) any tool that can search for archived Exchange data can also search for archived Skype for Business Server data.
  
One tool that makes it easy to search for archived data is Microsoft SharePoint Server 2013. If you would like to use SharePoint to search for Skype for Business Server data, you must first complete all the steps involved in configuring Exchange archiving in Skype for Business Server. After Exchange Server and Skype for Business Server have been successfully integrated, you must then install the Exchange [Web Services Managed API](https://go.microsoft.com/fwlink/p/?LinkId=258305) on your SharePoint Server. The downloaded file (EWSManagedAPI.msi) can be saved to any folder on your SharePoint server.
  
After the file has been downloaded complete the following procedure on the SharePoint server:
  
1. Open a command window by clicking **Start**, clicking **All Programs**, clicking **Accessories**, right-clicking **Command Prompt**, and then clicking **Run as administrator**.
    
2. In the command window, use the cd command to change the current directory to the folder where the file EWSManagedAPI.msi has been saved. For example, if you have saved the file to C:\Downloads type the following command in the command window and then press Enter:
    
   ```console
   cd C:\Downloads
   ```

3. To install the API, type the following command then press Enter:
    
   ```console
   msiexec /I EwsManagedApi.msi addlocal="ExchangeWebServicesApi_Feature,ExchangeWebServicesApi_Gac"
   ```

4. After the API has been installed, reset IIS by typing the following command and pressing Enter:
    
   ```console
   iisreset
   ```

After Exchange Web Services has been installed you must then configure server-to-server authentication between SharePoint Server and Exchange Server. To do this, first open the SharePoint Management Shell and run the following set of commands:
  
```powershell
New-SPTrustedSecurityTokenIssuer -Name "Exchange" -MetadataEndPoint "https://autodiscover.litwareinc.com/autodiscover/metadata/json/1"
$service = Get-SPSecurityTokenServiceConfig
$service.HybridStsSelectionEnabled = $True
$service.AllowMetadataOverHttp = $False
$service.AllowOAuthOverHttp = $False
$service.Update()
```

> [!NOTE]
> Be sure and use the URI for your autodiscover service. Do not use the sample URI https://autodiscover.litwareinc.com/autodiscover/metadata/json/1. 
  
After you have created the token issuer and configured the token service, run these commands, making sure to substitute the URL of your SharePoint site for the sample URL `http://atl-sharepoint-001`:
  
```powershell
$exchange = Get-SPTrustedSecurityTokenIssuer "Exchange"
$app = Get-SPAppPrincipal -Site "https://atl-sharepoint-001" -NameIdentifier $exchange.NameID
$site = Get-SPSite  "https://atl-sharepoint-001"
Set-SPAppPrincipalPermission -AppPrincipal $app -Site $site.RootWeb -Scope "SiteSubscription" -Right "FullControl" -EnableAppOnlyPolicy
```

To configure server-to-server authentication for Exchange Server, open the Exchange Management Shell and run a command similar to this (assuming that Exchange has been installed on drive C: and that it uses the default folder path):
  
```powershell
"C:\Program Files\Microsoft\Exchange Server\V15\Scripts\Configure-EnterprisePartnerApplication.ps1 -AuthMetaDataUrl 'https://atl-sharepoint-001/_layouts/15/metadata/json/1' -ApplicationType SharePoint"
```

After configuring the partner application it is recommended that you stop and restart Internet Information Services (IIS) on all your Exchange mailbox and client access servers. You can restart IIS by using a command similar to this, which restarts the service on the computer atl-exchange-001:
  
```powershell
iisreset atl-exchange-001
```

This command can be run from within the Exchange Management Shell or from any other command window.
  
Next, run a command similar to the following, which gives the specified user (in this example, kenmyer) the right to do discovery on Exchange:
  
```powershell
Add-RoleGroupMember "Discovery Management" -Member "kenmyer"
```

After server-to-server authentication has been established between Exchange and SharePoint, your next step is to create an eDiscovery site in SharePoint. That can be done by running commands similar to these from the SharePoint Management Shell:
  
```powershell
$template = Get-SPWebTemplate | Where-Object {$_.Title -eq "eDiscovery Center"}
New-SPSite -Url "https://atl-sharepoint-001/sites/discovery" -OwnerAlias "kenmyer" -Template $Template -Name "Discovery Center"
```

> [!NOTE]
> "eDiscovery" is short for "electronic discovery," and typically refers to the process of looking through electronic archives for items that can be "reasonably calculated to lead to admissible evidence" in a court of law. 
  
When the new site is ready, the next step is to configure Exchange Server to act as a result source for SharePoint. You can do that by completing the following procedure from the SharePoint Central Administration page:
  
1. On the Central Administration page click **Manage Service Applications** and then click **Search Service Application**.
    
2. On the Search Service Application: Search Administration page click **Result Sources** and then click **New Result Source**.
    
3. In the **New Result Source** pane enter a name for the new result source (for example, **Microsoft Exchange**) in the **Name** box. Select **Exchange** as the result source **Protocol**, and then enter the web services source URL for your Exchange server in the **Exchange Source URL** box. The source URL should look similar to this:
    
    `https://atl-exchange-001.litwareinc.com/ews/exchange.asmx`
    
4. Make sure that **Use Autodiscover** is not selected, and then click **OK**.
    
Finally, create a new eDiscovery case and a new eDiscovery set by completing the following procedure from the SharePoint Discovery site (for example, `https://atl-sharepoint-001/sites/discovery`):
  
1. On the Site Contents page click **Create a new case**.
    
2. On the Site Contents: New SharePoint Site page, enter the user's email alias (for example, **kenmyer**) in the **Title** box, then add that same URL to the **Web Site Address** box. That will result in a URL similar to this:
    
    `https://atl-sharepoint-001/sites/eDiscovery/kenmyer`
    
3. Click **Create**.
    
4. When the eDiscovery set page appears, click **new item** under **Identity and Preserve: Discovery Sets**.
    
5. On the New: Discovery Set page, enter the user's email alias in the **Discovery Set Name** box. Enter **eDiscovery Lync\\*** in the **Filter** box and then click **Add &amp; Manage Sources**.
    
6. On the Add &amp; Manage Sources page, enter the user's email alias in the first textbox under **Mailboxes**. Click the check mailbox icon located next to the textbook to verify that SharePoint can connect to the specified mailbox.
    
7. Click **OK**.
    
8. On the eDiscovery set page, click **Save** to save the new eDiscovery set.
    
At this point you can search the specified mailbox (kenmyer) and/or enable In-Place holds the same way you would for any other SharePoint content or result source.
  

