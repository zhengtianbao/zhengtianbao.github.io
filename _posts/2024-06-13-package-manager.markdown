---
layout: post
title: "包管理器的依赖管理"
date: 2024-06-13 14:13:08 +0800
categories: ["2024"]
tags: [algorithm]
---

设计一个包管理器需要考虑很多方面的问题：如何定义软件包的格式（zip，tar），如何定义软件包的元数据（名称，版本，作者信息，依赖包等），如何在安装包的时候解决依赖关系，如何进行版本管理，如何安装或者卸载软件包，如何进行软件包的存储和分发，如何支持多操作系统等问题。

本文主要针对包管理器在安装软件包过程中是如何解决依赖冲突和循环依赖的问题，用 go 语言进行了简单的描述。

## 依赖关系描述

如何描述依赖关系不同语言有不同的实现，在每个软件包的元数据文件中（如 package.json、setup.py、requirements.txt、POM.xml 等），定义依赖项和版本约束。例如：

- npm: 使用 package.json 中的 dependencies 字段。
- pip: 使用 setup.py 或 requirements.txt 文件。
- Maven: 使用 POM.xml 中的 `<dependencies>` 元素。

go 语言则是在 go.mod 中 require 定义了依赖的包以及对应的版本。

这里通过一个 `Package` 结构体表示解析完元数据文件后对应生成的依赖树。

```go
// Package represents a software package with a name, version, and dependencies.
type Package struct {
	Name         string
	Version      string
	Dependencies []*Package
}
```

`Package` 是一个树状的结构体，表示一个软件包，包括名称、版本和它的依赖项。

## 依赖解析

假设有以下依赖关系：

```
A@1.0.0
├── B@1.0.0
│   └── C@1.0.0
└── C@1.0.0
```

`A@1.0.0` 和 `B@1.0.0` 都依赖 `C@1.0.0`，**去重**后最终需要下载的包应该有 `A@1.0.0`，`B@1.0.0`，`C@1.0.0`。

这个问题其实可以理解为遍历一颗树，去除重复节点。下面代码通过 DFS 遍历，然后去重。

```go
// FlattenDependencies flattens the dependency tree into a map of unique dependencies.
func FlattenDependencies(pkg *Package) []*Package {
	flatDependencies := make(map[string]*Package)
	collectDependencies(pkg, flatDependencies)

	// Convert map to slice and sort
	var sortedDependencies []*Package
	for _, pkg := range flatDependencies {
		sortedDependencies = append(sortedDependencies, pkg)
	}
	sort.Slice(sortedDependencies, func(i, j int) bool {
		return sortedDependencies[i].Name < sortedDependencies[j].Name
	})
	return sortedDependencies
}

// collectDependencies recursively collects dependencies.
func collectDependencies(pkg *Package, flatDependencies map[string]*Package) {
	if _, exists := flatDependencies[pkg.Name]; exists {
		// We will compare version later
	} else {
		flatDependencies[pkg.Name] = pkg
	}

	for _, dep := range pkg.Dependencies {
		collectDependencies(dep, flatDependencies)
	}
}
```

通过 `FlattenDependencies` 方法将重复的依赖包去重后放在变量`flatDependencies` 中，注意这里将结果按字母排序后返回是为了方便测试，同样的 yarn 生成的 yarn.lock 文件按照字母排序是为了在依赖包变动时方便进行 `git diff` 比对差异。

测试用例：

