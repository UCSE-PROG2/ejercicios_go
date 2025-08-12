
# CRUD de Aviones - API REST en Go

## Objetivo
Crear una API REST en Go para gestionar aviones con operaciones CRUD completas.

## Entidad Avion
- **id**: identificador único
- **nombre**: nombre del avión
- **modelo**: modelo del avión
- **cantidadPasajeros**: capacidad de pasajeros

## Endpoints
- **POST /aviones**: Crear avión
- **GET /aviones**: Buscar aviones (con filtros)
- **GET /aviones/{id}**: Obtener avión por ID
- **PUT /aviones/{id}**: Modificar avión
- **DELETE /aviones/{id}**: Eliminar avión

## Arquitectura
```
├── main.go          # Rutas
├── handlers/        # Funciones que atienden rutas
├── services/        # Lógica de negocio
├── dto/            # Structs de request/response
├── utils/          # Funciones generales
```

## Paso a Paso

### 1. Inicializar el proyecto
```bash
mkdir crud_aviones
cd crud_aviones
go mod init crud_aviones
```

### 2. Instalar dependencias
```bash
go get github.com/gin-gonic/gin
go get github.com/google/uuid
```

### 3. Crear estructura de directorios
```bash
mkdir handlers services dto utils
```

### 4. Crear archivo dto/avion.go
```go
package dto

type AvionRequest struct {
    Nombre             string `json:"nombre" binding:"required"`
    Modelo             string `json:"modelo" binding:"required"`
    CantidadPasajeros  int    `json:"cantidad_pasajeros" binding:"required,min=1"`
}

type AvionResponse struct {
    ID                 string `json:"id"`
    Nombre             string `json:"nombre"`
    Modelo             string `json:"modelo"`
    CantidadPasajeros  int    `json:"cantidad_pasajeros"`
}

type SearchRequest struct {
    Nombre             string `form:"nombre"`
    Modelo             string `form:"modelo"`
    MinPasajeros       int    `form:"min_pasajeros"`
}
```

### 5. Crear archivo utils/avion_utils.go
```go
package utils

import (
    "strings"
    "crud_aviones/dto"
)

func ConvertToResponse(avion dto.AvionRequest, id string) dto.AvionResponse {
    return dto.AvionResponse{
        ID:                id,
        Nombre:            avion.Nombre,
        Modelo:            avion.Modelo,
        CantidadPasajeros: avion.CantidadPasajeros,
    }
}

func MatchesSearch(avion dto.AvionResponse, search dto.SearchRequest) bool {
    if search.Nombre != "" && !strings.Contains(strings.ToLower(avion.Nombre), strings.ToLower(search.Nombre)) {
        return false
    }
    
    if search.Modelo != "" && !strings.Contains(strings.ToLower(avion.Modelo), strings.ToLower(search.Modelo)) {
        return false
    }
    
    if search.MinPasajeros > 0 && avion.CantidadPasajeros < search.MinPasajeros {
        return false
    }
    
    return true
}
```

### 6. Crear archivo services/avion_service.go
```go
package services

import (
    "errors"
    "crud_aviones/dto"
    "crud_aviones/utils"
)

type Avion struct {
    ID     string
    Avion  dto.AvionRequest
}

var aviones []Avion

func CreateAvion(avion dto.AvionRequest) (dto.AvionResponse, error) {
    id := generateID()
    nuevoAvion := Avion{
        ID:    id,
        Avion: avion,
    }
    aviones = append(aviones, nuevoAvion)
    return utils.ConvertToResponse(avion, id), nil
}

func GetAvionByID(id string) (dto.AvionResponse, error) {
    for _, avion := range aviones {
        if avion.ID == id {
            return utils.ConvertToResponse(avion.Avion, id), nil
        }
    }
    return dto.AvionResponse{}, errors.New("avión no encontrado")
}

func UpdateAvion(id string, avion dto.AvionRequest) (dto.AvionResponse, error) {
    for i, avionItem := range aviones {
        if avionItem.ID == id {
            aviones[i].Avion = avion
            return utils.ConvertToResponse(avion, id), nil
        }
    }
    return dto.AvionResponse{}, errors.New("avión no encontrado")
}

func DeleteAvion(id string) error {
    for i, avion := range aviones {
        if avion.ID == id {
            aviones = append(aviones[:i], aviones[i+1:]...)
            return nil
        }
    }
    return errors.New("avión no encontrado")
}

func SearchAviones(search dto.SearchRequest) []dto.AvionResponse {
    var results []dto.AvionResponse
    
    for _, avion := range aviones {
        response := utils.ConvertToResponse(avion.Avion, avion.ID)
        if utils.MatchesSearch(response, search) {
            results = append(results, response)
        }
    }
    
    return results
}

func generateID() string {
    return "avion_" + string(rune(len(aviones)+1))
}
```

