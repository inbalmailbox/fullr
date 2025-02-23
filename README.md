# Full-Stack Application Setup Guide

> **Prerequisites:**  
> - Visual Studio 2022 (or your preferred IDE)  
> - .NET SDK (e.g., .NET 6/7/8)  
> - SQL Server (or SQL Server Express)  
> - Node.js (for the React client)

---

## PART 1: ASP.NET CORE WEB API (Server)

### 1. Create a New ASP.NET Core Web API Project
- In Visual Studio:  
  1. **File > New > Project**.
  2. Choose **ASP.NET Core Web API**.
  3. Name your project (e.g., `MyFullStackApp.Server`) and select your target framework.
  4. Enable HTTPS and Swagger if prompted.

### 2. Install Necessary Packages
Open the Package Manager Console (or use the CLI) and run:
```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

### 3. Create the Data Model
Create a file called **Product.cs** (e.g., in a `Models` folder):
```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}
```

### 4. Create the DbContext
Create a file called **AppDbContext.cs** (e.g., in a `Data` folder):
```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }
}
```

### 5. Configure the Connection String
Edit **appsettings.json** to include your connection string (update `YOUR_SERVER` and `YOUR_DB` as needed):
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=YOUR_SERVER;Database=YOUR_DB;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

### 6. Update Program.cs
Replace the contents of **Program.cs** with:
```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register DbContext with SQL Server
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// Enable Swagger in Development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    });
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

// Optional: Default route for testing
app.MapGet("/", () => "Server is running!");

app.Run();
```

### 7. Create the CRUD Controller
Create a new file **ProductsController.cs** in a `Controllers` folder:
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly AppDbContext _context;

    public ProductsController(AppDbContext context)
    {
        _context = context;
    }

    // GET: api/products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        return await _context.Products.ToListAsync();
    }

    // GET: api/products/{id}
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null)
            return NotFound();

        return product;
    }

    // POST: api/products
    [HttpPost]
    public async Task<ActionResult<Product>> PostProduct(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }

    // PUT: api/products/{id}
    [HttpPut("{id}")]
    public async Task<IActionResult> PutProduct(int id, Product product)
    {
        if (id != product.Id)
            return BadRequest();

        _context.Entry(product).State = EntityState.Modified;
        await _context.SaveChangesAsync();
        return NoContent();
    }

    // DELETE: api/products/{id}
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null)
            return NotFound();

        _context.Products.Remove(product);
        await _context.SaveChangesAsync();
        return NoContent();
    }
}
```

### 8. Create the Database
In the Package Manager Console (or terminal), run:
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

Your server is now ready. Run the project (F5) and verify:
- **Swagger UI:** `https://localhost:<port>/swagger/index.html`
- **API Endpoint:** `http://localhost:<port>/api/products`

---

## PART 2: REACT CLIENT (Using Create React App & Axios)

### 1. Create the React App
Open a terminal where you want the client project and run:
```bash
npx create-react-app client
```
This creates a folder named **client**.

### 2. Install Axios
Navigate into the **client** folder and run:
```bash
cd client
npm install axios
```

### 3. Configure the Proxy
Edit **client/package.json** to add the proxy setting (adjust the port if needed):
```json
{
  "name": "client",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "axios": "^1.3.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1",
    "web-vitals": "^2.1.4"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "proxy": "http://localhost:5191"
}
```

### 4. Set Up the React Code

#### File: `src/index.js`
```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

#### File: `src/App.js`
```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './App.css';

