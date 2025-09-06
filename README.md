# SESION DE LABORATORIO N° 03: PRUEBAS ESTATICAS DE SEGURIDAD DE APLICACIONES CON SEMGREP

## OBJETIVOS
  * Comprender el funcionamiento de las pruebas estaticas de seguridad de còdigo de las aplicaciones que desarrollamos utilizando Semgrep.

## REQUERIMIENTOS
  * Conocimientos: 
    - Conocimientos básicos de Bash (powershell).
    - Conocimientos básicos de Contenedores (Docker).
  * Hardware:
    - Virtualization activada en el BIOS..
    - CPU SLAT-capable feature.
    - Al menos 4GB de RAM.
  * Software:
    - Windows 10 64bit: Pro, Enterprise o Education (1607 Anniversary Update, Build 14393 o Superior)
    - Docker Desktop 
    - Powershell versión 7.x
    - Python 3.1x o superior
    - Net 8 o superior
    - Visual Studio Code

## CONSIDERACIONES INICIALES
  * Clonar el repositorio mediante git para tener los recursos necesarios
  * Tener una cuenta de Github valida. 

## DESARROLLO
### Parte I: Configuración de la herramienta de Pruebas Estaticas de Seguridad de la Aplicación
1. Iniciar la aplicación Powershell o Windows Terminal en modo administrador 
2. En el terminal, ejecutar los siguientes comandos para instalar SEMGREP.
```Bash
python -m pip install semgrep
python -m pip install prospector2html
```
3. Cerrar el terminal para que se actualicen las variables de entorno.

### Parte II: Creación de la aplicación
1. Iniciar la aplicación Powershell o Windows Terminal en modo administrador. 
2. En el terminal, Ejecutar el siguiente comando para crear una nueva solución
```
dotnet new sln -o Bank
dotnet tool install -g dll2mmd
dotnet tool install -g docfx
dotnet tool install -g dotnet-reportgenerator-globaltool
```
3. En el terminal, Acceder a la solución creada y ejecutar el siguiente comando para crear una nueva libreria de clases y adicionarla a la solución actual.
```
cd Bank
dotnet new webapi -o Bank.WebApi
dotnet sln add ./Bank.WebApi/Bank.WebApi.csproj
```
4. En el terminal, Ejecutar el siguiente comando para crear un nuevo proyecto de pruebas y adicionarla a la solución actual
```
dotnet new mstest -o Bank.WebApi.Tests
dotnet sln add ./Bank.WebApi.Tests/Bank.WebApi.Tests.csproj
dotnet add ./Bank.WebApi.Tests/Bank.WebApi.Tests.csproj reference ./Bank.WebApi/Bank.WebApi.csproj
```
5. Iniciar Visual Studio Code (VS Code) abriendo el folder de la solución como proyecto. En el proyecto Bank.Domain, si existe un archivo Class1.cs proceder a eliminarlo. Asimismo en el proyecto Bank.Domain.Tests si existiese un archivo UnitTest1.cs, también proceder a eliminarlo.

6. En el Visual Studio Code, en el proyecto Bank.WebApi proceder la carpeta `Models` y dentro de esta el archivo BankAccount.cs e introducir el siguiente código:
```C#
namespace Bank.WebApi.Models
{
    public class BankAccount
    {
        private readonly string m_customerName;
        private double m_balance;
        private BankAccount() { }
        public BankAccount(string customerName, double balance)
        {
            m_customerName = customerName;
            m_balance = balance;
        }
        public string CustomerName { get { return m_customerName; } }
        public double Balance { get { return m_balance; }  }
        public void Debit(double amount)
        {
            if (amount > m_balance)
                throw new ArgumentOutOfRangeException("amount");
            if (amount < 0)
                throw new ArgumentOutOfRangeException("amount");
            m_balance -= amount;
        }
        public void Credit(double amount)
        {
            if (amount < 0)
                throw new ArgumentOutOfRangeException("amount");
            m_balance += amount;
        }
    }
}
```
7. En el Visual Studio Code, en el proyecto Bank.WepApi.Tests añadir un nuevo archivo BanckAccountTests.cs e introducir el siguiente código:
```C#
using Bank.WebApi.Models;
using NUnit.Framework;

namespace Bank.Domain.Tests
{
    public class BankAccountTests
    {
        [Test]
        public void Debit_WithValidAmount_UpdatesBalance()
        {
            // Arrange
            double beginningBalance = 11.99;
            double debitAmount = 4.55;
            double expected = 7.44;
            BankAccount account = new BankAccount("Mr. Bryan Walton", beginningBalance);
            // Act
            account.Debit(debitAmount);
            // Assert
            double actual = account.Balance;
            Assert.AreEqual(expected, actual, 0.001, "Account not debited correctly");
        }
    }
}
```
8. En el Visual Studio Code, en la raiz del proyecto crear un archivo Dockerfile con el siguiente contenido:
```Yaml
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
WORKDIR "/src/."
RUN dotnet restore 
RUN dotnet build -o /app/build

FROM build AS publish
RUN dotnet publish -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Bank.WebApi.dll"]
```
9. En el Visual Studio Code, en la raiz del proyecto crear un archivo docker-compose.yml con el siguiente contenido:
```Yaml
version: '3.4'
services:
  webapi:
    image: api-bank
    build:
      context: .
    ports:
        - ${APP_HOST}:80
    environment:
        - "ConnectionStrings__DefaultConnection=Server=${DB_SERVER},${DB_PORT};Initial Catalog=${DB_NAME};Persist Security Info=False;User ID=${DB_USERNAME};Password=${DB_PASSWORD};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=${TRUST_SERVER_CERTIFICATE}; Integrated Security=${INTEGRATED_SECURITY};Connection Timeout=30;"
    depends_on:
        - "sqlServer"
    links:
        - "sqlServer"

  sqlServer:
    image: mcr.microsoft.com/mssql/server:2022-latest
    hostname: "sqlserver"
    environment:
        SA_PASSWORD: ${DB_PASSWORD}
        ACCEPT_EULA: "Y"
    restart: always
    ports:
        - "${DB_PORT}:1433"
```

