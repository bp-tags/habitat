---
title: "Running Habitat as a Windows Service"
date: 2017-10-31
author: Matt Wrock
tags: Windows, Supervisor
category: windows
classes: body-article
---

You can now run Habitat on Windows as a Windows Service. This is the recommended way to run a Windows Habitat production environment. All of the Windows service functionality resides in a new package [`core/windows-service`](https://github.com/habitat-sh/windows-service). This includes a [small C# program](https://github.com/habitat-sh/windows-service/blob/master/HabService.cs) that implements the service loop that starts and stops the Habitat service and redirects output to rotating log files. It also includes a service installer and uninstaller.

## Requirements

The only requirement to install and run the service is that you have the Habitat cli (hab.exe) installed and on your path. Preferably this should be version 0.38.0 or later. The service is supported on all versions of Windows from Server 2008 R2 to the current version of server 2016.

## Installation

To install the Habitat Windows service, run:

```console
hab pkg install core/windows-service
hab pkg exec core/windows-service install
```

This will install the service in the Windows Service Control Manager (SCM). You can locate the service in the Services console app. It will be named "Habitat." It will be set to start automatically and run under the `Local System` account. You may of course change the startup type and/or account identity in the service's properties.

## Uninstallation

To uninstall the Habitat service, run:

```console
hab pkg exec core/windows-service uninstall
```

This will stop the service if it is running and uninstall it from the Windows Service Control Manager (SCM).

## Supervisor Logs

The Habitat Supervisor logs will be located in `$env:systemdrive\hab\svc\windows-service\logs`. The log will rotate every 100MB and will archive up to 10 log files. These rotation settings are configurable (see below).

## Configuring the Habitat service

You may configure the Habitat service using its configuration file located at `$env:systemdrive\hab\svc\windows-service\HabService.exe.config`. Here you can configure settings for logging, arguments to be passed to `hab run`, and whether to include debug verbosity in the logs.

The Habitat Windows service uses [log4net](https://logging.apache.org/log4net/) to control where logs are saved and how they are rotated. You likely do not want to change these settings but if you are familiar with `log4net` logging appenders, you can certainly change these settings.

Other settings are located in the `appSettings` section of the configuration file. Here you can add or edit the following configuration keys:

* `debug` - When any value other than `false`, this will cause the service logs to include very verbose debug logging. This is likely only helpful to a Habitat developer troubleshooting problems with the Habitat Supervisor.
* `launcherArgs` - Arguments to forward on to `hab run`. You can see `hab run --help` for details but here you may pass peer information or override supervisor ports.
* `launcherPath` - The absolute path of the `hab-launch.exe` to invoke which will start the supervisor. By default this will point to the latest version of the launcher installed. This is useful by Habitat developers for debugging the Habitat launcher.

## Considerations for setting the Habitat service identity

By default the Habitat service will run under `Local System` and should have adequate permissions for running Habitat services. You may specify in your service `plan.ps1` a specific identity (`pkg_svc_user`) that you want an individual Habitat service to run as and the Habitat Supervisor (running under the Windows service) will launch that service with the specified identity. Note that identity must have the right to run as a service. If you do not specify a `pkg_svc_user` in your plan, the service will run under the same identity as the Windows service (`Local System` by default).

If you would like to have the Windows service run under an identity other than `Local System`, keep the following in mind:

* You may not use `Network Service` because that identity is given minimum local privileges and may not be able to access processes that the supervisor needs to access. For example, if the Supervisor cannot gracefully terminate a service, it will forcefully terminate the process along with all child processes. The `Network Service` does not have rights to terminate a process.
* The configured account must have local administrator privileges and be assigned the following rights:
  * Log on as a service
  * Adjust memory quotas for a process
  * Replace a process level token

The last two assignments are necessary in order for the Habitat Windows service to launch Habitat services under different identites.

## Running the Service and a Windows Studio on the same machine

You may encounter failures if you attempt to enter a Windows Studio using `hab studio enter -w` on a machine that is running the Habitat Windows service due to port conflicts. Currently this is not a supported scenario and it is recommended that you run the Windows service in non-development environments where you do not intend to use a Windows based Studio.