function App() {
  // State for products list
  const [products, setProducts] = useState([]);
  // State for new product form
  const [newProduct, setNewProduct] = useState({ name: '', price: 0 });
  // State to track which product is being edited
  const [editingProductId, setEditingProductId] = useState(null);
  // State for edit form data
  const [editFormData, setEditFormData] = useState({ name: '', price: 0 });

  // Fetch products when component mounts
  useEffect(() => {
    fetchProducts();
  }, []);

  // GET: Retrieve products
  const fetchProducts = () => {
    axios.get('/api/products')
      .then(response => setProducts(response.data))
      .catch(error => console.error('Error fetching products:', error));
  };

  // POST: Create a new product
  const handleCreateProduct = (e) => {
    e.preventDefault();
    axios.post('/api/products', newProduct)
      .then(() => {
        setNewProduct({ name: '', price: 0 });
        fetchProducts();
      })
      .catch(error => console.error('Error creating product:', error));
  };

  // DELETE: Remove a product
  const handleDeleteProduct = (id) => {
    axios.delete(`/api/products/${id}`)
      .then(() => fetchProducts())
      .catch(error => console.error('Error deleting product:', error));
  };

  // Set up edit form with product data
  const handleEditClick = (product) => {
    setEditingProductId(product.id);
    setEditFormData({ name: product.name, price: product.price });
  };

  // PUT: Update a product
  const handleUpdateProduct = (e, id) => {
    e.preventDefault();
    axios.put(`/api/products/${id}`, { id, ...editFormData })
      .then(() => {
        setEditingProductId(null);
        fetchProducts();
      })
      .catch(error => console.error('Error updating product:', error));
  };

  return (
    <div style={{ padding: '20px', fontFamily: 'Arial, sans-serif' }}>
      <h1>Product Manager</h1>

      {/* Create New Product */}
      <h2>Add New Product</h2>
      <form onSubmit={handleCreateProduct}>
        <input
          type="text"
          placeholder="Name"
          value={newProduct.name}
          onChange={(e) => setNewProduct({ ...newProduct, name: e.target.value })}
          required
          style={{ marginRight: '10px' }}
        />
        <input
          type="number"
          placeholder="Price"
          value={newProduct.price}
          onChange={(e) => setNewProduct({ ...newProduct, price: parseFloat(e.target.value) })}
          required
          style={{ marginRight: '10px' }}
        />
        <button type="submit">Add Product</button>
      </form>

      {/* Display Products List */}
      <h2>Product List</h2>
      {products.length === 0 ? (
        <p>No products found.</p>
      ) : (
        <ul>
          {products.map(product => (
            <li key={product.id} style={{ marginBottom: '10px' }}>
              {editingProductId === product.id ? (
                // Edit Form
                <form onSubmit={(e) => handleUpdateProduct(e, product.id)}>
                  <input
                    type="text"
                    value={editFormData.name}
                    onChange={(e) => setEditFormData({ ...editFormData, name: e.target.value })}
                    required
                    style={{ marginRight: '10px' }}
                  />
                  <input
                    type="number"
                    value={editFormData.price}
                    onChange={(e) => setEditFormData({ ...editFormData, price: parseFloat(e.target.value) })}
                    required
                    style={{ marginRight: '10px' }}
                  />
                  <button type="submit" style={{ marginRight: '5px' }}>Save</button>
                  <button type="button" onClick={() => setEditingProductId(null)}>Cancel</button>
                </form>
              ) : (
                // Display Product Details
                <>
                  <strong>{product.name}</strong> - ${product.price.toFixed(2)}{' '}
                  <button onClick={() => handleEditClick(product)} style={{ marginLeft: '10px' }}>Edit</button>
                  <button onClick={() => handleDeleteProduct(product.id)} style={{ marginLeft: '5px' }}>Delete</button>
                </>
              )}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

export default App;
```

#### File: `src/App.css` (Optional Styling)
```css
.App {
  text-align: center;
}
```

#### File: `src/index.css`
```css
body {
  margin: 0;
  font-family: Arial, Helvetica, sans-serif;
}
```

### 5. Run the React Client
- Open a terminal in the **client** folder and run:
```bash
npm start
```
This will launch the React development server. With the proxy set in **package.json**, API calls (e.g., `/api/products`) will be forwarded to your ASP.NET Core Web API.

---

## FINAL WORKFLOW

1. **Run the Server:**  
   - Open your ASP.NET Core project in Visual Studio and run it (F5).  
   - Verify your API (e.g., `http://localhost:5191/api/products`) or check Swagger at `https://localhost:<port>/swagger/index.html`.

2. **Run the Client:**  
   - Navigate to the **client** folder and run `npm start` to launch the React client.

3. **Test & Develop:**  
   - Use the React app to add, edit, or delete products.  
   - The React client communicates with your API using axios through the configured proxy.

---

This guide includes all instructions and code to set up a similar full‑stack application. Happy coding!
