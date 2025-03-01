---
title: "Configure recommendations and usage event types in SharePoint Server"
ms.reviewer: 
ms.author: serdars
author: SerdarSoysal
manager: serdars
ms.date: 9/11/2017
audience: ITPro
f1.keywords:
- NOCSH
ms.topic: article
ms.prod: sharepoint-server-itpro
ms.localizationpriority: medium
ms.collection: IT_Sharepoint_Server_Top
ms.assetid: 0830f8f5-4432-4a75-9974-0bb77268f2f1
description: "Usage events enable you to track how users interact with items on your site. Items can be documents, sites, or catalog items. When a user interacts with an item on your site, SharePoint Server generates a usage event for this action. For example, if you want to monitor how often a catalog item is viewed from a mobile phone, you can track this activity."
---

# Configure recommendations and usage event types in SharePoint Server

[!INCLUDE[appliesto-2013-2016-2019-xxx-md](../includes/appliesto-2013-2016-2019-xxx-md.md)]

Usage events enable you to track how users interact with items on your site. Items can be documents, sites, or catalog items. When a user interacts with an item on your site, SharePoint Server generates a usage event for this action. For example, if you want to monitor how often a catalog item is viewed from a mobile phone, you can track this activity. 
  
This article describes how to create custom usage event types, and how to add code to record custom usage events so that they can be processed by the analytics processing component.
  
 You can use the data that is generated by usage events to show recommendations or popular items on your site. This article also explains how to influence how recommendations are shown by changing the level of importance for a specific usage event type. For more information, see "Plan usage analytics, usage events and recommendations" in [Plan search for cross-site publishing sites in SharePoint Server 2016](plan-search-for-sharepoint-cross-site-publishing-sites.md).
  
You can view the statistics for all usage event types in Popularity Trends and Most Popular Items reports. For more information, see [View usage reports in SharePoint Server](view-usage-reports.md).
  
    
## Create a custom usage event type
<a name="BKMK_CreateCustomUsageEventType"> </a>

There are three default usage event types in SharePoint Server. You can create up to twelve custom usage event types by using Microsoft PowerShell.
  
 **To create a custom usage event type**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To get a site at the root site collection level:
  $Site = Get-SPSite "http://localhost"
  # To get a site below the root site collection level:
  $Site = Get-SPSite "http://localhost/sites/<SiteName>"
  # To create a custom usage event type:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $EventGuid = [Guid]::NewGuid()
  $EventName = "<EventTypeName>"
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $newEventType = $tenantConfig.RegisterEventType($EventGuid, $EventName, "")
  $tenantConfig.Update($SSP)
  
  ```

   Where:
    
  -  \<SiteName\> is the name of the site for which you want to create a custom usage event. 
    
  - \<EventTypeName\> is the name of the new custom usage event type that you want to create — for example,  *BuyEventType*  . 
    
    This procedure creates a random GUID for the usage event type. Use this GUID when you add code to record the custom usage event, as described in [Record a custom usage event](configure-recommendations-and-usage-event-types.md#BKMK_RecordCustomUsageEvent).
    
    > [!IMPORTANT]
    > It can take up to three hours for a custom usage event type to become available in the system. However, to speed up the process, you can alternatively restart the SharePoint Timer Service. 
  
> [!NOTE]
> We recommend that you use Microsoft PowerShell when performing command-line administrative tasks. The Stsadm command-line tool has been deprecated, but is included to support compatibility with previous product versions. 
  
## Record a custom usage event
<a name="BKMK_RecordCustomUsageEvent"> </a>

After you have created a custom usage event type, as described in [Create a custom usage event type](configure-recommendations-and-usage-event-types.md#BKMK_CreateCustomUsageEventType), you have to add code to the place where the event occurs — for example, when a page loads, or when a user clicks a link or a button. This data is then sent to the analytics processing component, where it is recorded and processed.
  
If you are using cross-site publishing, where you show catalog content on a publishing site, you must record the usage event on the URL of the indexed item, and override some site settings. For example, if you have a catalog in an authoring site that you have published on a publishing site, when a user interacts with a catalog item on the publishing site, this usage event must be recorded on the item in the authoring site. Furthermore, the code that you add to record the usage event must override the SiteId and the WebId of the publishing site, and be replaced with the SiteId and the WebId of the authoring site.
  
 **To add code to record a custom usage event**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view GUIDs for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  
  ```

