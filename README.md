![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/Mikes_Data_Work_Header_890x170.png "Mikes Data Work")        

# Create SQL Alert For Restore Latency With SQL HTML CSS
**Post Date: November 13, 2020**


## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)      

## About-Process

<p> I created this SQL Alert process to notify me where any latency is detected in a SQL Transactional Migration systems aka SQL Transaction Log Shipping. It’s built around SMTP notification, and this must be already configured on your system. All you need to do is plug in the necessary SMTP information in the logic below, and place it into a Job. That’s it. This can be copied to any destination (or DR) SQL system on the enterprise.

It shows the following information:

Server Name
Instance Name

SQL Build
SQL Edition
Complete Instance Connection String
Net Bios Name

Instance Name (if any)

Port
Dynamic Port (if any)
Source Database Server
Destination Database Server
Last Restore Date

Latency (in weeks and/or number of days)
Lists any database where latency is detected. </p>

![Create SQL Alert For Restore Latency With SQL HTML CSS]( https://mikesdatawork.files.wordpress.com/2020/11/image001-2.png "Create SQL Alert For Restore Latency With SQL HTML CSS")
 
     
## SQL-Logic
```SQL
use master 
set nocount on
go 
----------------------------------------------------------------------------------------
-- configure database mail and create SQLAlert profile with SMTP server name
 
if (select sum(cast([value_in_use] as int)) from master.sys.configurations where [configuration_id] in ('518', '16386', '16388', '16390'))  <> 4
    begin
        exec master..sp_configure   'show advanced options',	1 reconfigure
        exec master..sp_configure   'Ole Automation Procedures',1 reconfigure with override
        exec master..sp_configure   'Database Mail XPs',		1 reconfigure with override
	exec master..sp_configure		'xp_cmdshell',				1 reconfigure with override
    end
  
if not exists(select * from msdb.dbo.sysmail_profile where  name = 'SQLAlerts')  
  begin
    execute msdb.dbo.sysmail_add_profile_sp 
      @profile_name 	= 'SQLAlerts', 
      @description  	= 'SQLDatabaseMail'; 
  end
    
  if not exists(select * from msdb.dbo.sysmail_account where  name = 'SQLAlerts@MyDomain.com') 
  begin
    execute msdb.dbo.sysmail_add_account_sp 
    @account_name            	= 'SQLAlerts@MyDomain.com', 
    @email_address           	= '', 
    @mailserver_name        	= 'mailer.MyDomain.com',
    @mailserver_type         	= 'SMTP', 
    @port                   	= '25', 
    @use_default_credentials 	=  0 , 
    @enable_ssl              	=  0 ; 
  end
    
if not exists
			(select * 
              from msdb.dbo.sysmail_profileaccount spa
                inner join msdb.dbo.sysmail_profile p on spa.profile_id = p.profile_id 
                inner join msdb.dbo.sysmail_account a on spa.account_id = a.account_id   
              where p.name = 'SQLAlerts' 
                and a.name = 'SQLAlerts@MyDomain.com'
			)  
  begin
    execute msdb.dbo.sysmail_add_profileaccount_sp 
      @profile_name = 'SQLAlerts', 
      @account_name = 'SQLAlerts@MyDomain.com', 
      @sequence_number = 1; 
  end
 
----------------------------------------------------------------------------------------
-- create variables to capture server, instance and port info
 
declare     @instance_name_basic        varchar(255)
declare     @server_name_instance_name  varchar(255)
declare     @server_time_zone			varchar(255)
set	@instance_name_basic        		= (select cast(serverproperty('servername') as varchar(255)))
set	@server_name_instance_name			= (select upper(@@servername))
declare     @statements					nvarchar(max)
declare     @original					int
declare     @iid						int
declare     @maxid						int
declare     @domain						nvarchar(255)
declare     @ports						table ([PortType] nvarchar(180), Port int)
exec        master..xp_regread 'HKEY_LOCAL_MACHINE', 'SYSTEM\CurrentControlSet\services\Tcpip\Parameters', N'Domain',@Domain output
 
----------------------------------------------------------------------------------------
-- create and capture all other instance information found on the server
declare     @instances  table
(
    [InstanceID]    	int identity(1, 1) not null primary key
,   [InstanceName]  	nvarchar(180)
,   [InstanceFolder]	nvarchar(50)
,   [StaticPort]    	int
,   [DynamicPort]   	int
);
 
declare     @xp_msver_platform  table ([id] int,[name] varchar(180),[InternalValue] varchar(50), [Charactervalue] varchar (50))
declare     @platform       varchar(10)
insert into @xp_msver_platform exec master..xp_msver platform
select  @platform = (select 1 from @xp_msver_platform where [Charactervalue] like '%86%')
If  @platform is NULL
Begin
    insert into @instances ([InstanceName], [InstanceFolder])
    exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
end
else
Begin
    insert into @instances ([InstanceName], [InstanceFolder])
    exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
end
  
declare     @Key table ([KeyValue] int)
insert into @Key
exec master..xp_regread'HKEY_LOCAL_MACHINE', N'SOFTWARE\Wow6432Node\Microsoft\Microsoft SQL Server\Instance Names\SQL';
select  @original= [KeyValue] from @Key
If      @original=1
        insert into @instances ([InstanceName], [InstanceFolder])
        exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Wow6432Node\Microsoft\Microsoft SQL Server\Instance Names\SQL';
  
select  @maxid = max([InstanceID]), @iid = 1
from    @instances
while   @iid <= @maxid
    begin
        delete from @ports
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Microsoft\\Microsoft SQL Server\' 
        + [instancefolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPDynamicPorts'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Microsoft\\Microsoft SQL Server\' 
        + [instancefolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPPort'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Wow6432Node\Microsoft\\Microsoft SQL Server\' 
        + [instancefolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPDynamicPorts'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Wow6432Node\Microsoft\\Microsoft SQL Server\' 
        + [InstanceFolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPPort'''
        from        @instances  where [instanceid] = @iid
        insert into @ports  exec sp_executesql @statements
        update si   set [staticport] = p.port, [dynamicport] = dp.port from @instances si
        inner join @ports dp on dp.[porttype] = 'TCPDynamicPorts' join @ports p on p.[porttype] = 'TCPPort'
        where [instanceid] = @iid;
        set @iid = @iid + 1
    end
 
----------------------------------------------------------------------------------------
-- create and capture SQL Version info.
declare @version_info   table
(
    [Server_Name]       	varchar(255)
,   [SQL_Instance]      	varchar(255)
,   [SQL_Version]       	varchar(255)
,   [SQL_Build]				varchar(255)
,   [SQL_Edition]       	varchar(255)
)
 
insert into @version_info
select
    'ServerName'      = cast(serverproperty('machinename') as varchar)
,   'SQLInstance'     = cast(upper(@@servername) as varchar)
,   'SQLVersion'      =   
case
when left(cast(serverproperty('productversion') as varchar), 4) = '14.0' then 'SQL 2017 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '13.0' then 'SQL 2016 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '12.0' then 'SQL 2014 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '11.0' then 'SQL 2012 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '10.5' then 'SQL 2008 R2 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '10.0' then 'SQL 2008 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '9.00' then 'SQL 2005 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '8.00' then 'SQL 2000 ' + cast(serverproperty('productlevel') as varchar)
end
,   'SQLBuild'        = cast(serverproperty('productversion') as nvarchar(25))
,   'SQLEdition'      = cast(serverproperty('edition') as varchar)

----------------------------------------------------------------------------------------
-- create and capture latency info (for log shipping)

declare @ls_latency_info	table 
(
	[database]			    varchar(255)
,	[primary_server]	    varchar(255)
,	[secondary_server]	    varchar(255)
,	[last_restored_date] 	datetime
,	[latency]			    varchar(255)
)

insert into @ls_latency_info
select
	'database_name'	= [primary_database]
,	'from_primary'	= [primary_server]
,	'to_secondary'	= [secondary_server]
,	'last_restored'	= [last_restored_date]
,	'latency\delay'	= 
	case	
		when cast(datediff(second, [last_restored_date], getdate()) as varchar) between 1 and 59 
			then cast(datediff(second, [last_restored_date], getdate()) as varchar) + ' seconds'
		when cast(datediff(second, [last_restored_date], getdate()) as varchar) between 60 and 3599 
			then cast(datediff(hour, [last_restored_date], getdate()) as varchar) + ' hours'
		when cast(datediff(second, [last_restored_date], getdate()) as varchar) between 3600 and 86399 
			then cast(datediff(day, [last_restored_date], getdate()) as varchar) + ' days'
		when cast(datediff(second, [last_restored_date], getdate()) as varchar) between 86400 and 2591999 
			then cast(datediff(week, [last_restored_date], getdate()) as varchar) + ' weeks - (' + cast(datediff(day, [last_restored_date], getdate()) as varchar) + ' Days)'
		when cast(datediff(second, [last_restored_date], getdate()) as varchar) between 2592000 and 31535999 
			then cast(datediff(month, [last_restored_date], getdate()) as varchar) + ' months - (' + cast(datediff(week, [last_restored_date], getdate()) as varchar) + ' Weeks)'
		when cast(datediff(second, [last_restored_date], getdate()) as varchar) > 31536000 
			then cast(datediff(year, [last_restored_date], getdate()) as varchar) + ' years'
	end
from 
	msdb..[log_shipping_monitor_secondary]

select count(*) from @ls_latency_info


----------------------------------------------------------------------------------------
-- create email warning message

declare 	@shipping_latency	varchar(255)
set	        @shipping_latency = 'Warning: Restore latency detected in Log Shipping'


----------------------------------------------------------------------------------------
-- create HTML\xml variables

declare	@message_body		nvarchar(max)
declare @html_block_begin	nvarchar(1000)
declare	@html_block_version	nvarchar(max)
declare @html_block_instance	nvarchar(max)
declare	@html_block_latency	nvarchar(max)
declare @html_block_end		nvarchar(1000)

declare	@xml_latency_info	nvarchar(max)
declare @xml_version_info   nvarchar(max)
declare @xml_instance_info  nvarchar(max)
declare @xml_drive_info		nvarchar(max)
declare @xml_last_backup    nvarchar(max)
declare @xml_paths_used     nvarchar(max)

----------------------------------------------------------------------------------------
-- set XML latency_info
 
set @xml_latency_info = 
    cast(
        (select
            [database]                  as 'td'
        ,   ''
        ,   [primary_server]            as 'td'
        ,   ''
        ,   [secondary_server]          as 'td'
        ,   ''
        ,   cast([last_restored_date]   as varchar)	as 'td'
        ,   ''
        ,   [latency]			        as 'td'
 
        from @ls_latency_info 
        order by [database] asc 
        for xml raw('tr')
    ,   elements, type)
    as nvarchar(1000)
        )
 
----------------------------------------------------------------------------------------
-- set XML version_info
 
set @xml_version_info = 
    cast(
        (select
            [Server_Name]       as 'td'
        ,   ''
        ,   [SQL_Instance]      as 'td'
        ,   ''
        ,   [SQL_Version]       as 'td'
        ,   ''
        ,   [SQL_Build]	        as 'td'
        ,   ''
        ,   [SQL_Edition]       as 'td'
 
        from @version_info 
        order by [SQL_Instance] asc 
        for xml raw('tr')
    ,   elements, type)
    as nvarchar(1000)
        )
 
----------------------------------------------------------------------------------------
-- set XML for instance_info
set @xml_instance_info = 
    cast(
        (select
            cast(serverproperty('MachineName') as nvarchar) + '.' + lower(@domain) +  + case when [InstanceName] = 'MSSQLServer' then '' else '\' + [InstanceName] end
        +   case when [StaticPort] is null then ',' + cast([DynamicPort] as varchar) else ',' + cast([StaticPort] as varchar) end as 'td'
        ,   ''
        ,    cast(serverproperty('ComputerNamePhysicalNetBIOS') as varchar) as 'td'
        ,   ''
        ,    [InstanceName]					                                as 'td'
        ,   ''
        ,    isnull(cast([StaticPort] as varchar), '')				        as 'td'
        ,   ''
        ,    isnull(cast([DynamicPort] as varchar), '')			            as 'td'
 
        from @instances 
        order by [InstanceName] asc 
        for xml raw('tr')
    ,   elements, type)
    as nvarchar(2000)
        )

----------------------------------------------------------------------------------------
-- create html message blocks

set @html_block_begin =
'<html><head><style>
BODY {background-color:#141414; line-height:1px; -webkit-text-size-adjust:none; color: #ccc; font-family: sans-serif;}
        H1 {font-size: 90%; color: #bbb;}
        H2 {font-size: 90%; color: #bbb;}
        H3 {color: #bbb;}                  
        TABLE, TD, TH {
            font-size: 87%;
            border: 1px solid #bbb;
            border-collapse: collapse;
        }                         
        TH {
            font-size: 87%;
            text-align: left;
            background-color: #1A1B20;
            color: #f8ab0c;
            padding: 4px;
            padding-left: 7px;
            padding-right: 7px;
        }
        TD {
            font-size: 87%;
            padding: 4px;
            padding-left: 7px;
            padding-right: 7px;
            max-width: 100px;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }
</style></head><body>'

set @html_block_version = 
'
<p style="font-family:sans-serif; font-size:20px; color:#f8ab0c;">SQL Alert - Log Shipping Restore Latency</p>
<p style="color:#f8ab0c;">Database Server: ' + @server_name_instance_name + '    ' + @shipping_latency + '</p>
<p>SQL Version</p>
<table border="1">
        <tr>
            <th> SERVER NAME  </th>
            <th> SQL INSTANCE </th>
            <th> SQL VERSION  </th>
            <th> SQL BUILD	</th>
            <th> SQL EDITION  </th>
        </tr>'        
+ @xml_version_info + '</table>
<br>
'

set @html_block_instance = 
'
<p>SQL Instances On This Server</p>
<table border="1">
        <tr>
            <th> INSTANCE CONNECTION STRING </th>
            <th> NET BIOS NAME </th>
            <th> INSTANCE NAME </th>
            <th> STATIC PORT </th>
            <th> DYNAMIC PORT </th>
        </tr>'        
+ @xml_instance_info + '</table>
<br>
'

set @html_block_latency = 
'
<p>Log Shipping Restore Latency</p>
<table border="1">
        <tr>
            <th> DATABASE		    </th>
            <th> PRIMARY SERVER		</th>
            <th> SECONDARY SERVER	</th>
	        <th> LAST RESTORE DATE	</th>
            <th> LATENCY			</th>
        </tr>'
  
+ @xml_latency_info + '</table>
<br>
'
set @html_block_end =
'<br></body></html>'

----------------------------------------------------------------------------------------
-- combine HTML formatted blocks

set @message_body =
	@html_block_begin
+	@html_block_version
+	@html_block_instance
+	@html_block_latency
+	@html_block_end

/*
select * from @version_info
select * from @instances
select * from @ls_latency_info
*/

----------------------------------------------------------------------------------------
-- set message subject.

declare @message_subject    varchar(1000)
set		@message_subject	= 'SQL Latency detected on [' + @instance_name_basic + ']... '
select (@message_subject)
 
----------------------------------------------------------------------------------------
-- send email.

if (select top 1 cast(datediff(day, [last_restored_date], getdate()) as varchar) from msdb..[log_shipping_monitor_secondary]) > 3
    begin
        exec    msdb.dbo.sp_send_dbmail
                @PROFILE_NAME   = 'SQLAlerts'
            ,   @RECIPIENTS		= 'MyEmail@MyDomain.com'
            ,   @SUBJECT		= @message_subject
            ,   @BODY			= @message_body 
            ,   @BODY_FORMAT    = 'HTML';
    end
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/Mikes_Data_Work_Social.png "Mikes Data Work")
