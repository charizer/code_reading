# 【K8s源码品读】003：Phase 1 - kubectl - 设计模式中Visitor的实现

## 聚焦目标

理解kubectl的核心实现之一：`Visitor Design Pattern` 访问者模式



## 目录

1. [什么是访问者模式](#visitor-design-pattern)

2. [kubectl中的Visitor](#visitor)

3. [Visitor的链式处理](#chained)
   1. 多个对象聚合为一个对象
      1. [VisitorList](#VisitorList)
      2. [EagerVisitorList ](#EagerVisitorList)
   2. 多个方法聚合为一个方法
      1. [DecoratedVisitor](#DecoratedVisitor)
      2. [ContinueOnErrorVisitor](#ContinueOnErrorVisitor)
   3. 将对象抽象为多个底层对象，逐个调用方法
      1. [FlattenListVisitor](#FlattenListVisitor)
      2. [FilteredVisitor](#FilteredVisitor)

4. [Visitor的各类实现](#implements)
   1. [StreamVisitor](#StreamVisitor)
   2. [FileVisitor](#FileVisitor)
   3. [URLVisitor](#URLVisitor)
   4. [KustomizeVisitor](#KustomizeVisitor)



## visitor design pattern

在设计模式中，访问者模式的定义为：

> 允许一个或者多个操作应用到对象上，解耦操作和对象本身

那么，对一个程序来说，具体的表现就是：

1. 表面：某个对象执行了一个方法
2. 内部：对象内部调用了多个方法，最后统一返回结果

举个例子，

1. 表面：调用一个查询订单的接口
2. 内部：先从`缓存`中查询，没查到再去`热点数据库`查询，还没查到则去`归档数据库`里查询



## Visitor 

我们来看看kubeadm中的`访问者模式`的定义：

```go
// Visitor 即为访问者这个对象
type Visitor interface {
	Visit(VisitorFunc) error
}
// VisitorFunc对应这个对象的方法，也就是定义中的“操作”
type VisitorFunc func(*Info, error) error
```

基本的数据结构很简单，但从当前的数据结构来看，有两个问题：

1. `单个操作` 可以直接调用`Visit`方法，那`多个操作`如何实现呢？
2. 在应用`多个操作`时，如果出现了error，该退出还是继续应用`下一个操作`呢？



## Chained

### VisitorList

封装多个Visitor为一个，出现错误就立刻中止并返回

```go
// VisitorList定义为[]Visitor，又实现了Visit方法，也就是将多个[]Visitor封装为一个Visitor
type VisitorList []Visitor

// 发生error就立刻返回，不继续遍历
func (l VisitorList) Visit(fn VisitorFunc) error {
	for i := range l {
		if err := l[i].Visit(fn); err != nil {
			return err
		}
	}
	return nil
}
```



### EagerVisitorList

封装多个Visitor为一个，出现错误暂存下来，全部遍历完再聚合所有的错误并返回

```go
// EagerVisitorList 也是将多个[]Visitor封装为一个Visitor
type EagerVisitorList []Visitor

// 返回的错误暂存到[]error中，统一聚合
func (l EagerVisitorList) Visit(fn VisitorFunc) error {
	errs := []error(nil)
	for i := range l {
		if err := l[i].Visit(func(info *Info, err error) error {
			if err != nil {
				errs = append(errs, err)
				return nil
			}
			if err := fn(info, nil); err != nil {
				errs = append(errs, err)
			}
			return nil
		}); err != nil {
			errs = append(errs, err)
		}
	}
	return utilerrors.NewAggregate(errs)
}
```



### DecoratedVisitor

这里借鉴了装饰器的设计模式，将一个Visitor调用多个VisitorFunc方法，封装为调用一个VisitorFunc

```go
// 装饰器Visitor
type DecoratedVisitor struct {
	visitor    Visitor
	decorators []VisitorFunc
}

// visitor遍历调用decorators中所有函数，有失败立即返回
func (v DecoratedVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			return err
		}
		for i := range v.decorators {
			if err := v.decorators[i](info, nil); err != nil {
				return err
			}
		}
		return fn(info, nil)
	})
}
```



### ContinueOnErrorVisitor

```go
// 报错依旧继续
type ContinueOnErrorVisitor struct {
	Visitor
}

// 报错不立即返回，聚合所有错误后返回
func (v ContinueOnErrorVisitor) Visit(fn VisitorFunc) error {
	errs := []error{}
	err := v.Visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			errs = append(errs, err)
			return nil
		}
		if err := fn(info, nil); err != nil {
			errs = append(errs, err)
		}
		return nil
	})
	if err != nil {
		errs = append(errs, err)
	}
	if len(errs) == 1 {
		return errs[0]
	}
	return utilerrors.NewAggregate(errs)
}
```



## FlattenListVisitor

将runtime.ObjectTyper解析成多个runtime.Object，再转换为多个Info，逐个调用VisitorFunc

```go
type FlattenListVisitor struct {
	visitor Visitor
	typer   runtime.ObjectTyper
	mapper  *mapper
}
```



### FilteredVisitor

对Info资源的检验

```go
// 过滤的Info
type FilteredVisitor struct {
	visitor Visitor
	filters []FilterFunc
}

func (v FilteredVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			return err
		}
		for _, filter := range v.filters {
      // 检验Info是否满足条件，出错则退出
			ok, err := filter(info, nil)
			if err != nil {
				return err
			}
			if !ok {
				return nil
			}
		}
		return fn(info, nil)
	})
}
```



## Implements

### StreamVisitor

最基础的Visitor

```go
type StreamVisitor struct {
  // 读取信息的来源，实现了Read这个接口，这个"流式"的概念，包括了常见的HTTP、文件、标准输入等各类输入
	io.Reader
	*mapper

	Source string
	Schema ContentValidator
}
```



### FileVisitor

文件的访问，包括标准输入，底层调用StreamVisitor来访问

```go
type FileVisitor struct {
  // 表示文件路径或者STDIN
	Path string
	*StreamVisitor
}
```



### URLVisitor

HTTP用GET方法获取数据，底层也是复用StreamVisitor

```go
type URLVisitor struct {
	URL *url.URL
	*StreamVisitor
  // 提供错误重试次数
	HttpAttemptCount int
}
```



### KustomizeVisitor

自定义的Visitor，针对自定义的文件系统

```go
type KustomizeVisitor struct {
	Path string
	*StreamVisitor
}
```

