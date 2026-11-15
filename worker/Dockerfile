FROM mcr.microsoft.com/dotnet/sdk:10.0.404-alpine3.24 AS build

WORKDIR /src
COPY Worker.csproj .
RUN dotnet restore
COPY Program.cs .
RUN dotnet publish -c Release --self-contained false --no-restore -o /out

FROM mcr.microsoft.com/dotnet/runtime:10.0.15-alpine3.24

WORKDIR /app
USER 1654:1654
COPY --from=build --chown=1654:1654 /out .

ENTRYPOINT ["dotnet", "Worker.dll"]
