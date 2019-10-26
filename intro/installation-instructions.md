# Installation instructions
This article describes how to install the OPCAIC platform on a new machine. This text assumes a Linux based hosts, however, it should be possible to adapt it for Windows hosts.

The OPCAIC platform consists of three main components:
 - *Web application* written in JavaScript
 - *Application server* hosting main applicaton logic
 - *Worker service* for tournament execution

In this text, we will refer to these components simply as webapp, server and worker. It is recommended to deploy server and worker(s) on separate machines for performance reasons.

## Prerequsites

Many of the steps required for deploying server and workers are the same. This text describes the scenario of deploying both components on the same machine, with additional notes on the difference when deploying the components separately.

### User

Create a dedicated user which will be used to run the platform binaries. Constrain the privileges for the user as you see fit. In this text, we will use user `opcaic`.

### .NET Core 3.0 runtime

Follow instructions on [official website](https://dotnet.microsoft.com/download/) for your system.

### Setup PostgreSQL database (server only)

The web backend requires SQL database for data persistence.

#### Install PostgreSQL

Download and install PostgreSQL database. For Linux distributions using `apt`, run following command
as root:

    apt-get install postgresql postgresql-contrib

After installation, start it by:

    systemctl enable postgresql
    systemctl start postgresql

#### Change default user password

Issue command:

    passwd postgres

#### Change PostgreSQL admin password

Run `psql` as `postgres` user, and run 

    \password postgres;

#### Create a database for the server

Still in `psql` prompt, run:

    create database opcaic_server_db;

Check that the database was created by:

    \l

#### Create a database user for the server

Create user with same name as the dedicated linux user created earlier.

    create user opcaic;

Setup suitable password

    \password opcaic;

Grant user privileges to the created database

    grant all privileges on database opcaic_server_db to opcaic;

## Deploying the server

Create `/var/opcaic/server` directory and copy the server files there. The server also needs a directory for storing user submissions. For this we recommend creating directory `/var/opcaic/server_storage`. Make sure that the `opcaic` user has access to these directories.

### Configuring the server

The server requires additional configuration before starting. Namely the connection string to the database and the location of the storage folder. These can be provided either by writing their value into the `appsettings.json` configuration file, or through environment variables. Names of variable names are case insensitive. The environment variables take precedence over the configuration file, and their name is obtained by taking the JSON path and replacing any colons with two underscores (e.g. `Security:Key` becomes `Security__Key`). The list of required variables are: 

| **Variable name**               | **Description**                                                                    |
| -----------------------------   | ---------------------------------------------------------------------              |
| `FrontendUrl`                   | Url of the frontend application (to be used when generating links)                 |
| `Security:Key`                  | Key for signing JWT tokens provided by the web server.                             |
| `ConnectionStrings:DataContext` | Connection string to the PostgreSQL database                                       |
| `Storage:Directory`             | Path to the storage folder, recomended `/var/opcaic/server_storage`                |
| `Broker:ListeningAddress`       | Address to which worker processes will connect. Default is `tcp://localhost:6000`  |
| `Emails:SmtpServerurl`          | Url (without port) of the server used for sending emails.                          |
| `Emails:Port`                   | Port on smtp server to connect to.                                                 |
| `Emails:Username`               | Username used to authenticate to the smtp server.                                  |
| `Emails:Password`               | Password used to authenticate to the smtp server.                                  |
| `Emails:UseSsl`                 | Whether SSL connection should be enforced when communicating with the smtp server. |
| `Emails:SenderAddress`          | Email address to use as the sender address.                                        |

Additional configuration variables are described in separate section.

#### First run of the server

On the very first startup, it is needed to provide additional configuration variables for creating the first admin account.

| **Parameter**                 | **Description**                                                         |
| ----------------------------- | ----------------------------------------------------------------------- |
| `Seed:AdminUsername`          | The username under which the admin will be visible                      |
| `Seed:AdminEmail`             | The email address used for admin login. This needs to be a valid email. |
| `Seed:AdminPassword`          | Password which should be used for login.                                |

We recommend using command line parameters for the admin account credentials. Supposing that correct values for other variables have been provided either in `appconfig.json` or environment variables, you can use following command line command:

    dotnet OPCAIC.ApiService.dll \
        --Seed:AdminUsername=admin \
        --Seed:AdminEmail=admin@opcaic.com \
        --Seed:AdminPassword='P4$$w0rd'

The application will immediately try to verify the email address by sending an email to it. Once the email is sent, you may terminate the application. Note that confirming the email address requires working `web-app` to be deployed. If the application has been misconfigured (e.g. invalid frontend address in the configuration), you need to drop the SQL database to be able to repeat the process.

### Running the server as a service

We recommend using some service management tool such as `systemd`. Example systemd unit file can be found below:

```systemd
[Unit]
Description=OPCAIC.Web service
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=opcaic
WorkingDirectory=/var/opcaic/server
ExecStart=/usr/bin/dotnet /var/opcaic/server/OPCAIC.ApiService.dll

Environment=SECURITY__KEY=insert_security_key_here
Environment='CONNECTIONSTRINGS__DATACONTEXT=Server=127.0.0.1;Port=5432;Database=opcaic_server_db;User Id=opcaic;Password=long_live_opcaic;'
Environment=STORAGE__DIRECTORY=/var/opcaic/server_storage
Environment=BROKER__LISTENINGADDRESS=tcp://168.192.0.0:6000

[Install]
WantedBy=multi-user.target
```

Save this file as `/etc/systemd/system/opcaic.server.service` and issue following commands as root

    systemctl enable opcaic.server.service
    systemctl start opcaic.server.service

You can use 

    sudo journalctl -fu *opcaic*

to view latest logs from the server. For more information about `journalctl` see `man journalctl`

For other configuration options, see [Server configuration](server-configuration.md) section.

### Exposing the server

The server component does not provide support for HTTPS, nor accepts HTTP connections from remote hosts by default. The expected scenario is exposing the server through a *reverse proxy* like Nginx or Apache, which will handle HTTPS redirection and other security measures. The server by default listens on `http://localhost:5000/` so the reverse proxy should be pointed there. All routes that server handles start with `/api/` or `/swagger/`, so we need to map only those. Example `nginx.conf` follows:

```nginx
## beginning omitted
	location ~* /(api|swagger)/
	{
 		# configure client_max_body_size to allow larger submission uploads
		client_max_body_size 50m;

		proxy_pass         http://localhost:5000;
		proxy_http_version 1.1;
		proxy_set_header   Upgrade $http_upgrade;
		proxy_set_header   Connection keep-alive;
		proxy_set_header   Host $host;
		proxy_cache_bypass $http_upgrade;
		proxy_set_header   X-Forwarded-For
			$proxy_add_x_forwarded_for;
		proxy_set_header
			X-Forwarded-Proto $scheme;

		# add other settings as required
	}
## rest of the file omitted
```

The server also needs to communicate with workers. If worker(s) are deployed on different machines, make sure they can make connection to the address specified by the `Broker.ListeningAddress` config variable.

## Deploying the web application

The web-app component is a typical javascript SPA application and can be deployed e.g. by Apache or Nginx. We will show how to serve the application using Nginx. Copy the web-app files to `/var/opcaic/web-app` folder and add following configuration to `nginx.conf`:

```nginx
## beginning omitted
	location / {
		# First attempt to serve request as file
		# then attempt to redirect to /index.html and let app's client-side routing work it out,
		# else fallback to 404 error.
		try_files $uri /index.html =404;
		root /var/opcaic/web-app;
	}
## rest of the file omitted
```

## Deploying the worker

Deploying the worker is done similarly to deploying the server. We recommend following directories inside `/var/opcaic`:
 - `worker` - worker binaries
 - `worker_storage/work` - storing temporary data during match execution
 - `worker_storage/archive` - archive of executed matches for diagnostic purposes
 - `modules` - game modules handling execution of individual games.

Copy the worker binaries to `/var/opcaic/worker` directory and wanted game modules to the `/var/opcaic/modules` directory. Give appropriate access rights to the `opcaic` user for all above directories. Worker also needs to be configured, following table describes variables which need to be configured eithre via `appsettings.json` or environment variables

| **JSON configuration path**     | **Description**                                                                                       |
| ------------------------------- | ----------------------------------------------------------------------------------------------------- |
| ModulePath                      | Path to directory with game modules, recomended `/var/opcaic/modules`                                 |
| Execution:WorkingDirectoryRoot  | Path to dedicated working directory for in-process tasks                                              |
| Execution:ArchiveDirectoryRoot  | Path to dedicated archiving directory for executed tasks                                              |
| ConnectorConfig:BrokerAddress   | Address to which the worker should connect. Corresponds to Broker.ListeningAddress variable on server |

Example systemd unit file follows:

```systemd
[Unit]
Description=OPCAIC.Worker service
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=opcaic
WorkingDirectory=/var/opcaic/worker
ExecStart=/usr/bin/dotnet /var/opcaic/worker/OPCAIC.Worker.dll 

Environment=MODULEPATH=/var/opcaic/modules
Environment=EXECUTION__WORKINGDIRECTORYROOT=/var/opcaic/worker_root/work
Environment=EXECUTION__ARCHIVEDIRECTORYROOT=/var/opcaic/worker_root/archive
Environment=CONNECTORCONFIG__BROKERADDRESS=tcp://168.192.0.10:6000

[Install]
WantedBy=multi-user.target

```

Save this file as `/etc/systemd/system/opcaic.worker.service` and start the worker by following commands (as root)

    systemctl enable opcaic.worker.service
    systemctl start opcaic.worker.service

As with server, you can see debug output by running

    journalctl -fu *opcaic*

The output should now display both server and worker logs.

For information how to create your own game modules and deploy them, see [Adding a new game module to the OPCAIC platform](adding-new-game-modules.md).

## (Optional) Installing Graylog for log aggregation

Searching though the logs using `journalctl` is not very user friendly for inexperienced users. The OPCAIC platform can be configured to use [Graylog](https://www.graylog.org) which is a tool supporting log aggregation, structured log searching and even monitoring capabilities. Install graylog by following the [official installation guide](https://docs.graylog.org/en/3.1/pages/installation.html).

For the actual Graylog setup for consuming OPCAIC platform logs, we recommend setting up an GELF HTTP input. Both opcaic server and worker binaries can be configured by editing the `Serilog` configuration section in `appsettings.json` file. It is also good idea to raise the minimum level for console logger when using Graylog. Example configuration follows:

```js
{
	"Serilog": {
		"Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.Graylog" ],
		//... left out for brevity
		"WriteTo": [
			{
				"Name": "Console",
				"Args": {
					"restrictedToMinimumLevel": "Warning"
				}
			},
			{
				"Name": "Graylog",
				"Args": {
					"hostnameOrAddress": "localhost",
					"port": "12201",
					"transportType": "Http"
				}
			}
		],
		// ... rest of the section omitted for brevity
	}
}
```

Refer to the official documentation on how to use Graylog for querying the aggregated logs.