+++
title = 'go中使用代码测试'
date = 2024-06-02
draft = false
toc = true
tags = ['test', 'go']
categories = ['Golang']
+++

代码测试在开发中是非常重要的一环，做好代码的单元测试可以避免每次提交代码时去测试整个应用。go 中的代码测试主要用 go 自带的 testing 包，以及那个第三方的 testify 包。

<!--more-->

## 1. go代码测试的约定
约定大于配置是go的一个特色，关于代码测试的约定有2个：
1. 代码测试文件名以下划线加 test 结尾，比如 add.go 的测试文件的名字就是 add_test.go，至于源代码文件和测试代码文件放不放在同一个包里没有具体要求，看个人喜好，但是如果要测试包中的小写字母开头的函数，因为它是不可导出的，所以测试代码要和源代码放在一个包里。
2. 测试函数的命名要以 Test （注意一定要是大写的T）开头，比如要测试源代码中的 `Add` 函数，那么代码测试中测试函数的名字就必须是 `TestAdd`（以Test为命名前缀就可以了，后面的命名没有要求）。

## 2. go testing的基本使用
文件结构
```go
myapp/
├── main.go
└── main_test.go
```

main.go
```go
package main

func Add(a, b int) int {
    return a + b
}

func main() {
    result := Add(2, 3)
    println(result)
}
```

main_test.go
```go
package main

import (
    "testing"
)

// 函数参数是t *testing.T, 通过这个里的一系列方法来控制测试的流程（失败，显示日志等）
func TestAdd1(t *testing.T) {
	a := 1
	b := 2
	if Add(a, b) != 3 {
		t.Fatalf("Test failed, 1 + 2 should be 3")
	}
}

// 先设置一些 test case 然后一起测试
func TestAdd2(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 1, 2, 3},
        {"zero and positive number", 0, 5, 5},
        {"negative and positive number", -1, 1, 0},
        {"negative numbers", -2, -3, -5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

关于 go 自带的测试包 testing的用法，可以参考官方文档，提供了一系列方法用于不同的测试需求。
## 3. 指定运行的 test case
直接运行`go test` 会运行当前包下所有的test case，有的时候我们只想运行某个包下的 test cases 或者某个特定的test case.

**指定运行某个包下面的test**
```sh
go test -v ./bmaas/
```
-v 参数用于更详细的输出测试结果，`./bmaas/` 是测试文件的目录

**运行某个go测试文件里的所有test**  
需要把原代码文件也指定进去，否则测试时会报找不到函数错误
```sh
go test -v ./bmaas/server_test.go ./bmaas/server.go
```

**运行某个test**  
通过 -run 参数来指定要执行哪个 test case
```sh
go test -v -run TestSelectServers ./bmaas/
或者
go test -v -run ^TestSelectServer ./bmaas/
```
-run 参数后面是一个正则表达式， ^TestSelectServer 表示必须以 TestSelectServer 开头的test case

**运行go测试文件里的某个test**
```sh
go test -v -run ^TestSelectServer ./bmaas/server_test.go ./bmaas/server.go
```

## 4. cover 和 bench 参数
**指定 `cover` 参数来输出代码测试覆盖率**
```shell
go test -v -run ^TestSelectServer ./bmaas/. -cover

--- PASS: TestSelectServer (0.00s)
PASS
	go-test/bmaas	coverage: 100.0% of statements
ok  	go-test/bmaas	0.640s	coverage: 100.0% of statements
```
\
**指定 `bench` 参数来测试代码的性能，即benchmark测试**  
go中的benchmark测试和普通的单元测试用例一样，都位于 `_test.go` 文件中，函数名以 `Benchmark` 开头，参数是 `b *testing.B`，不同于以 `Test` 开头，参数是 `t *testing.T` 的普通单元测试。

```go
package bmaas

import "testing"

func BenchmarkSelectServer(b *testing.B) {
  selectServers()
}
```

执行测试的时候使用 `-bench` 参数，但是总是报没有go文件错误：
```shell
go test -bench ./bmaas

no Go files in /Users/xue.a.yu/Codes/my/go-test
```

需要为bench参数指定内容以匹配测试用例的名字😅
```shell
go test -v -bench=. ./bmaas/

BenchmarkSelectServer-10    	1000000000	         0.0000122 ns/op
PASS
ok  	go-test/bmaas	0.346s
```

真的不得不说go的测试用例执行真的是各种坑，这样运行不行那样运行也不行，属实折磨人。

## 5. 使用 testify
go testing没有断言和mock之类的功能，用起来不是很方便，可以用 testify 这个第三方的测试包。
[stretchr/testify: A toolkit with common assertions and mocks that plays nicely with the standard library (github.com)](https://github.com/stretchr/testify)
```shell
go get github.com/stretchr/testify
```

**使用断言来代替if error**
```go
import (
    "testing"
    
    "github.com/stretchr/testify/assert"
)

func TestAdd1(t *testing.T) {
	a := 1
	b := 2
//	if Add(a, b) != 3 {
//		t.Fatalf("Test failed, 1 + 2 should be 3")
//	}
     assert.Equal(t, a, b, 3)
}
```
使用`testify`编写测试代码与`testing`一样，测试文件为`_test.go`，测试函数为`TestXxx`。使用`go test`命令运行测试。

**使用testify的mock**  
一个典型的mock用例是，但需要从外部获取数据源（网络请求，数据库等）导致每次返回的数据不一致时，为了方便测试我们可以用mock来模拟数据源，从而测试其他用到这些数据的代码是否按预期工作。
```go
// data_fetcher.go
package main

import "errors"

// DataFetcher 是一个接口，定义了获取数据的方法
type DataFetcher interface {
    FetchData(id int) (string, error)
}