4. In an HTML editor, open the file where the custom usage event should be logged — for example, a display template for a Content Search Web Part, and add the following code:
    
  ```
  window.Log<CustomUsageEventType>ToEventStore = function(url)
  {
      ExecuteOrDelayUntilScriptLoaded(function()
      {
          var spClientContext = SP.ClientContext.get_current();
          var eventGuid = new SP.Guid("<GUID>");
          SP.Analytics.AnalyticsUsageEntry.logAnalyticsAppEvent(spClientContext, eventGuid, url);
          spClientContext.executeQueryAsync(null, Function.createDelegate(this, function(sender, e){ alert("Failed to log event for item: " + document.URL + " due to: " + e.get_message()) }));
      }, "SP.js");
  }
  ```

  - <CustomUsageEventType> is the name of the custom event — for example,  *BuyEventType*  . 
    
  - <GUID> is the numeric ID of the usage event type — for example,  *4e605543-63cf-4b5f-aab6-99a10b8fb257*. 
    
5. In an HTML editor, open the file that refers to the custom usage event, and add the following code:
    
  ```
  # The example below shows how a custom usage event type is referred to when a button is clicked: 
  <button onclick="Log<CustomUsageEventType>ToEventStore('<URL>')"></button>
  
  ```

   Where:
    
  - \<CustomUsageEventType\> is the name of the custom event type.
    
  - \<URL\> is the full URL of the item to which the usage event should be logged — for example,  *http://contoso.com/faq*  . 
    
 **To add code to record a custom usage event and override site settings**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view GUIDs for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  
  ```

4. In an HTML editor, open the file where the custom usage event should be logged — for example, a display template for a Content Search Web Part. The following example shows how to override the current SiteId, WebId and UserId.
    
  ```
  window.Log<CustomUsageEventType>ToEventStore = function(url, siteIdGuid, webIdGuid, spUser)
  {
      ExecuteOrDelayUntilScriptLoaded(function()
      {
        var spClientContext = SP.ClientContext.get_current();
        var eventGuid = new SP.Guid("<GUID>");
  SP.Analytics.AnalyticsUsageEntry.logAnalyticsAppEvent2(spClientContext, eventGuid, url, webIdGuid, siteIdGuid, spUser);
        spClientContext.executeQueryAsync(null, Function.createDelegate(this, function(sender, e){ alert("Failed to log event for item: " + document.URL + " due to: " + e.get_message()) }));
      }, "SP.js");
  }
  
  ```

   Where:
    
  - \<CustomUsageEventType\> is the name of the custom event type — for example,  *BuyEventType*  . 
    
  - \<GUID\> is the numeric ID of the usage event type — for example, *4e605543-63cf-4b5f-aab6-99a10b8fb257*  . 
    
5. In an HTML editor, open the file that refers to the custom usage event type, and add the following code:
    
  ```
  # The example below shows how a custom usage event type is referred to when the "Buy!" button is clicked:
  <button onclick="Log<CustomUsageEventType>ToEventStore('<URL>', new SP.Guid('{<SiteId GUID>}'), new SP.Guid('{<WebId GUID>}'), '<UserName>')">Buy!</button>
  
  ```

   Where:
    
  - \<CustomUsageEventType\> is the name of the custom event type — for example, BuyEventType.
    
  - \<URL\> is the URL found in the managed property OriginalPath.
    
  - \<SiteId GUID\> is the SiteId GUID of the authoring site. For information about how to get the SiteId GUID, see [Get SiteId GUID and WebId GUID for a site](configure-recommendations-and-usage-event-types.md#BKMK_GetGUID).
    
  - \<WebId GUID\> is the WebId GUID of the authoring site. For information about how to get the WebId GUID, see [Get SiteId GUID and WebId GUID for a site](configure-recommendations-and-usage-event-types.md#BKMK_GetGUID).
    
  - \<UserName\> can be a cookie ID that is used to identify users on a site that has anonymous users.
    
## Record a default usage event
<a name="BKMK_RecordDefaultUsageEvent"> </a>

If you want to add code that refers to a default usage event type — for example, views, you have to add code to the place where the event occurs.
  
If you are using cross-site publishing, which shows catalog content on a publishing site, you must record the usage event on the URL of the indexed item, and override some site settings. For example, if you have a catalog in an authoring site that you have published on a publishing site, when a user interacts with a catalog item on the publishing site, this usage event must be recorded on the item in the authoring site. Furthermore, the code that you add to record the usage event must override the SiteId and WebId of the publishing site, and be replaced with the SiteId and WebId of the authoring site.
  
 **To add code to record a default usage event**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view EventTypeId for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  
  ```