```go
// TestFlattenDependencies tests the FlattenDependencies function.
func TestFlattenDependencies(t *testing.T) {
	pkgC := &Package{Name: "C", Version: "1.0.0"}
	pkgB := &Package{Name: "B", Version: "1.0.0", Dependencies: []*Package{pkgC}}
	pkgA := &Package{Name: "A", Version: "1.0.0", Dependencies: []*Package{pkgB, pkgC}}

	expected := []*Package{
		{Name: "A", Version: "1.0.0"},
		{Name: "B", Version: "1.0.0"},
		{Name: "C", Version: "1.0.0"},
	}

	flattenedDeps := FlattenDependencies(pkgA)

	if len(flattenedDeps) != len(expected) {
		t.Fatalf("Unexpected number of flattened dependencies: got %d, want %d", len(flattenedDeps), len(expected))
	}

	for i := range flattenedDeps {
		if flattenedDeps[i].Name != expected[i].Name || flattenedDeps[i].Version != expected[i].Version {
			t.Errorf("Unexpected dependency at index %d: got %v, want %v", i, flattenedDeps[i], expected[i])
		}
	}
}
```

## 依赖冲突解决

上面的代码中没有考虑版本号冲突产生的问题，考虑下面的依赖：

```
A@1.0.0
├── B@1.0.0
│   └── C@1.2.0
└── C@1.1.1
```

出现了两个版本的 C，那应该保留哪个呢？根据 SemVer 规范的要求：

- 主版本号：不兼容的 API 修改
- 次版本号：向下兼容的功能性新增
- 修订号：向下兼容的问题修正

通过最小版本选择（Minimal Version Selection，MVS）算法来做选择的话，应该保留 `C@1.2.0`。

```go
// collectDependencies recursively collects dependencies.
func collectDependencies(pkg *Package, flatDependencies map[string]*Package) {
	if existingPkg, exists := flatDependencies[pkg.Name]; exists {
		if (compareVersions(pkg.Version, existingPkg.Version) > 0) {
			flatDependencies[pkg.Name] = pkg
		} else {
			return
		}
	} else {
		flatDependencies[pkg.Name] = pkg
	}

	for _, dep := range pkg.Dependencies {
		collectDependencies(dep, flatDependencies)
	}
}

// compareVersions compares two version strings.
func compareVersions(version1, version2 string) int {
	if version1 > version2 {
		return 1
	} else if version1 < version2 {
		return -1
	} else {
		return 0
	}
}
```

测试用例：

```go
func TestConfictDependencies(t *testing.T) {
	pkgC1_1_1 := &Package{Name: "C", Version: "1.1.1"}
	pkgC1_2_0 := &Package{Name: "C", Version: "1.2.0"}
	pkgB := &Package{Name: "B", Version: "1.0.0", Dependencies: []*Package{pkgC1_2_0}}
	pkgA := &Package{Name: "A", Version: "1.0.0", Dependencies: []*Package{pkgB, pkgC1_1_1}}

	expected := []*Package{
		{Name: "A", Version: "1.0.0"},
		{Name: "B", Version: "1.0.0"},
		{Name: "C", Version: "1.2.0"},
	}

	flattenedDeps := FlattenDependencies(pkgA)

	if len(flattenedDeps) != len(expected) {
		t.Fatalf("Unexpected number of flattened dependencies: got %d, want %d", len(flattenedDeps), len(expected))
	}

	for i := range flattenedDeps {
		if flattenedDeps[i].Name != expected[i].Name || flattenedDeps[i].Version != expected[i].Version {
			t.Errorf("Unexpected dependency at index %d: got %v, want %v", i, flattenedDeps[i], expected[i])
		}
	}
}
```

注：以上的代码没有考虑主版本号冲突的情况，如果出现这种情况，那么就需要报错提示手动解决依赖了。

## 循环依赖检测

最后需要补充的是循环依赖检测，假设有以下情况的依赖关系：

```
A@1.0.0
├── B@1.0.0
│   └── C@1.2.0
│       └── A@1.0.0
└── C@1.1.1
```

`C@1.2.0` 引入了依赖 `A@1.0.0`，这就会包解析过程中出现不断的循环，为了处理循环依赖，需要在 `collectDependencies` 函数中增加检测循环依赖的逻辑。可以使用一个辅助数据结构（如 map 或 set）来跟踪正在访问的节点。如果在访问过程中再次遇到已经存在于这个集合中的节点，就说明存在循环依赖。

