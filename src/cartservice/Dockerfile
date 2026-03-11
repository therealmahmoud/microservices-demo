# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS builder
WORKDIR /app

# Copy project file first
COPY cartservice.csproj ./

# Restore dependencies
RUN dotnet restore cartservice.csproj

# Copy all service source files (exclude tests folder)
COPY . ./
RUN rm -rf tests

# Publish the service
RUN dotnet publish cartservice.csproj \
    -c Release \
    -o /cartservice \
    -p:PublishTrimmed=true \
    -p:TrimMode=full

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app

# Copy the published app from builder
COPY --from=builder /cartservice .

# Run the service
ENTRYPOINT ["dotnet", "cartservice.dll"]
