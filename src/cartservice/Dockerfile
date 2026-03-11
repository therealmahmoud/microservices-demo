# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS builder
WORKDIR /app

# Copy the project file and restore dependencies
COPY cartservice.csproj ./
RUN dotnet restore cartservice.csproj

# Copy only the source files, **exclude tests**
COPY ./src/cartservice/*.cs ./

# Publish the service
RUN dotnet publish cartservice.csproj \
    -c Release \
    -o /cartservice \
    -p:PublishTrimmed=true \
    -p:TrimMode=full

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=builder /cartservice .

ENTRYPOINT ["dotnet", "cartservice.dll"]
