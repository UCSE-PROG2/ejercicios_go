
# CRUD de Aviones - API REST en Go (MongoDB)

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
├── main.go          # Rutas y configuración
├── handlers/        # Funciones que atienden rutas
├── services/        # Lógica de negocio
├── repositories/    # Acceso a base de datos
├── database/        # Configuración de conexión
├── model/           # Modelos de datos
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
go get go.mongodb.org/mongo-driver/mongo
go get go.mongodb.org/mongo-driver/bson
```

### 3. Crear estructura de directorios
```bash
mkdir handlers services repositories database model dto utils
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

### 5. Crear archivo model/avion.go
```go
package model

import "go.mongodb.org/mongo-driver/bson/primitive"

type Avion struct {
    ID                 primitive.ObjectID `bson:"_id,omitempty"`
    Nombre             string             `bson:"nombre"`
    Modelo             string             `bson:"modelo"`
    CantidadPasajeros  int                `bson:"cantidad_pasajeros"`
}
```

### 6. Crear archivo database/db.go
```go
package database

import (
    "go.mongodb.org/mongo-driver/mongo"
)

type DB interface {
    Connect() error
    Disconnect() error
    GetClient() *mongo.Client
}
```

### 7. Crear archivo database/mongo.go
```go
package database

import (
    "context"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type MongoDB struct {
    Client *mongo.Client
}

func NewMongoDB() *MongoDB {
    instancia := &MongoDB{}
    instancia.Connect()
    return instancia
}

func (mongoDB *MongoDB) GetClient() *mongo.Client {
    return mongoDB.Client
}

func (mongoDB *MongoDB) Connect() error {
    clientOptions := options.Client().ApplyURI("mongodb://localhost:27017")
    
    client, err := mongo.Connect(context.Background(), clientOptions)
    if err != nil {
        return err
    }
    
    err = client.Ping(context.Background(), nil)
    if err != nil {
        return err
    }
    
    mongoDB.Client = client
    return nil
}

func (mongoDB *MongoDB) Disconnect() error {
    return mongoDB.Client.Disconnect(context.Background())
}
```

### 8. Crear archivo repositories/avion_repository.go
```go
package repositories

import (
    "context"
    "crud_aviones/model"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
)

type AvionRepositoryInterface interface {
    ObtenerAviones(nombre string) ([]model.Avion, error)
    ObtenerAvionPorID(id string) (model.Avion, error)
    InsertarAvion(avion model.Avion) (*mongo.InsertOneResult, error)
    ModificarAvion(avion model.Avion) (*mongo.UpdateResult, error)
    EliminarAvion(id primitive.ObjectID) (*mongo.DeleteResult, error)
}

type AvionRepository struct {
    db DB
}

func NewAvionRepository(db DB) *AvionRepository {
    return &AvionRepository{
        db: db,
    }
}

func (repository AvionRepository) ObtenerAviones(nombre string) ([]model.Avion, error) {
    collection := repository.db.GetClient().Database("crud_aviones").Collection("aviones")
    
    var filtro bson.M
    if nombre != "" {
        // Búsqueda por aproximación usando regex (case insensitive)
        filtro = bson.M{"nombre": bson.M{"$regex": nombre, "$options": "i"}}
    } else {
        filtro = bson.M{}
    }
    
    cursor, err := collection.Find(context.TODO(), filtro)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(context.Background())
    
    var aviones []model.Avion
    for cursor.Next(context.Background()) {
        var avion model.Avion
        err := cursor.Decode(&avion)
        if err != nil {
            continue
        }
        aviones = append(aviones, avion)
    }
    
    return aviones, nil
}

func (repository AvionRepository) ObtenerAvionPorID(id string) (model.Avion, error) {
    collection := repository.db.GetClient().Database("crud_aviones").Collection("aviones")
    objectID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return model.Avion{}, err
    }
    
    filtro := bson.M{"_id": objectID}
    var avion model.Avion
    
    err = collection.FindOne(context.TODO(), filtro).Decode(&avion)
    return avion, err
}

func (repository AvionRepository) InsertarAvion(avion model.Avion) (*mongo.InsertOneResult, error) {
    collection := repository.db.GetClient().Database("crud_aviones").Collection("aviones")
    resultado, err := collection.InsertOne(context.TODO(), avion)
    return resultado, err
}

func (repository AvionRepository) ModificarAvion(avion model.Avion) (*mongo.UpdateResult, error) {
    collection := repository.db.GetClient().Database("crud_aviones").Collection("aviones")
    
    filtro := bson.M{"_id": avion.ID}
    actualizacion := bson.M{"$set": bson.M{
        "nombre":              avion.Nombre,
        "modelo":              avion.Modelo,
        "cantidad_pasajeros":  avion.CantidadPasajeros,
    }}
    
    resultado, err := collection.UpdateOne(context.TODO(), filtro, actualizacion)
    return resultado, err
}

func (repository AvionRepository) EliminarAvion(id primitive.ObjectID) (*mongo.DeleteResult, error) {
    collection := repository.db.GetClient().Database("crud_aviones").Collection("aviones")
    
    filtro := bson.M{"_id": id}
    resultado, err := collection.DeleteOne(context.TODO(), filtro)
    return resultado, err
}
```


