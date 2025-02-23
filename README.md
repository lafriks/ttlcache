# TTLCache - an in-memory cache with item expiration

[![Go Reference](https://pkg.go.dev/badge/github.com/lafriks/ttlcache/v3.svg)](https://pkg.go.dev/github.com/lafriks/ttlcache/v3)
[![Build Status](https://github.com/lafriks/ttlcache/actions/workflows/go.yml/badge.svg)](https://github.com/lafriks/ttlcache/actions/workflows/go.yml)
[![Coverage Status](https://coveralls.io/repos/github/lafriks/ttlcache/badge.svg?branch=master)](https://coveralls.io/github/lafriks/ttlcache?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/lafriks/ttlcache/v3)](https://goreportcard.com/report/github.com/lafriks/ttlcache/v3)

## Features
- Simple API.
- Type parameters.
- Item expiration and automatic deletion.
- Automatic expiration time extension on each `Get` call.
- `Loader` interface that is used to load/lazily initialize missing cache 
items.
- Subscription to cache events (insertion and eviction).
- Metrics.
- Configurability.

## Installation
```
go get github.com/lafriks/ttlcache/v3
```

## Usage
The main type of `ttlcache` is `Cache`. It represents a single 
in-memory data store.

To create a new instance of `ttlcache.Cache`, the `ttlcache.New()` function 
should be called:
```go
func main() {
	cache := ttlcache.New[string, string]()
}
```

Note that by default, a new cache instance does not let any of its
items to expire or be automatically deleted. However, this feature
can be activated by passing a few additional options into the 
`ttlcache.New()` function and calling the `cache.Start()` method:
```go
func main() {
	cache := ttlcache.New[string, string](
		ttlcache.WithTTL[string, string](30 * time.Minute),
	)

	go cache.Start() // starts automatic expired item deletion
}
```

Even though the `cache.Start()` method handles expired item deletion well,
there may be times when the system that uses `ttlcache` needs to determine 
when to delete the expired items itself. For example, it may need to 
delete them only when the resource load is at its lowest (e.g., after 
midnight, when the number of users/HTTP requests drops). So, in situations 
like these, instead of calling `cache.Start()`, the system could 
periodically call `cache.DeleteExpired()`:
```go
func main() {
	cache := ttlcache.New[string, string](
		ttlcache.WithTTL[string, string](30 * time.Minute),
	)

	for {
		time.Sleep(4 * time.Hour)
		cache.DeleteExpired()
	}
}
```

The data stored in `ttlcache.Cache` can be retrieved and updated with 
`Set`, `Get`, `Delete`, etc. methods:
```go
func main() {
	cache := ttlcache.New[string, string](
		ttlcache.WithTTL[string, string](30 * time.Minute),
	)

	// insert data
	cache.Set("first", "value1", ttlcache.DefaultTTL)
	cache.Set("second", "value2", ttlcache.NoTTL)
	cache.Set("third", "value3", ttlcache.DefaultTTL)

	// retrieve data
	item, _ := cache.Get("first")
	fmt.Println(item.Value(), item.ExpiresAt())

	// delete data
	cache.Delete("second")
	cache.DeleteExpired()
	cache.DeleteAll()
}
```

To subscribe to insertion, eviction and clear events, `cache.OnInsertion()`, 
`cache.OnEviction()` and `cache.OnClear()` methods should be used:
```go
func main() {
	cache := ttlcache.New[string, string](
		ttlcache.WithTTL[string, string](30 * time.Minute),
		ttlcache.WithCapacity[string, string](300),
	)

	cache.OnInsertion(func(item *ttlcache.Item[string, string]) {
		fmt.Println(item.Value(), item.ExpiresAt())
	})
	cache.OnEviction(func(reason ttlcache.EvictionReason, item *ttlcache.Item[string, string]) {
		if reason == ttlcache.EvictionReasonCapacityReached {
			fmt.Println(item.Key(), item.Value())
		}
	})
	cache.OnClear(func() {
		fmt.Println("cache has been cleared")
	})

	cache.Set("first", "value1", ttlcache.DefaultTTL)
	cache.DeleteAll()
}
```

To load data when the cache does not have it, a custom or
existing implementation of `ttlcache.Loader` can be used:
```go
func main() {
	loader := ttlcache.LoaderFunc[string, string](
		func(c *ttlcache.Cache[string, string], key string) (*ttlcache.Item[string, string], error) {
			// load from file/make an HTTP request
			if err != nil {
				return nil, err
			}
			item := c.Set("key from file", "value from file")
			return item, nil
		},
	)
	cache := ttlcache.New[string, string](
		ttlcache.WithLoader[string, string](loader),
	)

	item, err := cache.Get("key from file")
}
```
