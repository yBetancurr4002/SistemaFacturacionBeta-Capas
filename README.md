# SistemaFacturacionBeta-Capas
Built just for academic purposes

## Capa de datos

### **1. Estructura del Proyecto**

Organiza tu solución en Visual Studio con las siguientes carpetas:

```css
SistemaFacturacion
│
├── CapaDatos
│   ├── Contextos
│   │   └── SistemaFacturacionContext.cs
│   ├── Entidades
│   │   ├── Persona.cs
│   │   ├── Empresa.cs
│   │   ├── TipoCliente.cs
│   │   ├── Cliente.cs
│   │   ├── Usuario.cs
│   │   ├── Rol.cs
│   │   ├── Permiso.cs
│   │   ├── Producto.cs
│   │   ├── Factura.cs
│   │   ├── DetalleFactura.cs
│   │   └── Pago.cs
│   └── Repositorios
│       ├── GenericRepository.cs
│       ├── PersonaRepository.cs
│       ├── EmpresaRepository.cs
│       └── ... (otros repositorios específicos)
│
└── Migraciones
    └── ... (archivos de migración generados por EF Core)
  ```

### **2. Configuración del Contexto (DbContext)**

#### **Configurar `appsettings.json`**

Crea un archivo llamado `appsettings.json` en la raíz del proyecto. Este archivo almacenará la cadena de conexión y otras configuraciones globales.

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=.;Initial Catalog=SistemaFacturacionDB;Integrated Security=true;Encrypt=True;TrustServerCertificate=true;"
  }
}
```

El `DbContext` es el punto central de Entity Framework Core para interactuar con la base de datos. Define las entidades y sus relaciones.

```cs
using Microsoft.EntityFrameworkCore;

namespace SistemaFacturacion.CapaDatos.Contextos
{
    public class SistemaFacturacionContext : DbContext
    {
        private readonly string _connectionString;

        public SistemaFacturacionContext(DbContextOptions<SistemaFacturacionContext> options, string connectionString)
            : base(options)
        {
            _connectionString = connectionString;
        }

        public DbSet<Persona> Personas { get; set; }
        public DbSet<Empresa> Empresas { get; set; }
        public DbSet<TipoCliente> TiposCliente { get; set; }
        public DbSet<Cliente> Clientes { get; set; }
        public DbSet<Usuario> Usuarios { get; set; }
        public DbSet<Rol> Roles { get; set; }
        public DbSet<Permiso> Permisos { get; set; }
        public DbSet<Producto> Productos { get; set; }
        public DbSet<Factura> Facturas { get; set; }
        public DbSet<DetalleFactura> DetallesFactura { get; set; }
        public DbSet<Pago> Pagos { get; set; }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            if (!optionsBuilder.IsConfigured)
            {
                optionsBuilder.UseSqlServer(_connectionString);
            }
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Configuración de relaciones y restricciones adicionales
            modelBuilder.Entity<Cliente>()
                .HasCheckConstraint("CHK_Cliente_PersonaEmpresa", "(PersonaId IS NOT NULL AND EmpresaId IS NULL) OR (PersonaId IS NULL AND EmpresaId IS NOT NULL)");

            modelBuilder.Entity<DetalleFactura>()
                .Property(d => d.Subtotal)
                .HasComputedColumnSql("Cantidad * PrecioUnitario");

            base.OnModelCreating(modelBuilder);
        }
    }
}
```

### **Configurar `Program.cs` para Leer `appsettings.json`**

En el archivo `Program.cs`, configuraremos la aplicación para leer el archivo `appsettings.json` y pasar la cadena de conexión al `DbContext`.

```cs
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;
using SistemaFacturacion.CapaDatos.Contextos;

var builder = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);

IConfiguration configuration = builder.Build();

var services = new ServiceCollection();
services.AddDbContext<SistemaFacturacionContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

var serviceProvider = services.BuildServiceProvider();
```

### **3. Definición de Entidades**

Las entidades representan las tablas de la base de datos. Cada entidad debe mapear los campos de las tablas.

Ejemplo. Persona

```cs
using System;

namespace SistemaFacturacion.CapaDatos.Entidades
{
    public class Persona
    {
        public int Id { get; set; }
        public string Nombre { get; set; }
        public string Apellido { get; set; }
        public DateTime FechaNacimiento { get; set; }
        public string TipoDocumento { get; set; }
        public string NumeroDocumento { get; set; }
        public string Direccion { get; set; }
        public string Telefono { get; set; }
        public string Email { get; set; }
        public string CreadoPor { get; set; }
        public DateTime FechaCreacion { get; set; }
        public string ModificadoPor { get; set; }
        public DateTime? FechaModificacion { get; set; }
    }
}
```

### **4. Creación de Repositorios**

Los repositorios encapsulan la lógica de acceso a datos para cada entidad. Implementamos un repositorio genérico para reutilizar código común.

`GenericRepository<T>`:
```cs
using Microsoft.EntityFrameworkCore;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace SistemaFacturacion.CapaDatos.Repositorios
{
    public class GenericRepository<TEntity> where TEntity : class
    {
        private readonly SistemaFacturacionContext _context;

        public GenericRepository(SistemaFacturacionContext context)
        {
            _context = context;
        }

        public async Task<IEnumerable<TEntity>> GetAllAsync()
        {
            return await _context.Set<TEntity>().ToListAsync();
        }

        public async Task<TEntity> GetByIdAsync(int id)
        {
            return await _context.Set<TEntity>().FindAsync(id);
        }

        public async Task AddAsync(TEntity entity)
        {
            await _context.Set<TEntity>().AddAsync(entity);
            await _context.SaveChangesAsync();
        }

        public async Task UpdateAsync(TEntity entity)
        {
            _context.Set<TEntity>().Update(entity);
            await _context.SaveChangesAsync();
        }

        public async Task DeleteAsync(int id)
        {
            var entity = await _context.Set<TEntity>().FindAsync(id);
            if (entity != null)
            {
                _context.Set<TEntity>().Remove(entity);
                await _context.SaveChangesAsync();
            }
        }
    }
}
```

**Archivo: PersonaRepository.cs**  
```cs
using SistemaFacturacion.CapaDatos.Entidades;
using System.Threading.Tasks;

namespace SistemaFacturacion.CapaDatos.Repositorios
{
    public class PersonaRepository : GenericRepository<Persona>
    {
        private readonly SistemaFacturacionContext _context;

        public PersonaRepository(SistemaFacturacionContext context) : base(context)
        {
            _context = context;
        }

        public async Task<Persona> GetByNumeroDocumentoAsync(string numeroDocumento)
        {
            return await _context.Personas.FirstOrDefaultAsync(p => p.NumeroDocumento == numeroDocumento);
        }
    }
}
```
