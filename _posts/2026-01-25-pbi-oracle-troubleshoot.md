---
title: "From Client to Server: An End-to-End Deep Dive Into Power BI-Oracle Connection Timeouts"
---

<span style="color:grey"> _Disclaimer: Everything you’ll find here reflects my personal views and is not affiliated with Microsoft or Oracle._ </span>

<br>

If you've ever investigated a generic Power BI-Oracle connectivity error, you know how quickly it can turn into a rabbit hole.<br>

In this blog post, I walk through a real troubleshooting journey where a simple Power BI report turned into a full-blown investigation across Power BI and Oracle logs, network traces and considerations about Azure infrastructure. <br>

The example used was very simple but the same approach can scale to complex enterprise setups including scenarios with on-premises data gateways. For those scenarios, you may face hundreds of logs to go through, so knowing exactly where to look becomes essential.<br>


🔍 In this post you'll find:
- An approach to simplify the scenarios as much as possible.
- How to set up end‑to‑end log collection.
- Using real-life logs as examples, how to investigate the logs by knowing quickly where and how to look.
- The final “aha!” moment that solved the entire mystery.


Whether you're building reports, supporting Power BI-Oracle integrations, or simply love a good troubleshooting story, this one’s for you.

<br>

# The Beginning of a Very Annoying Mystery

One day, while working on a Power BI report connected to Oracle and multitasking (as we all do), I kept running into a generic error: _**ORA-03135: connection lost contact.**_ <br>
Forcing a **Refresh Preview** fixed it temporarily, but that only confirmed something wasn't right. 

⚠️ Was something on my machine interfering with the connection? <br>

<img width="1200" alt="image" src="https://github.com/user-attachments/assets/4298e793-56b9-45d7-888a-0fca614b3c6c" />

