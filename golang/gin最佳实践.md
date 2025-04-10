## Gin 框架的最佳实践

**案例：一个简单的 "待办事项 (Todo)" API**

这个案例将涵盖以下方面：

1.  **项目结构 (Project Structure)**：清晰的分层和模块化。
2.  **配置管理 (Configuration Management)**：使用 `viper` 加载配置。
3.  **路由组织 (Routing)**：分组和模块化路由。
4.  **分层设计 (Layered Architecture)**：Handler (Controller), Service, Repository (Data Access)。
5.  **依赖注入 (Dependency Injection)**：手动注入或使用简单容器概念。
6.  **中间件 (Middleware)**：日志、错误处理、恢复。
7.  **模型定义 (Models)**：数据结构定义。
8.  **错误处理 (Error Handling)**：统一的错误响应。
9.  **请求验证 (Request Validation)**：使用Gin内置的绑定和验证。
10. **结构化日志 (Structured Logging)**：使用标准库 `log/slog` (Go 1.21+)。
11. **优雅关停 (Graceful Shutdown)**：处理系统信号，确保服务平稳停止。

---
**1. 项目结构**

```
todo-api/
├── cmd/
│   └── server/
│       └── main.go           # 程序入口
├── config/
│   ├── config.go           # 配置加载逻辑
│   └── config.yaml         # 配置文件 (示例)
├── internal/
│   ├── api/
│   │   ├── handler/          # HTTP 请求处理器 (Controllers)
│   │   │   ├── todo_handler.go
│   │   │   └── routes.go       # 路由注册
│   │   ├── middleware/       # 中间件
│   │   │   ├── error_handler.go
│   │   │   └── logger.go
│   │   └── request/          # 请求体结构定义与验证
│   │       └── todo_request.go
│   ├── domain/               # 核心领域模型 (Entities)
│   │   └── todo.go
│   ├── repository/           # 数据访问层接口与实现
│   │   ├── todo_repository.go # 接口定义
│   │   └── memory_repo.go    # 内存实现 (示例, 可替换为 DB 实现)
│   └── service/              # 业务逻辑层
│       └── todo_service.go
├── pkg/                    # 可共享的通用库 (如果需要)
│   └── httputil/           # HTTP 相关的工具 (例如: 统一响应)
│       └── error_response.go
├── go.mod
├── go.sum
└── .env                    # 环境变量文件 (可选, 用于覆盖配置)
```

**2. 配置文件 (`config/config.yaml`)**

```yaml
server:
  port: 8080
  mode: "debug" # Gin mode: debug, release, test

# 可以添加数据库配置等
# database:
#   dsn: "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
```

**3. 配置加载 (`config/config.go`)**

```go
package config

import (
	"log/slog"
	"strings"

	"github.com/spf13/viper"
)

type Config struct {
	Server ServerConfig `mapstructure:"server"`
	// Database DatabaseConfig `mapstructure:"database"`
}

type ServerConfig struct {
	Port int    `mapstructure:"port"`
	Mode string `mapstructure:"mode"`
}

// type DatabaseConfig struct {
// 	DSN string `mapstructure:"dsn"`
// }

// LoadConfig 从文件和环境变量加载配置
func LoadConfig(configPath string) (*Config, error) {
	// 默认值
	viper.SetDefault("server.port", 8080)
	viper.SetDefault("server.mode", "debug")

	// 从文件读取
	if configPath != "" {
		viper.SetConfigFile(configPath)
		if err := viper.ReadInConfig(); err != nil {
			slog.Warn("Could not read config file", "path", configPath, "error", err)
			// 文件不存在或解析错误可以接受，将使用默认值或环境变量
		}
	} else {
		viper.SetConfigName("config") // name of config file (without extension)
		viper.SetConfigType("yaml")   // REQUIRED if the config file does not have the extension in the name
		viper.AddConfigPath("./config") // path to look for the config file in
		viper.AddConfigPath(".")        // optionally look for config in the working directory
		if err := viper.ReadInConfig(); err != nil {
			slog.Warn("Could not find or read config file", "error", err)
		}
	}

	// 允许环境变量覆盖 (例如: SERVER_PORT=9090)
	viper.SetEnvPrefix("APP") // 例如: APP_SERVER_PORT
	viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
	viper.AutomaticEnv()

	var cfg Config
	if err := viper.Unmarshal(&cfg); err != nil {
		slog.Error("Unable to decode config into struct", "error", err)
		return nil, err
	}

	slog.Info("Configuration loaded successfully", "port", cfg.Server.Port, "mode", cfg.Server.Mode)
	return &cfg, nil
}
```

