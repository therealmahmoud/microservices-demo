# Build stage
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /app

# Copy csproj and restore dependencies
COPY cartservice.csproj ./
RUN dotnet restore cartservice.csproj

# Copy all source files except tests
COPY . ./
RUN rm -rf ./tests     # remove tests folder if present

# Publish the project
RUN dotnet publish cartservice.csproj \
    -c Release \
    -o /cartservice \
    -p:PublishSingleFile=true \
    -a amd64 \
    --self-contained true \
    -p:PublishTrimmed=true \
    -p:TrimMode=full

# Runtime stage
FROM mcr.microsoft.com/dotnet/runtime:7.0
WORKDIR /app
COPY --from=builder /cartservice .
ENTRYPOINT ["./cartservice"]
