## Title: 干净简洁的实现Go项目结构

### 1. 引言
今天，我要分享一个在 GitHub 上获得了 4.8k 星的 Go 项目：go-backend-clean-arch。

这是一个采用Gin框架、使用MongoDB、具备JWT认证中间件、包含测试以及支持Docker的Go（Golang）后端整洁架构项目。该项目通过一个HTTP示例展示了一种优雅的项目结构。

### 2. 项目架构
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/go_backend_architecture1.jpg)

3. 目录说明
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/go_backend_architecture2.jpg)

#### 3.1 参数配置与项目启动
`./cmd/main.go` 文件内容如下：

```go
type Applicationstruct{
   Env*Env
   Mongo mongo.Client
}

typeEnvstruct{
   AppEnvstring`mapstructure:"APP_ENV"`
   ServerAddressstring`mapstructure:"SERVER_ADDRESS"`
   ContextTimeoutint`mapstructure:"CONTEXT_TIMEOUT"`
   DBHoststring`mapstructure:"DB_HOST"`
   DBPortstring`mapstructure:"DB_PORT"`
   ...
}

func main(){
// app is the instance of the entire application, managing key resources throughout its lifecycle
    app := bootstrap.App()

// Configuration variables
    env := app.Env

// Database instance
    db := app.Mongo.Database(env.DBName)
defer app.CloseDBConnection()

    timeout := time.Duration(env.ContextTimeout)* time.Second

// Creating a gin instance
    gin := gin.Default()

// Route binding
    route.Setup(env, timeout, db, gin)

// Running the server
    gin.Run(env.ServerAddress)
}
```

以下将以登录逻辑为例来阐释这个三层架构。

#### 3.2 接口层

`./api/controller/login_controller.go` 文件内容如下：

`./api/controller/login_controller.go`

LoginController 结构体持有配置类以及 LoginUsecase 接口（该接口定义了业务层的行为）。

```go
// Business layer interface


typeSignupUsecaseinterface{
   Create(c context.Context, user *User)error
   GetUserByEmail(c context.Context, email string)(User,error)
   CreateAccessToken(user *User, secret string, expiry int)(accessToken string, err error)
   CreateRefreshToken(user *User, secret string, expiry int)(refreshToken string, err error)
}

typeLoginControllerstruct{
   LoginUsecase domain.LoginUsecase
   Env*bootstrap.Env
}

func (lc *LoginController)Login(c *gin.Context){
   var request domain.LoginRequest

   err := c.ShouldBind(&request)
   if err !=nil{
      c.JSON(http.StatusBadRequest, domain.ErrorResponse{Message: err.Error()})
      return
   }

   user, err := lc.LoginUsecase.GetUserByEmail(c, request.Email)
   if err !=nil{
      c.JSON(http.StatusNotFound, domain.ErrorResponse{Message:"User not found with the given email"})
      return
   }

   if bcrypt.CompareHashAndPassword([]byte(user.Password),[]byte(request.Password))!=nil{
      c.JSON(http.StatusUnauthorized, domain.ErrorResponse{Message:"Invalid credentials"})
      return
   }

   accessToken, err := lc.LoginUsecase.CreateAccessToken(&user, lc.Env.AccessTokenSecret, lc.Env.AccessTokenExpiryHour)
   if err !=nil {
      c.JSON(http.StatusInternalServerError, domain.ErrorResponse{Message: err.Error()})
      return
   }

   refreshToken, err := lc.LoginUsecase.CreateRefreshToken(&user, lc.Env.RefreshTokenSecret, lc.Env.RefreshTokenExpiryHour)
   if err !=nil {
      c.JSON(http.StatusInternalServerError, domain.ErrorResponse{Message: err.Error()})
      return
   }

   loginResponse := domain.LoginResponse{
      AccessToken:  accessToken,
      RefreshToken: refreshToken,
   }

   c.JSON(http.StatusOK, loginResponse)
}
```

#### 3.3 业务层

`./usecase/login_usecase.go` 

loginUsecase 结构体实现了 LoginUsecase 接口。

```go
// Data anti-corruption layer interface
typeUserRepositoryinterface{
   Create(c context.Context, user *User)error
   Fetch(c context.Context)([]User,error)
   GetByEmail(c context.Context, email string)(User,error)
   GetByID(c context.Context, id string)(User,error)
}

type loginUsecase struct{
   userRepository domain.UserRepository
   contextTimeout time.Duration
}

func NewLoginUsecase(userRepository domain.UserRepository, timeout time.Duration) domain.LoginUsecase{
   return&loginUsecase{
         userRepository: userRepository,
         contextTimeout: timeout,
   }
}

func (lu *loginUsecase)GetUserByEmail(c context.Context, email string)(domain.User,error){
   ctx, cancel := context.WithTimeout(c, lu.contextTimeout)
   defer cancel()
   return lu.userRepository.GetByEmail(ctx, email)
}

func (lu *loginUsecase)CreateAccessToken(user *domain.User, secret string, expiry int)(accessToken string, err error){
   return tokenutil.CreateAccessToken(user, secret, expiry)
}

func (lu *loginUsecase)CreateRefreshToken(user *domain.User, secret string, expiry int)(refreshToken string, err error){
   return tokenutil.CreateRefreshToken(user, secret, expiry)
}
```