以下是添加循环依赖检测后的完整代码：

```go
// FlattenDependencies flattens the dependency tree into a sorted slice of unique dependencies.
func FlattenDependencies(pkg *Package) ([]*Package) {
	flatDependencies := make(map[string]*Package)
	visited := make(map[string]bool)

	collectDependencies(pkg, flatDependencies, visited)

	// Convert map to slice and sort
	var sortedDependencies []*Package
	for _, pkg := range flatDependencies {
		sortedDependencies = append(sortedDependencies, pkg)
	}
	sort.Slice(sortedDependencies, func(i, j int) bool {
		return sortedDependencies[i].Name < sortedDependencies[j].Name
	})
	return sortedDependencies
}

// collectDependencies recursively collects dependencies and detects cycles.
func collectDependencies(pkg *Package, flatDependencies map[string]*Package, visited map[string]bool) {
	if visited[pkg.Name] {
		// detected cycle in dependencies, skip
		return
	}

	if existingPkg, exists := flatDependencies[pkg.Name]; exists {
		if compareVersions(pkg.Version, existingPkg.Version) > 0 {
			flatDependencies[pkg.Name] = pkg
		} else {
			return
		}
	} else {
		flatDependencies[pkg.Name] = pkg
	}

	visited[pkg.Name] = true
	defer func() { visited[pkg.Name] = false }()

	for _, dep := range pkg.Dependencies {
		collectDependencies(dep, flatDependencies, visited)
	}
}

```

`FlattenDependencies` 函数：

- 增加一个 `visited` map 来跟踪正在访问的节点。
- 修改调用 `collectDependencies` 函数的方式，以便传递 `visited` map。

`collectDependencies` 函数：

- 增加 `visited` map 参数。
- 检查当前依赖包是否在 `visited` map 中。如果存在，说明检测到循环依赖，直接跳过。
- 将当前依赖包添加到 `visited` map 中。
- 使用 `defer` 语句确保在函数退出前将当前依赖包从 `visited` map 中移除。
- 递归调用 `collectDependencies` 函数处理依赖项。

通过这种方式，我们可以检测和处理循环依赖，确保依赖树的扁平化过程不会陷入无限循环。

增加测试用例：

```go
func TestCircleDependencies(t *testing.T) {
	pkgC1_1_1 := &Package{Name: "C", Version: "1.1.1"}
	pkgC1_2_0 := &Package{Name: "C", Version: "1.2.0", Dependencies: []*Package{}}
	pkgB := &Package{Name: "B", Version: "1.0.0", Dependencies: []*Package{pkgC1_2_0}}
	pkgA := &Package{Name: "A", Version: "1.0.0", Dependencies: []*Package{pkgB, pkgC1_1_1}}
	pkgC1_2_0.Dependencies = append(pkgC1_2_0.Dependencies, pkgA)

	expected := []*Package{
		{Name: "A", Version: "1.0.0"},
		{Name: "B", Version: "1.0.0"},
		{Name: "C", Version: "1.2.0"},
	}

	flattenedDeps := FlattenDependencies(pkgA)

	if len(flattenedDeps) != len(expected) {
		t.Fatalf("Unexpected number of flattened dependencies: got %d, want %d", len(flattenedDeps), len(expected))
	}

	for i := range flattenedDeps {
		if flattenedDeps[i].Name != expected[i].Name || flattenedDeps[i].Version != expected[i].Version {
			t.Errorf("Unexpected dependency at index %d: got %v, want %v", i, flattenedDeps[i], expected[i])
		}
	}
}
```

运行结果：

```
$ go test -v
=== RUN   TestFlattenDependencies
--- PASS: TestFlattenDependencies (0.00s)
=== RUN   TestConfictDependencies
--- PASS: TestConfictDependencies (0.00s)
=== RUN   TestCircleDependencies
--- PASS: TestCircleDependencies (0.00s)
PASS
ok      github.com/zhengtianbao/pm      0.001s
```