**4. 核心领域模型 (`internal/domain/todo.go`)**

```go
package domain

import "time"

type Todo struct {
	ID          int       `json:"id"`
	Title       string    `json:"title"`
	Description string    `json:"description"`
	Completed   bool      `json:"completed"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}
```

**5. 数据访问层 (`internal/repository/`)**

*   **接口 (`todo_repository.go`)**

    ```go
    package repository

    import (
    	"context"
    	"errors"

    	"todo-api/internal/domain"
    )

    var ErrTodoNotFound = errors.New("todo not found")

    // TodoRepository 定义了与待办事项数据存储交互的方法
    type TodoRepository interface {
    	GetAll(ctx context.Context) ([]domain.Todo, error)
    	GetByID(ctx context.Context, id int) (*domain.Todo, error)
    	Create(ctx context.Context, todo *domain.Todo) error
    	Update(ctx context.Context, todo *domain.Todo) error
    	Delete(ctx context.Context, id int) error
    }
    ```
*   **内存实现 (`memory_repo.go`)** (用于演示，实际应用替换为数据库实现)

    ```go
    package repository

    import (
    	"context"
    	"sync"
    	"time"
    	"todo-api/internal/domain"
    )

    type InMemoryTodoRepository struct {
    	mu      sync.RWMutex
    	todos   map[int]domain.Todo
    	nextID  int
    }

    func NewInMemoryTodoRepository() *InMemoryTodoRepository {
    	return &InMemoryTodoRepository{
    		todos:  make(map[int]domain.Todo),
    		nextID: 1,
    	}
    }

    func (r *InMemoryTodoRepository) GetAll(ctx context.Context) ([]domain.Todo, error) {
    	r.mu.RLock()
    	defer r.mu.RUnlock()
    	
        // 检查上下文是否已取消
        if err := ctx.Err(); err != nil {
             return nil, err
        }

    	list := make([]domain.Todo, 0, len(r.todos))
    	for _, todo := range r.todos {
    		list = append(list, todo)
    	}
    	return list, nil
    }

    func (r *InMemoryTodoRepository) GetByID(ctx context.Context, id int) (*domain.Todo, error) {
    	r.mu.RLock()
    	defer r.mu.RUnlock()

        if err := ctx.Err(); err != nil {
             return nil, err
        }

    	todo, exists := r.todos[id]
    	if !exists {
    		return nil, ErrTodoNotFound
    	}
    	return &todo, nil
    }

    func (r *InMemoryTodoRepository) Create(ctx context.Context, todo *domain.Todo) error {
    	r.mu.Lock()
    	defer r.mu.Unlock()

        if err := ctx.Err(); err != nil {
             return nil, err
        }

    	now := time.Now()
    	todo.ID = r.nextID
    	todo.CreatedAt = now
    	todo.UpdatedAt = now
    	r.todos[todo.ID] = *todo
    	r.nextID++
    	return nil
    }

    func (r *InMemoryTodoRepository) Update(ctx context.Context, todo *domain.Todo) error {
    	r.mu.Lock()
    	defer r.mu.Unlock()

        if err := ctx.Err(); err != nil {
             return nil, err
        }

    	_, exists := r.todos[todo.ID]
    	if !exists {
    		return ErrTodoNotFound
    	}
    	todo.UpdatedAt = time.Now()
    	// 确保 CreatedAt 不被覆盖
    	originalTodo := r.todos[todo.ID]
    	todo.CreatedAt = originalTodo.CreatedAt
    	r.todos[todo.ID] = *todo
    	return nil
    }

    func (r *InMemoryTodoRepository) Delete(ctx context.Context, id int) error {
    	r.mu.Lock()
    	defer r.mu.Unlock()

        if err := ctx.Err(); err != nil {
             return nil, err
        }

    	_, exists := r.todos[id]
    	if !exists {
    		return ErrTodoNotFound
    	}
    	delete(r.todos, id)
    	return nil
    }
    ```

**6. 业务逻辑层 (`internal/service/todo_service.go`)**

```go
package service

