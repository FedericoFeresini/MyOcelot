# OCELOT

## API Gateway open source in .NET Core.
<details>
<summary>Offre i seguenti servizi</summary>
<p>
* Routing
* Request Aggregation
* Server discovery
* Authentication
* Authorization
* Rate Limiting
* Caching
* Load balancing
* Logging
</p>
</details>

Per capire come si usa immaginiamo di creare due applicazioni API:
* **CatalogAPI**
* **OrderesAPI**
E poi creare il Gateway che gestisce le chiamate e le reidirizza al servizio corretto.

1. Creo i due microservizi indipendenti che vogliamo mettere dietro al nostro Gateway creato con Ocelot (CatalogAPI e OrdersAPI):
```bash
                      -----------> CatalogAPI
                      |
                      |
Client- -------->   OCELOT 
                      |
                      |
                      -----------> OrderesAPI   
```
Per farlo creo semplicemente due progetti WebAPI con Visual studio.

#### Se intendiamo fare esperimenti in locale, serve che questi due microservizi API, quando lanciati sulla stessa macchina, girino su porte diverse. Per farlo basta moficiare il file launchSettings.json <br/>
<details>
<summary><p>**launchSettings.json** di CatalogAPI</p></summary>
<p>

```bash
[...]

"CatalogAPI": {
      "commandName": "Project",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "applicationUrl": "https://localhost:7000"
    },

[...]
```
</p>
</details>

<details>
<summary><b>**launchSettings.json** di OrdersAPI</b></summary>
<p>

```bash
[...]

"OrdersAPI": {
      "commandName": "Project",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "applicationUrl": "https://localhost:7001"
    },

[...]
```
</p>
</details>

Cosi facendo, una volta lanciati, avremo: 
* OrdersAPI sulla porta 7001
* CatalogAPI sulla porta 7000

2. Importo il NuGet necessario per implementare Ocelot:
```bash
Install-Package Ocelot
```
_O possiamo utilizzare direttamente il NuGet manager da Visual Studio_

3. Creo il GatewayApi. Anche questo e'un altro progetto WebAPI indipendente dai precedenti.
 1. Crea il progetto 
 2. Modifica **Program.cs** perche' il programma venga lanciato con le configurazioni di Ocelot
```csharp
 //Asp.Net Core 2
 public class Program
    {
        public static void Main(string[] args)
        {
            CreateWebHostBuilder(args).Build().Run();
        }

        public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .ConfigureAppConfiguration((host, config) =>
                {
                    config
                    	.SetBasePath(AppContext.BaseDirectory)
                        .AddJsonFile("appsettings.json", false, true)
                        .AddJsonFile("ocelot.json", false, true)
                        .Build();
                })
            .UseStartup<Startup>();
    }
```
Verra' utilizzato il file _ocelot.json_ per configurare GatewayAPI quando viene lanciato.

#### NB: In Asp.Net Core 3.0 WebHost e' stato sostituito con il piu generico Host. Come tale in un progetto Asp.Net Core 3.0 Program.cs risulterebbe: <br/>

```csharp
    //Asp.Net Core 3
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.ConfigureAppConfiguration((host, config) => {
                        config
                        	.SetBasePath(AppContext.BaseDirectory)
                        	.AddJsonFile("appsettings.json", false, true)
                        	.AddJsonFile("ocelot.json", false, true)
                        	.Build();
                    })

                    webBuilder.UseStartup<Startup>();
                });
    }
```
	3. Aggiungi il Middleware di Ocelot in **Startup.cs**
```csharp
    public class Startup
    {
        public IConfiguration Configuration { get; }

        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }
       
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddOcelot(Configuration); // DA AGGIUNGERE
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
               app.UseDeveloperExceptionPage();
            }
            else
            {
                app.UseHsts(); // DA AGGIUNGERE
            }

            app.UseHttpsRedirection(); // DA AGGIUNGERE
            app.UseRouting();
            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGet("/", async context =>
                {
                    await context.Response.WriteAsync("Hello World!");
                });
            });
            app.UseOcelot().Wait(); // DA AGGIUNGERE ALLA FINE DI TUTTO!

        }
    }
```

