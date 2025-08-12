# CRUD de Productos - API REST en Go

## Objetivo
Crear una API REST en Go para gestionar productos con operaciones CRUD completas.

## Entidad Producto
- **id**: identificador único
- **nombre**: nombre del producto
- **descripcion**: descripción del producto
- **precio**: precio del producto
- **categoria**: categoría del producto (estructura anidada)

## Entidad Categoria
- **id**: identificador único de la categoría
- **nombre**: nombre de la categoría
- **descripcion**: descripción de la categoría

## Endpoints
- **POST /productos**: Crear producto
- **GET /productos**: Buscar productos (con filtros)
- **GET /productos/{id}**: Obtener producto por ID
- **PUT /productos/{id}**: Modificar producto
- **DELETE /productos/{id}**: Eliminar producto

## Arquitectura
```
├── main.go          # Rutas
├── handlers/        # Funciones que atienden rutas
├── services/        # Lógica de negocio
├── dto/            # Structs de request/response
├── utils/          # Funciones generales
```

## Estructuras Anidadas (DTOs)

### Definición de structs anidados
```go
type Categoria struct {
    ID          string `json:"id"`
    Nombre      string `json:"nombre"`
    Descripcion string `json:"descripcion"`
}

type ProductoRequest struct {
    Nombre      string     `json:"nombre" binding:"required"`
    Descripcion string     `json:"descripcion"`
    Precio      float64    `json:"precio" binding:"required,min=0"`
    Categoria   Categoria  `json:"categoria" binding:"required"`
}
```

### Instanciación y asignación de valores
```go
// Crear una categoría
categoria := Categoria{
    ID:          "cat_1",
    Nombre:      "Electrónicos",
    Descripcion: "Productos electrónicos y tecnología",
}

// Crear un producto con categoría
producto := ProductoRequest{
    Nombre:      "Smartphone",
    Descripcion: "Teléfono inteligente de última generación",
    Precio:      599.99,
    Categoria:   categoria,
}

// Asignar valores por separado
producto.Categoria.Nombre = "Nueva Categoría"
```

## Resultado
Al completar este ejercicio tendrás una API REST completa para gestionar productos con todas las operaciones CRUD implementadas, incluyendo el manejo de categorías como estructuras anidadas.
