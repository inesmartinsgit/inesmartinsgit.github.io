---
title: "Seamless Power BI and Oracle Integration: Key Learnings & Setup Tips"
---

*Disclaimer: Everything you’ll find here reflects my personal views and is not affiliated with Microsoft or Oracle.*

<br>

**Connecting Power BI to Oracle on-premises databases** is a common scenario for business intelligence professionals, but it’s also a source of frequent frustration. 

Over the years, I’ve encountered and helped configure and troubleshoot a wide range of issues that almost always trace back to **setup and configuration missteps**.

This post is for Power BI users and IT administrators who want to avoid the most common pitfalls when integrating Power BI with Oracle. <br>
Rather than repeating the standard connection steps found in several community posts and blogs, I’ll focus on the **key learnings and practical tips** that are often overlooked but critical for a smooth setup.

I will focus only on the following scenarios:
- Using Power BI desktop or the on-premises data gateway (OPDG) for semantic models and dataflows.
- Using the Oracle built-in connector that requires installing an oracle driver (not the connector in preview which already includes the driver).

By the end of this post, you will know:
- Which components are required and why.
- How to avoid configuration mistakes that can block your connection.
- How to install and validate your setup correctly.

Let’s dive in and make your Power BI and Oracle integration as seamless as possible !

 <br>

# Step 1. Confirm you have a supported database server

At the time of this post, it's required to have **Oracle Database Server 12c (12.1.0.2) and later.**

<br>

# Step 2. Understand the driver requirements

You can connect to oracle using several drivers but to use the built-in oracle connector, you need to install a specific oracle driver.

In particular, for Power BI Desktop, semantic models in Power BI Service and dataflows in Power BI Service, that driver is **ODP.NET unmanaged** (ODP.NET stands for **Oracle Data Provider for .NET**) and it **needs to be 64-bit** version since both Power BI Desktop and OPDG are now only available in 64-bit.

If you have **another driver** (such as a ODBC driver or another flavor of ODP.NET such as ODP.NET managed) it **cannot be used**. The connector was designed and is supported only for ODP.NET unmanaged.

Also, you need to ensure that your ODP.NET unmanaged driver is **compatible** with your Oracle Database Server: Oracle support provides matrices of combability to check this.

<br>

# Step 3. Install the ODP.NET unmanaged driver

