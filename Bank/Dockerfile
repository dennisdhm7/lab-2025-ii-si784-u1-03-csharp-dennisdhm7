# Imagen base para ejecutar la aplicación
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 80

# Imagen SDK para compilar
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copiar todos los archivos al contenedor
COPY . .

# Restaurar dependencias
RUN dotnet restore

# Compilar el proyecto en modo Release
RUN dotnet build ./Bank.WebApi/Bank.WebApi.csproj -c Release -o /app/build

# Publicar el proyecto
FROM build AS publish
RUN dotnet publish ./Bank.WebApi/Bank.WebApi.csproj -c Release -o /app/publish /p:UseAppHost=false

# Imagen final lista para ejecutar la API
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Bank.WebApi.dll"]
