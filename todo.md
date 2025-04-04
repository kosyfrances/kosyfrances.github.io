
Given that `go-redis` is the official and more widely used Redis client for Go, it makes sense to make the article about `go-redis` and `redismock/v9` as Part One. This will provide beginners with the most relevant and up-to-date information first. The article about `redigomock` for Redigo can then be Part Two.

### Part One: Writing Unit Tests for Redis with Redismock/v9 for go-redis

#### Introduction

Unit testing is an essential part of software development. It ensures that your code works as expected and helps catch bugs early. When working with Redis and the `go-redis` library in Go, you can use the `redismock/v9` package to create mock Redis connections for testing. This article will guide you through writing unit tests for Redis using `redismock/v9`.

#### Prerequisites

- Basic knowledge of Go programming language
- Go installed on your machine
- Redis installed and running (optional for this tutorial)

#### Setting Up

First, you need to install the `redismock/v9` package. You can do this by running:

```sh
go get github.com/go-redis/redismock/v9
```

#### Writing Unit Tests with Redismock/v9

Let's start by creating a simple Redis client using `go-redis`. We'll then write unit tests for this client using `redismock/v9`.

##### Step 1: Create a Redis Client

Create a file named `redis_client.go` and add the following code:

```go
package main

import (
	"context"
	"github.com/go-redis/redis/v9"
)

type RedisClient struct {
	client *redis.Client
	ctx    context.Context
}

func NewRedisClient(addr string) *RedisClient {
	client := redis.NewClient(&redis.Options{
		Addr: addr,
	})

	return &RedisClient{
		client: client,
		ctx:    context.Background(),
	}
}

func (c *RedisClient) Set(key, value string) error {
	return c.client.Set(c.ctx, key, value, 0).Err()
}

func (c *RedisClient) Get(key string) (string, error) {
	return c.client.Get(c.ctx, key).Result()
}
```

##### Step 2: Write Unit Tests

Create a file named `redis_client_test.go` and add the following code:

```go
package main

import (
	"testing"

	"github.com/go-redis/redismock/v9"
	"github.com/stretchr/testify/assert"
)

func TestRedisClient_Set(t *testing.T) {
	db, mock := redismock.NewClientMock()

	mock.ExpectSet("key", "value", 0).SetVal("OK")

	client := &RedisClient{
		client: db,
		ctx:    context.Background(),
	}

	err := client.Set("key", "value")
	assert.NoError(t, err)
	assert.NoError(t, mock.ExpectationsWereMet())
}

func TestRedisClient_Get(t *testing.T) {
	db, mock := redismock.NewClientMock()

	mock.ExpectGet("key").SetVal("value")

	client := &RedisClient{
		client: db,
		ctx:    context.Background(),
	}

	value, err := client.Get("key")
	assert.NoError(t, err)
	assert.Equal(t, "value", value)
	assert.NoError(t, mock.ExpectationsWereMet())
}
```

##### Step 3: Run the Tests

Run the tests using the following command:

```sh
go test -v
```

You should see output indicating that the tests passed successfully.

#### Conclusion

In this article, we covered how to write unit tests for a Redis client using the `redismock/v9` package with the `go-redis` library. This approach allows you to mock Redis commands and test your code without needing a running Redis instance. By following these steps, you can ensure that your Redis interactions are correctly implemented and tested.

---

### Part Two: Writing Unit Tests for Redis with Redigomock for Redigo

#### Introduction

In the previous article, we learned how to write unit tests for Redis using `redismock/v9` with the `go-redis` library. In this part, we will focus on using the `redigomock` package to write unit tests for Redis with the Redigo library.

#### Prerequisites

- Basic knowledge of Go programming language
- Go installed on your machine
- Redis installed and running (optional for this tutorial)

#### Setting Up

First, you need to install the `redigomock` package. You can do this by running:

```sh
go get github.com/rafaeljusto/redigomock
```

#### Writing Unit Tests with Redigomock

Let's start by creating a simple Redis client using Redigo. We'll then write unit tests for this client using `redigomock`.

##### Step 1: Create a Redis Client

Create a file named `redis_client.go` and add the following code:

```go
package main

import (
	"github.com/gomodule/redigo/redis"
)

type RedisClient struct {
	pool *redis.Pool
}

func NewRedisClient(addr string) *RedisClient {
	return &RedisClient{
		pool: &redis.Pool{
			Dial: func() (redis.Conn, error) {
				return redis.Dial("tcp", addr)
			},
		},
	}
}

func (c *RedisClient) Set(key, value string) error {
	conn := c.pool.Get()
	defer conn.Close()

	_, err := conn.Do("SET", key, value)
	return err
}

func (c *RedisClient) Get(key string) (string, error) {
	conn := c.pool.Get()
	defer conn.Close()

	return redis.String(conn.Do("GET", key))
}
```

##### Step 2: Write Unit Tests

Create a file named `redis_client_test.go` and add the following code:

```go
package main

import (
	"testing"

	"github.com/rafaeljusto/redigomock"
	"github.com/stretchr/testify/assert"
)

func TestRedisClient_Set(t *testing.T) {
	mockConn := redigomock.NewConn()
	mockConn.Command("SET", "key", "value").Expect("OK")

	client := &RedisClient{
		pool: &redis.Pool{
			Dial: func() (redis.Conn, error) {
				return mockConn, nil
			},
		},
	}

	err := client.Set("key", "value")
	assert.NoError(t, err)
}

func TestRedisClient_Get(t *testing.T) {
	mockConn := redigomock.NewConn()
	mockConn.Command("GET", "key").Expect("value")

	client := &RedisClient{
		pool: &redis.Pool{
			Dial: func() (redis.Conn, error) {
				return mockConn, nil
			},
		},
	}

	value, err := client.Get("key")
	assert.NoError(t, err)
	assert.Equal(t, "value", value)
}
```

##### Step 3: Run the Tests

Run the tests using the following command:

```sh
go test -v
```

You should see output indicating that the tests passed successfully.

#### Conclusion

In this article, we covered how to write unit tests for a Redis client using the `redigomock` package with the Redigo library. This approach allows you to mock Redis connections and test your code without needing a running Redis instance. By following these steps, you can ensure that your Redis interactions are correctly implemented and tested.
