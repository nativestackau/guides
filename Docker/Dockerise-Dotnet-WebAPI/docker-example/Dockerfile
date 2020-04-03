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