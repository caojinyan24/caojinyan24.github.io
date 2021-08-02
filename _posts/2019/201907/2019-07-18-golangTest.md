---
layout: post
title: "golang 集成测试实践"
date: 2019-07-04 22:14:26 +0800
categories: 基础
tags: go
---

写代码,开发功能,不可避免地就要涉及的单元测试和集成测试.最近一直在折腾组内的集成测试,探索了很多框架的用法,这里沉淀下我最终采用的一套测试实践.
首先了解了下单元测试和集成测试的区别,单元测试关注局部的代码和功能;而集成测试关注整个功能模块.针对单个项目来说,如 kite,gin 框架的代码,最外层的 handler 测试可以认为是集成测试.

写一个测试用例需要关注哪些问题?

1. 数据表结构的同步
2. 测试数据的准备
3. 外部依赖的 mock

# 数据表结构的同步

由于内部项目使用 kite+gorm 做开发,所以对于以上第三点,gorm 提供了 migration 的相关功能,可以做到数据表结构的同步,同时 gorm 一个比较好的一点是,可以将表结构信息定义在 struct model 中.
如:

```
type TradeInfo struct {
	Id        int64     `gorm:"column:id;primary_key" json:"id"`
	OrderId   string    `gorm:"column:order_id" json:"order_id"`
	RequestId string    `gorm:"column:request_id;unique_index:uniq_merchant_id_request_id" json:"request_id"`
	FlowId    string    `gorm:"column:flow_id;unique_index" json:"flow_id"`
}
```

做数据表结构的同步

```
dbConn.AutoMigrate(&model.PayInTradeInfo{})
```

# 测试数据准备

测试数据的一个基本要求是,每次跑 case 使用的数据都是全新未污染的;之前接触到的方案是执行之前先清数据,每次执行前先插数据,这样的问题是数据内容一样,在使用同一个数据库的时候,多个集成任务同时跑的时候,会互相影响.

为了解决这个问题,我一开始考虑每次执行前创建一个数据库,数据库名称拼一个时间戳,执行完之后再 drop 调.这样多个 CI 同时跑的数据在不同的数据库,可以保证不会互相影响;进而考虑到其实可以对每个单测的单号拼时间戳和随机串(`固定前缀+单测case名称+时间戳`)的方式来保证各个测试用例的 id 不一样.

以下是一个示例代码:

```
var _ = Describe("TestDemo", func() {
	var sufix = ""
	BeforeEach(func() {
		sufix = fmt.Sprintf("%s_%d_%s", "_Payin", time.Now().Unix(), rand.String(5))
	})
	AfterEach(func() {
		sufix = fmt.Sprintf("%s_%d_%s", "_Payin", time.Now().Unix(), rand.String(5))
	})
	BeforeEach(func() {
		InsertInfo(sufix)
	})
	Describe("Test1", func() {
		Context("Test1", func() {
			It("Test1", func() {
				//todo
			})
		})
    })
})
var (
	MerchantId           = "merchantId"
	MerchantUId          = "merchantUId"
)
func InsertInfo(sufix string) {
	err := dbConn.Create(&model.User{
		MerchantId:  MerchantId + sufix,
		MerchantUid: MerchantUId + sufix,
		UserType:    "USER",
		UserId:      UserId + sufix,
		CreatedAt:   time.Now().UTC(),
	}).Error
	if err != nil {
		logs.Error("error,err=%v", err)
	}
}
```

# 外部依赖的 mock

gomock 只能对接口做 mock,所以在定义需要 mock 的方法的时候,确保定义一个接口类,然后针对接口类做 mock

1.定义接口,接口文件:mockgen -source=./rpc/rpc_inf.go -package=mock -destination=./mock/rpc_inf_mock.go 2.生成 mock 代码:mockgen -source=./rpc/rpc_inf.go -package=mock -destination=./mock/rpc_inf_mock.go 3.使用 set 方法注入

```
var _ = Describe("Test", func() {
	var (
		mockCtrl *gomock.Controller
		MockIndex *mock.MockRpcInf
	)
	BeforeEach(func() {
		mockCtrl = gomock.NewController(GinkgoT())
		MockIndex = mock.NewMockRpcInf(mockCtrl)
		rpc.SetRpcService(MockIndex)
	})
	AfterEach(func() {
		mockCtrl.Finish()
	})
	Context("fail", func() {
		It("MockFail", func() {
			MockIndex.EXPECT().Invoke(gomock.Any(), gomock.Any(), gomock.Any()).Return(failResp, nil)
		})
	})
})
```

