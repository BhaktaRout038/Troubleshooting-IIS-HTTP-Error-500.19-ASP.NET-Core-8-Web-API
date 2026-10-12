# Troubleshooting IIS HTTP Error 500.19 – ASP.NET Core 8 Web API

## 1. Overview

This document describes how to troubleshoot and resolve the following IIS error when hosting an ASP.NET Core 8 Web API on Windows.

**Application:** `DataAPI`

**Deployment path:** `D:\Host`

**Target framework:** .NET 8

**IIS URL:** `http://192.168.29.82:8032/`

### Error Details

| Property | Value |
|---|---|
| HTTP Error | 500.19 – Internal Server Error |
| Error Code | `0x8007000d` |
| Configuration File | `D:\Host\web.config` |
| IIS Module | IIS Web Core |
| Required Module | `AspNetCoreModuleV2` |

### Root Cause

The ASP.NET Core Module V2 was initially missing from the full IIS installation.

The module DLL and IIS configuration schema were not found in their expected locations. The DLL was initially present only under IIS Express:

```text
C:\Program Files\IIS Express\Asp.Net Core Module\V2\aspnetcorev2.dll
```

IIS Express and full IIS use separate module installations. Having the module installed for IIS Express does not automatically register it for full IIS.

The issue was resolved by installing or repairing the appropriate ASP.NET Core Hosting Bundle, restarting IIS, and verifying that `AspNetCoreModuleV2` was registered.

---

## 2. Prerequisites

Before troubleshooting, verify the following:

- Windows with IIS installed.
- Administrator access to the server.
- .NET 8 ASP.NET Core application published successfully.
- Application files available under `D:\Host`.
- IIS website configured to use the correct physical path and port.

The published application should contain files similar to:

```text
D:\Host\
├── DataAPI.dll
├── DataAPI.exe
├── DataAPI.deps.json
├── DataAPI.runtimeconfig.json
├── web.config
└── appsettings.json
```

Additional application files and folders may be present.

---

## 3. Step 1 – Verify the Target Framework

Open Command Prompt as Administrator and execute:

```cmd
type D:\Host\DataAPI.runtimeconfig.json
```

The application configuration showed the following target framework:

```json
{
  "runtimeOptions": {
    "tfm": "net8.0",
    "frameworks": [
      {
        "name": "Microsoft.NETCore.App",
        "version": "8.0.0"
      },
      {
        "name": "Microsoft.AspNetCore.App",
        "version": "8.0.0"
      }
    ]
  }
}
```

This confirms that the application targets .NET 8.

**Important:** Install the Hosting Bundle that supports the application's target framework. For this application, use the .NET 8 Hosting Bundle.

---

## 4. Step 2 – Verify the IIS ASP.NET Core Module

Run the following command:

```cmd
%windir%\System32\inetsrv\appcmd.exe list modules
```

To display only ASP.NET Core-related modules:

```cmd
%windir%\System32\inetsrv\appcmd.exe list modules | findstr /i "AspNetCore"
```

### Expected output

```text
MODULE "AspNetCoreModuleV2" ( native, preCondition: )
```

### Problem identified

Initially, the command returned no ASP.NET Core module entries.

This indicated that the required module was not registered in full IIS.

---

## 5. Step 3 – Verify the Module DLL

Check whether the ASP.NET Core Module V2 DLL exists:

```cmd
dir "%ProgramFiles%\IIS\Asp.Net Core Module\V2\aspnetcorev2.dll"
```

The expected path is:

```text
C:\Program Files\IIS\Asp.Net Core Module\V2\aspnetcorev2.dll
```

Initially, this file was missing.

The following command also returned no result:

```cmd
dir "%windir%\System32\inetsrv\config\schema\aspnetcore_schema_v2.xml"
```

