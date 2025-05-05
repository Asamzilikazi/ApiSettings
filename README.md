# launchSettings

      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "Redis": "localhost:6379",
        "StorageAccountName": "rgcontainersprodfile",
        //DEV INT
        "Catalogue": "Data Source=10.11.3.11;Initial Catalog=DEVCatalogue;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        "LoadCatalogue": "Data Source=10.11.3.11;Initial Catalog=DEVLoadCatalogue;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        "RuntimeCatalogue": "Data Source=10.11.3.11;Initial Catalog=DEVRunTimeCatalogue;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        "LIVE": "Data Source=10.11.3.11;Initial Catalog=DEV_RTC_International;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        "CMS": "Data Source=10.11.3.11;Initial Catalog=DEVCMS;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;"
        //DEV ZA
        //"Catalogue": "Data Source=sqld-01.dev.flash.co.za;Initial Catalog=DEVCatalogue;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        //"LoadCatalogue": "Data Source=sqld-01.dev.flash.co.za;Initial Catalog=DEVLoadCatalogue;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        //"RuntimeCatalogue": "Data Source=sqld-01.dev.flash.co.za;Initial Catalog=DEVRunTimeCatalogue;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        //"LIVE": "Data Source=sqld-01.dev.flash.co.za;Initial Catalog=DEV_RTC_ZA;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;",
        //"CMS": "Data Source=sqld-01.dev.flash.co.za;Initial Catalog=DEVCMS;User Id=svc-DevCatalogue;Password=Flash2019@;Integrated Security=False;TrustServerCertificate=true;"
      },
      "dotnetRunMessages": true,
      "applicationUrl": "http://localhost:5169"
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "StorageAccountName": "rgcontainersprodfile"
      }
    },
    "WSL": {
      "commandName": "WSL2",
      "launchBrowser": true,
      "launchUrl": "https://localhost:7206/swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "ASPNETCORE_URLS": "https://localhost:7206;http://localhost:5169",
        "StorageAccountName": "rgcontainersprodfile"
      },
      "distributionName": ""
    }
  },
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:37920",
      "sslPort": 44312
    }
  }
}



# Program

using AutoMapper;
using Flash.ProductCatalogue.Api.Service;
using Flash.ProductCatalogue.API.Mapping;
using Flash.ProductCatalogue.Data.Repository;
using Flash.ProductCatalogue.Data.Utils;
using Flash.ProductCatalogue.Data.Repository.Implementation;
using Flash.ProductCatalogue.Data.Utils.Implementation;
using Flash.ProductCatalogue.API.Middleware;
using Serilog;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using HealthChecks.UI.Client;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Flash.ProductCatalogue.Infrastructure.BlobStore;
using Flash.ProductCatalogue.Infrastructure.BlobStore.Implementation;
using Flash.ProductCatalogue.Infrastructure;
using Flash.ProductCatalogue.Infrastructure.Cache;
using StackExchange.Redis;