#### 3.4 Anti-Corruption 层

`./repository/user_repository.go` 

userRepository 结构体实现了 UserRepository 接口。它内部持有 mongo.Database 接口（该接口定义了数据层的行为）以及集合实例的名称。

```go
// Data operation layer interface

typeDatabaseinterface{
   Collection(string)Collection
   Client()Client
}

type userRepository struct{
   database   mongo.Database
   collection string
}

func NewUserRepository(db mongo.Database, collection string) domain.UserRepository{
   return&userRepository{
         database:   db,
         collection: collection,
   }
}

func (ur *userRepository)Create(c context.Context, user *domain.User)error{
   collection := ur.database.Collection(ur.collection)

   _, err := collection.InsertOne(c, user)

   return err
}

func (ur *userRepository)Fetch(c context.Context)([]domain.User,error){
   collection := ur.database.Collection(ur.collection)

   opts := options.Find().SetProjection(bson.D{{Key:"password",Value:0}})
   cursor, err := collection.Find(c, bson.D{}, opts)

   if err !=nil{
      returnnil, err
   }

   var users []domain.User

   err = cursor.All(c,&users)
   if users ==nil{
      return[]domain.User{}, err
   }

   return users, err
}

func (ur *userRepository)GetByEmail(c context.Context, email string)(domain.User,error){
   collection := ur.database.Collection(ur.collection)
   var user domain.User
   err := collection.FindOne(c, bson.M{"email": email}).Decode(&user)
   return user, err
}

func (ur *userRepository)GetByID(c context.Context, id string)(domain.User,error){
   collection := ur.database.Collection(ur.collection)

   var user domain.User

   idHex, err := primitive.ObjectIDFromHex(id)
   if err !=nil{
      return user, err
   }

   err = collection.FindOne(c, bson.M{"_id": idHex}).Decode(&user)
   return user, err
}
```

#### 3.5 数据层
`./mongo/mongo.go` 实现了 mongo.Database 接口。
通过 mongoDatabase 结构体的两个方法，可以获取相应的 Client 实例和 Collection 实例来操作数据库。

```go
type mongoDatabase struct{
   db *mongo.Database
}

func (md *mongoDatabase)Collection(colName string)Collection{
   collection := md.db.Collection(colName)
   return&mongoCollection{coll: collection}
}

func (md *mongoDatabase)Client()Client{
   client := md.db.Client()
   return&mongoClient{cl: client}
}
```

### 4. 单例与封装
查看 `./cmd/main.go` 中的路由绑定逻辑：`route.Setup(env, timeout, db, gin)`。

```go
func Setup(env *bootstrap.Env, timeout time.Duration, db mongo.Database, gin *gin.Engine) {
   publicRouter := gin.Group("")
   // All Public APIs
   NewSignupRouter(env, timeout, db, publicRouter)
   NewLoginRouter(env, timeout, db, publicRouter)
   NewRefreshTokenRouter(env, timeout, db, publicRouter)

   protectedRouter := gin.Group("")
   // Middleware to verify AccessToken
   protectedRouter.Use(middleware.JwtAuthMiddleware(env.AccessTokenSecret))
   // All Private APIs
   NewProfileRouter(env, timeout, db, protectedRouter)
   NewTaskRouter(env, timeout, db, protectedRouter)
}
```

进一步查看 `NewLoginRouter` 函数会发现，在注册由路由触发的控制器方法时，所需的 db（数据库实例）已经创建好，并在数据层内共享。

此外，Anti-Corruption 层、业务层和控制器层的实例在服务启动前就已创建，它们依次嵌套并持有相关实例。

```go
func NewLoginRouter(env *bootstrap.Env, timeout time.Duration, db mongo.Database, group *gin.RouterGroup) {
   ur := repository.NewUserRepository(db, domain.CollectionUser)
   lc := &controller.LoginController{
      LoginUsecase: usecase.NewLoginUsecase(ur, timeout),
      Env:          env,
   }
   group.POST("/login", lc.Login)
}
```

因此，所有的结构体都是单例，并且类似树形结构，按顺序相互关联。

这种方式对资源进行了约束，防止开发者跨模块调用实例，否则可能会导致诸如循环依赖以及其他安全问题的出现。


来源：
https://mp.weixin.qq.com/s/OgznZDG4jypdFOzekj8Rkg