The module registration was also absent from the registry:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\IIS Extensions\IIS AspNetCore Module V2"
```

These checks indicated that the full IIS module installation was incomplete or missing.

---

## 6. Step 4 – Download the .NET 8 Hosting Bundle

Download the Hosting Bundle from Microsoft's official website:

**Official download page:**  
https://dotnet.microsoft.com/en-us/download/dotnet/8.0

### Select the correct package

1. Open the .NET 8 download page.
2. Find the ASP.NET Core Runtime section.
3. Download the **Hosting Bundle for Windows x64**.
4. Do not download only the standard .NET Runtime.
5. Run the installer as Administrator.
6. If an existing Hosting Bundle installation is detected, use the Repair option if available.
7. Wait for the installer to finish.

The Hosting Bundle installs the ASP.NET Core runtime components and IIS integration required to host ASP.NET Core applications in full IIS.

Official Microsoft documentation:

https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/hosting-bundle

---

## 7. Step 5 – Restart IIS

After the Hosting Bundle installation or repair, open Command Prompt as Administrator.

Execute:

```cmd
iisreset
```

Expected output:

```text
Attempting stop...
Internet services successfully stopped
Attempting start...
Internet services successfully restarted
```

If necessary, restart the Windows Process Activation Service and IIS services during a maintenance window:

```cmd
net stop was /y
net start w3svc
```

**Warning:** Restarting IIS affects all websites hosted by that IIS instance. Perform this operation during an appropriate maintenance window on production servers.

---

## 8. Step 6 – Verify the Module After Installation

Run:

```cmd
%windir%\System32\inetsrv\appcmd.exe list modules | findstr /i "AspNetCore"
```

The module should now appear:

```text
MODULE "AspNetCoreModuleV2" ( native, preCondition: )
```

Verify that the DLL exists:

```cmd
dir "%ProgramFiles%\IIS\Asp.Net Core Module\V2\aspnetcorev2.dll"
```

Expected directory:

```text
C:\Program Files\IIS\Asp.Net Core Module\V2
```

The module DLL was successfully found at this location after the installation was corrected.

This confirmed that the module was installed and registered in full IIS.

---

## 9. Step 7 – Verify the web.config File

Open:

```text
D:\Host\web.config
```

The configuration for this application is:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore"
             path="*"
             verb="*"
             modules="AspNetCoreModuleV2"
             resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet"
                  arguments=".\DataAPI.dll"
                  stdoutLogEnabled="false"
                  stdoutLogFile=".\logs\stdout"
                  hostingModel="inprocess" />
    </system.webServer>
  </location>
</configuration>
```

### Configuration checks

- The file must be named `web.config`, not `web.config.txt`.
- The `AspNetCoreModuleV2` module name must be correct.
- The DLL referenced by `arguments` must exist in `D:\Host`.
- The configuration must be valid XML.
- The IIS website's physical path must point to the published application directory.
- The Hosting Bundle must support the application's target framework.

The XML provided for this application was structurally valid. The primary issue identified during troubleshooting was the missing IIS module.

**Do not replace a working configuration unnecessarily.** Preserve any additional application-specific settings in your existing file.

---

## 10. Step 8 – Validate the IIS Configuration

Run the following command:

```cmd
%windir%\System32\inetsrv\appcmd.exe list config /section:system.webServer/aspNetCore
```

If IIS recognizes the ASP.NET Core configuration section, the command should return the section's configuration rather than an unknown-section or invalid-configuration error.

You can also check the handlers section:

```cmd
%windir%\System32\inetsrv\appcmd.exe list config /section:system.webServer/handlers
```

If either command fails, record the complete error message and inspect the IIS configuration schema and module installation.

---

## 11. Step 9 – Test the API Directly

Before testing through IIS, verify that the application can start independently.

Run:

```cmd
cd /d D:\Host
dotnet --info
dotnet DataAPI.dll
```

Check for startup exceptions, missing dependencies, and runtime errors.

If the application starts successfully, it should report the URL on which it is listening.

Press `Ctrl+C` to stop the manually started application after testing.

**Note:** Running the DLL directly is a diagnostic test. It does not prove that IIS configuration, application-pool settings, or file permissions are correct.

---

