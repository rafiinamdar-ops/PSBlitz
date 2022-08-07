# PSBlitz
<a name="header1"></a>
## Navigation
- [Intro](#Intro)
- [What it does](#What-it-does)
- [Prerequisites](#Prerequisites)
- [Installation](#Installation)
- [What it runs](#What-it-runs)
- [Default check VS in-depth check](#Default-check-VS-in-depth-check)
- [Output files](#Output-files)
- [Usage examples](#Usage-examples)
- [License](/LICENSE)

## Intro

Since I'm a big fan of [Brent Ozar's](https://www.brentozar.com/) [SQL Server First Responder Kit](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit) and I've found myself in many situations where I would have liked a quick way to easily export the output of sp_Blitz, sp_BlitzCache, sp_BlitzFirst, sp_BlitzIndex, sp_BlitzLock, and sp_BlitzWho to Excel, as well as saving to disk the execution plans identified by sp_BlitzCache and deadlock graphs from sp_BlitzLock, I've decided to put together a PowerShell script that does just that.

## What it does

Outputs relevant diagnostics data about your instance and database(s) to an Excel file, as well as writing execution plans and deadlock graphs to disk.

## Prerequisites
1. In order to be able to run PowerShell scripts, you'll need to run this (if you haven't already changed the execution policy)
    ```PowerShell
    Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
    ```
2. Excel needs to be installed for the output to Excel to actually work.
3. Sufficient permissions to query DMVs, server state, and get database objects' definitions.

This should be ran from your workstation and not from the instance's host itself (why would you have the MS Word suite installed on a database server anyway?)

You don't need to have any of the sp_Blitz stored procedures present on the instance that you're executing PSBlitz.ps1 for, all the scripts are contained in the `PSBlitz\Resources` directory in non-stored procedure format.

Limitations:
- For the time being PSBlitz.ps1 can only run against SQL Server instances, not Azure SQL DB.

[*Back to top*](#header1)

## Installation

Download the latest zip file from [Releases](https://github.com/VladDBA/PSBlitz/releases) section of the repository and extract its contents. 

Do not change the directory structure and file names.

[*Back to top*](#header1)

## What it runs
PSBlitz.ps1 uses slightly modified, non-stored procedure versions, of the following components 
from [Brent Ozar's](https://www.brentozar.com/) [SQL Server First Responder Kit](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit):
- sp_Blitz
- sp_BlitzCache
- sp_BlitzFirst
- sp_BlitzIndex
- sp_BlitzLock
- sp_BlitzWho

[*Back to top*](#header1)

## Paramaters
| Parameter | Description|
|-----------|------------|
|`-ServerName`| Accepts either `HostName\InstanceID` (for named instances) or just `HostName` for default instances. If you provide either `?` or `Help` as a value for `-ServerName`, the script will return a brief help menu. | your SQL Server instance, `?`, `Help` |
|`-SQLLogin`| The name of the SQL login used to run the script. If not provided, the script will use integrated security. | the name of your SQL Login, empty | empty|
|`-SQLPass` | The password for the SQL login provided via the -SQLLogin parameter, omit if `-SQLLogin` was not used. |
|`-IsIndepth` | Providing Y as a value will tell PSBlitz.ps1 to run a more in-depth check against the instance/database. Omit for default check. |
|`-CheckDB` | Used to provide the name of a specific database against which sp_BlitzIndex, sp_BlitzCache, and sp_BlitzLock will be ran. Omit to run against the whole instance.|

[*Back to top*](#header1)

## Default check VS in-depth check

- The default check will run the following:
```SQL
sp_Blitz @CheckServerInfo = 1
sp_BlitzFirst @ExpertMode = 1, @Seconds = 30
sp_BlitzIndex @GetAllDatabases = 1, @Mode = 0
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'CPU'/'avg cpu'	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'duration'/'avg duration'
sp_BlitzWho @ExpertMode = 1
sp_BlitzLock @StartDate = DATEADD(DAY,-30, GETDATE()), @EndDate = GETDATE()
```

- The in-depth check will run the following:
```SQL
sp_Blitz @CheckServerInfo = 1, @CheckUserDatabaseObjects = 1	
sp_BlitzFirst @ExpertMode = 1, @Seconds = 30	
sp_BlitzFirst @SinceStartup = 1
sp_BlitzIndex @GetAllDatabases = 1, @Mode = 0	
sp_BlitzIndex @GetAllDatabases = 1, @Mode = 1	
sp_BlitzIndex @GetAllDatabases = 1, @Mode = 2	
sp_BlitzIndex @GetAllDatabases = 1, @Mode = 4	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'CPU'/'avg cpu'	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'reads'/'avg reads'	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'writes'/'avg writes'
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'duration'/'avg duration'	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'executions'/'xpm'	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'memory grant'	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'recent compilations', @Top = 50	
sp_BlitzCache @ExpertMode = 1, @SortOrder = 'spills'/'avg spills'	
sp_BlitzWho @ExpertMode = 1	
sp_BlitzLock @StartDate = DATEADD(DAY,-30, GETDATE()), @EndDate = GETDATE()
```

- Using `-CheckDB SomeDB` will modify the executions of sp_BlitzCache, sp_BlitzIndex, and sp_BlitzLoc as follows:
```SQL
sp_BlitzIndex @GetAllDatabases = 0, @DatabaseName = 'SomeDB', @Mode = ...
sp_BlitzCache @ExpertMode = 1, @DatabaseName = 'SomeDB', @SortOrder = ...
sp_BlitzLock @StartDate = DATEADD(DAY,-30, GETDATE()), @EndDate = GETDATE(), @DatabaseName = 'SomeDB'
```
[*Back to top*](#header1)

## Output files
The output directory will be created in the PSBlitz directory where the PSBlitz.ps1 script lives.

Output directory name `[Instance]_[TimeStamp]` for an instance-wide check, or `[Instance]_[TimeStamp]_[Database]` for a database-specific check.

Deadlocks will be saved in the Deadlocks directory under the output directory.

Deadlock file naming convention - `[EventDate]_[EventTime]_[RecordNumberOfDistinctDeadlockGroupVictim].xdl`

Execution plans will be saved in the Plans directory under the output directory.

Execution plans file naming convention - `[SortOrder]_[RowNumber].sqlplan`

[*Back to top*](#header1)

## Usage examples
You can run PSBlitz.ps1 by simply right-clicking on the script and then clicking on "Run With PowerShell" which will execute the script in interactive mode, prompting you for the required input.

Otherwise you can navigate to the directory where the script is in PowerShell and execute it by providing parameters and appropriate values.
- Examples:
1. Print the help menu
    ```PowerShell
    .\PSBlitz.ps1 ?
    ```
    or
    ```PowerShell
    .\PSBlitz.ps1 Help
    ```
2. Run it against the whole instance (named instance SQL01), with default checks via integrated security
    ```PowerSHell
    .\PSBlitz.ps1 Server01\SQL01
    ```
3. Run it against the whole instance, with in-depth checks via integrated security
    ```PowerSHell
    .\PSBlitz.ps1 Server01\SQL01 -IsIndepth Y
    ```
4. Run it with in-depth checks, limit sp_BlitzIndex, sp_BlitzCache, and sp_BlitzLock to YourDatabase only, via integrated security
    ```PowerSHell
    .\PSBlitz.ps1 Server01\SQL01 -IsIndepth Y -CheckDB YourDatabase
    ```
5. Run it against the whole instance, with default checks via SQL login and password
    ```PowerSHell
    .\PSBlitz.ps1 Server01\SQL01 -SQLLogin DBA1 -SQLPass SuperSecurePassword
    ```
6. Run it against a default instance residing on Server02, with in-depth checks via SQL login and password, while limmiting sp_BlitzIndex, sp_BlitzCache, and sp_BlitzLock to YourDatabase only
    ```PowerSHell
    .\PSBlitz.ps1 Server02 -SQLLogin DBA1 -SQLPass SuperSecurePassword -IsIndepth Y -CheckDB YourDatabase
    ```
Note that `-ServerName` is a positional parameter, so you don't necessarily have to specify the parameter's name as long as the first thing after the script's name is the instance 

[*Back to top*](#header1)
