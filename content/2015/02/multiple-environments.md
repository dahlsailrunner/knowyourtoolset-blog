---
title: "Supporting Multiple Environments with a single XML Configuration file" # Title of the blog post.
date: 2015-02-17T15:46:10-05:00 # Date of post creation.
description: "Using an XML configuration file that can support multiple runtime environments in .NET" # Description used for search engine.
codeMaxLines: 20 # Override global value for how many lines within a code block before auto-collapsing.
tags:
  - C#
  - .NET
---

## Abstract 
Provide a technique for setting up configuration files that can support multiple environments (e.g. staging and production) and allow for central deployment without having to update config files on the fly during deployment.  Benefits of the approach described:

* config files are not touched during deployment
* developers are forced to think about configuration for EVERY environment at the start of the development process if their effort will involve new configuration points (like a new database or something)
* key configuration treated as a cross-cutting concern and managed centrally rather than repeating items in web.config files and other places (this makes a big difference if you have different applications that should be sharing information like database connections, server names, etc)

## Background
Any good development organization should have at least a couple of different environments that support a full application life cycle management: development itself, one or more testing areas, and a production environment are good starting points.  This article is not meant to comment on which precise environments you have and their purpose — but rather acknowledges that you should probably have more than one and then addresses how to add support in your application code and deployment practices for those environments.

For each environment you probably have some metadata that helps define the environment.  Such pieces of metadata might be:

{{% notice warning CAUTION %}}
This approach should not be used to store any ***secret*** values in a configuration file that finds its way into source control.  At a minimum, inject the secret values
during a deployment or override the non-secret values in this file with secret ones from another configuration source (e.g. environment variables).  But this approach 
would work very well for non-secret values that differ from environment to environment.
{{% /notice %}}

* Database connection strings
* Network folders (logging, files, reports, etc)
* Web service URLs
* Server names

## Approach
The basic approach involves three basic parts:

* A `common.config` file that will contain the configuration information for ALL of your environments in different sections
* A separate `environment.config` file that is placed on every machine running any of your applications and simply indicates which environment that machine is configured to use
* A simple custom architecture assembly that will be referenced by your applications and provide the means to access the configuration

### The Config files and Code
#### common.config
Create a configuration file called `common.config` (or some other name you like) and make it look something like this (I’m  guessing you can figure out what the sample values are — you’ll see how to make these your own and even add more in the custom architecture code below) — specifically note the three SECTIONS for dev (development), tes (test), and prd (production) — you can create whatever sections you want:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="CustomConfigSection.dev" type="MyCompany.Architecture.Common.CustomConfiguration, MyCompany.Architecture.Common"/>
    <section name="CustomConfigSection.tes" type="MyCompany.Architecture.Common.CustomConfiguration, MyCompany.Architecture.Common"/> 
    <section name="CustomConfigSection.prd" type="MyCompany.Architecture.Common.CustomConfiguration, MyCompany.Architecture.Common"/>
  </configSections>
  <CustomConfigSection.dev    
     myMainDbConnection="Server=myDevDbServer;Initial Catalog=myDevDb;Connection Timeout=60;Trusted_Connection=Yes"
     myFileDir="C:\Files"
     myLogPath="C:\Logs"
     myReportService="http://myreportserver/reportserver/reportservice2010.asmx"
     myReportExecutionService="http://myreportserver/reportserver/reportexecution2005.asmx"    
  />
  <CustomConfigSection.tes
     myMainDbConnection="Server=myTesDbServer;Initial Catalog=myTesDb;Connection Timeout=60;Trusted_Connection=Yes"
     myFileDir="C:\Files"
     myLogPath="C:\Logs"
     myReportService="http://testreportserver/reportserver/reportservice2010.asmx"
     myReportExecutionService="http://testreportserver/reportserver/reportexecution2005.asmx"    
  />
  <CustomConfigSection.prd    
     myMainDbConnection="Server=myDevDbServer;Initial Catalog=myDevDb;Connection Timeout=60;Trusted_Connection=Yes"
     myFileDir="\\prdfileshare\files"
     myLogPath="\\prdfilesshare\logs"
     myReportService="http://prdreports/reportserver/reportservice2010.asmx"
     myReportExecutionService="http://prdreports/reportserver/reportexecution2005.asmx"    
  />
</configuration>
```

As noted, the above is merely a sample and should contain the environment-specific configuration values appropriate for your company / project.

#### environment.config

Create a second configuration file called `environment.config` (or a name of your choosing) and make it look like this:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
<appSettings>
  <add key="environment" value="dev"/>
</appSettings>
</configuration>
```
#### The Library Code
The library code is what will expose your configuration to the application.  You need a class that represents your configuration options, and then an object that reads and exposes that configuration to the rest of your application code, wherever it may be.