import (
	"context"
	"log/slog"
	"todo-api/internal/domain"
	"todo-api/internal/repository"
)

// TodoService 定义了待办事项相关的业务逻辑操作
type TodoService interface {
	GetAllTodos(ctx context.Context) ([]domain.Todo, error)
	GetTodoByID(ctx context.Context, id int) (*domain.Todo, error)
	CreateTodo(ctx context.Context, title, description string) (*domain.Todo, error)
	UpdateTodo(ctx context.Context, id int, title, description string, completed *bool) (*domain.Todo, error)
	DeleteTodo(ctx context.Context, id int) error
}

type todoService struct {
	repo   repository.TodoRepository
	logger *slog.Logger
}

// NewTodoService 创建一个新的 TodoService 实例
func NewTodoService(repo repository.TodoRepository, logger *slog.Logger) TodoService {
	return &todoService{
		repo:   repo,
		logger: logger.With("service", "TodoService"), // 添加上下文日志信息
	}
}

func (s *todoService) GetAllTodos(ctx context.Context) ([]domain.Todo, error) {
	s.logger.InfoContext(ctx, "Fetching all todos")
	todos, err := s.repo.GetAll(ctx)
	if err != nil {
		s.logger.ErrorContext(ctx, "Error fetching all todos from repository", "error", err)
		return nil, err // 可以包装错误，但这里简单返回
	}
	s.logger.DebugContext(ctx, "Successfully fetched todos", "count", len(todos))
	return todos, nil
}

func (s *todoService) GetTodoByID(ctx context.Context, id int) (*domain.Todo, error) {
	s.logger.InfoContext(ctx, "Fetching todo by ID", "id", id)
	todo, err := s.repo.GetByID(ctx, id)
	if err != nil {
		if errors.Is(err, repository.ErrTodoNotFound) {
			s.logger.WarnContext(ctx, "Todo not found", "id", id)
		} else {
			s.logger.ErrorContext(ctx, "Error fetching todo by ID from repository", "id", id, "error", err)
		}
		return nil, err
	}
	s.logger.DebugContext(ctx, "Successfully fetched todo", "id", id)
	return todo, nil
}

func (s *todoService) CreateTodo(ctx context.Context, title, description string) (*domain.Todo, error) {
	s.logger.InfoContext(ctx, "Creating new todo", "title", title)
	todo := &domain.Todo{
		Title:       title,
		Description: description,
		Completed:   false, // 默认未完成
	}
	err := s.repo.Create(ctx, todo)
	if err != nil {
		s.logger.ErrorContext(ctx, "Error creating todo in repository", "title", title, "error", err)
		return nil, err
	}
	s.logger.InfoContext(ctx, "Successfully created todo", "id", todo.ID)
	return todo, nil
}

func (s *todoService) UpdateTodo(ctx context.Context, id int, title, description string, completed *bool) (*domain.Todo, error) {
	s.logger.InfoContext(ctx, "Updating todo", "id", id)
	existingTodo, err := s.repo.GetByID(ctx, id)
	if err != nil {
		// GetByID 已经记录了日志
		return nil, err
	}

	// 更新字段
	if title != "" {
		existingTodo.Title = title
	}
	if description != "" {
		existingTodo.Description = description
	}
	if completed != nil {
		existingTodo.Completed = *completed
	}

	err = s.repo.Update(ctx, existingTodo)
	if err != nil {
		s.logger.ErrorContext(ctx, "Error updating todo in repository", "id", id, "error", err)
		return nil, err
	}
	s.logger.InfoContext(ctx, "Successfully updated todo", "id", id)
	return existingTodo, nil
}