### 9. Crear archivo utils/avion_utils.go
```go
package utils

import (
    "strings"
    "crud_aviones/dto"
)

import (
    "strings"
    "crud_aviones/dto"
    "crud_aviones/model"
)

func ConvertModelToResponse(avion model.Avion) dto.AvionResponse {
    return dto.AvionResponse{
        ID:                avion.ID.Hex(),
        Nombre:            avion.Nombre,
        Modelo:            avion.Modelo,
        CantidadPasajeros: avion.CantidadPasajeros,
    }
}

func ConvertRequestToModel(avion dto.AvionRequest) model.Avion {
    return model.Avion{
        Nombre:            avion.Nombre,
        Modelo:            avion.Modelo,
        CantidadPasajeros: avion.CantidadPasajeros,
    }
}

func MatchesSearch(avion model.Avion, search dto.SearchRequest) bool {
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

### 10. Crear archivo services/avion_service.go
```go
package services

import (
    "errors"
    "crud_aviones/dto"
    "crud_aviones/model"
    "crud_aviones/repositories"
    "crud_aviones/utils"
    "go.mongodb.org/mongo-driver/bson/primitive"
)

type AvionInterface interface {
    CreateAvion(avion dto.AvionRequest) (dto.AvionResponse, error)
    GetAvionByID(id string) (dto.AvionResponse, error)
    UpdateAvion(id string, avion dto.AvionRequest) (dto.AvionResponse, error)
    DeleteAvion(id string) error
    SearchAviones(search dto.SearchRequest) ([]dto.AvionResponse, error)
}

type AvionService struct {
    repository repositories.AvionRepositoryInterface
}

func NewAvionService(repository repositories.AvionRepositoryInterface) *AvionService {
    return &AvionService{
        repository: repository,
    }
}

func (service *AvionService) CreateAvion(avion dto.AvionRequest) (dto.AvionResponse, error) {
    modelAvion := utils.ConvertRequestToModel(avion)
    
    result, err := service.repository.InsertarAvion(modelAvion)
    if err != nil {
        return dto.AvionResponse{}, err
    }
    
    // Obtener el avión insertado para devolver la respuesta completa
    if oid, ok := result.InsertedID.(primitive.ObjectID); ok {
        modelAvion.ID = oid
        return utils.ConvertModelToResponse(modelAvion), nil
    }
    
    return dto.AvionResponse{}, errors.New("error al obtener ID del avión insertado")
}

func (service *AvionService) GetAvionByID(id string) (dto.AvionResponse, error) {
    avion, err := service.repository.ObtenerAvionPorID(id)
    if err != nil {
        return dto.AvionResponse{}, errors.New("avión no encontrado")
    }
    
    return utils.ConvertModelToResponse(avion), nil
}

func (service *AvionService) UpdateAvion(id string, avion dto.AvionRequest) (dto.AvionResponse, error) {
    objectID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return dto.AvionResponse{}, errors.New("ID inválido")
    }
    
    modelAvion := utils.ConvertRequestToModel(avion)
    modelAvion.ID = objectID
    
    _, err = service.repository.ModificarAvion(modelAvion)
    if err != nil {
        return dto.AvionResponse{}, err
    }
    
    return utils.ConvertModelToResponse(modelAvion), nil
}

func (service *AvionService) DeleteAvion(id string) error {
    objectID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return errors.New("ID inválido")
    }
    
    _, err = service.repository.EliminarAvion(objectID)
    return err
}

func (service *AvionService) SearchAviones(search dto.SearchRequest) ([]dto.AvionResponse, error) {
    aviones, err := service.repository.ObtenerAviones(search.Nombre)
    if err != nil {
        return nil, err
    }
    
    var results []dto.AvionResponse
    for _, avion := range aviones {
        if utils.MatchesSearch(avion, search) {
            results = append(results, utils.ConvertModelToResponse(avion))
        }
    }
    
    return results, nil
}
```

### 11. Crear archivo handlers/avion_handler.go
```go
package handlers