<br>
All I wanted was to get back to building my report, so I created a clean Azure VM to host my Power BI Desktop and Oracle driver ODP.NET unmanaged (for guidance on this, refer to my previous blog post - [Seamless Power BI and Oracle Integration: Key Learnings & Setup Tips](https://inesmartinsgit.github.io/2026/01/14/pbi-oracle-seamless.html) ).  <br>
<br>

My Oracle server was already running on a clean Azure VM so I thought this setup was good enough.

⛔ I was wrong.

Same exact error continued to happen.  <br>
So I decided to put my reporting on hold and to dig deeper into the issue.

<br>

# The Hunt Begins
Like any good hunt, mine started with research. 

I began by checking the Oracle error code using the [Oracle Database Error Messages](https://docs.oracle.com/en/error-help/db/ora-03135/) for this error code: 
- Cause
  - Server unexpectedly terminated or was forced to terminate.
  - Server timed out the connection.
- Action 
  - Check if the server session was terminated.
  - Check if the timeout parameters are set properly in sqlnet.ora.

This offered a few clues, but **nothing specific** enough to point me toward a solution.

<br>
Next, I went into my "smart-lazy mode" digging through community posts and hoping to find a quick fix, but **no luck.**

<br>
Before diving deep into a time-consuming investigation, I **ran a few tests:**
- The connection lost contact while I was **multitasking**, so I tried staying fully active in the report and got no errors.
- That gave me the hint that it was the **inactivity causing the error** (some kind of **idle timeout**).
- After a few more attempts I understood it took around **five minutes to break.**
	
It looked like a classic idle timeout of some component so I thought it was worth the investment on learning how to troubleshoot this rather than continuing with random tests.

<br>
📌 For this blog post, the scenario is intentionally simple, so only a few log files are generated. <br>
In **real enterprise environments**, though, you may face **hundreds of logs to go through**, so knowing exactly where to look becomes essential.

<br>

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

To simplify the scenario and remove Power BI from the picture, I tested the same scenario using **PowerShell** (for more on this, refer to my previous blog post: - [Seamless Power BI and Oracle Integration: Key Learnings & Setup Tips](https://inesmartinsgit.github.io/2026/01/14/pbi-oracle-seamless.html) ).  <br>

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

💡 **PowerShell had the same problem** therefore the issue was not specific to the application (PowerShell/Power BI).

<img width="700" alt="image" src="https://github.com/user-attachments/assets/f4194716-62d5-42bc-9d23-894aefa9d81d" />

# Collecting Clues Through Logs

To trace what was happening I started gathering logs from every part of the setup.<br>
Here’s what I enabled:
- On the Client:
  - The Client tools (Power BI Desktop / PowerShell) - Client logs
  - ODP.NET unmanaged driver - Oracle driver logs
  - Client VM connectivity:
    - Network trace - I prefer using Wireshark but you can use any tool you are comfortable.
    - SQL NET logs - Oracle client
			
- On the Server:
  - Oracle server session and process information.
  - Server VM connectivity: 
	- Network trace
	- SQL NET logs - Oracle server
		

📌 The guidance to enable the logs explained below comes from my Oracle 19 database and from a bit of trial‑and‑error testing. <br>
You may need to adjust the steps for your own version, but the overall approach remains the same. <br>
If you have feedback or ideas to improve it, I’d love to hear them!<br>

<br>

### Setting Up Client Logs

<br>

#### Power BI Desktop logs
- Collect following the steps from Microsoft's documentation: [Power BI Desktop diagnostics collection](https://learn.microsoft.com/en-us/power-bi/fundamentals/desktop-diagnostics).

<br>

#### ODP.NET unmanaged driver logs
- **Create a folder** where the files will be located. Be careful with possible disk space issues and if possible select a non C:\drive. For scenarios with on-premises data gateway, don't choose a folder within a user folder.
- Using the **registry editor**, navigate to the driver folder. It should be a similar to the following: _Computer\HKEY_LOCAL_MACHINE\SOFTWARE\ORACLE\ODP.NET\4.122.19.1_
- Manually create **2 string values**:
  - _TraceFileLocation_ - specifies the location of logs
  - _TraceLevel = 7_ - to enable all traces
- **Restart** the machine.
- To collect the logs, navigate to the specified folder.

<img width="600" alt="image" src="https://github.com/user-attachments/assets/9b4605b6-15f6-4277-add4-4c8fec191165" />

<br>

#### SQL NET logs - Oracle client

- **Create a folder** where the files will be located. Be careful with possible disk space issues and if possible select a non C:\ drive. For scenarios with on-premises data gateway, don't choose a folder within a user folder.
- Edit the **sqlnet.ora** file and add the following:

<div style="font-size:12px; overflow-x:auto;">
<table>
  <thead>
    <tr>
      <th>Trace Config to Add</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>TRACE_LEVEL_CLIENT = 16</td>
      <td>Support level logs (maximum info).</td>
    </tr>
    <tr>
      <td>TRACE_FILE_CLIENT = client</td>
      <td>
        Name of the log file.<br>
        It includes the process identifier (PID) appended to the name automatically.
      </td>
    </tr>
    <tr>
      <td>TRACE_DIRECTORY_CLIENT = C:\Users\inmartin\Oracle\network\admin\sqlnetlogs</td>
      <td>Folder where the traces are located.</td>
    </tr>
    <tr>
      <td>TRACE_TIMESTAMP_CLIENT = ON</td>
      <td>To add a timestamp to the logs inside the file.</td>
    </tr>
    <tr>
      <td>DIAG_ADR_ENABLED = OFF</td>
      <td>Required so we can configure all these parameters for the logs.</td>
    </tr>
    <tr>
      <td>TRACE_UNIQUE_CLIENT = ON</td>
      <td>Unique trace file is created for each client trace session.</td>
    </tr>
  </tbody>
</table>
</div>

- To collect the logs, navigate to the specified folder.
- Logs can be opened using notepad or another text editor.

<br>

### Setting Up Sever Logs

#### Oracle server: Sessions and Processes
	
- Using **Oracle SQL developer**, connect to system and run:
  - _select * from v$session;_
  - _select * from v$process;_

- Logs can be exported in several formats, though I prefer CSV.

#### SQL NET logs - Oracle server and listener
	
- For this scenario there are no errors related to listener but I'll include them in the configuration in case you will find it useful.

**SQL NET logs for Oracle server**

- **Create a folder** where the files will be located. Be careful with possible disk space issues and if possible select a non C:\ drive. For scenarios with on-premises data gateway, don't choose a folder within a user folder.
- Edit the **sqlnet.ora** file and add the following:

<div style="font-size:12px; overflow-x:auto;">
<table>
  <thead>
    <tr>
      <th>Trace Config to Add</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>TRACE_LEVEL_SERVER = 16</td>
      <td>Support level logs (maximum info).</td>
    </tr>
    <tr>
      <td>TRACE_FILE_SERVER = server</td>
      <td>
        Name of the log file.<br>
        It will include the process identifier (pid) appended to the name automatically.
      </td>
    </tr>
    <tr>
      <td>TRACE_DIRECTORY_SERVER = C:\OracleServer\network\admin\sqlnetlogs</td>
      <td>Folder where the traces are located.</td>
    </tr>
    <tr>
      <td>TRACE_TIMESTAMP_SERVER = ON</td>
      <td>To add a timestamp to the logs inside the file.</td>
    </tr>
    <tr>
      <td>DIAG_ADR_ENABLED = OFF</td>
      <td>
        Required so we can configure all these parameters for the logs 
        (ADR = Automatic Diagnostic Repository).
      </td>
    </tr>
  </tbody>
</table>
</div>

- To collect the logs, navigate to the specified folder.
- Logs can be opened using notepad or another text editor.

<br>

**SQL NET logs for Oracle listener**

- **Create a folder** where the files will be located. Be careful with possible disk space issues and if possible select a non C:\ drive. For scenarios with on-premises data gateway, don't choose a folder within a user folder.
- Edit the **listener.ora** file and add the following:

<div style="font-size:12px; overflow-x:auto;">
<table>
  <thead>
    <tr>
      <th>Trace Config to Add</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>TRACE_LEVEL_LISTENER = 16</td>
      <td>Support level logs (maximum info).</td>
    </tr>
    <tr>
      <td>TRACE_FILE_LISTENER = listener</td>
      <td>
        Name of the log file. It will include the process identifier (pid) appended to the name automatically.
      </td>
    </tr>
    <tr>
      <td>TRACE_DIRECTORY_LISTENER = C:\OracleServer\network\admin\listenerlogs</td>
      <td>Folder where the traces are located.</td>
    </tr>
    <tr>
      <td>TRACE_TIMESTAMP_LISTENER = ON</td>
      <td>To add a timestamp to the logs inside the file.</td>
    </tr>
    <tr>
      <td>DIAG_ADR_ENABLED_LISTENER = OFF</td>
      <td>
        Required so we can configure all these parameters for the logs 
        (ADR = Automatic Diagnostic Repository).
      </td>
    </tr>
  </tbody>
</table>
</div>

- **Restart the listener** using cmd
  - _lsnrctl stop_
  - _lsnrctl start_

- To collect the logs, navigate to the specified folder.
- Logs can be opened using notepad or another text editor.

<br>

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

<br>

### Oracle Server - Session and Process

When checking the active **Oracle session**, the technical information in the error correlates directly with the information I needed to proceed further on checking the client side logs (below table is cropped to include relevant information).


<div style="font-size:12px; overflow-x:auto;">

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
- The **Mashup container PID =  <span style="background-color:#D9FAAA">4960</span>.** This allowed me to identify the Power BI mashup log file directly.
- Was leveraging the **Oracle driver TID = <span style="background-color:#FADAAA">10144</span>.** This allowed me to identify the sqlnet logs from client side directly.
- Using the **port from client side = <span style="background-color:#FAAABF">50679</span>.** This helped me narrow down the network traces.

When checking the active **Oracle process**, I repeat a similar similar analysis, this time correlating the information from the server side:

<div style="font-size:12px; overflow-x:auto;">

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

<br>

### Client Log Analysis

#### Power BI Desktop Logs

Knowing the mashup container process ID PID = <span style="background-color:#D9FAAA">4960</span> obtained from the session info,  I was able to quickly locate the log file to analyze: _**Microsoft.Mashup.Container.NetFX45.<span style="background-color:#D9FAAA">4960</span>.2026-01-24T15-02-09-350038.log**_

<div style="white-space: pre-wrap; font-style: italic; font-size:12px; word-wrap: break-word; max-width: 100%;">
	
	{"Start":"<span style='background:#DEDEDE;'>2026-01-24T15:09:48.9553912Z</span>","Action":"<strong>Engine/IO/Db/Oracle/Connection/Open</strong>","ResourceKind":"Oracle","ResourcePath":"20.163.1.157:1521/ines.internal.cloudapp.net","HostProcessId":"956","PartitionKey":"Section1/LOCATIONS/2","Server":"20.163.1.157:1521/ines.internal.cloudapp.net","RequireEncryption":"False","ConnectionTimeout":"15","ConnectionId":"957c17d0-db30-4833-a3ab-70b9c1e43935","ProductVersion":"2.149.1054.0 (25.11)+fe0a1c0f2fc6e9cf0939a7efabedc7cdb3358bad","ActivityId":"86d59d77-7be2-43f3-afc3-8dfab4ef4b1b","Process":"Microsoft.Mashup.Container.NetFX45","Pid":4960,"Tid":1,"Duration":"00:00:00.0004251"}
	
	{"Start":"<span style='background:#DEDEDE;'>2026-01-24T15:09:48.9558422Z</span>","Action":"<strong>Engine/IO/Db/Oracle/Command/ExecuteDbDataReader</strong>","ResourceKind":"Oracle","ResourcePath":"20.163.1.157:1521/ines.internal.cloudapp.net","HostProcessId":"956","PartitionKey":"Section1/LOCATIONS/2","Server":"20.163.1.157:1521/ines.internal.cloudapp.net","CommandText":"<span style='background:#F6FAAA;'>select \"$Ordered\".\"REGION_ID\",\n    \"$Ordered\".\"REGION_NAME\"\nfrom\n(\n    select \"_\".\"REGION_ID\",\n        \"_\".\"REGION_NAME\"\n    from \"HR\".\"REGIONS\" \"_\"\n    where \"_\".\"REGION_ID\" = 3\n) \"$Ordered\"\norder by \"$Ordered\".\"REGION_ID\"\nfetch next 4096 rows only</span>","CommandTimeout":"600","ConnectionId":"957c17d0-db30-4833-a3ab-70b9c1e43935","Behavior":"Default","Exception":"Exception:\nExceptionType: Oracle.DataAccess.Client.OracleException\nMessage: ORA-03135: connection lost contact\nProcess ID: 7900\nSession ID: 396 Serial number: 44130\nStackTrace:\n   at Oracle.DataAccess.Client.OracleException.HandleErrorHelper(...)\n   at Oracle.DataAccess.Client.OracleCommand.ExecuteReader(...)\n","ProductVersion":"2.149.1054.0 (25.11)+fe0a1c0f2fc6e9cf0939a7efabedc7cdb3358bad","ActivityId":"86d59d77-7be2-43f3-afc3-8dfab4ef4b1b","Process":"Microsoft.Mashup.Container.NetFX45","Pid":4960,"Tid":1,"Duration":"00:00:19.4224248"}
	
	{"Start":"<span style='background:#DEDEDE;'>2026-01-24T15:10:08.3951188Z</span>","Action":"<strong>Engine/IO/Db/Oracle/Connection/Close</strong>","ResourceKind":"Oracle","ResourcePath":"20.163.1.157:1521/ines.internal.cloudapp.net","HostProcessId":"956","PartitionKey":"Section1/LOCATIONS/2","Server":"20.163.1.157:1521/ines.internal.cloudapp.net","ConnectionId":"957c17d0-db30-4833-a3ab-70b9c1e43935","ProductVersion":"2.149.1054.0 (25.11)+fe0a1c0f2fc6e9cf0939a7efabedc7cdb3358bad","ActivityId":"86d59d77-7be2-43f3-afc3-8dfab4ef4b1b","Process":"Microsoft.Mashup.Container.NetFX45","Pid":4960,"Tid":1,"Duration":"00:00:00.0099732"}

</div>


💡 From this analysis, we can see that just before the error occurs, the connection **open request** and the **query sent** to the server are clearly visible.”

<br>

#### ODP.NET Driver Logs

Knowing the Power BI mashup container process ID PID = <span style="background-color:#D9FAAA">4960</span> obtained from the session info, I was able to **convert it to HEX** to find the driver log file: **<span style="background-color:#D9FAAA">4960</span> (DEC) = <span style="background-color:#D1B69D">1360</span> (HEX)**: <br>
_**MICROSOFT.MASHUP.CONTAINER.NETFX45.EXE_PID_<span style="background-color:#D1B69D">1360</span>_DATE_2026_01_24_TIME_15_02_14_000333.trc**_

<div style="white-space: pre-wrap; font-style: italic; font-size:12px; word-wrap: break-word; max-width: 100%;">
	
	TIME:<span style="background:#DEDEDE;">2026/01/24-15:09:48:961</span> TID:27a0  OracleConnection::Open() => Calling RequestBegin
	(…)
	TIME:2026/01/24-15:09:48:961 TID:27a0  (INFO) (OracleConnectionOCP.Open) Pool Id=531559270 PooledConCtx=40026340 sessid=<span style="background-color:#AAFABE">396</span> serial=<span style="background-color:#42D4BC">44130</span> inst=ines ConCtx=(2405dcbd1a0)
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


💡 Once again, just before the error occurs, the **connection open** request and the **query sent** to the server are visible, with timestamps and details matching those in the Power BI mashup logs.

<br>

#### SQLNET Client Logs

Knowing it was leveraging the driver TID = <span style="background-color:#FADAAA">10144</span> I was able to find the SLQ.NET logs: client_<span style="background-color:#FADAAA">10144</span>.trc

<div style="white-space: pre-wrap; font-style: italic; font-size:12px; word-wrap: break-word; max-width: 100%;">
	
	(10144) <span style="background:#DEDEDE;">[24-JAN-2026 15:02:14:738]</span> nsmore2recv: exit (0)
	(10144) <span style="background:#DEDEDE;">[24-JAN-2026 15:09:48:960]</span> nioctl: entry
	(…)
	(10144) <span style="background:#DEDEDE;">[24-JAN-2026 15:09:48:961]</span> <strong>nsdofls: sending NSPTDA packet</strong>
	(10144) [24-JAN-2026 15:09:48:961] nspsend: entry
	(10144) [24-JAN-2026 15:09:48:961] nspsend: plen=761, type=6
	(10144) [24-JAN-2026 15:09:48:961] nttmwr: entry
	(10144) [24-JAN-2026 15:09:48:962] nttmwr: socket 1712 had bytes written=761
	(10144) [24-JAN-2026 15:09:48:962] nttmwr: exit
	(10144) [24-JAN-2026 15:09:48:962] <strong>nspsend: packet dump</strong>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 02 F9 06 20 00 00  |........|</span>
	(…)
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 00 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: FE 04 01 00 00 73 65 6C  |.....sel|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 65 63 74 20 22 24 4F 72  |ect."$Or|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 64 65 72 65 64 22 2E 22  |dered"."|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 52 45 47 49 4F 4E 5F 49  |REGION_I|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 44 22 2C 0D 0A 20 20 20  |D",.....|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 20 22 24 4F 72 64 65 72  |."$Order|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 65 64 22 2E 22 52 45 47  |ed"."REG|</span>)
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 64 22 2E 22 52 45 47 49  |d"."REGI|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 4F 4E 5F 49 44 22 0D 0A  |ON_ID"..|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 66 65 74 63 68 20 6E 65  |fetch.ne|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 78 74 20 34 30 39 36 20  |xt.4096.|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 72 6F 77 73 20 6F 6E 6C  |rows.onl|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 79 00 00 00 00 01 00 00  |y.......|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 00 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 00 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 00 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 01 00 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 80 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00 00 00 00 00 00 00 00  |........|</span>
	(10144) [24-JAN-2026 15:09:48:962] <span style="background-color:#F6FAAA">nspsend: 00                       |.       |</span>
	(10144) [24-JAN-2026 15:09:48:962] nspsend: 761 bytes to transport
	(10144) [24-JAN-2026 15:09:48:962] <strong>nspsend: normal exit</strong>
	(10144) [24-JAN-2026 15:09:48:962] nsdofls: exit (0)
	(…)
	(10144) [24-JAN-2026 15:10:07:880] ntt2err: entry
	(10144) <span style="background:#DEDEDE;">[24-JAN-2026 15:10:07:897]</span> ntt2err: soc 1712 error - operation=5, ntresnt[0]=517, ntresnt[1]=54, ntresnt[2]=0
	(10144) [24-JAN-2026 15:10:07:897] ntt2err: exit
	(10144) [24-JAN-2026 15:10:07:897] <strong>nttmrd: socket 1712 had bytes read=-1</strong>
	(10144) [24-JAN-2026 15:10:07:897] nttmrd: exit
	(10144) [24-JAN-2026 15:10:07:897] nsprecv: error exit
	(10144) [24-JAN-2026 15:10:07:897] nserror: entry
	(10144) <span style="background:#DEDEDE;">[24-JAN-2026 15:10:07:897]</span> nserror: nsres: id=0, op=68, ns=<strong>12547</strong>, ns2=<strong>12560</strong>; nt[0]=<strong>517</strong>, nt[1]=<strong>54</strong>, nt[2]=0; ora[0]=0, ora[1]=0, ora[2]=0
	(10144) [24-JAN-2026 15:10:07:897] nsrdr: error exit
	(…)
	(10144) [24-JAN-2026 15:10:07:904] nioqer:  <strong>returning err = 3135</strong>
	
</div>

<br>

💡 From this analysis, there is a clear **time gap between 15:02 and 15:09** (while I was not using the Power BI desktop). <br>
Then at 15h09 the logs continue and the data packet being sent is visible: from the [Oracle's documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/troubleshooting-oracle-net-services.html), **NSPTDA** is used with **data packet types**. <br>
Since there was no encryption I could see the packet dump and then correlate with the timestamps and error. <br>

📖 Then, based on the **ntt2err** and **nserror** error codes, I was able to extract additional information:
- **Error Stack Component: NS - Network Session (main and secondary layers)**
  - Main TNS error: ns=**12547**
	- [TNS-12547](https://docs.oracle.com/en/error-help/db/tns-12547/?r=19c): TNS:lost contact - cause: Partner has unexpectedly gone away, usually during process startup.
  - Secondary error: ns2=**12560**;
	- [TNS-12560](https://docs.oracle.com/en/error-help/db/tns-12560/?r=19c): TNS:protocol adapter error - cause: A generic protocol adapter error occurred.
- **Error Stack Component: NT - Network Transport** (main, secondary, and operating system layers)
  - Protocol adapter error: nt[0]=**517**
	- [TNS-00517](https://docs.oracle.com/en/error-help/db/tns-00517/?r=19c): Lost contact - cause: Partner has unexpectedly gone away.
  - OS (windows) error code:  nt[1]=**54**
	- An existing connection was forcibly closed by the remote host.

<img width="500" alt="image" src="https://github.com/user-attachments/assets/62305484-3b72-49ef-a33f-8fb981eb4387" />

<br>

### Server Log Analysis

#### SQLNET Server Logs

Knowing the server process ID  PID = <span style="background-color:#829FED">7900</span> obtained from the process info, I was able to quickly locate the log file to analyze: server_<span style="background-color:#829FED">7900</span>.trc

<div style="white-space: pre-wrap; font-style: italic; font-size:12px; word-wrap: break-word; max-width: 100%;">
	<span style="background:#DEDEDE;">[24-JAN-2026 15:02:14:741]</span> nsbasic_bsd: <strong>packet dump</strong>
	[24-JAN-2026 15:02:14:741] nsbasic_bsd: 00 00 02 BF 06 00 00 00  |........|
(…)
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: 00 00 00 00 DB D9 FF 7F  |........|
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: 00 00 07 02 41 52 09 41  |....AR.A|
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: 72 67 65 6E 74 69 6E 61  |rgentina|
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: 02 C1 03 07 02 42 52 06  |.....BR.|
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: 42 72 61 7A 69 6C 02 C1  |Brazil..|
	(…)
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: 20 66 6F 75 6E 64 0A     |.found. |
	[24-JAN-2026 15:02:14:742] nsbasic_bsd: exit (0)
	[24-JAN-2026 15:02:14:742] nsbasic_brc: entry: oln/tot=0,prd=0
	<span style="background:#DEDEDE;">[24-JAN-2026 15:02:14:742]</span> nttfprd: entry
</div>

💡 The last traces show the **last packet dump** which was successful but the **timestamp was before the issue happens**, matching the previous set of logs that I could see from the client logs at 15:02.

<br>


### Client-Server Network Trace (wireshark)

From the **client network trace**:
- I confirmed the **port** from client side is <span style="background-color:#FAAABF">50679</span> aligning with the Oracle session details.
- I could see the **packet** matching the query that I saw from the client side logs.
- But then, the same query was trying to be **retransmitted**.
- Ultimately, the **connection was closed.**

<img width="1200" alt="image" src="https://github.com/user-attachments/assets/eadf2a67-4c0c-4340-922b-3c6d6d9cc00a" />
<br>
From the **server network trace**, the **same packet could not be found**.
<br>
<img width="1200" alt="image" src="https://github.com/user-attachments/assets/3d9bdd92-ce86-4825-8a5a-3d66b6b57de3" />
<br>
The previous packet in the same stream can be found on the server side matching the previous query at 15:02 as seen in the Power BI mashup logs and SQL NET logs.
<br>
<img width="1200" alt="image" src="https://github.com/user-attachments/assets/5667c792-1b83-47a5-8227-4d8d89a2a842" />
<br>
Checking on the server network trace using the client VM IP 20.14.72.115, there is a time gap with **no traces between 15:02 and 15:11.**
<br>
<img width="1200" alt="image" src="https://github.com/user-attachments/assets/4bc35660-6ff0-4301-81b8-eacfd1ac170f" />

<br>
💡 To summarize:
- Power BI Desktop correctly generated the **query** and used the Oracle driver to send it to the server.
- The client machine did send the packet, but it **never reached the server**.
- Neither the client nor the server VMs had **idle‑timeout settings** or **firewall** rules that could account for the drop.

<br>

❓ Somehow the **connection was getting "lost in Azureland"** and driving me a bit crazy.

<br> 

# The Truth Behind the Mystery

After correlating the logs and realizing the packet never reached the server, I went back to the basics and reviewed my setup.

That’s when I uncovered a critical detail in [Azure’s networking documentation](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses).<br> 
💡 There is a by design **limitation affecting the public IPs of Azure VMs**: 
- _"An adjustable **inbound originated flow idle timeout of 4-30 minutes**, with a **default of 4 minutes**, and **fixed outbound originated flow idle timeout of 4 minutes**"_
Unfortunately I couldn't find any logs from Azure showing the connection drop explicitly but was able to confirm this was the root cause by applying the resolution steps described below.

<br>

# What Actually Solved It

🔧 To **fix the issue**, I narrowed it down to a few workable options:
- **Option 1:** If client machine is not an azure VM, I can **increase the inbound idle timeout up to 30m** (inbound is configurable for the server VM).
- **Option 2:** If I want to keep using the Azure VM as the client:
  - **2A:** Switch to using the **private IP**, avoiding the outbound idle timeout entirely, or
  - **2B:** Configure a **keep‑alive mechanism** so the connection never becomes idle long enough to be dropped.

📌 For Option 2B, enabling a keep‑alive was simple on the server side. <br>
I added the following parameter to the sqlnet config file, setting the interval to the number of minutes I wanted (in this case, 1 minute): _SQLNET.EXPIRE_TIME=1_

✅ I tested each of these options independently, and **all of them successfully resolved the error**, confirming that the root cause was the Azure VM public IP idle‑timeout limitation.

<br>

# Wrapping Up the Mystery

🚀 In the end, what looked like an Oracle‑side issue turned out to be a much simpler but easily overlooked network behavior dictated by Azure VM Public IP’s default idle timeout. 

By walking through logs at each layer (Power BI desktop, ODP.NET unmanaged Oracle driver, SQLNet logs on server and client and network traces), the root cause revealed itself clearly: the query never reached the server because the connection was silently dropped mid‑path. <br>
Understanding the full chain of dependencies was what ultimately gave me clarity. 

I hope this breakdown helps you accelerate your own investigations the next time a connection mysteriously “vanishes.”  <br>
If you’ve encountered similar challenges or found alternative approaches, I’d love to learn from your experience too!

<br>
<br>

---

References:
- [Seamless Power BI and Oracle Integration: Key Learnings & Setup Tips](https://inesmartinsgit.github.io/2026/01/14/pbi-oracle-seamless.html)
- [Power Query Oracle database connector](https://learn.microsoft.com/en-us/power-query/connectors/oracle-database)
- [Power BI Desktop diagnostics collection](https://learn.microsoft.com/en-us/power-bi/fundamentals/desktop-diagnostics)
- [Database Error Messages](https://docs.oracle.com/en/error-help/db/)
- [Introducing Oracle Net Services](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/introducing-oracle-net-services.html)
- [Understanding Oracle Net Architecture](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/understanding-oracle-net-architecture.html)
- [Parameters for the sqlnet.ora File](https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/parameters-for-the-sqlnet.ora.html)
- [Oracle Net Listener Parameters in the listener.ora File](https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/oracle-net-listener-parameters-in-listener-ora.html)
- [Oracle Database Background Process](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/background-processes.html)
- [Evaluating Oracle Net Services Trace Files](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/troubleshooting-oracle-net-services.html)
- [TNS Database Error Messages](https://docs.oracle.com/en/error-help/db/tns-index.html)
