# SQLRoboFailover

All in one solution to failover AlwaysOn Availability Groups in SQL Server.

Functions to run comprehensive health checks, failover availability groups, set AG replicas to sync / async modes. Can be combined with the [Update-DbaInstance](https://docs.dbatools.io/#Update-DbaInstance) function for a fully automated patching solution. 

Requires the `SqlServer` module.  
Optional: Install [dbatools](https://dbatools.io/) module for patching functionality.

## Core Functions

This solution is built to be flexible. Functions can be pieced together to build a fully automated solution. Detailed documentation to core functions can be found [here](./docs/CoreFunctions.md). 

## Example Usage

### Patching a SQL Server Instance

#### Import modules

```powershell
Import-Module SqlServer -Force
Import-Module .\src\SQLRoboFailover -Force
Import-Module dbatools -Force
```

#### Prepare primary node ready for restarts
All-in-one function to failover all primary AGs to a sync-commit replica and set all local replicas to async commit.

- Checks general sql health
- Fails over all primary AG's on the server to an available sync-commit replica
- Run post failover checks
- Set all sync-commit AGs on the server to async-commit
- Run health checks to ensure SQL is ready for restarts 

```powershell
Invoke-MakeSQLServerRestartReady -ServerInstance <ServerInstance> -RunPostFailoverChecks -ScriptOnly:$false -Confirm
```


#### Patch server using dbatools module
```powershell
[bool]$IsSQLServerHealthy = Test-IsSQLServerHealthy -ServerInstance <ServerInstance> -Verbose
[bool]$IsRestartReady = Test-IsRestartReady -ServerInstance <ServerInstance> -Verbose

if ($IsSQLServerHealthy -and $IsRestartReady) {
    $Credential = Get-Credential
    Update-DbaInstance -ComputerName $ServerInstance -Version 2017CU20 -Path "\\fileshare.prod\dfs\FileShare\Database\PatchAutomation\UpgradeMedia\SQL2017" -Credential $Credential -Confirm
}

else {
    Write-Output "SQL Server - [$ServerInstance] is NOT ready to patch"
}
```

#### Check server health after patching
```powershell
[bool]$IsSQLServerHealthy = Test-IsSQLServerHealthy -ServerInstance <ServerInstance> -RunExtendedAGChecks -Verbose
```

#### Set all secondary asynchronous_commit AGs back to synchronous_commit
```powershell
if ($IsSQLServerHealthy){
  Set-AllSecondaryAsyncReplicasToSync -ServerInstance <ServerInstance> -ForceSingleSyncCopy -ScriptOnly:$false -Confirm
}
```









### Patching a SQL Server Instance

#### Import modules

```powershell
Import-Module SqlServer -Force
Import-Module .\src\SQLRoboFailover -Force
Import-Module dbatools -Force
```

#### Failover all Primary AGs to an available sync commit replica
Comprehensive health checks will be completed for each AG pre and post failover with a built in health polling mechanism.

```powershell
Invoke-FailoverAllPrimaryAGsOnServer -ServerInstance <ServerName> -RunPostFailoverChecks -ScriptOnly:$false -Confirm
```

#### Set all secondary synchronous_commit AGs to asynchronous_commit
```powershell
Set-AllSecondarySyncReplicasToAsync -ServerInstance <ServerInstance> -MaintainHAForAGs -ScriptOnly:$false -Confirm
```

#### Test if server is ready to be patched or restarted
```powershell
[bool]$IsRestartReady = Test-IsRestartReady -ServerInstance <ServerInstance> -Verbose
```

#### Patch server using dbatools module
```powershell
if ($IsRestartReady){
  Update-DbaInstance -ComputerName <ServerInsance> -Version <PatchVersion> -Path \\network\share
}
```

#### Check server health after patching
```powershell
[bool]$IsSQLServerHealthy = Test-IsSQLServerHealthy -ServerInstance <ServerInstance> -RunExtendedAGChecks -Verbose
```

#### Set all secondary asynchronous_commit AGs back to synchronous_commit
```powershell
if ($IsSQLServerHealthy){
  Set-AllSecondaryAsyncReplicasToSync -ServerInstance <ServerInstance> -ForceSingleSyncCopy -ScriptOnly:$false -Confirm
}
```


