# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS builder
WORKDIR /app

# Copy project file first and restore dependencies
COPY cartservice.csproj ./
RUN dotnet restore cartservice.csproj

# Copy all source files (exclude tests)
COPY ./src/cartservice/*.cs ./ 

# Publish as framework-dependent (smaller!)
RUN dotnet publish cartservice.csproj \
    -c Release \
    -o /cartservice \
    -p:PublishTrimmed=true \
    -p:TrimMode=full

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app

# Copy published files from builder
COPY --from=builder /cartservice .

# Run your service
ENTRYPOINT ["dotnet", "cartservice.dll"]