namespace Flash.ProductCatalogue.API
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args);

            // Add AppSettings
            ApplicationSettings appSettings = new ApplicationSettings();
            builder.Configuration.Bind(appSettings);
            builder.Services.AddSingleton(appSettings);

            // Add Infrastructure
            builder.Services.AddInfrastructure(builder.Configuration);

            // Environment Variables: retrieve any to be used via AppSettings
            BlobStoreSettings? blobSettings = appSettings.BlobStoreSettings;
            if (blobSettings != null)
            {
                blobSettings.StorageAccountName = Environment.GetEnvironmentVariable("StorageAccountName");
            }

            // Register IConnectionMultiplexer
            string redisConnectionString = builder.Configuration.GetConnectionString("Redis");
            builder.Services.AddSingleton<IConnectionMultiplexer>(ConnectionMultiplexer.Connect(redisConnectionString));

            // Add services to the container.
            builder.Services.AddControllers();
            builder.Services.AddEndpointsApiExplorer();
            builder.Services.AddSwaggerGen();

            // Automapper
            var mapperConfig = new MapperConfiguration(mc =>
            {
                mc.AddProfile(new MappingProfile());
            });

            IMapper mapper = mapperConfig.CreateMapper();
            builder.Services.AddSingleton(mapper);
            builder.Services.AddSingleton<IRedisCacheService, RedisCacheService>();

            // Serilog
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Warning()
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.File("logs/ProductCatalogueAPI.log", rollOnFileSizeLimit: true)
                .CreateLogger();

            builder.Host.UseSerilog();

            // Global Exception Handler
            builder.Services.AddTransient<GlobalExceptionHandler>();

            // Register other services
            builder.Services.AddTransient<IDapperDataAccess, DapperDataAccess>();
            builder.Services.AddTransient<ISystemDataRepo, SystemDataRepo>();
            builder.Services.AddTransient<ISystemDataService, SystemDataService>();
            builder.Services.AddHttpClient();
            builder.Services.AddTransient<IExternalDataProviderRepo, ExternalDataProviderRepo>();
            builder.Services.AddTransient<IExternalDataProviderService, ExternalDataProviderService>();
            builder.Services.AddTransient<IReferenceDataRepo, ReferenceDataRepo>();
            builder.Services.AddTransient<IReferenceDataService, ReferenceDataService>();
            builder.Services.AddTransient<IMenuGenService, MenuGenService>();
            builder.Services.AddTransient<IMenuGenRepo, MenuGenRepo>();
            builder.Services.AddTransient<IWalletGenerationService, WalletGenerationService>();
            builder.Services.AddTransient<IWalletGenerationRepo, WalletGenerationRepo>();
            builder.Services.AddTransient<IPromotionsService, PromotionsService>();
            builder.Services.AddTransient<IPromotionsRepo, PromotionsRepo>();
            builder.Services.AddTransient<ILogoService, LogoService>();
            builder.Services.AddHttpContextAccessor();

            // Blob Store
            bool startupNullBlobStore = appSettings.StartupNullBlobStore;
            builder.Services.AddSingleton(blobSettings);
            if (startupNullBlobStore)
            {
                builder.Services.AddScoped<IBlobStore, NullBlobStore>();
            }
            else
            {
                builder.Services.AddScoped<IBlobStore, AzureBlobStore>();
            }

            // Health check
            string sqlCatalogueConnection = Environment.GetEnvironmentVariable(nameof(Databases.Catalogue));
            string sqlLoadCatalogueConnection = Environment.GetEnvironmentVariable(nameof(Databases.LoadCatalogue));
            string sqlRuntimeCatalogueConnection = Environment.GetEnvironmentVariable(nameof(Databases.RuntimeCatalogue));
            string sqlLIVEConnection = Environment.GetEnvironmentVariable(nameof(Databases.LIVE));
            string sqlCMSConnection = Environment.GetEnvironmentVariable(nameof(Databases.CMS));

            builder.Services.AddHealthChecks().AddSqlServer(connectionString: sqlCatalogueConnection, name: "Catalogue", timeout: TimeSpan.FromSeconds(30), tags: new[] { "startup", "full" });
            builder.Services.AddHealthChecks().AddSqlServer(connectionString: sqlLoadCatalogueConnection, name: "LoadCatalogue", timeout: TimeSpan.FromSeconds(30), tags: new[] { "startup", "full" });
            builder.Services.AddHealthChecks().AddSqlServer(connectionString: sqlRuntimeCatalogueConnection, name: "RuntimeCatalogue", timeout: TimeSpan.FromSeconds(30), tags: new[] { "startup", "full" });
            builder.Services.AddHealthChecks().AddSqlServer(connectionString: sqlLIVEConnection, name: "LIVE", timeout: TimeSpan.FromSeconds(30), tags: new[] { "startup", "full" });
            builder.Services.AddHealthChecks().AddSqlServer(connectionString: sqlCMSConnection, name: "CMS", timeout: TimeSpan.FromSeconds(30), tags: new[] { "startup", "full" });
            builder.Services.AddHealthChecks().AddRedis(redisConnectionString, name: "Redis", timeout: TimeSpan.FromSeconds(30), tags: new[] { "startup", "full" });

            // Connection health check
            builder.Services.AddHealthChecks().AddCheck(name: "Catalogue_API", check: () => { return HealthCheckResult.Healthy(); }, tags: new[] { "ready" });

            var app = builder.Build();


            //Configure the HTTP request pipeline.
            app.UseSwagger();
            app.UseSwaggerUI();

            app.UseMiddleware<GlobalExceptionHandler>();

            app.UseRouting();

            app.UseAuthorization();

            app.UseStatusCodePages();

            app.MapControllers();

            //Add health check endpoints
            app.UseHealthChecks("/health", new HealthCheckOptions
            {
                Predicate = healthCheck => healthCheck.Tags.Contains("ready"),
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });
            app.UseHealthChecks("/health/ready", new HealthCheckOptions
            {
                Predicate = healthCheck => healthCheck.Tags.Contains("ready"),
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });
            app.UseHealthChecks("/health/live", new HealthCheckOptions
            {
                Predicate = healthCheck => healthCheck.Tags.Contains("ready"),
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });
            app.UseHealthChecks("/health/full", new HealthCheckOptions
            {
                Predicate = healthCheck => healthCheck.Tags.Contains("full"),
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });
            app.UseHealthChecks("/health/startup", new HealthCheckOptions
            {
                Predicate = healthCheck => healthCheck.Tags.Contains("startup"),
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });

            app.Run();
        }
    }
}


#appsettings
{
  "ApplicationName": "Product Catalogue API",
  "BlobStoreSettings": {
    "ContainerPrefix": "ProductCatalogue_LOCALDEV",
    "DummyFilePath": "C:\\Code\\Flash\\PCM\\BlobStore\\",
    "StorageAccountName": "thisWillBeSetInProgram.cs"
  },
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false"
  },
  "RabbitMQ": {
  },
  "RedisConfig": {
    "TraderProfileExpiryDays": 7,
    "PromoProductsExpiryDays": 7
  },
  "InstanceFlags": {
    "UseRedis": false,
    "UseRabbitMQ": true
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "StartupNullBlobStore": true
}  ![image](https://github.com/user-attachments/assets/026f0d9e-2d03-4404-89fa-3fac371cdcae)
url/path for sourcetreer, api ;: https://flashmobilevending.visualstudio.com/DefaultCollection/Product%20Catalogue%20Management/_git/Product%20Catalogue%20API

ui sourcetree https://flashmobilevending.visualstudio.com/DefaultCollection/Product%20Catalogue%20Management/_git/Product%20Catalogue%20Management  with azuredevops login
