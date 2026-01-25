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