### 7. Crear archivo handlers/avion_handler.go
```go
package handlers

import (
    "net/http"
    "strconv"
    "crud_aviones/dto"
    "crud_aviones/services"
    "github.com/gin-gonic/gin"
)

func CreateAvion(c *gin.Context) {
    var request dto.AvionRequest
    if err := c.ShouldBindJSON(&request); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    avion, err := services.CreateAvion(request)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, avion)
}

func GetAvionByID(c *gin.Context) {
    id := c.Param("id")
    
    avion, err := services.GetAvionByID(id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, avion)
}

func UpdateAvion(c *gin.Context) {
    id := c.Param("id")
    
    var request dto.AvionRequest
    if err := c.ShouldBindJSON(&request); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    avion, err := services.UpdateAvion(id, request)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, avion)
}

func DeleteAvion(c *gin.Context) {
    id := c.Param("id")
    
    err := services.DeleteAvion(id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"message": "Avión eliminado correctamente"})
}

func SearchAviones(c *gin.Context) {
    var search dto.SearchRequest
    
    search.Nombre = c.Query("nombre")
    search.Modelo = c.Query("modelo")
    
    if minPasajerosStr := c.Query("min_pasajeros"); minPasajerosStr != "" {
        if minPasajeros, err := strconv.Atoi(minPasajerosStr); err == nil {
            search.MinPasajeros = minPasajeros
        }
    }
    
    aviones := services.SearchAviones(search)
    c.JSON(http.StatusOK, aviones)
}
```

### 8. Crear archivo main.go
```go
package main

import (
    "crud_aviones/handlers"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    // Rutas para aviones
    aviones := r.Group("/aviones")
    {
        aviones.POST("", handlers.CreateAvion)
        aviones.GET("", handlers.SearchAviones)
        aviones.GET("/:id", handlers.GetAvionByID)
        aviones.PUT("/:id", handlers.UpdateAvion)
        aviones.DELETE("/:id", handlers.DeleteAvion)
    }
    
    r.Run(":8080")
}
```

### 9. Ejecutar la aplicación
```bash
go run main.go
```

### 10. Probar la funcionalidad

#### Crear aviones:
```bash
curl -X POST http://localhost:8080/aviones \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Boeing 737","modelo":"737-800","cantidad_pasajeros":189}'

curl -X POST http://localhost:8080/aviones \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Airbus A320","modelo":"A320neo","cantidad_pasajeros":180}'
```

#### Buscar todos los aviones:
```bash
curl http://localhost:8080/aviones
```

#### Buscar con filtros:
```bash
curl "http://localhost:8080/aviones?nombre=boeing&min_pasajeros=150"
```

#### Obtener avión por ID:
```bash
curl http://localhost:8080/aviones/avion_1
```

#### Modificar avión:
```bash
curl -X PUT http://localhost:8080/aviones/avion_1 \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Boeing 737 MAX","modelo":"737-8","cantidad_pasajeros":200}'
```

#### Eliminar avión:
```bash
curl -X DELETE http://localhost:8080/aviones/avion_1
```

## Resultado
Al seguir estos pasos tendrás una API REST completa para gestionar aviones con todas las operaciones CRUD implementadas siguiendo el patrón de arquitectura especificado.