ODP.NET unmanaged is free and can be downloaded from Oracle documentation: [Download OCMT](https://www.oracle.com/database/technologies/appdev/ocmt.html)

The latest versions are packaged into **OCMT - Oracle Client for Microsoft Tools**, a graphical installer.


⚠️ **Warning:** The installer is easy to use but try to resist the temptation of clicking NEXT without reading. There are critical steps that require your input and will break your setup if not correct. 


In the example below, if I leave the default location under my user, I can only use the driver with Power BI Desktop and not the OPDG:

<img height="400" alt="image" src="https://github.com/user-attachments/assets/08d8b503-7a78-4cfd-a78d-86876e010423" />


This driver needs to be **installed on each client machine** where you want to connect to Oracle. 

For the purpose of this post it needs to be on the same machine as your Power BI Desktop or OPDG. If your OPDG is a cluster with more than one node/machine, it needs to be installed on all of them and ensure it's the same version.

I strongly recommend **having only one** ODP.NET unmanaged driver version installed **on each machine**. If you have multiple, use the graphical installer to remove the others.

<br>

# Step 4. Validate the Installation

After you completed the installation on the client machine, you can leverage **PowerShell to confirm the driver is correctly installed.**

To do this, run the following command: 
```
[System.Data.Common.DbProviderFactories]::GetFactoryClasses()|?{$_.InvariantName -eq "Oracle.DataAccess.Client"}
```

You will see something like this:

<img  height="130" alt="image" src="https://github.com/user-attachments/assets/9646c2e8-eb62-4380-8630-829597106a3d" />


This confirms that the driver is installed and, for this example, it confirms that is Version 4.122.19.1 (the naming convention is explained in depth in Oracle documentation pages if you want more information).

If this result is blank, it means the driver wasn't properly installed. In that case, use the graphical installer (OCMT) to repair or uninstall and install again.

<br>

# Step 5. Test the connection

If possible, before making the scenario complex, **try to simplify it**.

From the Oracle side, before jumping to complex config files such as TNSNAMES.ORA, I recommend creating a simple test with EZCONNECT (more on this below).

From the BI side, you can consider testing with Power BI desktop which is easier than with the OPDG. 

You can even remove the BI client tool from the setup and test the driver directly using PowerShell or writing your own C\# application (more on that here: [Using ODP.NET Client Provider in a Simple Application (oracle.com)](https://docs.oracle.com/database/121/ODPNT/intro005.htm))

If you want to use **PowerShell**, below is a sample script to open the connection, getting the schema, running a query, reading some results and closing the schema. You can customize it with your own connection string, query and results to be read.

This script uses **EZCONNECT** (this is called **easy connect naming** and, in essence, allows specifying directly the host name, port and service name).

⚠️**Warning**: The script is using hard-coded username/password which is not advised for production environments. For those scenarios, I recommend using secure methods to store the credentials.


```
#Create the connection string
$conn = [System.Data.Common.DbProviderFactories]::GetFactory("Oracle.DataAccess.Client").CreateConnection()
$conn.ConnectionString = "Data Source=20.163.1.157:1521/ines.internal.cloudapp.net;User Id=HR;Password=xxxx“ #---> CHANGE TO YOUR CONNECTION STRING

#Open the connection
$conn.Open()

#Get schema
$conn.GetSchema()

#Run a query - EDIT the select statement below
$sel_stmt = $conn.CreateCommand()
$sel_stmt.CommandText = "SELECT * FROM HR.JOBS WHERE MIN_SALARY > 10000" # ---> CHANGE QUERY
$rdr = $sel_stmt.ExecuteReader()
while ($rdr.Read()) {
   write-host "$($rdr.GetOracleValue(0).Value)" ---> ADAPT TO THE RESULT OF YOUR QUERY
}

#Close the connection
$conn.Close()
```
 
If you want to make it a bit more complex, you can consider including a config file such as **TNSNAMES.ORA** (this is called **local naming** and allows you to create an alias which enables locating the network address leveraging the information on this file).

I recommend by starting to copy the sample of this file provided by OCMT. Depending on where you installed it, you will find a folder including the samples such as: ***C:\Program Files\Oracle Client for Microsoft Tools\network\admin\sample***

<img  height="200" alt="image" src="https://github.com/user-attachments/assets/24b54eae-9b57-4422-abf0-03b99dad6093" />

You can copy both files to the location you specified during the OCMT installation.

For my example, I specified: **C:\Oracle\network\admin** so this is where I copied the samples to.

Then I edited the samples and specified SQLNET.AUTHENTICATION_SERVICES to NONE (username/password access) instead of NTS (Microsoft Windows native operating system authentication). 

For more information on this, consult Oracle's documentation such as [Local Naming Parameters in the tnsnames.ora File](https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/local-naming-parameters-in-tns-ora-file.html).

<img height="600" alt="image" src="https://github.com/user-attachments/assets/3e390bcf-6f13-4283-8d09-9f2a56db7c69"/>

And finally, I connected by using the same PowerShell sample with the **alias** defined by me in the TNSNAMES.ORA file (*inesalias*) instead of EZCONNECT.

```
#Create the connection string
$conn = [System.Data.Common.DbProviderFactories]::GetFactory("Oracle.DataAccess.Client").CreateConnection()
$conn.ConnectionString = "Data Source=inesalias;User Id=HR;Password=xxxx“ #---> CHANGE TO YOUR CONNECTION STRING
 
#Open the connection
$conn.Open()
 
#Get schema
$conn.GetSchema()
 
#Run a query - EDIT the select statement below
$sel_stmt = $conn.CreateCommand()
$sel_stmt.CommandText = "SELECT * FROM HR.JOBS WHERE MIN_SALARY > 10000" # ---> CHANGE TO YOUR QUERY
$rdr = $sel_stmt.ExecuteReader()
while ($rdr.Read()) {
   write-host "$($rdr.GetOracleValue(0).Value)" ---> ADAPT TO THE RESULT OF YOUR QUERY
}
 
#Close the connection
$conn.Close()

```
 
<br>
Thank you for following along with these key learnings and practical tips for connecting Power BI to Oracle on-premises databases. 

With careful setup and validation, you’ll save time and avoid the most common frustrations. 

If you run into unique challenges, remember that both Microsoft and Oracle communities are great places to seek help. 

I’ll be sharing more real-world troubleshooting scenarios and advanced tips in future posts, stay tuned ! 

Your feedback and topic suggestions are always welcome.

<br>
<br>
---

References:

- Microsoft's official documentation on the Oracle connector:
   - [PowerQuery Oracle Connector](https://learn.microsoft.com/en-us/power-query/connectors/oracle-database)
   - [Connect to an Oracle database with Power BI Desktop](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-connect-oracle-database)

- Oracle's official documentation to download the driver: [
Connect Microsoft Tools to Oracle Databases](https://www.oracle.com/database/technologies/appdev/ocmt.html)