func (s *todoService) DeleteTodo(ctx context.Context, id int) error {
	s.logger.InfoContext(ctx, "Deleting todo", "id", id)
	err := s.repo.Delete(ctx, id)
	if err != nil {
		if errors.Is(err, repository.ErrTodoNotFound) {
			s.logger.WarnContext(ctx, "Attempted to delete non-existent todo", "id", id)
		} else {
			s.logger.ErrorContext(ctx, "Error deleting todo from repository", "id", id, "error", err)
		}
		return err
	}
	s.logger.InfoContext(ctx, "Successfully deleted todo", "id", id)
	return nil
}

```

**7. API 层 (`internal/api/`)**

*   **请求结构与验证 (`request/todo_request.go`)**

    ```go
    package request

    // CreateTodoRequest 定义了创建 Todo 的请求体结构
    type CreateTodoRequest struct {
    	Title       string `json:"title" binding:"required,min=3,max=100"`
    	Description string `json:"description" binding:"max=500"`
    }

    // UpdateTodoRequest 定义了更新 Todo 的请求体结构
    type UpdateTodoRequest struct {
    	Title       *string `json:"title,omitempty" binding:"omitempty,min=3,max=100"` // omitempty 让字段可选, 指针类型用于区分零值和未提供
    	Description *string `json:"description,omitempty" binding:"omitempty,max=500"`
    	Completed   *bool   `json:"completed,omitempty"`
    }
    ```

*   **通用错误响应 (`pkg/httputil/error_response.go`)**

    ```go
    package httputil

    import "github.com/gin-gonic/gin"

    // ErrorResponse 定义了标准的错误响应结构
    type ErrorResponse struct {
    	Error   string `json:"error"`
    	Details any    `json:"details,omitempty"` // 可以是字符串、map 或 validation errors
    }

    // NewErrorResponse 创建一个新的错误响应并中止请求
    func NewErrorResponse(c *gin.Context, status int, err error, details ...any) {
    	resp := ErrorResponse{
    		Error: err.Error(),
    	}
    	if len(details) > 0 {
    		resp.Details = details[0]
    	}
    	c.AbortWithStatusJSON(status, resp)
    }
    ```

*   **中间件 (`middleware/`)**

    *   **日志中间件 (`logger.go`)** (使用 `log/slog`)

        ```go
        package middleware

        import (
        	"log/slog"
        	"time"

        	"github.com/gin-gonic/gin"
        )

        func StructuredLogger(logger *slog.Logger) gin.HandlerFunc {
        	return func(c *gin.Context) {
        		start := time.Now()
        		path := c.Request.URL.Path
        		query := c.Request.URL.RawQuery

        		// 处理请求
        		c.Next()

        		// 请求处理完毕后记录日志
        		latency := time.Since(start)
        		statusCode := c.Writer.Status()
        		clientIP := c.ClientIP()
        		method := c.Request.Method

        		logAttrs := []slog.Attr{
        			slog.Int("status", statusCode),
        			slog.String("method", method),
        			slog.String("path", path),
        			slog.String("query", query),
        			slog.String("ip", clientIP),
        			slog.Duration("latency", latency),
        			slog.String("user_agent", c.Request.UserAgent()),
        		}

        		if len(c.Errors) > 0 {
        			// 只记录最后一个错误以便简化
        			logger.Error("Request failed", append(logAttrs, slog.String("error", c.Errors.String()))...)
        		} else {
        			logger.Info("Request processed", logAttrs...)
        		}
        	}
        }
        ```
    *   **错误处理中间件 (`error_handler.go`)**

        ```go
        package middleware

        import (
        	"errors"
        	"log/slog"
        	"net/http"
        	"todo-api/internal/repository"
        	"todo-api/pkg/httputil"

        	"github.com/gin-gonic/gin"
        	"github.com/go-playground/validator/v10"
        )

        // ErrorHandler 中间件捕获后续处理中发生的错误并将其转换为标准的 JSON 错误响应
        func ErrorHandler(logger *slog.Logger) gin.HandlerFunc {
        	return func(c *gin.Context) {
        		c.Next() // 先执行路由处理器和其他中间件

        		if len(c.Errors) == 0 {
        			return // 没有错误，直接返回
        		}

        		// 只处理第一个错误，或者可以迭代处理
        		err := c.Errors[0].Err

        		// 在这里根据错误类型决定 HTTP 状态码和响应体
        		var ve validator.ValidationErrors
        		if errors.As(err, &ve) {
        			// 处理验证错误
        			errs := make(map[string]string)
        			for _, fe := range ve {
        				errs[fe.Field()] = fe.Tag() // 可以自定义更友好的错误消息
        			}
        			logger.WarnContext(c.Request.Context(), "Validation error", "details", errs, "error", err)
        			httputil.NewErrorResponse(c, http.StatusBadRequest, errors.New("validation failed"), errs)
        			return
        		}

        		if errors.Is(err, repository.ErrTodoNotFound) {
        			logger.WarnContext(c.Request.Context(), "Resource not found", "error", err)
        			httputil.NewErrorResponse(c, http.StatusNotFound, err)
        			return
        		}

                // 添加对 context canceled 或 deadline exceeded 的处理
                if errors.Is(err, context.Canceled) {
        			logger.WarnContext(c.Request.Context(), "Request canceled by client", "error", err)
                    httputil.NewErrorResponse(c, 499, err) // 499 Client Closed Request (非标准，但常用)
                    return
                }
                if errors.Is(err, context.DeadlineExceeded) {
        			logger.WarnContext(c.Request.Context(), "Request deadline exceeded", "error", err)
                    httputil.NewErrorResponse(c, http.StatusGatewayTimeout, err)
                    return
                }


        		// 其他未知错误，视为服务器内部错误
        		logger.ErrorContext(c.Request.Context(), "Internal server error", "error", err)
        		httputil.NewErrorResponse(c, http.StatusInternalServerError, errors.New("internal server error")) // 避免暴露内部错误细节
        	}
        }
        ```

*   **Handler (`handler/todo_handler.go`)**

    ```go
    package handler

    import (
    	"errors"
    	"net/http"
    	"strconv"
    	"todo-api/internal/api/request"
    	"todo-api/internal/service"
    	"todo-api/pkg/httputil"

    	"github.com/gin-gonic/gin"
    )

    type TodoHandler struct {
    	todoService service.TodoService
    }

    func NewTodoHandler(ts service.TodoService) *TodoHandler {
    	return &TodoHandler{todoService: ts}
    }

    // GetAllTodos godoc
    // @Summary Get all todos
    // @Description Get a list of all todo items
    // @Tags todos
    // @Accept json
    // @Produce json
    // @Success 200 {array} domain.Todo
    // @Failure 500 {object} httputil.ErrorResponse
    // @Router /todos [get]
    func (h *TodoHandler) GetAllTodos(c *gin.Context) {
    	todos, err := h.todoService.GetAllTodos(c.Request.Context())
    	if err != nil {
    		// 使用 c.Error 将错误传递给错误处理中间件
    		_ = c.Error(err) // The error middleware will handle the response
    		return
    	}
    	c.JSON(http.StatusOK, todos)
    }

    // GetTodoByID godoc
    // @Summary Get a todo by ID
    // @Description Get details of a specific todo item by its ID
    // @Tags todos
    // @Accept json
    // @Produce json
    // @Param id path int true "Todo ID"
    // @Success 200 {object} domain.Todo
    // @Failure 400 {object} httputil.ErrorResponse "Invalid ID format"
    // @Failure 404 {object} httputil.ErrorResponse "Todo not found"
    // @Failure 500 {object} httputil.ErrorResponse
    // @Router /todos/{id} [get]
    func (h *TodoHandler) GetTodoByID(c *gin.Context) {
    	idStr := c.Param("id")
    	id, err := strconv.Atoi(idStr)
    	if err != nil {
    		httputil.NewErrorResponse(c, http.StatusBadRequest, errors.New("invalid ID format"))
    		return
    	}

    	todo, err := h.todoService.GetTodoByID(c.Request.Context(), id)
    	if err != nil {
    		_ = c.Error(err) // Let error middleware handle based on type (e.g., ErrTodoNotFound -> 404)
    		return
    	}
    	c.JSON(http.StatusOK, todo)
    }

    // CreateTodo godoc
    // @Summary Create a new todo
    // @Description Add a new item to the todo list
    // @Tags todos
    // @Accept json
    // @Produce json
    // @Param todo body request.CreateTodoRequest true "Todo object to create"
    // @Success 201 {object} domain.Todo
    // @Failure 400 {object} httputil.ErrorResponse "Invalid request body or validation error"
    // @Failure 500 {object} httputil.ErrorResponse
    // @Router /todos [post]
    func (h *TodoHandler) CreateTodo(c *gin.Context) {
    	var req request.CreateTodoRequest
    	// 绑定并验证请求体
    	if err := c.ShouldBindJSON(&req); err != nil {
    		// 验证错误会被 validator 捕获，并通过 error middleware 处理
    		_ = c.Error(err)
    		return
    	}

    	todo, err := h.todoService.CreateTodo(c.Request.Context(), req.Title, req.Description)
    	if err != nil {
    		_ = c.Error(err)
    		return
    	}
    	c.JSON(http.StatusCreated, todo)
    }

    // UpdateTodo godoc
    // @Summary Update an existing todo
    // @Description Update details of a specific todo item by its ID
    // @Tags todos
    // @Accept json
    // @Produce json
    // @Param id path int true "Todo ID"
    // @Param todo body request.UpdateTodoRequest true "Todo object with updated fields"
    // @Success 200 {object} domain.Todo
    // @Failure 400 {object} httputil.ErrorResponse "Invalid ID format or request body/validation error"
    // @Failure 404 {object} httputil.ErrorResponse "Todo not found"
    // @Failure 500 {object} httputil.ErrorResponse
    // @Router /todos/{id} [put]
    func (h *TodoHandler) UpdateTodo(c *gin.Context) {
    	idStr := c.Param("id")
    	id, err := strconv.Atoi(idStr)
    	if err != nil {
    		httputil.NewErrorResponse(c, http.StatusBadRequest, errors.New("invalid ID format"))
    		return
    	}

    	var req request.UpdateTodoRequest
    	if err := c.ShouldBindJSON(&req); err != nil {
    		_ = c.Error(err) // Validation errors handled by middleware
    		return
    	}

        // 注意：直接传递指针可能为 nil，service 层需要处理
        var title, description string
        if req.Title != nil { title = *req.Title }
        if req.Description != nil { description = *req.Description }


    	updatedTodo, err := h.todoService.UpdateTodo(c.Request.Context(), id, title, description, req.Completed)
    	if err != nil {
    		_ = c.Error(err) // Handles NotFound and other errors
    		return
    	}
    	c.JSON(http.StatusOK, updatedTodo)
    }

    // DeleteTodo godoc
    // @Summary Delete a todo by ID
    // @Description Remove a specific todo item by its ID
    // @Tags todos
    // @Accept json
    // @Produce json
    // @Param id path int true "Todo ID"
    // @Success 204 "No Content"
    // @Failure 400 {object} httputil.ErrorResponse "Invalid ID format"
    // @Failure 404 {object} httputil.ErrorResponse "Todo not found"
    // @Failure 500 {object} httputil.ErrorResponse
    // @Router /todos/{id} [delete]
    func (h *TodoHandler) DeleteTodo(c *gin.Context) {
    	idStr := c.Param("id")
    	id, err := strconv.Atoi(idStr)
    	if err != nil {
    		httputil.NewErrorResponse(c, http.StatusBadRequest, errors.New("invalid ID format"))
    		return
    	}

    	err = h.todoService.DeleteTodo(c.Request.Context(), id)
    	if err != nil {
    		_ = c.Error(err) // Handles NotFound and other errors
    		return
    	}
    	c.Status(http.StatusNoContent) // 204 No Content for successful deletion
    }
    ```

*   **路由注册 (`handler/routes.go`)**

    ```go
    package handler

    import "github.com/gin-gonic/gin"

    // RegisterRoutes 将 Todo 相关的路由注册到 Gin 引擎
    func (h *TodoHandler) RegisterRoutes(router *gin.RouterGroup) {
    	todoRoutes := router.Group("/todos")
    	{
    		todoRoutes.GET("", h.GetAllTodos)
    		todoRoutes.POST("", h.CreateTodo)
    		todoRoutes.GET("/:id", h.GetTodoByID)
    		todoRoutes.PUT("/:id", h.UpdateTodo)
    		todoRoutes.DELETE("/:id", h.DeleteTodo)
    	}
    }
    ```

**8. 程序入口 (`cmd/server/main.go`)**

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
	"todo-api/config"
	apihandler "todo-api/internal/api/handler"
	"todo-api/internal/api/middleware"
	"todo-api/internal/repository"
	"todo-api/internal/service"

	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"github.com/lmittmann/tint" // Optional: for colorful console output
)

func main() {
	// --- 配置加载 ---
	cfg, err := config.LoadConfig("./config/config.yaml") // 或者从命令行参数获取路径
	if err != nil {
		slog.Error("Failed to load configuration", "error", err)
		os.Exit(1)
	}

	// --- 设置日志 ---
	logLevel := slog.LevelInfo
	if cfg.Server.Mode == "debug" {
		logLevel = slog.LevelDebug
	}
	// 可选：使用 tint 实现彩色日志输出到控制台
	logger := slog.New(tint.NewHandler(os.Stdout, &tint.Options{Level: logLevel, TimeFormat: time.Kitchen}))
	// 或者使用标准 JSON Handler
	// logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: logLevel}))
	slog.SetDefault(logger) // 设置为全局默认 logger

	// --- 依赖注入 (手动) ---
	// 选择 Repository 实现 (这里用内存，实际项目中可能根据配置选择)
	todoRepo := repository.NewInMemoryTodoRepository()
	// 可以添加数据库 Repository
	// db, err := gorm.Open(mysql.Open(cfg.Database.DSN), &gorm.Config{})
	// if err != nil {
	// 	logger.Error("Failed to connect database", "error", err)
	// 	os.Exit(1)
	// }
	// todoRepo := repository.NewGormTodoRepository(db)

	todoService := service.NewTodoService(todoRepo, logger)
	todoHandler := apihandler.NewTodoHandler(todoService)

	// --- Gin 引擎设置 ---
	gin.SetMode(cfg.Server.Mode)
	router := gin.New() // 使用 New() 以便完全控制中间件

	// --- 全局中间件 ---
	// 结构化日志中间件
	router.Use(middleware.StructuredLogger(logger))
	// 恢复中间件 (Panic Recovery)
	router.Use(gin.Recovery()) // Gin 自带的就很好用
	// 错误处理中间件 (应该放在 Recovery 之后，路由之前)
	router.Use(middleware.ErrorHandler(logger))
	// CORS 中间件 (根据需要配置)
	router.Use(cors.New(cors.Config{
		AllowOrigins:     []string{"*"}, // 生产环境应配置具体的域
		AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowHeaders:     []string{"Origin", "Content-Type", "Accept", "Authorization"},
		ExposeHeaders:    []string{"Content-Length"},
		AllowCredentials: true,
		MaxAge:           12 * time.Hour,
	}))

	// --- 路由注册 ---
	apiGroup := router.Group("/api/v1") // API 版本控制
	todoHandler.RegisterRoutes(apiGroup)

	// 添加健康检查端点
	router.GET("/health", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"status": "UP"})
	})

	// --- 启动 HTTP 服务器 ---
	addr := fmt.Sprintf(":%d", cfg.Server.Port)
	srv := &http.Server{
		Addr:    addr,
		Handler: router,
		ReadTimeout: 10 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout: 60 * time.Second,
	}

	logger.Info("Starting server", "address", addr)

	// 启动服务器 (goroutine)
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("Server failed to start", "error", err)
			os.Exit(1)
		}
	}()

	// --- 优雅关停 ---
	quit := make(chan os.Signal, 1)
	// signal.Notify 会监听指定信号： interrupt (Ctrl+C) 和 termination (kill)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit // 阻塞，直到接收到信号

	logger.Info("Shutting down server...")

	// 创建一个带超时的 context，用于通知服务器有 5 秒时间来完成当前处理的请求
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("Server forced to shutdown", "error", err)
		os.Exit(1)
	}

	logger.Info("Server exiting")
}

```

