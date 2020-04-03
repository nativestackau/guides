# Dockerising a .Net Core Web API

## Aim
- Create a .Net Core Web API using the .Net Core CLI.
- Dockerise the .Net Core Web API.
- Run the Docker container locally.

## Prerequisites
- .Net Core CLI (Included in the .Net Core SDK): https://docs.microsoft.com/en-us/dotnet/core/install/sdk?pivots=os-windows
- Docker Desktop: https://www.docker.com/products/docker-desktop

## Creating the Web API
1. To create a new Web API project using the .Net Core CLI open a new shell session and run the following command:

~~~
dotnet new webapi --name docker-example
~~~

- The `new` command can take many arguments which tell it what to create. In this case, we specify `webapi` so that it will create a new .Net Core Web API project.
- The `--name` argument is used to name the project, and creates a new directory with the specified name. If the `--name` argument is not provided, the name of the current directory will be used.
- Documentation for the `dotnet new` command can be found here: https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-new

2. Navigate into the newly created project, and run it to confirm everything is working as expected:

~~~
$ cd docker-example

$ dotnet run
~~~

You should see the following output:

~~~powershell
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: https://localhost:5001
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: C:\Dockerise-Dotnet-WebAPI\docker-example
~~~

- The `run` command is used to run the project, and will automatically **build** any code, and **restore** any nuget packages.
- Documentation for the `dotnet run` command can be found here: https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-run

## Dockerising the Web API
1. Create a new file called `Dockerfile` inside the project directory, and add the following content:

~~~docker
#multi-stage build:
#stage 1
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 as build-env
WORKDIR /app

COPY *.csproj ./
RUN dotnet restore

COPY . ./
RUN dotnet publish -c Release -o release

#stage 2 - only copying over the release artifact from the build-env image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=build-env /app/release .
ENTRYPOINT ["dotnet", "docker-example.dll"]
~~~

- A `Dockerfile` defines the steps needed to build a Docker image.
- This `Dockerfile` utilises the multi-stage build pattern which helps to reduce the final image size by only including the necessary artifacts in the final image.
    - **Stage 1**
        1. **FROM mcr.microsoft.com/dotnet/core/sdk:3.1 as build-env** - This line sets the base image to Microsoft's .Net Core 3.1 sdk image.
        2. **WORKDIR /app** - This line sets the working directory for the subsequent copy command.
        3. **COPY \*.csproj ./** - This line copys the project's `docker-example.csproj` into the images **/app** directory, as the previous step set the working directory to **/app**.
        4. **RUN dotnet restore** - This line runs `dotnet restore` from the **/app** directory.
        5. **COPY . ./** - This line copies over all other files into the **/app** directory. This step was seperated from step **3** to take advantage of Docker's **build cache**. Documentation can be found here: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache
        6. **RUN dotnet publish -c Release -o release** - This line creates the release artifact using the `dotnet publish` command/

    - **Stage 2**
        1. **FROM mcr.microsoft.com/dotnet/core/aspnet:3.1** - This line sets the base image to of this new stage to Microsoft's ASP.NET Core 3.1 Runtime. This image is used instead of the SDK because it is smaller.
        2. **WORKDIR /app** - This line again sets the working directory for subsequent commands.
        3. **COPY --from=build-env /app/release .** - This line copies our release artifact that was created in stage 1 into **/app**. This means that this image only has the release artifact and nothing else adding to the images size *(ie source code or build files).*
        4. **ENTRYPOINT ["dotnet", "docker-example.dll"]** - In this line we are specifying a command to run our project within the container.

        
- Documentation on `Dockerfile` can be found here: https://docs.docker.com/engine/reference/builder/#dockerfile-reference

2. Add a `.dockerignore` file to the project and add the following:

~~~docker
bin\
obj\
~~~

- This will ensure that the build context is as small as possible. Docker commands such as **COPY** will ignore files listed in `.dockerignore`.

## Running the Docker container
1. To build the Docker image using the `Docker CLI` run the following command from the same directory that the `Dockerfile` is located:

~~~powershell
$ docker build -t mydockerexampleapp .
~~~

- The Docker `build` command builds a Docker image from a `Dockerfile` and a build context, where context is the set of files located in a specified **PATH** or **URL**. This example with be using **PATH** but you could specify the **URL** of a remote container registry like *Dockerhub* or *Azure Container Registry*.
- The `-t` flag is used to **tag** the image. This gives the image a name that can be referenced and used in other commands. Tagging is very important for pushing Docker images to remote image repositories like *Dockerhub* and *Azure Container Regesitry*.

You should see the following output:

~~~powershell
Step 1/10 : FROM mcr.microsoft.com/dotnet/core/sdk:3.1 as build-env
3.1: Pulling from dotnet/core/sdk
f15005b0235f: Pull complete                                                                                                                                                                                                                                                    41ebfd3d2fd0: Pull complete                                                                                                                                                                                                                                                    b998346ba308: Pull complete                                                                                                                                                                                                                                                    f01ec562c947: Pull complete                                                                                                                                                                                                                                                    de2914a3bce9: Pull complete                                                                                                                                                                                                                                                    e1060ee932d6: Pull complete                                                                                                                                                                                                                                                    b2ee07dee813: Pull complete                                                                                                                                                                                                                                                    Digest: sha256:bf755bf00ee2712af5fb71af0aea57e8e65dc2cc191f28c2d1f95c325f00177f
Status: Downloaded newer image for mcr.microsoft.com/dotnet/core/sdk:3.1
 ---> fc3ec13a2fac
