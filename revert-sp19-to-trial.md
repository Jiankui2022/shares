# Revert SP19 to trial

> Microsoft strictly prohibits all third-party changes to SharePoint Server databases. During a support incident, database changes that are made under the guidance of a Microsoft SharePoint Server support agent won't cause an unsupported database state. You shouldn't reapply the scripts or changes that are provided by Microsoft SharePoint Server Support outside a support incident.

There are no built-in commands to configure and change the license status for current farm. After reviewing the product code carefully, we have to update the configuration database directly to fix this issue.

Here are the steps to revert the farm to trial status and provide a chance to type the product key. I’ve tested in my lab for these steps.

**Attention**: Please backup the data before any changes.

1. Backup the following registry keys on all SharePoint servers:

    ```
    HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office Server\16.0\ServerSettings
    HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Shared Tools\Web Server Extensions\16.0\WSS\InstalledProducts
    ```

2. Backup the SharePoint Config database.

3. Update the following registry keys on all SharePoint servers:
    
      * HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Shared Tools\Web Server Extensions\16.0\WSS\InstalledProducts
      * Change the value of {90160000-1167-0000-1000-0000000FF1CE} from 8662EA53-55DB-4A44-B72A-5E10822F9AAB to AE423093-CA28-4C04-A566-631C39C5D186

4. Update the SharePoint Config database.

    ```sql
    Update SharePoint_Config.dbo.Objects SET Properties = REPLACE(Properties,'8662ea53-55db-4a44-b72a-5e10822f9aab','AE423093-CA28-4C04-A566-631C39C5D186')
    WHERE Properties like '%Microsoft.SharePoint.Administration.SPInstallState%' 

    Update SharePoint_Config.dbo.Objects SET Properties = REPLACE(Properties,'9223372036854775807','9223372036854700')
    WHERE Properties like '%9223372036854775807%' AND Properties like '%OfficeServerServiceInstance%'
    ```

5. Clean SharePoint cache on all SharePoint servers by the script:

    ```PowerShell
    Add-PSSnapin -Name Microsoft.SharePoint.PowerShell -ErrorAction SilentlyContinue

    Stop-Service SPTimerV4
    Write-Host "Stopped Timer Service."

    $configID = (Get-SPDatabase|?{$_.Type -eq 'Configuration Database'}).id
    $cachefolder = "$Env:ProgramData\Microsoft\SharePoint\Config\$configID"
    $cachefolderitems = Get-ChildItem $cachefolder
    foreach ($cachefolderitem in $cachefolderitems){
        if ($cachefolderitem -like "*.xml")
            {
                Write-Host "Deleted $cachefolderitem"
                $cachefolderitem.Delete()
            }
        }

    Set-Content "1" -Path "$cachefolder\cache.ini"
    Write-Host "Updated cache.ini"
    Read-Host "Run this script on all your SharePoint Servers and THEN press ENTER!"

    Start-Service SPTimerV4
    Write-Host "Started Timer Service"
    Write-Host "Completed on $($Env:COMPUTERNAME)."

    ```

6. **IISREST** on all SharePoint servers.

7. Visit Conversion.aspx to and you should be able to type the product key.
