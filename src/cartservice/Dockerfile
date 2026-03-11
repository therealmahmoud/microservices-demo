FROM mcr.microsoft.com/dotnet/sdk:10.0 AS builder
WORKDIR /app

COPY cartservice.csproj ./
RUN dotnet restore cartservice.csproj

COPY . ./
RUN rm -rf ./tests

RUN dotnet publish cartservice.csproj \
    -c Release \
    -o /cartservice \
    -p:PublishSingleFile=true \
    -a amd64 \
    --self-contained true \
    -p:PublishTrimmed=true \
    -p:TrimMode=full

FROM mcr.microsoft.com/dotnet/runtime:10.0
WORKDIR /app
COPY --from=builder /cartservice .
ENTRYPOINT ["./cartservice"]