// MyService 是依赖 DataFetcher 的服务
type MyService struct {
    fetcher DataFetcher
}

// NewMyService 创建一个 MyService 实例
func NewMyService(fetcher DataFetcher) *MyService {
    return &MyService{fetcher: fetcher}
}

// ProcessData 根据 id 获取数据并进行处理
func (s *MyService) ProcessData(id int) (string, error) {
    data, err := s.fetcher.FetchData(id)
    if err != nil {
        return "", err
    }
    return "Processed: " + data, nil
}
```
测试 `ProcessData`
```go
// data_fetcher_test.go
package main

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

// MockDataFetcher 是 DataFetcher 接口的 mock 实现
type MockDataFetcher struct {
    mock.Mock
}

// FetchData 是 MockDataFetcher 对 DataFetcher 接口的 FetchData 方法的实现
func (m *MockDataFetcher) FetchData(id int) (string, error) {
    args := m.Called(id)
    return args.String(0), args.Error(1)
}

func TestMyService_ProcessData(t *testing.T) {
    // 创建 MockDataFetcher 实例
    mockFetcher := new(MockDataFetcher)

    // 定义 mock 对象的行为
    mockFetcher.On("FetchData", 1).Return("Mock Data", nil)
    mockFetcher.On("FetchData", 2).Return("", errors.New("not found"))

    // 创建 MyService 实例，并注入 mockFetcher
    service := NewMyService(mockFetcher)

    // 测试成功情况
    result, err := service.ProcessData(1)
    assert.NoError(t, err)
    assert.Equal(t, "Processed: Mock Data", result)

    // 测试错误情况
    result, err = service.ProcessData(2)
    assert.Error(t, err)
    assert.Equal(t, "", result)

    // 验证 mock 对象的行为
    mockFetcher.AssertExpectations(t)
}
```
\
**使用testify的suite**  
可以将多个测试用例组织到一个测试套件中，并且可以在测试前后统一处理一些东西，可以更方便的批量运行测试用例。
```go
package example

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
)

// 定义一个测试套件
type ExampleTestSuite struct {
    suite.Suite
}

// SetupSuite 在整个套件运行之前执行一次
func (suite *ExampleTestSuite) SetupSuite() {
    // 这里可以做一些全局的初始化操作
}

// TearDownSuite 在整个套件运行完毕之后执行一次
func (suite *ExampleTestSuite) TearDownSuite() {
    // 这里可以做一些全局的清理操作
}

// SetupTest 在每个测试运行之前执行
func (suite *ExampleTestSuite) SetupTest() {
    // 这里可以做一些每个测试前的初始化操作
}

// TearDownTest 在每个测试运行之后执行
func (suite *ExampleTestSuite) TearDownTest() {
    // 这里可以做一些每个测试后的清理操作
}

// 示例测试方法
func (suite *ExampleTestSuite) TestExample() {
    result := 1 + 1
    expected := 2
    assert.Equal(suite.T(), expected, result, "他们应该是相等的")
}

// 示例另一个测试方法
func (suite *ExampleTestSuite) TestAnotherExample() {
    result := "Hello, World"
    expected := "Hello, World"
    assert.Equal(suite.T(), expected, result, "他们应该是相等的")
}

// 运行测试套件
func TestExampleTestSuite(t *testing.T) {
    suite.Run(t, new(ExampleTestSuite))
}
```
## 6. 在vscode中设置go test
vscode中可以直接运行test case，当case过不了需要定位问题时也可从 vscode 来调试，非常方便。
![](/images/vscode_test_debug.png)

有时候我们需要设置一些测试用的环境变量或者参数，可以在项目根目录中的 .vscode 目录中的`launch.json` 中配置（没有这个文件的话可以手动创建目录和文件，或者从vcode那个 `RUN AND DEBUG` 下拉框中选择 `Add Config`）。
比如如果测试用例是在项目根目录下的 `app/provider/server_test.go`，那么 `launch.json` 就可以配置为下面这个样子：
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "test",
            "program": "${workspaceRoot}/app/provider",
            "args": [
                "-test.v",
                "-test.run",
                "TestGetServers"
            ],
            "env": {
                "secret": "secret"
            },
            "showLog": true
        }
    ]
}
```
将 `program` 设置为测试代码所在的包并在 `args` 中设置测试用例的名字便可从 vscode 中的 `RUN AND DEBUG` 运行测试用例。
[VSCodeでGoのユニットテストをデバッグしよう #VisualStudioCode - Qiita](https://qiita.com/momotaro98/items/10ae87b21903dd54601c)
## 7.  如何让代码更容易测试
测试时一个比较棘手的问题是代码中依赖其他库需要mock，mock有的时候没那么容易mock，比如这个问题：https://stackoverflow.com/questions/19167970/mock-functions-in-go

如果一个函数里用到另一个函数，可以让这个使用不是直接的强使用，把对函数的使用放到参数里，这样可以在测试的时候直接将其替换成测试函数。
```go
func GetResult(url string) {
	// ...
	response := lib.GetContent(url) // 不好测试GetResult因为这里有个对库函数的依赖
	// ...
}

func TestGetResult(t *testing.T) {
	// ...
	result := GetResult(productionURL) // 在本地测试环境这个url比较难弄
	// ...
}
```

将对函数的依赖放到参数中来解耦，从而方便测试：
```go
type ResutlGetter func(url string) string

func GetResult(resutlGetterFunc ResutlGetter) { 
	// ...
	result := resutlGetterFunc(BASE_URL)
	// ...
}

// mock
func mock_get_result(url string) string {
    // mock some data here
}

func TestGetResult(t *testing.T) {
    GetResult(mock_get_result)
}
```