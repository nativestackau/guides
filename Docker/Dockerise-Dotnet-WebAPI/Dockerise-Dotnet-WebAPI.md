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
cd docker-example

dotnet run
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
ENTRYPOINT ["dotnet", "docker-example.csproj"]
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
        4. **ENTRYPOINT ["dotnet", "docker-example.csproj"]** - In this line we are specifying a command to run our project within the container.

        
- Documentation on `Dockerfile` can be found here: https://docs.docker.com/engine/reference/builder/#dockerfile-reference


## Running the Docker container

## Cleaning up

## Summary