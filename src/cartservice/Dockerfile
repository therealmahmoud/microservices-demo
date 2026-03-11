# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS builder
WORKDIR /app

# Copy project file and restore dependencies
COPY cartservice.csproj ./
RUN dotnet restore cartservice.csproj

# Copy only the service source files (exclude tests)
COPY *.cs ./   # <-- relative to the Dockerfile in cartservice/

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
