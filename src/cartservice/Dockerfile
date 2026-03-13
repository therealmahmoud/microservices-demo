# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY . .
RUN rm -rf tests

RUN dotnet publish cartservice.csproj \
    -c Release \
    -o /app/publish \
    --no-self-contained

# Runtime stage (small image)
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app/publish .

ENTRYPOINT ["dotnet","cartservice.dll"]
