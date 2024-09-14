
# Prueba Práctica: Creando una API de Productos

## Objetivo

Desarrollar una API REST utilizando .NET Core 7 y C# que permita realizar operaciones CRUD en una tienda de productos.

## Especificaciones

### Modelo de Datos

Crea una clase `Product` que incluya propiedades para `Id`, `Name`, `Description`, `Price` y `Category`.

### Operaciones CRUD

Implementa las operaciones CRUD (Create, Read, Update, Delete) para `Product`.

### API Endpoints que debes implementar

- `GET /api/products` - Devuelve una lista de todos los productos.
- `GET /api/products/{id}` - Devuelve un solo producto basado en su ID.
- `POST /api/products` - Crea un nuevo producto.
- `PUT /api/products/{id}` - Actualiza un producto existente.
- `DELETE /api/products/{id}` - Elimina un producto.

### Base de Datos

Usa Dapper para implementar el patrón de repositorio. La base de datos puede ser una base de datos en memoria para simplificar la prueba o en SQL Server en tu máquina local.

## Requerimientos

- El proyecto debe ser creado en .NET Core 7 con C#.
- Debe implementar las operaciones CRUD mencionadas anteriormente.

## Entrega

Por favor, crea un repositorio público en GitHub con un archivo README que explique los requerimientos necesarios para poder ejecutar la aplicación.

## Evaluación

La prueba se evaluará en base a la calidad del código, la funcionalidad, el manejo de errores y el cumplimiento de las especificaciones.

---

### SOLUCIÓN

Aquí te dejo el paso a paso para crear una API de productos con .NET Core 7 y Dapper para implementar el patrón de repositorio, siguiendo las especificaciones que me diste.

#### Paso 1: Crear un proyecto de API en .NET Core

1. Abre Visual Studio 2022.
2. Crea un nuevo proyecto seleccionando "ASP.NET Core Web API".
3. Configura el proyecto con .NET 7 y nómbralo `ProductApi`.

#### Paso 2: Crear el modelo `Product`

Dentro de la carpeta `Models`, crea la clase `Product.cs`:

```C#
// Models/Product.cs
namespace ProductApi.Models
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Description { get; set; }
        public decimal Price { get; set; }
        public string Category { get; set; }
    }
}
```

#### Paso 3: Configurar la conexión a la base de datos y Dapper

Agrega el paquete NuGet de Dapper. En la terminal de Visual Studio, ejecuta:

```
dotnet add package Dapper
```

Luego, en `appsettings.json`, configura la cadena de conexión a SQL Server o una base de datos en memoria. Por ejemplo, para SQL Server local:

```json
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ProductDb;Trusted_Connection=True;"
  },
  "AllowedHosts": "*"
}
```

#### Paso 4: Crear el repositorio para manejar operaciones de base de datos

Crea la interfaz `IProductRepository.cs` dentro de una carpeta `Repositories`:

```C#
// Repositories/IProductRepository.cs
using ProductApi.Models;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace ProductApi.Repositories
{
    public interface IProductRepository
    {
        Task<IEnumerable<Product>> GetAllAsync();
        Task<Product> GetByIdAsync(int id);
        Task<int> CreateAsync(Product product);
        Task<int> UpdateAsync(Product product);
        Task<int> DeleteAsync(int id);
    }
}
```

Implementa la interfaz en la clase `ProductRepository.cs`:

```C#
// Repositories/ProductRepository.cs
using Dapper;
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Configuration;
using ProductApi.Models;
using System.Collections.Generic;
using System.Data;
using System.Threading.Tasks;

namespace ProductApi.Repositories
{
    public class ProductRepository : IProductRepository
    {
        private readonly string _connectionString;

        public ProductRepository(IConfiguration configuration)
        {
            _connectionString = configuration.GetConnectionString("DefaultConnection");
        }

        public async Task<IEnumerable<Product>> GetAllAsync()
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                return await connection.QueryAsync<Product>("SELECT * FROM Products");
            }
        }

        public async Task<Product> GetByIdAsync(int id)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                return await connection.QueryFirstOrDefaultAsync<Product>("SELECT * FROM Products WHERE Id = @Id", new { Id = id });
            }
        }

        public async Task<int> CreateAsync(Product product)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                var sql = "INSERT INTO Products (Name, Description, Price, Category) VALUES (@Name, @Description, @Price, @Category)";
                return await connection.ExecuteAsync(sql, product);
            }
        }

        public async Task<int> UpdateAsync(Product product)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                var sql = "UPDATE Products SET Name = @Name, Description = @Description, Price = @Price, Category = @Category WHERE Id = @Id";
                return await connection.ExecuteAsync(sql, product);
            }
        }

        public async Task<int> DeleteAsync(int id)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                var sql = "DELETE FROM Products WHERE Id = @Id";
                return await connection.ExecuteAsync(sql, new { Id = id });
            }
        }
    }
}
```

#### Paso 5: Registrar el repositorio en el contenedor de dependencias

Abre `Program.cs` y agrega el repositorio en el contenedor de servicios:

```C#
// Program.cs
using ProductApi.Repositories;

var builder = WebApplication.CreateBuilder(args);

// Agregar el repositorio
builder.Services.AddScoped<IProductRepository, ProductRepository>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

#### Paso 6: Crear el controlador `ProductsController`

Crea el archivo `ProductsController.cs` dentro de la carpeta `Controllers`:

```C#
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using ProductApi.Models;
using ProductApi.Repositories;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace ProductApi.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ProductsController : ControllerBase
    {
        private readonly IProductRepository _productRepository;

        public ProductsController(IProductRepository productRepository)
        {
            _productRepository = productRepository;
        }

        [HttpGet]
        public async Task<ActionResult<IEnumerable<Product>>> GetAll()
        {
            var products = await _productRepository.GetAllAsync();
            return Ok(products);
        }

        [HttpGet("{id}")]
        public async Task<ActionResult<Product>> GetById(int id)
        {
            var product = await _productRepository.GetByIdAsync(id);
            if (product == null) return NotFound();
            return Ok(product);
        }

        [HttpPost]
        public async Task<ActionResult> Create(Product product)
        {
            await _productRepository.CreateAsync(product);
            return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
        }

        [HttpPut("{id}")]
        public async Task<ActionResult> Update(int id, Product product)
        {
            if (id != product.Id) return BadRequest();
            await _productRepository.UpdateAsync(product);
            return NoContent();
        }

        [HttpDelete("{id}")]
        public async Task<ActionResult> Delete(int id)
        {
            await _productRepository.DeleteAsync(id);
            return NoContent();
        }
    }
}
```

#### Paso 7: Crear la base de datos

Ejecuta el siguiente script SQL para crear la tabla `Products`:

```sql
CREATE TABLE Products (
    Id INT PRIMARY KEY IDENTITY(1,1),
    Name NVARCHAR(100),
    Description NVARCHAR(255),
    Price DECIMAL(18, 2),
    Category NVARCHAR(50)
);
```