**9. 如何运行**

1.  确保安装了 Go (推荐 1.21+ 以使用 `log/slog`)。
2.  将以上代码保存到对应的文件和目录结构中。
3.  在 `todo-api` 目录下创建 `go.mod`:
    ```bash
    go mod init todo-api
    ```
4.  下载依赖:
    ```bash
    go get github.com/gin-gonic/gin
    go get github.com/spf13/viper
    go get github.com/go-playground/validator/v10
    go get github.com/gin-contrib/cors
    go get github.com/lmittmann/tint # 可选，用于彩色日志
    # 如果使用数据库，还需要添加对应的驱动和 ORM，例如:
    # go get gorm.io/gorm gorm.io/driver/mysql
    ```
5.  运行服务:
    ```bash
    go run cmd/server/main.go
    ```
    或者编译后运行:
    ```bash
    go build -o todo-server cmd/server/main.go
    ./todo-server
    ```
6.  使用 `curl` 或 Postman 等工具测试 API:
    *   `GET http://localhost:8080/api/v1/todos`
    *   `POST http://localhost:8080/api/v1/todos` (Body: `{"title": "Learn Go", "description": "Read Gin docs"}`)
    *   `GET http://localhost:8080/api/v1/todos/1`
    *   `PUT http://localhost:8080/api/v1/todos/1` (Body: `{"title": "Learn Go Best Practices", "completed": true}`)
    *   `DELETE http://localhost:8080/api/v1/todos/1`
    *   `GET http://localhost:8080/health`