4. Andiamo ora a vedere come creare il file di configurazione _ocelot.json_
**ocelot.json**
```json
{
  "ReRoutes": [
    {
      "DownstreamPathTemplate": "/api/{version}/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "catalog-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/{version}/{everything}",
      "UpstreamHttpMethod": [ "POST", "PUT", "GET" ],
      "HttpHandlerOptions" : {
      	"AllowAutoRedirect" : true
      }
    },
    {
      "DownstreamPathTemplate": "/api/{version}/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "basket-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/v1/{everything}",
      "UpstreamHttpMethod": [ "POST", "PUT", "GET" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "IdentityApiKey",
        "AllowedScopes": []
      }
    }

  ],
    "GlobalConfiguration": {
      "BaseUrl" : "https://localhost:7003",
      "RequestIdKey": "OcRequestId",
      "AdministrationPath": "/administration"
    }
  }
```
Parti salienti:
* **"ReRoutes": []** - In questa sezione viene spiegato il comportamento di Ocelot per le chiamate che arrivano
	* **UpstreamPathTemplate** - Enpoint del Gateway
	* **DownstreamPathTemplate** - Endpoint che il gateway contattera'
	* **Priority** - Quando ci sono due UpstreamPathTemplate che possono combaciare, il Gateway decidera' quale utilizzare in base a chi ha Priority piu' alta (Nell'esempio sopra _/api/{version}/{everything}_ e _/api/v1/{everything}_ sono due UpstreamPathTemplate che possono combaciare)
	* **AllowAutoRedirect** - Si spiega bene con un esempio. Se questo parametro non venisse settato (o settato **false**), quando col Browser chiamo GET _https://localhost:7003/api/v1/c/1_, il browser verra' reindirizzato al link _http://catalog-api:80//api/v1/1_. Se invece voglio nascondere il secondo link e mantenere il Browser sul primo link pur ottenendo lo stesso risultato, allora questo paramtro va settato **true**
* **"GlobalConfiguration": {}** - Si definiscono dei settings che sovrascrivono quelli di ReRoutes, utile perche' altrimenti ripetere un sacco di informazioni in ReRoutes.
	* **BaseUrl** - Questo parametro e' molto importante e rappresenta l'url con cui servizi esterni vedranno Ocelot. Ad esempio supponiamo che Gateway venga lanciato in un container all' url http://123.12.1.1:6342, questo e' l'indirizzo da settare.
	Mettiamo ora che pero' davanti al Gateway Ocelot ci sia un load balancer Ngix all'indirizzo http://myApp.com, sara' questo allora il valore da settare.
	* **RequestIdKey** - Definisce un Header contenente il CorrelationId della richiesta. Questo Header poi verra' passato anche alla richiestra tra Gateway e Servizio. In caso questo header non venisse trovato, il middleware lo andra' a creare.
	* **AdministrationPath** - Definisce l'endpoint con la quale sara' possibile fare delle operazioni runtime su Ocelot (Vedi avanti)

#### Aggregation: Ocelot offre anche una feature che con una sola chiamata permette in chiamare due servizi diversi dietro le quinte e ritornare una rsisposta che e' l'aggregazione delle due:
```json
{
//-----------------------------ReRoutes
  "ReRoutes": [
    {
//----->GetCatalog
      "DownstreamPathTemplate": "/api/{version}/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "catalog-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/{version}/{everything}",
      "UpstreamHttpMethod": [ "GET" ],
      "Key" : "GetCatalog",
      "UpstreamHeaderTransform" : {
      	"Accept-Encoding" : "gzip;q=0"
      }
    },
//----->GetBasket
    {
      "DownstreamPathTemplate": "/api/{version}/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "basket-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/{version}/{everything}",
      "UpstreamHttpMethod": [ "GET" ],
      "Key" : "GetBasket",
      "UpstreamHeaderTransform" : {
      	"Accept-Encoding" : "gzip;q=0"
      }
    }
  ],
//-----------------------------GlobalConfiguration
    "GlobalConfiguration": {
      "BaseUrl" : "https://localhost:7003",
    },
//-----------------------------Aggregates
	"Aggregates" : [
		{
			"ReRouteKeys" : [
				"GetCatalog",
				"GetBasket"
			],
			"UpstreamPathTemplate": "/api/aggregate/{version}/{everything}"
		}
	]
  }
```
Dove:
* **Aggregates** - Sezione che crea un endpoint vituale che verra' utilizzato per aggregare piu richieste (nell'esempio l'endpoint sara' _/api/aggregate/{version}/{everything}_ e aggrega le chiamate ley cui **Key** sono _GetCatalog_ e _GetBasket_)
* **UpstreamHeaderTransform** - Il Gateway aggiunge Headers alle chiamate che saranno inviate ai Backend (Nel nostro caso disabilitiamo la compressione, altrimenti quando il messaggio verra' ritornato dai due backend otterremo qualcosa di decifrato ed incomprensibile).

La risposta che si ottiene e' di questo genere:
```json
/* 
 * GET https://localhost:7003/api/aggregate/v1/5
 */

//RESPONSE
{
	"GetCatalog" : {
		//Valori ritornati da GetCatalog
	},
	"GetBasket" : {
		//Valori ritornati da GetBasket
	}
}
```
#### E' possibile modificare il modo con cui vengono ritornati i dati aggreati (tipo eliminare le due sottocategorie, e ritornare un corpo unico, o ritornare solo alcuni dei campi). Per farlo sara' necessario creare una classe che implementi IDefinedAggregator. Non vediamo questo caso.

### Throttling 
Ocelot offre anche metodi di Throttling per evitare che un utente chiami troppe volte lo stesso metodo.
1. Per attivare queste feature innanzi tutto andiamo ad aggiungere _"RateLimitOptions"_ ad ocelet.json
```json
//-----------------------------GlobalConfiguration
    "GlobalConfiguration": {
      "BaseUrl" : "https://localhost:7003",
      "RateLimitOptions" : {
      	"DisableRateLimitHeaders" : false,
      	"QuotaExceededMessage" : "Please stop!",
      	"HttpStatusCode" : 419,
      	"ClientIdHeader" : "TestHeader" 
      }
    }
```
Dove: 
* **QuotaExceededMessage** - E' il messaggio che viene ritornato quando si eccede il throttling
* **HttpStatusCode** - E' uno Status Code customizzato per questa risposta
* **ClientIdHeader** - Questo parametro definisce un Header da cercare nella richiesta. Questo Header serve ad identificare l'utente che ha fatto la chiamata (posso immaginare di mettere come valore a questo Header il ClientId preso dal Claim estratto da un JWT). Questo perche' ovviamente il Throttling viene applicato al singolo utente.
2. Andiamo ora a decidere quale Route avra' il Throttling e secondo quale politica usando _RateLimitOptions{}_:
```json
"ReRoutes": [
    {
//----->GetCatalog
      "DownstreamPathTemplate": "/api/{version}/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "catalog-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/{version}/{everything}",
      "UpstreamHttpMethod": [ "GET" ],
      "RateLimitOptions" : {
      	"ClientWhiteList" : [],
      	"EnableRateLimiting" : true,
      	"Period" : "10s",
      	"PeriodTimespan" : 30,
      	"limit" : 3
      }
    }
  ]
```
Dove:
* **Period** - Periodo di tempo da considerare
* **limit** - Numero di chiamate (quindi in questo caso 3 chiamate negli ultimi 10 secondi)
* **PeriodTimespan** - Se esageri con le chiamate, per quanto tempo vieni fermato (Se fai piu di 3 chiamate in 10 secondi riceverai il messaggio di Stop e verrai bloccato per 30 secondi)

