# Guía rápida: migraciones, nuevo controlador y pruebas (ESP)

Este README describe, paso a paso, cómo realizar migraciones de Entity Framework Core, añadir un nuevo controlador y exponer una nueva API en este proyecto ASP.NET Core (.NET 8). También incluye ejemplos de prueba desde PowerShell, curl y el archivo `ApiExample.http` incluido.

Requisitos previos
- .NET 8 SDK instalado (dotnet)
- dotnet-ef (opcional, puede usarse desde CLI o Package Manager Console)
- PowerShell (se muestran ejemplos con `Invoke-RestMethod`)

Comprobaciones rápidas

1. Abrir una terminal en la carpeta del proyecto (donde está `ApiExample.csproj`).
2. Comprobar la SDK: `dotnet --info`.
3. Restaurar paquetes: `dotnet restore`.

1) Configurar y ejecutar migraciones EF Core

Nota: Este proyecto tiene una carpeta `Context/AppDbContext.cs` y el modelo `Models/User.cs`.

a) Instalar herramientas (si no las tienes):

PowerShell (ejecutar desde la carpeta del proyecto):

```powershell
# Instala la herramienta de EF Core localmente si no existe
dotnet tool install --global dotnet-ef
```

b) Añadir una nueva migración

1. Abre PowerShell en la raíz del proyecto (misma carpeta que `ApiExample.csproj`).
2. Crea la migración:

```powershell
dotnet ef migrations add NombreDeLaMigracion
```

Ejemplo:

```powershell
dotnet ef migrations add AddIsActiveToUser
```

c) Aplicar la migración a la base de datos

```powershell
dotnet ef database update
```

d) Problemas comunes
- Error: "No se puede inferir el proveedor de la base de datos" → Asegúrate de que `AppDbContext` esté registrado en `Program.cs` con `builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString));` o `UseInMemoryDatabase` para pruebas.
- Error: "Unable to create an object of type 'AppDbContext'" → Revisa el constructor del contexto y si tienes un `DesignTimeDbContextFactory` o si `Program.cs` tiene la configuración correcta.

2) Añadir un nuevo controlador (ejemplo)

Archivo sugerido: `Controllers/ProductsController.cs`

Código de ejemplo (C#):

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products
    [HttpGet]
    public ActionResult<IEnumerable<object>> Get()
    {
        var list = new [] { new { id = 1, name = "Widget" }, new { id = 2, name = "Gadget" } };
        return Ok(list);
    }

    // GET: api/products/1
    [HttpGet("{id:int}")]
    public ActionResult<object> Get(int id)
    {
        return Ok(new { id = id, name = $"Product {id}" });
    }

    // POST: api/products
    [HttpPost]
    public ActionResult Post([FromBody] object model)
    {
        // Aquí normalmente guardas en la BBDD
        return CreatedAtAction(nameof(Get), new { id = 123 }, model);
    }
}
```

Pasos para agregarlo:
1. Crea `Controllers/ProductsController.cs` con el código anterior.
2. Ejecuta la aplicación:

```powershell
dotnet run
```

3. Prueba los endpoints: ver sección "Probar la API".

3) Añadir una nueva API que use el DbContext (ejemplo con `User`)

Ejemplo breve para `UsersController` ya existe en este proyecto. Para crear uno nuevo que use `AppDbContext`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Context; // Ajusta el namespace según tu proyecto
using Models;

[ApiController]
[Route("api/[controller]")]
public class SampleUsersController : ControllerBase
{
    private readonly AppDbContext _db;
    public SampleUsersController(AppDbContext db) => _db = db;

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var users = await _db.Users.ToListAsync();
        return Ok(users);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] User user)
    {
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return CreatedAtAction(nameof(GetAll), new { id = user.Id }, user);
    }
}
```

Notas:
- Asegúrate de que `AppDbContext` exponga `DbSet<User> Users { get; set; }` y que el `User` tenga una propiedad `Id`.
- Revisa nombres de namespaces: en este repo `Context/AppDbContext.cs` y `Models/User.cs`.

4) Probar la API

a) Usando PowerShell (Invoke-RestMethod)

GET:

```powershell
Invoke-RestMethod -Method Get -Uri http://localhost:5000/api/products
```

POST (JSON):

```powershell
$body = @{ name = 'Nuevo' } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri http://localhost:5000/api/products -Body $body -ContentType 'application/json'
```

b) Usando curl (si lo tienes instalado en Windows):

```powershell
curl -X GET http://localhost:5000/api/products
curl -X POST http://localhost:5000/api/products -H "Content-Type: application/json" -d '{"name":"Nuevo"}'
```

c) Usando `ApiExample.http`

Abre el archivo `ApiExample.http` que ya está en el proyecto. Este tipo de archivos funciona con extensiones como REST Client en VS Code. Añade o modifica una petición como:

```
### Get products
GET http://localhost:5000/api/products

### Create product
POST http://localhost:5000/api/products
Content-Type: application/json

{
  "name": "Nuevo"
}
```

5) Debug y puertos

Por defecto, `dotnet run` en lanzamiento de desarrollo puede exponer Kestrel en puertos 5000/5001 o el puerto definido por `launchSettings.json` o variables de entorno. Observa la salida de `dotnet run` para la URL exacta (ej. "Now listening on: http://localhost:5234").

6) Buenas prácticas y siguientes pasos

- Usa migraciones pequeñas y con nombres significativos.
- Si haces cambios grandes en modelos, crea una migración y revisa el SQL generado (usa `dotnet ef migrations script` si quieres revisar).
- Añade validación de modelos en controllers con `[ApiController]` y `ModelState.IsValid` o atributos de data annotations.
- Añade tests unitarios o de integración para endpoints críticos.

7) Troubleshooting rápido

- Error: 500 Interno → Mira salida de la consola y logs. Habilita Developer Exception Page en `Program.cs` cuando estés en development.
- Error: 404 → Verifica rutas y atributos `[Route]`/`[HttpGet]` y que la app esté corriendo en el puerto correcto.
- Error de migraciones: revisa dependencias de EF Core en `.csproj` y versiones compatibles con .NET 8.

8) Recursos útiles

- Documentación EF Core: https://learn.microsoft.com/ef/core/
- ASP.NET Core Web API: https://learn.microsoft.com/aspnet/core/web-api/
