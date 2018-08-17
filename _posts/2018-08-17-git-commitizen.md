---
layout: post
title: git commit和change log
categories: [编程]
tags: [git, commit, change log]
---


> `git`提交代码时常用`-m`来指定`commit message`，例如`git commit -m 'update xxx'`，一个格式良好的`commit message`应该能够清楚的说明本次代码提交的改动

#### 1. commitizen

`commitizen`是一个用来编写`commit message`的工具，该工具出自`Angular规范`，所以本文到处都有前端的影子

使用`npm`安装

```
$ npm install -g commitizen
```

现在，可以用`git cz`来代替`git commit`命令了

```
$ git cz
```

> 但是`git cz`命令只是简单的打开了一个`vi`编辑器，用于编写`commit message`，对其书写格式并未做要求

#### 2. 安装cz-conventional-changelog

`cz-conventional-changelog`是`commitizen`工具的一个适配器，用于提供一套规范化的`commit message`格式

使用`npm`安装

```
$ npm install -g cz-conventional-changelog
```

初始化项目

```
$ commitizen init cz-conventional-changelog --save-dev --save-exact
```

> 对于非`npm`项目，需要创建一个`package.json`文件内容可输入`{}`

再次运行`git cz`命令，这次有交互式提示了

```
Line 1 will be cropped at 100 characters. All other lines will be wrapped after 100 characters.

? Select the type of change that you're committing: (Use arrow keys)
> feat:     A new feature
  fix:      A bug fix
  docs:     Documentation only changes
  style:    Changes that do not affect the meaning of the code (white-space, fo
rmatting, missing semi-colons, etc)
  refactor: A code change that neither fixes a bug nor adds a feature
  perf:     A code change that improves performance
  test:     Adding missing tests or correcting existing tests
(Move up and down to reveal more choices)
```

#### 3. cz-conventional-changelog规范

cz-conventional-changelog格式：

```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```

参考[AngularJS's commit message convention](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)

##### 3.1 Type

提交类型，支持如下几种：

* `feat`: A new feature
* `fix`: A bug fix
* `docs`: Documentation only changes
* `style`: Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
* `refactor`: A code change that neither fixes a bug nor adds a feature
* `perf`: A code change that improves performance
* `test`: Adding missing or correcting existing tests
* `chore`: Changes to the build process or auxiliary tools and libraries such as documentation generation

##### 3.2 Scope

影响范围，例如表示层、控制层、数据层等，不同项目有不同的定义

> The scope could be anything specifying place of the commit change. For example $location, $browser, $compile, $rootScope, ngHref, ngClick, ngView, etc...
> You can use * when the change affects more than a single scope.

##### 3.3 Subject

一句简单的主题说明，格式如下：

* use the imperative, present tense: "change" not "changed" nor "changes"
* don't capitalize first letter
* no dot (.) at the end

##### 3.4 Body

比较详细的说明，可分多行，例如：

```
用户查询接口更新.
- 更改参数名'user_name'为'username'
- 修正用户不存在时空指针异常的bug
```

> Just as in the subject, use the imperative, present tense: "change" not "changed" nor "changes". The body should include the motivation for the change and contrast this with previous behavior.

##### 3.5 Footer

Footer主要用于两种情况：

* 不兼容变动：以`BREAKING CHANGE:`开头，其后是描述，例如：

```
BREAKING CHANGE: 用户查询接口参数名变化.
用户查询接口，参数名从'user_name'改为'username'.
```

* 关闭 Issue，例如：

```
fix #20
```

#### 4. changelog

使用工具`conventional-changelog-cli`从`commit message`生成`change log`

```
$ npm install -g conventional-changelog-cli
```

生成`change log`(增量)：

```
$ conventional-changelog -p angular -i CHANGELOG.md -s
```

生成`change log`(全量)：

```
$ conventional-changelog -p angular -i CHANGELOG.md -w -r 0
```

#### 5. 延伸

对于非`npm`工程，例如`java`工程，也不想增加一个`package.json`文件，如何使用本规范呢？

直接编写`cz-conventional-changelog`格式的`commit message`即可，例如：

```
feat(controller): update user query interface

用户查询接口更新.
- 更改参数名'user_name'为'username'
- 修正用户不存在时空指针异常的bug

BREAKING CHANGE: 用户查询接口参数名变化.
用户查询接口，参数名从'user_name'改为'username'.
fix #20
```

#### 6. 参考

* [Commitizen for contributors](https://github.com/commitizen/cz-cli)

* [conventional-changelog-cli](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog-cli)