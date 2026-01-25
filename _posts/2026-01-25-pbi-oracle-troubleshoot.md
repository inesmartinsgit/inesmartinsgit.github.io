---
title: "From Client to Server: A Full End-to-End Deep Dive Into a Power BI - Oracle Connection Timeout"
---

<span style="color:red">Warning: Post under development, not complete yet.</span>

<span style="color:grey"> _Disclaimer: Everything you’ll find here reflects my personal views and is not affiliated with Microsoft or Oracle._ </span>

<br>

Troubleshooting an Oracle connection error can be challenging, especially when the error message is frustratingly generic.


In this post, I’ll share my troubleshooting workflow and hands-on tips, using a real-life scenario as an example.<br>
While the example involves Power BI Desktop, the same approach can scale to complex enterprise setups including scenarios with on-premises data gateways. <br>

# The Beginning of a Very Annoying Mystery

One day, while working on a Power BI report connected to Oracle and multitasking (as we all do), I kept running into a generic error: _**ORA-03135: connection lost contact.**_ <br>
Forcing a **Refresh Preview** fixed it temporarily, but that only confirmed something wasn't right. 

⚠️ Was something on my machine interfering with the connection? <br>

<img width="1200" alt="image" src="https://github.com/user-attachments/assets/4298e793-56b9-45d7-888a-0fca614b3c6c" />