4. In an HTML editor, open the file where the custom usage event should be logged — for example, a display template for a Content Search Web Part, and add the following code:
    
  ```
  window.Log<DefaultUsageEventType>ToEventStore = function(url)
  {
      ExecuteOrDelayUntilScriptLoaded(function()
      {
          var spClientContext = SP.ClientContext.get_current();
          SP.Analytics.AnalyticsUsageEntry.logAnalyticsEvent(spClientContext, <EventTypeId>, url);
          spClientContext.executeQueryAsync(null, Function.createDelegate(this, function(sender, e){ alert("Failed to log event for item: " + document.URL + " due to: " + e.get_message()) }));
      }, "SP.js");
  }
  
  ```

   Where:
    
  - \<DefaultUsageEventType\> is the name of the default usage event type — for example, *Views*. 
    
  - \<EventTypeId\> is the numeric ID of the usage event type — for example, *1*. 
    
5. In an HTML editor, open the file that refers to the default usage event, and add the following code:
    
  ```
  # The example below shows how a default usage event type is referred to on a page load:
  <body onload="Log<DefaultUsageEventType>ToEventStore('<URL>')"> 
  ```

   Where:
    
  - \<DefaultUsageEventType\> is the name of the default usage event type — for example,  *Views*  . 
    
  - \<URL\> is the full URL of the item to which the usage event should be logged, — for example,  *http://contoso.com/careers* 
    
6. Save the file.
    
 **To add code to record a default usage event and override site settings**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view EventTypeId for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  
  ```

4. In an HTML editor, open the file where the custom usage event should be logged — for example, a display template for a Content Search Web Part. The example below shows how to override the current SiteId, the WebId and the UserId.
    
  ```
  window.Log<DefaultUsageEventType>ToEventStore = function(url, siteIdGuid, webIdGuid, spUser)
  {
      ExecuteOrDelayUntilScriptLoaded(function()
      {
        var spClientContext = SP.ClientContext.get_current();
        SP.Analytics.AnalyticsUsageEntry.logAnalyticsEvent(spClientContext, <EventTypeId>, url, webIdGuid, siteIdGuid, spUser);
  spClientContext.executeQueryAsync(null, Function.createDelegate(this, function(sender, e){ alert("Failed to log event for item: " + document.URL + " due to: " + e.get_message()) }));
      }, "SP.js");
  }
  
  ```

   Where:
    
  - \<DefaultUsageEventType\> is the name of the default event type — for example,  *Views*  . 
    
  - \<EventTypeId\> is the numeric ID of the usage event type — for example,  *1*  . 
    
5. In an HTML editor, open the file that refers to the default usage event type, and add the following code:
    
  ```
  # The example below shows how a default usage event type is referred to on a page load:
  <body onload="Log<DefaultUsageEventType>ToEventStore('<URL>', new SP.Guid('{<SiteId GUID>}'), new SP.Guid('{<WebId GUID>}'), '<UserName>')">
  
  ```

   Where:
    
  - \<DefaultUsageEventType\> is the name of the default event type — for example,  *Views*  . 
    
  - \<URL\> is the URL in the managed property  *OriginalPath*  . 
    
  - \<SiteId GUID\> is the SiteId GUID of the authoring site. For information on how to get the SiteId GUID, see [Get SiteId GUID and WebId GUID for a site](configure-recommendations-and-usage-event-types.md#BKMK_GetGUID).
    
  - \<WebId GUID\> is the WebId GUID of the authoring site. For information on how to get the WebId GUID, see [Get SiteId GUID and WebId GUID for a site](configure-recommendations-and-usage-event-types.md#BKMK_GetGUID).
    
  - \<UserName\> can be a cookie ID that is used to identify users on a site that has anonymous users, for example.
    
> [!NOTE]
> We recommend that you use Microsoft PowerShell when performing command-line administrative tasks. The Stsadm command-line tool has been deprecated, but is included to support compatibility with previous product versions. 
  
## Change the level of importance of a usage event type
<a name="BKMK_ChangeImportanceLevel"> </a>

The usage event type property, **RecommendationWeight**, is a numeric value that shows the level of importance of a usage event type compared to other usage event types that are used in the recommendations calculation. The default **Views** usage event type has a preconfigured **RecommendationWeight** value of 1. The other default usage event types, **Recommendations displayed**, and **Recommendations clicked**, and all custom usage event types, have a **RecommendationWeight** value of 0. To increase the importance of a usage event type in the recommendations calculation, change the value of the **RecommendationWeight** parameter. The highest level of importance available is 10. 
  
 **To change the level of importance of a usage event type**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view EventTypeId for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  # To get a usage event type:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  # To change the importance level of a usage event type:
  $event.RecommendationWeight = <RecommendationWeightNumber>
  $tenantConfig.Update($SSP)
  # To verify the changed importance level for the usage event type:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  $event
  
  ```

   Where:
    
  - \<EventTypeId\> is the numeric ID of the usage event type for which you want to change the weight — for example, *256*. 
 
  - \<RecommendationWeightNumber\> is the level of importance that you want to apply to the user event type — for example, *4*. 
    