---

**关键最佳实践总结:**

1.  **清晰的结构**: 将代码按功能（配置、API、领域、数据访问、业务逻辑）分层，便于维护和扩展。`internal` 目录保护内部实现。
2.  **依赖倒置**: Service 和 Handler 依赖 Repository 和 Service 的 *接口* 而非具体实现，便于测试（mocking）和替换实现（如从内存切换到数据库）。
3.  **依赖注入**: 依赖关系在 `main.go` 中显式创建并注入，避免全局变量和隐式依赖。对于更复杂的应用，可以考虑使用 DI 容器（如 `google/wire`）。
4.  **配置外部化**: 使用 `viper` 管理配置，支持文件、环境变量等多种来源，方便不同环境部署。
5.  **路由集中管理**: 使用 `RegisterRoutes` 方法将特定模块的路由聚合，并在 `main.go` 中统一挂载到 Gin 引擎，通常会按 API 版本分组。
6.  **标准化的中间件**: 使用中间件处理通用逻辑，如日志记录、Panic恢复、CORS、统一错误处理。
7.  **结构化日志**: 使用 `log/slog` 记录带有上下文信息的结构化日志，方便日志聚合、查询和分析。
8.  **统一错误处理**: 中间件捕获错误，根据错误类型返回标准化的 JSON 响应和合适的 HTTP 状态码，避免向客户端暴露过多内部细节。
9.  **请求验证**: 在 Handler 层或使用专用 Request 结构体验证输入，确保数据有效性，尽早失败。
10. **优雅关停**: 监听系统信号，确保服务器在停止前完成正在处理的请求，防止数据丢失或连接中断。
11. **Context 传递**: 将 `context.Context` 贯穿请求处理链路（Handler -> Service -> Repository），用于控制超时、取消操作和传递请求范围的值。

这个案例提供了一个坚实的基础，你可以根据实际项目的复杂度和需求进行调整和扩展（例如，添加认证中间件、使用数据库、集成 Swagger/OpenAPI 文档生成等）。