import (
    "net/http"
    "strconv"
    "crud_aviones/dto"
    "crud_aviones/services"
    "github.com/gin-gonic/gin"
)

type AvionHandler struct {
    service services.AvionInterface
}

func NewAvionHandler(service services.AvionInterface) *AvionHandler {
    return &AvionHandler{
        service: service,
    }
}

func (handler *AvionHandler) CreateAvion(c *gin.Context) {
    var request dto.AvionRequest
    if err := c.ShouldBindJSON(&request); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    avion, err := handler.service.CreateAvion(request)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, avion)
}

func (handler *AvionHandler) GetAvionByID(c *gin.Context) {
    id := c.Param("id")
    
    avion, err := handler.service.GetAvionByID(id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, avion)
}

func (handler *AvionHandler) UpdateAvion(c *gin.Context) {
    id := c.Param("id")
    
    var request dto.AvionRequest
    if err := c.ShouldBindJSON(&request); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    avion, err := handler.service.UpdateAvion(id, request)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, avion)
}

func (handler *AvionHandler) DeleteAvion(c *gin.Context) {
    id := c.Param("id")
    
    err := handler.service.DeleteAvion(id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"message": "Avión eliminado correctamente"})
}

func (handler *AvionHandler) SearchAviones(c *gin.Context) {
    var search dto.SearchRequest
    
    search.Nombre = c.Query("nombre")
    search.Modelo = c.Query("modelo")
    
    if minPasajerosStr := c.Query("min_pasajeros"); minPasajerosStr != "" {
        if minPasajeros, err := strconv.Atoi(minPasajerosStr); err == nil {
            search.MinPasajeros = minPasajeros
        }
    }
    
    aviones, err := handler.service.SearchAviones(search)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, aviones)
}
```

### 12. Crear archivo main.go
```go
package main

import (
    "log"
    "crud_aviones/database"
    "crud_aviones/handlers"
    "crud_aviones/repositories"
    "crud_aviones/services"
    "github.com/gin-gonic/gin"
)

var (
    avionHandler *handlers.AvionHandler
    router       *gin.Engine
)

func main() {
    router = gin.Default()
    
    // Inicializar dependencias
    dependencies()
    
    // Configurar rutas
    mappingRoutes()
    
    log.Println("Iniciando el servidor...")
    router.Run(":8080")
}

func mappingRoutes() {
    // Rutas para aviones
    aviones := router.Group("/aviones")
    {
        aviones.POST("", avionHandler.CreateAvion)
        aviones.GET("", avionHandler.SearchAviones)
        aviones.GET("/:id", avionHandler.GetAvionByID)
        aviones.PUT("/:id", avionHandler.UpdateAvion)
        aviones.DELETE("/:id", avionHandler.DeleteAvion)
    }
}

// Generación de los objetos que se van a usar en la API
func dependencies() {
    // Definición de variables de interface
    var databaseInstance database.DB
    var avionRepository repositories.AvionRepositoryInterface
    var avionService services.AvionInterface
    
    // Creamos los objetos reales y los pasamos como parámetro
    databaseInstance = database.NewMongoDB()
    avionRepository = repositories.NewAvionRepository(databaseInstance)
    avionService = services.NewAvionService(avionRepository)
    avionHandler = handlers.NewAvionHandler(avionService)
}
```

### 13. Configurar MongoDB
Antes de ejecutar la aplicación, asegúrate de tener MongoDB instalado y ejecutándose en tu sistema:

```bash
# En macOS con Homebrew
brew install mongodb-community
brew services start mongodb-community

# En Ubuntu/Debian
sudo apt-get install mongodb
sudo systemctl start mongodb

# En Windows, descarga e instala desde mongodb.com
```

### 14. Ejecutar la aplicación
```bash
go run main.go
```

### 15. Probar la funcionalidad

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
Al seguir estos pasos tendrás una API REST completa para gestionar aviones con todas las operaciones CRUD implementadas siguiendo el patrón de arquitectura especificado, incluyendo:

- **Persistencia de datos**: Almacenamiento en MongoDB
- **Arquitectura en capas**: Separación clara entre handlers, services, repositories y database
- **Inyección de dependencias**: Uso de interfaces para desacoplar componentes
- **Patrón Repository**: Abstracción del acceso a datos
- **Manejo de errores**: Gestión adecuada de errores en cada capa