> [!NOTE]
> We recommend that you use Microsoft PowerShell when performing command-line administrative tasks. The Stsadm command-line tool has been deprecated, but is included to support compatibility with previous product versions. 
  
## Change the Recent time period for a usage event type
<a name="BKMK_ChangeRecentTimePeriod"> </a>

The usage event type property **RecentPopularityTimeframe** is a numeric value that defines the **Recent** time period in the **Most Popular Items** report. The Most Popular Items report shows the most popular items per usage event type for all items in a library or list — for example, the most viewed items in a library or list. The report can be sorted by the time periods **Recent** or **Ever**. By default, the Recent time period is set to the last 14 days for each usage event. You can change this to a time period between one and 14 days.
  
 **To change the Recent time period for a usage event type**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view EventTypeId for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  # To get a usage event type:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  # To change the Recent time span for a usage event type:
  $event.RecentPopularityTimeFrame = <TimeFrame>
  $tenantConfig.Update($SSP)
  # To verify the changed Recent time frame for the usage event type:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  $event
  
  ```

   Where:
    
  - \<EventTypeId\> is the numeric ID of the usage event type for which you want to change the **Recent** time frame — for example,  *256*  . 
    
  - \<TimeFrame\> is the new **Recent** time frame that you want to apply to the user event type — for example, *7*. 
    
    > [!NOTE]
    > The system updates any changes to the Recent time period only after the Usage Analytics Timer Job has run. 
  
> [!NOTE]
> We recommend that you use Microsoft PowerShell when performing command-line administrative tasks. The Stsadm command-line tool has been deprecated, but is included to support compatibility with previous product versions. 
  
## Enable and disable the logging of usage events of anonymous users
<a name="BKMK_LoggingAnonymousUsers"> </a>

Users that are browsing the contents of a site without being connected to an account are known as anonymous users. Only the  *Views*  event type is enabled for the logging of anonymous users. By default, the logging of custom usage events is disabled for anonymous users. 
  
 **To enable the logging of usage events of anonymous users**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view EventTypeId for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  # To get a usage event type:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  # To enable the logging of anonymous users:
  $event.Options = [Microsoft.Office.Server.Search.Analytics.EventOptions]::AllowAnonymousWrite
  $tenantConfig.Update($SSP)
  # To verify that the logging of anonymous users has been enabled, i.e. that the Options property is set to AllowAnonymousWrite:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  $event
  
  ```

   Where:
    
  - \<EventTypeId\> is the numeric ID of the usage event type that you want to enable for the logging of anonymous users — for example, *256*. 
    
 **To disable the logging of usage events of anonymous users**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To view EventTypeId for all usage event types:
  $SSP = Get-SPEnterpriseSearchServiceApplicationProxy
  $SSP.GetAnalyticsEventTypeDefinitions([Guid]::Empty, 3) | ft
  # To get a usage event type:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  $event = $tenantConfig.EventTypeDefinitions | where-object { $_.EventTypeId -eq <EventTypeId> }
  # To disable the logging of anonymous users:
  $event.Options = [Microsoft.Office.Server.Search.Analytics.EventOptions]::None
  $tenantConfig.Update($SSP)
  # To verify that logging of anonymous users has been disabled, i.e. that the Options property is set to None:
  $tenantConfig = $SSP.GetAnalyticsTenantConfiguration([Guid]::Empty)
  
  ```

   Where:
    
  - \<EventTypeId\> is the numeric ID of the usage event type that you want to disable for the logging of anonymous users — for example, *256*. 
    
    > [!NOTE]
    > For the default usage event type  *Views*  , you cannot disable the logging of anonymous users. 
  
> [!NOTE]
> We recommend that you use Microsoft PowerShell when performing command-line administrative tasks. The Stsadm command-line tool has been deprecated, but is included to support compatibility with previous product versions. 
  
## Get SiteId GUID and WebId GUID for a site
<a name="BKMK_GetGUID"> </a>

You can use the following PowerShell commands to get the SiteId GUID and the WebId GUID for a site.
  
 **To get SiteId GUID and WebId GUID for a site**
  
1. Verify that you have the following memberships:
    
  - **securityadmin** fixed server role on the SQL Server instance. 
    
  - **db_owner** fixed database role on all databases that are to be updated. 
    
  - Administrators group on the server on which you are running the PowerShell cmdlets.
    
  - Add memberships that are required beyond the minimums above.
    
    An administrator can use the **Add-SPShellAdmin** cmdlet to grant permissions to use SharePoint Server cmdlets. 
    
    > [!NOTE]
    > If you do not have permissions, contact your Setup administrator or SQL Server administrator to request permissions. For additional information about PowerShell permissions, see [Add-SPShellAdmin](/powershell/module/sharepoint-server/Add-SPShellAdmin?view=sharepoint-ps). 
  
2. Start the SharePoint Management Shell.
    
3. At the PowerShell command prompt, type the following command:
    
  ```
  # To get the SiteId GUID and the WebId GUID for a root site collection:
  $site = Get-SPSite "<RootSiteURL>"
  $web = $site.RootWeb
  $site.id
  $web.id
  # To get the WebId GUID for a site below the root site collection:
  $site = Get-SPSite "<RootSiteURL>"
  $web = $site.OpenWeb("<SubSiteLocation>")
  $web.id
  
  ```

   Where:
    
  -  _\<RootSiteURL\>_ is the URL of the root site that you want to get the SiteId GUID and the WebId GUID of — for example,  _http://contoso.com/sites/catalog_.
    
  -  _\<SubSiteLocation\>_ is the remainder of the URL path to the subsite after the root site URL. For example, if your root site URL is  _http://contoso.com/sites/catalog_, and your subsite URL is  _http://contoso.com/sites/catalog/products_, type  _products_ for this placeholder. 
    
> [!NOTE]
> We recommend that you use Microsoft PowerShell when performing command-line administrative tasks. The Stsadm command-line tool has been deprecated, but is included to support compatibility with previous product versions. 
  
## See also
<a name="BKMK_GetGUID"> </a>

#### Concepts

[View usage reports in SharePoint Server](view-usage-reports.md)
#### Other Resources

[How to display recommendations and popular items on a SharePoint Server 2013 site](/archive/blogs/tothesharepoint/how-to-display-recommendations-and-popular-items-on-a-sharepoint-server-2013-site)