Here’s the class representing the configuration options presented in the common.config above – note that the class name and assembly name for this should match what you have in the TYPE value of the common.config section nodes.  You should note the connection between the configuration property attribute values and the attribute values from the common.config file above — this file defines what you expect that section to look like — so make it whatever you want.  Refer to the .Net ConfigurationProperty documentation for more information if needed.

```csharp
public class CustomConfiguration : ConfigurationSection
{
 [ConfigurationProperty("myMainDbConnection", IsRequired = true)]
 public string MyMainDbConnection
 {
  get { return (string)this["myMainDbConnection"]; }
  set { this["myMainDbConnection"] = value; }
 }
 [ConfigurationProperty("myFileDir", IsRequired = true)]
 public string MyFileDir
 {
  get { return (string)this["myFileDir"]; }
  set { this["myFileDir"] = value; }
 }
 [ConfigurationProperty("myLogPath", IsRequired = true)]
 public string MyLogPath
 {
  get { return (string)this["myLogPath"]; }
  set { this["myLogPath"] = value; }
 }
 [ConfigurationProperty("myReportService", IsRequired = true)]
 public string MyReportService
 {
  get { return (string)this["myReportService"]; }
  set { this["myReportService"] = value; }
 }
 [ConfigurationProperty("myReportExecutionService", IsRequired = true)]
 public string MyReportExecutionService
 {
  get { return (string)this["myReportExecutionService"]; }
  set { this["myReportExecutionService"] = value; }
 }
}
```
Ok, so now that we have the class representing the options, we need to read it and provide it to the application.  Turns out that’s pretty simple.  Here’s some more code (notes follow the code):

```csharp
public class Config
{
    public CustomConfiguration CustomConfig { get; private set; }
 
    private static Config _instance;
    static readonly object Singleton = new object();
 
    private static string _cfgPath = @"..\cfg";
 
    public static Config Instance
    {
        get
        {
            lock (Singleton)
            {
                if (_instance == null)
                {
                    _instance = new Config();
                }
                return _instance;
            }
        }
    }
 
    private Config()
    {
        const string envConfigFile = @"environment.config";
        const string configFile = @"common.config";
 
        //if (!Directory.Exists(_cfgPath))
        // _cfgPath = UtilityMethods.GetCommonPath();
 
        var envConfigFileMap = new ExeConfigurationFileMap {ExeConfigFilename = Path.Combine(_cfgPath, envConfigFile)};
        var envConfig = ConfigurationManager.OpenMappedExeConfiguration(envConfigFileMap, ConfigurationUserLevel.None);
 
        var configFileMap = new ExeConfigurationFileMap { ExeConfigFilename = Path.Combine(_cfgPath, configFile) };
 
        var config = ConfigurationManager.OpenMappedExeConfiguration(configFileMap, ConfigurationUserLevel.None);
 
        EnvCd = envConfig.AppSettings.Settings["environment"].Value.ToLower();
        CustomConfig = (CustomConfiguration)config.GetSection("CustomConfigSection." + EnvCd);
    }
}
```

This code is where it all comes together.  A utility class called `Config` is built as a singleton class to avoid reading the config file 
over and over.  The first time it is referenced it will read the file with the private constructor.  If you ignore the location of the 
file (`_cfgPath`) for a second, the code simply reads your `environment.config` file to see which environment applies here.  Then it reads 
the corresponding section from `common.config` and stores that configuration in the CustomConfig property which any piece of application 
code can read from.

Regarding the file location:  you can easily pick some standard area (we picked a custom folder under `Program Files (x86)` for ours 
that will ALWAYS be the location of your config files.  Then just leave the 2 commented out lines in the constructor commented out and make 
sure you initialize the path correctly in the top of the class.  We chose to read from a local path to the source executables when available 
to aid developers, and to fall back to a standard path if it was not available.  It’s not a huge difference, so just choose 
an approach for locating the files that works for you.

#### The Application Code

The application code is really simple, then, for ANY application (web site, web service, web api, console app, wcf service, wpf app; even other 
class libraries) to access.  You just include a reference to your library assembly containing the `Config` class (and 
your `CustomConfiguration` class as well, I would imagine) and then write some code that looks like this:

```csharp
var connStr = Config.Instance.CustomConfig.MyMainDbConnection;
```
Pretty easy — and you get strong types with IntelliSense to boot!!!

###  Deployment
In order to set this up on all of your different machines, you need to get the `common.config` and `environment.config` put on 
each machine.  Then you have a one-time-only touch of the `environment.config` on each machine to set which environment it belongs to.  You can 
deploy the `common.config` file with every deploy action or however you deem appropriate.  The environment.config file is generally 
never touched unless your machines change the environments they support.  For us, staging machines will always be staging machines, 
and the same for production, so we never touch them.

That’s it!  Happy configuration.  🙂