## 12. Step 10 – Test Through IIS

Open the application URL:

```text
http://192.168.29.82:8032/
```

If the application has a health-check endpoint, test that endpoint as well.

For example:

```text
http://192.168.29.82:8032/health
```

Use `/health` only if that endpoint is implemented in the application.

### Interpreting the result

| Result | Meaning |
|---|---|
| HTTP 200 | The requested endpoint responded successfully. |
| HTTP 404 | The server responded, but the requested route may not exist. |
| HTTP 500.19 | IIS configuration is still invalid. |
| HTTP 500.30 | The ASP.NET Core application failed to start in-process. |
| HTTP 502.5 | The application process failed to start under the relevant hosting configuration. |
| HTTP 401 or 403 | Authentication or authorization may be required. |

A response other than HTTP 200 does not automatically mean the Hosting Bundle installation failed. Diagnose the new error separately.

---

## 13. Step 11 – If HTTP 500.19 Still Occurs

If the original error persists after the module is registered, collect the updated IIS error details.

### Check the IIS error page

Record:

- HTTP Error
- Error Code
- Module
- Notification
- Config Error
- Config File
- Config Source

### Check Event Viewer

1. Press `Win + R`.
2. Enter `eventvwr.msc`.
3. Open **Windows Logs → Application**.
4. Review recent IIS, WAS, and ASP.NET Core Module errors.
5. Record the complete error details.

### Check the IIS module schema

Run:

```cmd
dir "%windir%\System32\inetsrv\config\schema\aspnetcore_schema_v2.xml"
```

If the schema is missing, investigate the Hosting Bundle installation and its installer logs.

### Check the IIS application pool

In IIS Manager:

1. Open **Application Pools**.
2. Select the pool assigned to the application.
3. Open **Basic Settings**.
4. For a standalone ASP.NET Core application, set **.NET CLR Version** to **No Managed Code**.
5. Verify the website is assigned to the correct application pool.

Also confirm that the IIS application-pool identity has the necessary read and execute permissions on the published application directory.

---

## 14. Final Verification Checklist

Use this checklist after troubleshooting:

- [ ] Confirmed the application targets .NET 8.
- [ ] Installed or repaired the .NET 8 Hosting Bundle.
- [ ] Restarted IIS.
- [ ] Confirmed `AspNetCoreModuleV2` appears in the IIS module list.
- [ ] Confirmed `aspnetcorev2.dll` exists in the full IIS module directory.
- [ ] Verified the published DLL and `web.config` exist.
- [ ] Checked the IIS ASP.NET Core configuration section.
- [ ] Confirmed the IIS website points to the correct physical path.
- [ ] Tested the application directly using `dotnet`.
- [ ] Tested the application through the IIS URL.
- [ ] Reviewed Event Viewer if another error appeared.

---

## 15. Useful Microsoft References

- [.NET 8 downloads](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)
- [ASP.NET Core Hosting Bundle](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/hosting-bundle)
- [Host ASP.NET Core on Windows with IIS](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/)
- [Troubleshoot HTTP Error 500.19 in IIS](https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/iis/site-behavior-performance/http-error-500-19-webpage)

---

## Conclusion

The original HTTP 500.19 error (`0x8007000d`) occurred while IIS was processing the application's configuration.

The investigation identified that `AspNetCoreModuleV2` was missing from full IIS, even though its DLL existed under IIS Express.

After installing or repairing the appropriate Hosting Bundle and restarting IIS, the following verification succeeded:

```cmd
%windir%\System32\inetsrv\appcmd.exe list modules | findstr /i "AspNetCore"
```

Output:

```text
MODULE "AspNetCoreModuleV2" ( native, preCondition: )
```

The module DLL was also found at:

```text
C:\Program Files\IIS\Asp.Net Core Module\V2\aspnetcorev2.dll
```

These results confirm that the IIS module installation and registration were corrected. If HTTP 500.19 remains after these changes, use the updated error page and Event Viewer to identify the remaining configuration problem.
