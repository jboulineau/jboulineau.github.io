---
layout: post
title: "Automating SQL Server Databases with Docker"
date: 2019-05-13
categories: "sql-server"
excerpt: "Installing and managing SQL Server on a development workstation has never been an attractive prospect. It is time consuming, installs _tons_ of dependencies that are not removed even if you uninstall SQL Server, and can drain resources even when it's not in use. Managing the database is often a drag on contemporary development practices. Automating database deployments in a Docker container can be a big boost in development efficiency."
---
Running SQL Server in a Docker container has been possible for quite a while now. Very shortly after it was announced that SQL Server 2017 could run on Linux, Docker images were [available](https://blog.docker.com/2017/09/microsoft-sql-on-docker-ee/) on the [Docker repository](https://hub.docker.com/_/microsoft-mssql-server). The speed and convenience of being able to have a running SQL Server instance in seconds is a huge productivity saver. Installing and managing SQL Server on a development workstation has never been an attractive prospect. It is time consuming, installs _tons_ of dependencies that are not removed even if you uninstall SQL Server, and can drain resources even when it's not in use. Managing the database is often a drag on contemporary development practices.

But, having SQL Server in Docker isn't really enough. Even after having the instance available, it is still often an onerous process to deploy the database. What I need is the ability to spin up one or more **databases** with the same ease as deploying an instance of my service. 'Dockerizing' my database projects has had an immediate impact on my work efficiency due to the ease in being able to create on demand database instances with the most recent version of the code, unattended, in a matter of minutes. Best of all, the image can be automagically registered in my company's Docker Trusted Repository (DTR) so that anyone can include the image in, say, `docker-compose` for their own purposes.

Fortunately, by leveraging the capabilities of the [Data-Tier Application](https://docs.microsoft.com/en-us/sql/relational-databases/data-tier-applications/data-tier-applications?view=sql-server-2017) and related DacFx framework it is possible to include one or more databases in the SQL Server container. The path to the Data-Tier Application is through [SQL Server Data Tools](https://docs.microsoft.com/en-us/sql/ssdt/download-sql-server-data-tools-ssdt?view=sql-server-2017) within Visual Studio. A tutorial on using SSDT for SQL Server database projects is beyond the scope of this post. However, [here](https://www.mssqltips.com/sqlservertutorial/9000/overview-of-ssdt-sql-project-tutorial/), [here](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-data-tools/hh272702(v=vs.103)), or [here](https://sqlbits.com/Sessions/Event11/Real_world_SSDT) are some places to start.

Once you have your database project created (which is possible using an [existing database](https://www.mssqltips.com/sqlservertip/2971/creating-a-visual-studio-database-project-for-an-existing-sql-server-database/)), you're ready to start.

Now, let's create a [Dockerfile](https://docs.docker.com/engine/reference/builder/) in the project folder. Start with the Microsoft base image.

``` Dockerfile
FROM mcr.microsoft.com/mssql/server:2017-latest
```

The default product for the image is 'Developer,' the free single-user edition of SQL Server Enterprise. If used properly there is no cost for this edition, but make sure you understand the license so you stay in compliance. Next, set the required environment variables for SQL Server.

``` Dockerfile
ENV ACCEPT_EULA=Y
ENV SA_PASSWORD=<strong_password>
```

Yes, the SA password is set in plain text in the Dockerfile, which is typically committed to version control. For development activities this is probably fine (just don't use a **REAL** password), but if you are intending on utilizing the image as part of a production deployment of your service, it is most certainly not. It is possible to externalize the secret to an [environment file](https://vsupalov.com/docker-arg-env-variable-guide/) or pass the value through `docker build` using `ARG` in such a scenario.

Next, the `COPY` command here is copying everything in the same directory as the Dockerfile into /var/opt/ within the container.

``` Dockerfile
COPY . /var/opt/<project_name>
```

A digression is in order. Although at this point the entire database project exists within the container, to date Microsoft has not added support for database projects in dotnet core. This means it is not possible to generate the deployment artifact (.dacpac) within the Linux container. It is especially frustrating since they **have** ported the CLI for .dacpac deployments to Linux, so you can deploy one, you just can't create one. At time of writing, there doesn't seem to be an open issue for the [MSBuild project](https://github.com/microsoft/msbuild/issues?utf8=%E2%9C%93&q=database), either. Therefore, it is necessary to ensure that a build is run for the project either through MSBuild or Visual Studio **before** the build for the image, an annoying extra step that can't be automated in the Dockerfile. In projects I manage this is currently addressed by prohibiting PRs that do not include a .dacpac for code changes. At some point I'll try to figure out how to enforce that further with a [pre-commit](https://pre-commit.com/) hook. Let's get back to the Dockerfile.

``` Dockerfile
RUN apt-get update \
    && apt-get install unzip dotnet-sdk-2.2 -y --no-install-recommends \
    && wget -q -O /var/opt/sqlpackage.zip https://go.microsoft.com/fwlink/?linkid=2069122 \
    && unzip -qq /var/opt/sqlpackage.zip -d /var/opt/sqlpackage \
    && rm /var/opt/sqlpackage.zip \
    && chmod +x /var/opt/sqlpackage/sqlpackage \
```

Microsoft's base image for SQL Server is missing software we need. If you are going to run a unit test suite, the dotnet core sdk is required. If you're not unit testing, perhaps because all of your DML is in your app, you can probably remove dotnet-sdk-2.2 from the install. You will, however, need to download [sqlpackage](https://docs.microsoft.com/en-us/sql/tools/sqlpackage?view=sql-server-2017) (the DacFX CLI). In order to extract from the .zip, unzip is required.

``` Dockerfile
    && mv /var/opt/<project_dir>/bin/Debug/master.dacpac /var/opt/<project_dir>/bin/Debug/MASTER.DACPAC \
    && mv /var/opt/<project_dir>/bin/Debug/msdb.dacpac /var/opt/<project_dir>/bin/Debug/MSDB.DACPAC \
```

Often T-SQL objects will include references to objects stored in system databases. These dependencies can be added to the project in the form of their respective .dacpac, which allows Visual Studio to resolve the references. In a discovery that wasted a significant chunk of time, sqlpackage only looks for dependencies in upper case names `¯\_(ツ)_/¯`. In Windows, this doesn't matter, but in Linux it certainly does. It looks like a missed test case in the dotnet core port. Let's move on...

``` Dockerfile
    && (/opt/mssql/bin/sqlservr --accept-eula & ) | grep -q "Service Broker manager has started" \
    && /var/opt/sqlpackage/sqlpackage /a:Publish /tdn:<db_name> \
         /pr:/var/opt/<project_dir>/docker.publish.xml /sf:/var/opt/<project_dir>/bin/Debug/<project-name>.dacpac \
         /p:IncludeCompositeObjects=true /tu:sa /tp:$SA_PASSWORD \
    && (/opt/mssql-tools/bin/sqlcmd -S localhost -d <db_name> -U SA -P $SA_PASSWORD -Q 'ALTER DATABASE <db_name> SET RECOVERY SIMPLE') \
```

Here we start SQL Server and use sqlpackage to deploy the database. You'll notice that a publish profile has been created and stored with the project specifically for deploying to SQL Server running in the container. It's also a good idea to make sure the database is set to `SIMPLE` recovery for testing purposes to keep the log file small. You probably don't want to do this in prod. Note also that this Dockerfile doesn't establish external volumes for the data and log files, which you will probably want to do if using this in a production environment.

This is also a good place to insert any other SQL commands that makes sense for you using `sqlcmd`. For instance, this is where I also include the creation of any service accounts that need to have access to my database for whatever it is I'm testing. The key is to strive for zero administration or setup required whenever the container is run.

``` Dockerfile
    && cd /var/opt/<project_dir>/<test_dir>/ && dotnet test \
```

In most database projects I have a testing suite. MS Test, xUnit, and NUnit are available in dotnet core. Visual Studio works very nicely with MS Test, which includes a SQL specific test harness that can be handy, if somewhat restricted. This line in the Dockerfile executes the tests in the project.

``` Dockerfile
    && pkill sqlservr \
    && apt-get remove dotnet-sdk-2.2 -y \
    && apt-get autoremove -y \
    && rm -rf /usr/share/dotnet \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /var/opt/<project_dir> \
    && rm -rf /var/opt/sqlpackage
```

Finally, the last steps shutdown SQL Server and remove the software that was installed. This helps to keep an already large image as small as possible. The end result, depending on what is involved in deploying your database and running your tests, should be an image which is 1.6 - 2 GB in size.

## Conclusion

For future enhancement I intend on implementing test data population as part of this process, so that it will be possible to spin up databases very quickly for integration testing purposes. Restoring a database from backup should be quite simple to automate, assuming the backups are stored on a network-accessible path, so that is one option I am considering. That would also be a great option to rapidly deploy databases for shared environments, though that would call for the additional enhancement of being able to run masking/anonymization operations on the restored data set. But, like you have seen, it is possible to automate just about anything in the build of a Docker image.
