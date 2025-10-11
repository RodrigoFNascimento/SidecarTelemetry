# This stage is used when running from VS in fast mode (Default for Debug configuration)
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# --- OpenTelemetry Environment Variables ---
# Set the home directory for the agent
ENV OTEL_DOTNET_AUTO_HOME=/otel-dotnet-auto
# Enable instrumentation injection
ENV OTEL_DOTNET_AUTO_INJECT=true
# Set the service name for traces (important for traceability)
ENV OTEL_SERVICE_NAME=WebApi
# Set the OTLP endpoint (collector address).
ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
# Set the exporter protocol (gRPC is standard for 4317)
ENV OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# This stage is used to download the OpenTelemetry .NET Auto-Instrumentation Agent
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS otel_installer
WORKDIR /tmp
RUN apt-get update && apt-get install -y --no-install-recommends unzip curl && rm -rf /var/lib/apt/lists/*
ADD https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/otel-dotnet-auto-install.sh otel-dotnet-auto-install.sh
RUN sh otel-dotnet-auto-install.sh
RUN chmod +x /otel-dotnet-auto/instrument.sh

# This stage is used to build the service project
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["WebApi/WebApi.csproj", "WebApi/"]
RUN dotnet restore "./WebApi/WebApi.csproj"
COPY . .
WORKDIR "/src/WebApi"
RUN dotnet build "./WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

# This stage is used to publish the service project to be copied to the final stage
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# This stage is used in production or when running from VS in regular mode (Default when not using the Debug configuration)
FROM base AS final
COPY --from=otel_installer /otel-dotnet-auto /otel-dotnet-auto
COPY --from=publish /app/publish /app/
ENTRYPOINT ["/bin/bash", "-c", "source /otel-dotnet-auto/instrument.sh && exec dotnet WebApi.dll"]