<br>
All I wanted was to get back to building my report, so I created a clean Azure VM to host my Power BI desktop and Oracle driver ODP.NET unmanaged (for guidance on this, refer to my previous blog post - [Seamless Power BI and Oracle Integration: Key Learnings & Setup Tips](https://inesmartinsgit.github.io/2026/01/14/pbi-oracle-seamless.html) ).  <br>
<br>

My Oracle server was already running on a clean Azure VM so I thought this setup was good enough.

⛔ **I was wrong.**

Same exact error continued to happen.  <br>
So I decided to put my reporting on hold and to dig deeper into the issue.

# The Hunt Begins
Like any good hunt, mine started with research. 

I began by checking the Oracle error code using the **Oracle database error messages** for this error code - [ORA-03135 - Database Error Messages](https://docs.oracle.com/en/error-help/db/ora-03135/) :

- Cause <br>
		a. Server unexpectedly terminated or was forced to terminate.<br>
		b. Server timed out the connection.<br>
	
- Action <br>
		a. Check if the server session was terminated <br>
		b. Check if the timeout parameters are set properly in sqlnet.ora. <br>

This offered a few clues, but **nothing specific** enough to point me toward a solution.

Next, I went into my "smart-lazy mode" digging through community posts and hoping to find a quick fix, but **no luck.**


Before diving deep into a time-consuming investigation, I **ran a few tests:**
- The connection lost contact while I was **multitasking**, so I tried staying fully active in the report and got no errors.
- That gave me the hint that it was the **inactivity causing the error** (some kind of **idle timeout**).
- After a few more attempts I understood it took around **five minutes to break.**
	
It looked like a classic idle timeout of some component so I thought it was worth the investment on learning how to troubleshoot this rather than continuing with random tests.

📌 For this blog post, the scenario is intentionally simple, so only a few log files are generated. <br>
In **real enterprise environments**, though, you may face **hundreds of logs to go through**, so knowing exactly where to look becomes essential.


# Figuring Out Who Talks to Whom

The simplified components and flow of my setup were the following:
- **Azure VM - Client**: Public IP 20.14.72.115
  - Power BI Desktop generates queries using the Oracle connector.
	- The connector leverages the ODP.NET unmanaged driver to establish the connection and send queries.
- **Azure VM - Server**: Public IP 20.163.1.157
  - The Oracle server (and listener) receives and processes the queries (dedicated server architecture).
		
I spent some time understanding each component and their connectivity by leveraging Microsoft and Oracle documentation. 

I’m intentionally keeping this section simple, especially on the server side, by including only the essentials. <br>
If want to dive deeper, I’ve listed some references at the end of the post.

# When Simplifying Made Things Interesting

To simplify the scenario and remove Power BI from the picture, I tested the same scenario using PowerShell (for more on this, refer to my previous blog post: - [Seamless Power BI and Oracle Integration: Key Learnings & Setup Tips](https://inesmartinsgit.github.io/2026/01/14/pbi-oracle-seamless.html) ).  <br>

Since my connection broke after around 5 minutes, I wrote a quick PowerShell script to wait 5 minutes before sending the next query.

```powershell
# Create connection
$conn = [System.Data.Common.DbProviderFactories]::GetFactory("Oracle.DataAccess.Client").CreateConnection()
$conn.ConnectionString = "Data Source=20.163.1.157:1521/ines.internal.cloudapp.net;User Id=HR;Password=xxxx“

# Open connection
$conn.Open()
Write-Host "Open connection at: $(Get-Date -Format 'HH:mm:ss')" -ForegroundColor Cyan

# Run query
Write-Host "Running query at: $(Get-Date -Format 'HH:mm:ss')" -ForegroundColor Cyan
$sel_stmt = $conn.CreateCommand()
$sel_stmt.CommandText = "SELECT * FROM HR.JOBS WHERE MIN_SALARY > 10000"
$rdr = $sel_stmt.ExecuteReader()
while ($rdr.Read()) {
   write-host "$($rdr.GetOracleValue(0).Value)"
}

# Wait 5 minutes
Write-Host "Waiting 5 minutes" -ForegroundColor Red
Start-Sleep -Seconds 300

# Run query again
Write-Host "Running query at: $(Get-Date -Format 'HH:mm:ss')" -ForegroundColor Cyan
$sel_stmt = $conn.CreateCommand()
$sel_stmt.CommandText = "SELECT * FROM HR.JOBS WHERE MIN_SALARY > 10000"
$rdr = $sel_stmt.ExecuteReader()
while ($rdr.Read()) {
   write-host "$($rdr.GetOracleValue(0).Value)"
}

# Close connection
$conn.Close()
```

Guess what? 

💡 **PowerShell had the same problem therefore the issue was not specific to the application (PowerShell/Power BI).**

<img width="700" alt="image" src="https://github.com/user-attachments/assets/f4194716-62d5-42bc-9d23-894aefa9d81d" />

# Collecting Clues Through Logs

To trace what was happening I started gathering logs from every part of the setup.
Here’s what I enabled:
- On the Client:
  - The Client tools (Power BI Desktop / PowerShell) - client logs
  - ODP.NET unmanaged driver - Oracle driver logs
  - Client VM connectivity:
    - Network trace - I prefer using Wireshark but you can use any tool you are comfortable.
    - SQL NET logs - Oracle client
			
- On the Server:
  - Oracle server:
    - Sessions
	- Processes
  - Server VM connectivity: 
	- Network trace
	- SQL NET logs - oracle server
		

The guidance below come from my Oracle 19 database and from a bit of trial‑and‑error testing. <br>
You may need to adjust the steps for your own version, but the overall approach remains the same. <br>
If you have feedback or ideas to improve it, I’d love to hear them!<br>

### Setting Up Client Logs

#### Power BI Desktop logs
- Collect following the steps from Microsoft's documentation https://learn.microsoft.com/en-us/power-bi/fundamentals/desktop-diagnostics

#### ODP.NET unmanaged driver logs
- Using the registry editor, navigate to the driver folder: e.g. _Computer\HKEY_LOCAL_MACHINE\SOFTWARE\ORACLE\ODP.NET\4.122.19.1_
- Manually create 2 string values:
  - **TraceFileLocation** - specifies the location of logs
  - **TraceLevel = 7** - to enable all traces
- Restart the machine

<img width="700" alt="image" src="https://github.com/user-attachments/assets/9b4605b6-15f6-4277-add4-4c8fec191165" />

#### SQL NET logs - Oracle client
	
- Edit the sqlnet.ora file and add the following:

| Trace Config to Add | Description |
|----------|----------|
| TRACE_LEVEL_CLIENT = 16 | Support level logs (maximum info). |
| TRACE_FILE_CLIENT = client | Name of the log file. <br> It includes the process identifier (PID) appended to the name automatically. |
| TRACE_DIRECTORY_CLIENT = C:\Users\inmartin\Oracle\network\admin\sqlnetlogs | Folder where the traces are located. <br> Be careful with possible disk space issues and if possible select a non C:\ drive. |
| TRACE_TIMESTAMP_CLIENT = ON | To add a timestamp to the logs inside the file. |
| DIAG_ADR_ENABLED=OFF | Required so we can configure all these parameters for the logs. |
| TRACE_UNIQUE_CLIENT=ON | Unique trace file is created for each client trace session. |

- Navigate to the folder.
- Logs can be opened using notepad or another text editor.

### Setting Up Sever Logs

#### Oracle server: Sessions and Processes
	
- Using Oracle SQL developer, connect to system and run:
  - **select * from v$session;**
  - **select * from v$process;**

- Logs can be exported to several format (I prefer csv).

#### SQL NET logs - Oracle server and listener
	
- For this scenario there are no errors related to listener but I'll include them in the configuration in case you will find it useful.

**SQL NET logs for Oracle server**
	
- Edit the sqlnet.ora file and add the following:

| Trace Config to Add | Description |
|----------|----------|
| TRACE_LEVEL_SERVER = 16  | Support level logs (maximum info). |
| TRACE_FILE_SERVER = server  | Name of the log file. <br> It will include the process identifier (pid) is appended to the name automatically. |
| TRACE_DIRECTORY_SERVER = C:\OracleServer\network\admin\sqlnetlogs | Folder where the traces are located. <br> | Be careful with possible disk space issues and if possible select a non C:\drive. |
| TRACE_TIMESTAMP_SERVER = ON | To add a timestamp to the logs inside the file. |
| DIAG_ADR_ENABLED= OFF | Required so we can configure all these parameters for the logs (ADR = Automatic Diagnostic Repository). |

- Navigate to the folder.
- Logs can be opened using notepad or another text editor.

**SQL NET logs for Oracle listener**
	
- Edit the listener.ora file and add the following:

| Trace Config to Add | Description |
|----------|----------|
| TRACE_LEVEL_LISTENER = 16 | Support level logs (maximum info). |
| TRACE_FILE_LISTENER = listener | Name of the log file. It will include the process identifier (pid) is appended to the name automatically.|
| TRACE_DIRECTORY_LISTENER = C:\OracleServer\network\admin\listenerlogs | Folder where the traces are located. Be careful with possible disk space issues and if possible select a non C:\drive. |
| TRACE_TIMESTAMP_LISTENER = ON | To add a timestamp to the logs inside the file. |
| DIAG_ADR_ENABLED_LISTENER= OFF | Required so we can configure all these parameters for the logs (ADR = Automatic Diagnostic Repository). |
	
- Restart the listener using cmd
  - **lsnrctl stop**
  - **lsnrctl start**

- To collect logs:
  - Navigate to the folder.
  - Logs can be opened using notepad or another text editor.


# Walking Through the Failing Scenario

The error message provided me useful information to directly correlate with the logs.

_DataSource.Error: Oracle: ORA-03135: connection lost contact<br>
<span style="background-color:#829FED">Process ID: 7900</span><br>
<span style="background-color:#AAFABE">Session ID: 396</span> <span style="background-color:#42D4BC">Serial number: 44130 </span><br>
Details:<br>
&nbsp;&nbsp;&nbsp;&nbsp;DataSourceKind=Oracle<br>
&nbsp;&nbsp;&nbsp;&nbsp;DataSourcePath=20.163.1.157:1521/ines.internal.cloudapp.net<br>
&nbsp;&nbsp;&nbsp;&nbsp;Message=ORA-03135: connection lost contact<br>
Process ID: 7900<br>
Session ID: 396 Serial number: 44130<br>
&nbsp;&nbsp;&nbsp;&nbsp;ErrorCode=-2147467259<br>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="background-color:#EBAAFA">NativeError=3135</span><br>_

### Oracle Server - Session and Process

When checking the active **Oracle session**, the technical information in the error correlates directly with the information I needed to proceed further on checking the client side logs (below table is cropped to include relevant information).


<div style="overflow-x:auto;">

<table>
  <tr>
    <th>SADDR</th>
    <th>SID</th>
    <th>SERIAL#</th>
    <th>AUDSID</th>
    <th>PADDR</th>
    <th>USER#</th>
    <th>USERNAME</th>
    <th>COMMAND</th>
    <th>OWNERID</th>
    <th>TADDR</th>
    <th>LOCKWAIT</th>
    <th>STATUS</th>
    <th>SERVER</th>
    <th>SCHEMA#</th>
    <th>SCHEMANAME</th>
    <th>OSUSER</th>
    <th>PROCESS</th>
    <th>MACHINE</th>
    <th>PORT</th>
    <th>TERMINAL</th>
    <th>PROGRAM</th>
    <th>TYPE</th>
  </tr>

  <tr>
    <td>00007FF82FAEB420</td>
    <td><span style="background-color:#AAFABE">396</span></td>
    <td>44130</td>
    <td>40031</td>
    <td>00007FF82F573A20</td>
    <td>107</td>
    <td>HR</td>
    <td>0</td>
    <td>2147483644</td>
    <td></td>
    <td></td>
    <td>INACTIVE</td>
    <td>DEDICATED</td>
    <td>107</td>
    <td>HR</td>
    <td>inmartin</td>
    <td><span style="background-color:#D9FAAA">4960</span>:<span style="background-color:#FADAAA">10144</span></td>
    <td>WORKGROUP\VMoracleclient</td>
    <td><span style="background-color:#FAAABF">50679</span></td>
    <td>VMoracleclient</td>
    <td><b>Microsoft.Mashup.Container.NetFX45.exe</b></td>
    <td>USER</td>
  </tr>
</table>

</div>

<br>
From this session information I knew:
- The **Mashup container PID =  <span style="background-color:#D9FAAA">4960</span>.** This allowed me to identify the mashup file directly
- Was leveraging the **Oracle driver TID = <span style="background-color:#FADAAA">10144</span>.** This allowed me to identify the sqlnet logs from client side directly	
- Using the **port from client side = <span style="background-color:#FAAABF">50679</span>.** This helped me narrow down the network traces.

When checking the active **Oracle process**, I can do the same but correlating on the server side:


<div style="overflow-x:auto;">

<table>
  <tr>
    <th>ADDR</th>
    <th>PID</th>
    <th>SOSID</th>
    <th>SPID</th>
    <th>STID</th>
    <th>EXECUTION_TYPE</th>
    <th>PNAME</th>
    <th>USERNAME</th>
    <th>SERIAL#</th>
    <th>TERMINAL</th>
    <th>PROGRAM</th>
  </tr>
	
  <tr>
    <td>00007FF82F573A20</td>
    <td>59</td>
    <td>7900</td>
    <td><span style="background-color:#829FED">7900</span></td>
    <td>0</td>
    <td>THREAD</td>
    <td></td>
    <td>OracleServiceIN</td>
    <td>8</td>
    <td>VMoracleserver</td>
    <td>ORACLE.EXE (SHAD)</td>
  </tr>
</table>

</div>

From this process information I knew:
- The **Server PID = <span style="background-color:#829FED">7900</span>.** This allowed me to identify the sqlnet logs from server side directly.

### Oracle Client log analysis

#### Power BI Desktop logs


Knowing the mashup container process ID PID = <span style="background-color:#D9FAAA">4960</span> obtained from the session info,  I was able to quickly locate the log file to analyze: _**Microsoft.Mashup.Container.NetFX45.<span style="background-color:#D9FAAA">4960</span>.2026-01-24T15-02-09-350038.log**_

<div style="font-style: italic; white-space: pre-wrap; word-wrap: break-word; max-width: 100%;">

{"Start":<span style="background:#DEDEDE;">"2026-01-24T15:09:48.9553912Z"</span>,"Action":<strong>"Engine/IO/Db/Oracle/Connection/Open"</strong>strong>,"ResourceKind":"Oracle","ResourcePath":"20.163.1.157:1521/ines.internal.cloudapp.net","HostProcessId":"956","PartitionKey":"Section1/LOCATIONS/2","Server":"20.163.1.157:1521/ines.internal.cloudapp.net","RequireEncryption":"False","ConnectionTimeout":"15","ConnectionId":"957c17d0-db30-4833-a3ab-70b9c1e43935","ProductVersion":"2.149.1054.0 (25.11)+fe0a1c0f2fc6e9cf0939a7efabedc7cdb3358bad","ActivityId":"86d59d77-7be2-43f3-afc3-8dfab4ef4b1b","Process":"Microsoft.Mashup.Container.NetFX45","Pid":4960,"Tid":1,"Duration":"00:00:00.0004251"}

{"Start":<span style="background:#DEDEDE;">"2026-01-24T15:09:48.9558422Z"</span>,"Action":"<strong>Engine/IO/Db/Oracle/Command/ExecuteDbDataReader"</strong>,"ResourceKind":"Oracle","ResourcePath":"20.163.1.157:1521/ines.internal.cloudapp.net","HostProcessId":"956","PartitionKey":"Section1/LOCATIONS/2","Server":"20.163.1.157:1521/ines.internal.cloudapp.net","CommandText":<span style="background-color:#F6FAAA">"select \"$Ordered\".\"REGION_ID\",\r\n    \"$Ordered\".\"REGION_NAME\"\r\nfrom \r\n(\r\n    select \"_\".\"REGION_ID\",\r\n        \"_\".\"REGION_NAME\"\r\n    from \"HR\".\"REGIONS\" \"_\"\r\n    where \"_\".\"REGION_ID\" = 3\r\n) \"$Ordered\"\r\norder by \"$Ordered\".\"REGION_ID\"\r\nfetch next 4096 rows only"</span>,"CommandTimeout":"600","ConnectionId":"957c17d0-db30-4833-a3ab-70b9c1e43935","Behavior":"Default","Exception":"Exception:\r\nExceptionType: Oracle.DataAccess.Client.OracleException, Oracle.DataAccess, Version=4.122.19.1, Culture=neutral, PublicKeyToken=89b483f429c47342\r\n<span style="background-color:#F77059">Message: ORA-03135: connection lost contact\nProcess ID: 7900\nSession ID: 396 Serial number: 44130</span>\r\nStackTrace:\n   at Oracle.DataAccess.Client.OracleException.HandleErrorHelper(Int32 errCode, OracleConnection conn, IntPtr opsErrCtx, OpoSqlValCtx* pOpoSqlValCtx, Object src, String procedure, Boolean bCheck, Int32 isRecoverable, OracleLogicalTransaction m_OracleLogicalTransaction)\r\n   at Oracle.DataAccess.Client.OracleException.HandleError(Int32 errCode, OracleConnection conn, String procedure, IntPtr opsErrCtx, OpoSqlValCtx* pOpoSqlValCtx, Object src, Boolean bCheck, OracleLogicalTransaction m_OracleLogicalTransaction)\r\n   at Oracle.DataAccess.Client.OracleCommand.ExecuteReader(Boolean requery, Boolean fillRequest, CommandBehavior behavior)\r\n   at Oracle.DataAccess.Client.OracleCommand.ExecuteDbDataReader(CommandBehavior behavior)\r\n   at Microsoft.Mashup.Engine1.Library.Common.TracingDbCommand.<>n__0(CommandBehavior behavior)\r\n   at Microsoft.Mashup.Engine1.Library.Common.TracingDbCommand.<>c__DisplayClass6_0.<ExecuteDbDataReader>b__0(IHostTrace trace)\r\n   at Microsoft.Mashup.Engine1.Library.Common.Tracer.TraceCommon[T](IHostTrace trace, Func`2 func)\r\n\r\n\r\n","ProductVersion":"2.149.1054.0 (25.11)+fe0a1c0f2fc6e9cf0939a7efabedc7cdb3358bad","ActivityId":"86d59d77-7be2-43f3-afc3-8dfab4ef4b1b","Process":"Microsoft.Mashup.Container.NetFX45","Pid":4960,"Tid":1,"Duration":"00:00:19.4224248"}
	
{"Start":<span style="background:#DEDEDE;">"2026-01-24T15:10:08.3951188Z"</span>,"Action":<strong>"Engine/IO/Db/Oracle/Connection/Close"</strong>,"ResourceKind":"Oracle","ResourcePath":"20.163.1.157:1521/ines.internal.cloudapp.net","HostProcessId":"956","PartitionKey":"Section1/LOCATIONS/2","Server":"20.163.1.157:1521/ines.internal.cloudapp.net","ConnectionId":"957c17d0-db30-4833-a3ab-70b9c1e43935","ProductVersion":"2.149.1054.0 (25.11)+fe0a1c0f2fc6e9cf0939a7efabedc7cdb3358bad","ActivityId":"86d59d77-7be2-43f3-afc3-8dfab4ef4b1b","Process":"Microsoft.Mashup.Container.NetFX45","Pid":4960,"Tid":1,"Duration":"00:00:00.0099732"}

</div>

Before the error occurs, it's visible the connection open request and the query sent to the server.

#### ODP.NET driver logs

Knowing the Power BI mashup container process ID PID = <span style="background-color:#D9FAAA">4960</span> obtained from the session info, I was able to** convert it to HEX** to find the driver log file: **<span style="background-color:#D9FAAA">4960</span> (DEC) = <span style="background-color:#D1B69D">1360</span> (HEX)** - _**MICROSOFT.MASHUP.CONTAINER.NETFX45.EXE_PID_<span style="background-color:#D1B69D">1360</span>_DATE_2026_01_24_TIME_15_02_14_000333.trc**_

<div style="font-style: italic; white-space: pre-wrap; word-wrap: break-word; max-width: 100%;">
	
TIME:<span style="background:#DEDEDE;">2026/01/24-15:09:48:961</span> TID:27a0  OracleConnection::Open() => Calling RequestBegin
(…)
TIME:2026/01/24-15:09:48:961 TID:27a0  (INFO) (OracleConnectionOCP.Open) Pool Id=531559270 PooledConCtx=40026340 sessid=396 serial=44130 inst=ines ConCtx=(2405dcbd1a0)
(…)
TIME:<span style="background:#DEDEDE;">2026/01/24-15:09:48:961</span> TID:27a0  SqlInit(): SQL: <span style="background-color:#F6FAAA">select "$Ordered"."REGION_ID",
    "$Ordered"."REGION_NAME"
from 
(
    select "_"."REGION_ID",
        "_"."REGION_NAME"
    from "HR"."REGIONS" "_"
    where "_"."REGION_ID" = 3
) "$Ordered"
order by "$Ordered"."REGION_ID"
fetch next 4096 rows only</span>
(…)
TIME:2026/01/24-15:10:08:007 TID:27a0  (EXIT)  OpsSqlExecuteReader(): RetCode=-1 Line=783
TIME:2026/01/24-15:10:08:013 TID:27a0  (ENTRY) OpsErrGetOpoCtx()
TIME:<span style="background:#DEDEDE;">2026/01/24-15:10:08:014</span> TID:27a0  <span style="background:#F77059;">(ERROR) Oracle error code=3135; Oracle msg=ORA-03135: connection lost contact
Process ID: 7900
Session ID: 396 Serial number: 44130</span>

</div>