#测试用例的组织

最后, 对于集成测试来说,一个接口可能存在多个测试用例,为了更好的管理这些测试用例,可以使用 ginkgo 框架.ginkgo 完全支持 golang 原生的 testing 包,所以不用担心后续升级和兼容性的问题

ginkgo 对每个包只生成一个 Test 方法

首先进入要生成测试用例的包下:

```
$ cd $GOPATH/src/project/service
$ ginkgo bootstrap
```

以上命令将在 service 文件夹下生成一个文件`service_suite_test.go`,文件内容如下

```
package main_test

import (
"testing"

. "github.com/onsi/ginkgo"
. "github.com/onsi/gomega"
)

func TestService(t *testing.T) {
RegisterFailHandler(Fail)
RunSpecs(t, "Service Suite")
}
```

接下来为 service 文件夹下的`handler.go` 生成测试代码:`ginkgo generate handler.go`,文件内容如下

```
package main_test

import (
. "github.com/onsi/ginkgo"
)

var _ = Describe("Handler", func() {

})
```

编写测试代码,以下做示例:

```
    var _ = Describe("Handler", func() {
        BeforeEach(func() {
            fmt.Println("before each outer")
        })
        AfterEach(func() {
            fmt.Println("after each outer")
        })
        Describe("describe", func() {
            BeforeEach(func() {
                fmt.Println("before each describe")
            })
            AfterEach(func() {
                fmt.Println("after each describe")
            })
            Context("context", func() {
                BeforeEach(func() {
                    fmt.Println("before each context")
                })
                AfterEach(func() {
                    fmt.Println("after each context")
                })
                It("case_1", func() {
                    fmt.Println("case_1")
                })
                It("case_2", func() {
                    fmt.Println("case_2")
                })
            })
        })
    })
```

执行单测:ginkgo 或者 ginkgo -r

运行以上单测时留意下打印的日志,顺便了解下 AfterEach 和 BeforeEach 的执行顺序.

- 每个 It 里是一个测试 case,各个 It 在 ginkgo 运行时会并发执行,写单测的时候需要考虑下这个因素.希望每个单测顺序执行? ginkgo -nodes=1
- 每个 Describe,Context,It 可以嵌套.
- ginkgo 可以自定义单测的报告信息
- 希望只执行一个 case:`ginkgo --focus '$DescribeName'`
- 希望排除某个 case:`ginkgo --ignore '$DescribeName'`
- 希望忽略某个包下的case:`ginkgo -r --skipPackage=util,test/service` (其中的`util`和`test/service`是项目中的文件夹名称)

需要说明的是,对于`--skipPackage`标签,指定的并不是golang的包名,而是文件夹名称,在执行时,会进行文件夹名称的匹配,所有匹配上的文件夹下的测试case,都不会被执行.

# 执行初始化

在执行测试case之前还需要对相关配置做初始化,这些配置只初始化一次,不需要在每次执行case前做.golang提供了init()方法,但是testing包下还有一个更好的选择:`TestMain`

TestMain 可以用来做单测的初始化和清理工作,[点击查看详细参考]( https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter09/09.5.html)

针对TestMain的特性,可以做数据库的连接初始化,配置文件读取等.

# 生成测试覆盖率分析文件

ginkgo 提供了覆盖率相关的选项,包括
`-cover` 指定ginkgo在当前文件夹下生成测试覆盖率分析文件,文件命名规则是`PACKAGE.coverprofile`
`-coverpkg=<PKG1>,<PKG2>` 默认情况下,golang只统计当前文件夹下test在当前文件夹下的代码行数执行比例,如果希望在统计的时候,包含其他包的代码执行情况,需要通过`-coverpkg`来指定要统计的包.
`-coverprofile=<FILENAME>` 指定覆盖率分析文件的文件名
`-outputdir=<DIRECTORY>` 指定覆盖率分析文件的生成地址
示例:
`ginkgo -r -cover -coverpkg=a/b/c/handlers,a/b/c/service/...,a/b/c/rpc,a/b/c/db/... -coverprofile=coverage.output -outputdir=./`

最后查看生成的覆盖率文件
`go tool cover -html=coverage.output -o coverage.html`

# **参考**
****
[gomock](https://github.com/golang/mock)
[ginkgo](https://onsi.github.io/ginkgo/)
[golang-testing](https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter09/09.0.html)