Step 2/10 : WORKDIR /app
 ---> Running in f25a507d7744
Removing intermediate container f25a507d7744
 ---> 93c5bd2cf645
Step 3/10 : COPY *.csproj ./
 ---> 060bf66da146
Step 4/10 : RUN dotnet restore
 ---> Running in 5efaa28017b9
  Restore completed in 126.11 ms for /app/docker-example.csproj.
Removing intermediate container 5efaa28017b9
 ---> ce2beab47c20
Step 5/10 : COPY . ./
 ---> b2df396f5698
Step 6/10 : RUN dotnet publish -c Release -o release
 ---> Running in 1d4790590bb7
Microsoft (R) Build Engine version 16.5.0+d4cbfca49 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 34.16 ms for /app/docker-example.csproj.
  docker-example -> /app/bin/Release/netcoreapp3.1/docker-example.dll
  docker-example -> /app/release/
Removing intermediate container 1d4790590bb7
 ---> eae4abf5162c
Step 7/10 : FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
3.1: Pulling from dotnet/core/aspnet
c499e6d256d6: Pull complete                                                                                                                                                                                                                                                    251bcd0af921: Pull complete                                                                                                                                                                                                                                                    852994ba072a: Pull complete                                                                                                                                                                                                                                                    f64c6405f94b: Pull complete                                                                                                                                                                                                                                                    9347e53e1c3a: Pull complete                                                                                                                                                                                                                                                    Digest: sha256:31355469835e6df7538dbf5a4100c095338b51cbe52154aa23ae79d87585d404
Status: Downloaded newer image for mcr.microsoft.com/dotnet/core/aspnet:3.1
 ---> c819eb4381e7
Step 8/10 : WORKDIR /app
 ---> Running in caae8e5cdbb3
Removing intermediate container caae8e5cdbb3
 ---> 938774a6650b
Step 9/10 : COPY --from=build-env /app/release .
 ---> bffb2bc06bfe
Step 10/10 : ENTRYPOINT ["dotnet", "docker-example.dll"]
 ---> Running in de53f67f1555
Removing intermediate container de53f67f1555
 ---> d76bc07f9787
Successfully built d76bc07f9787
Successfully tagged mydockerexampleapp:latest
~~~

2. To run a Docker image and create a Docker container use the following command:

~~~powershell
$ docker run -d -p 8080:80 --name examplecontainer mydockerexampleapp
~~~

- The `-d` flag is used to run the container in **Detached** mode. This means that the Docker container will run in the background of your shell.
- The `-p` flag maps maps ports on your local machine, to ports on the Docker container. `-p <port-on-local-machine>:<port-on-container>`.
- The `--name` flag is used to name the container. This makes it easier to identify and target with other Docker commands. By default you can target the container with its unique identifier that is output to the shell when it is started.

You should see similar output:

~~~powershell
4ea71e780239e38cffb953375ae3de9dc4b42d4cfec7163694e4dd0984154318
~~~

- The output observed is the unique identifier for the container instance that was just created.

3. To see our running container, and any other running containers on the machine, run the following command:

~~~powershell
$ docker container ls
~~~

You should see see similar output:

~~~powershell
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS                  NAMES
30881a67ccd5        mydockerexampleapp   "dotnet docker-examp…"   6 seconds ago       Up 5 seconds        0.0.0.0:8080->80/tcp   examplecontainer
~~~

4. Browse to [http://localhost:8080/weatherforecast](http://localhost:8080/weatherforecast) and you should see similar output:

~~~json
[{"date":"2020-04-04T04:53:09.2297095+00:00","temperatureC":-7,"temperatureF":20,"summary":"Bracing"},{"date":"2020-04-05T04:53:09.2310717+00:00","temperatureC":54,"temperatureF":129,"summary":"Freezing"},{"date":"2020-04-06T04:53:09.231074+00:00","temperatureC":13,"temperatureF":55,"summary":"Sweltering"},{"date":"2020-04-07T04:53:09.2310742+00:00","temperatureC":29,"temperatureF":84,"summary":"Freezing"},{"date":"2020-04-08T04:53:09.2310744+00:00","temperatureC":24,"temperatureF":75,"summary":"Hot"}]
~~~

5. To stop the container from running use the following command:

~~~powershell
$ docker stop examplecontainer
~~~

- The command `docker stop` takes in the name or unique identifier of a running container and stops it.
- To start the container again you can use `docker start <container-name>`.

## Cleaning up
1. To remove a container permantly use the following command:

~~~powershell
$ docker container rm examplecontainer
~~~

2. To remove an image use the following command:

~~~powershell
$ docker image rm mydockerexampleapp
~~~

You should see similar output:

~~~powershell
Untagged: mydockerexampleapp:latest
Deleted: sha256:c4b04dbfeff96e7265754fdf6d788d9ac9aa25ba1cf90a7b9320b119525c5ee7
~~~

3. To list all images that you might want to remove, use the following command:

~~~powershell
$ docker image ls
~~~
## Summary
In this guide we did the following:
- Created a new .Net Core Web API project using the .Net Core CLI.
- Created a `Dockerfile` which defined the steps to build a `Docker image`.
- Used the `Docker CLI` to **build** and **tag** a `Docker image`.
- Used the `Docker CLI` to **run** the container in **detatched mode**, mapping ports from the *local machine* to the *container*.
- Used the `Docker CLI` to clean up unused **containers** and **images**.