10. En el terminal, ejecutar el siguiente comando para ejecutar las pruebas unitarias y el reporte de cobertura
```Bash
dotnet test --collect:"XPlat Code Coverage"
ReportGenerator "-reports:./*/*/*/coverage.cobertura.xml" "-targetdir:Cobertura" -reporttypes:MarkdownSummaryGithub
``` 
11. En el terminal, ejecutar el siguiente comando para optener el diagrama de clases.
```Bash
dll2mmd -f Bank.WebApi/bin/Debug/net8.0/Bank.WebApi.dll -o disenio.md
``` 
12. En el terminal, ejecutar el siguiente comando para generar el diagrama de clases respectivo
```Bash
docfx init -y
```
13. En el Visual Studio Code, eliminar los archivos de la carpeta o directorio docs, seguidamente modificar los archivos con el siguiente contenido:
> docfx.json
```Json
{
  "$schema": "https://raw.githubusercontent.com/dotnet/docfx/main/schemas/docfx.schema.json",
  "metadata": [
    {
      "src": [
        {
          "src": ".",
          "files": [
            "**/*.csproj"
          ]
        }
      ],
      "dest": "docs"
    }
  ],
  "build": {
    "content": [
      {
        "files": [
          "**/*.{md,yml}"
        ],
        "exclude": [
          "_site/**"
        ]
      }
    ],
    "resource": [
      {
        "files": [
          "images/**"
        ]
      }
    ],
    "output": "_site",
    "template": [
      "default",
      "modern"
    ],
    "globalMetadata": {
      "_appName": "Bank.App",
      "_appTitle": "Bank App",
      "_enableSearch": true,
      "pdf": true
    }
  }
}
```
> toc.yml
```Yaml
- name: Docs
  href: docs/
```
> index.md
```Markdown
---
_layout: landing
---

# This is the **HOMEPAGE**.

## [Diagrama de Clases](disenio.md)

## [Pruebas](Cobertura/SummaryGithub.md)
```
14. En el terminal, ejecutar el siguiente comando para generar la documentacion
```Bash
docfx metadata docfx.json
docfx build
```
15. En el terminal, ejecutar el(los) siguiente(s) comando(s) para realizar el analisis de codigo:
```Bash
semgrep scan --config='p/default' . --json --output semgrep.json
prospector-html --input semgrep.json --output semgrep-report.html --filter semgrep
```
16. En el paso anterior se genero un archivo `semgrep-report.html` el cual contiene el resultado del analisis que deberia ser similar a lo siguiente:

![image](https://github.com/user-attachments/assets/253bde0f-76e9-46bd-83b7-8cc17bb604ab)

17. En el VS Code, proceder a crear la carpeta .github/workflow y dentro de esta crear el archivo snyk.yml con el siguiente contenido.
```Yaml
name: Semgrep Analysis
env:
  DOTNET_VERSION: '8.x'                     # la versión de .NET
on: push
jobs:
  security:
    runs-on: ubuntu-latest
    container:
      # A Docker image with Semgrep installed. Do not change this.
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/setup@master
      # - name: Configurando la versión de NET
      #   uses: actions/setup-dotnet@v4
      #   with:
      #     dotnet-version: ${{ env.DOTNET_VERSION }}  
      - name: Semgrep scan
        run: semgrep scan --config="p/default" --sarif --output=report.sarif --metrics=off
      - name: Upload result to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: report.sarif
```

---
## Actividades Encargadas
2. Adicionar al archivo de semgrep.yml los pasos necesarios para generar el reporte en formato HTML y publicarlo dentro de un Github Page
2. Completar la documentación de todas las clases y generar una automatizaciòn .github/workflows/publish_docs.yml (Github Workflow) utilizando DocFx (init, metadata y build) y publicar el site de documentaciòn generado en un Github Page.
3. Generar una automatización de nombre .github/workflows/package_nuget.yml (Github Workflow) que ejecute:
   * Pruebas unitarias y reporte de pruebas automatizadas
   * Realice el analisis con SonarCloud.
   * Contruya un archivo .nuget a partir del proyecto Bank.Domain y lo publique como un Paquete de Github
4. Generar una automatización de nombre .github/workflows/release_version.yml (Github Workflow) que contruya la version (release) del paquete y publique en Github Releases e incluya pero ahi no esta el test unitarios
