---
layout: post
title: GraphQL on Android
summary: A Tutorial to GraphQL on Android With Apollo
featured-img: sleek
categories: [Android, GraphQL]
---

##  什么是 GraphQL
>	GraphQL is a query language for APIs and a runtime for fulfilling those queries with your existing data. 

如果你是一个移动开发者, 相信你对 `RESTful API` 一定不陌生. GraphQL 由 Facebook 在 2012 年开发并开源, 是一门 **查询语言**, 可作为 `RESTful API` 的替代方案. GraphQL 并不针对某个特定平台, Android, iOS, Web 等均可使用. 在语言实现上, JavaScript 版本的[实现](https://github.com/graphql/graphql-js)和 GraphQL 标准一同推出, 随后包括 [C#/.NET](https://graphql.org/code/#c-net), [GO](https://graphql.org/code/#go), [Java](https://graphql.org/code/#java), [PHP](https://graphql.org/code/#php), [Python](https://graphql.org/code/#python), [Scala](https://graphql.org/code/#scala), [Ruby](https://graphql.org/code/#ruby) 等也实现了标准.

访问 [http://graphql.org/](http://graphql.org/) 可以获取更新信息.

## 为什么要使用 GraphQL
### 传统的 RESTful 存在着一些不足

+ 扩展性较低，且容易造成数据冗余: 
		
	我们以 GitHub 的 [REST API v3](https://developer.github.com/v3/) 举例, 如果我想要获取某个用户的 followers, 应该调用 `GET /users/:username/followers` , 返回数据为
	
	```json
	[
	  {
	    "login": "octocat",
	    "id": 1,
	    "node_id": "MDQ6VXNlcjE=",
	    "avatar_url": "https://github.com/images/error/octocat_happy.gif",
	    "gravatar_id": "",
	    "url": "https://api.github.com/users/octocat",
	    "html_url": "https://github.com/octocat",
	    "followers_url": "https://api.github.com/users/octocat/followers",
	    "following_url": "https://api.github.com/users/octocat/following{/other_user}",
	    "gists_url": "https://api.github.com/users/octocat/gists{/gist_id}",
	    "starred_url": "https://api.github.com/users/octocat/starred{/owner}{/repo}",
	    "subscriptions_url": "https://api.github.com/users/octocat/subscriptions",
	    "organizations_url": "https://api.github.com/users/octocat/orgs",
	    "repos_url": "https://api.github.com/users/octocat/repos",
	    "events_url": "https://api.github.com/users/octocat/events{/privacy}",
	    "received_events_url": "https://api.github.com/users/octocat/received_events",
	    "type": "User",
	    "site_admin": false
	  }
	]
	```
	如果我们需要在页面上展示用户名, 则上述 API 实际上并不能满足我们的需求, 因为返回数据中并没有包含 `name` 字段, 仅有 `login` 字段. 另外返回数据中还包含一些页面中并不需要展示的数据如 `site_admin`, `node_id` 等 , 而这就增加了网络传输量. 
	
+ 多次请求的问题: 
	
	仍然以  GitHub 的 [REST API v3](https://developer.github.com/v3/) 举例, 我需要在一个页面上展示用户的信息包括用户名, 头像, email, 个性签名和其所属的 organizations , 由于 GitHub 并没有提供一个 API 可以直接获取以上所有内容，我们就需要用两次请求完成，`GET /users/:username` 和 `GET /repos/:owner/:repo` .

+ 响应结果不可预期:

	发出一个资源请求之后, 你并不知道响应结果是什么. 你可能会说, 查询文档就好了呀. 但是文档有可能是过期的, 并且, 程序员最讨厌两件事: 1. 别人的代码没有文档; 2. 给自己的代码写文档. 你可以直接发送一个请求查看响应结果, 但是你并不能保证覆盖了所有可能的情况.
	
+ 没有参数类型校验:

	RESTful 方案本身并没有对参数的类型作出规定, 造成安全隐患, 需要自行实现参数的校验.
	
### GraphQL 的一些优点

+ 所见即所得: 查询的返回结果就是输入的查询结构的精确映射, 这样就解决了 RESTful API 响应结果不可预期的问题.
	
	我们仍然以 GitHub 的 [GraphQL API v4](https://developer.github.com/v4/) 举例, 根据用户名查询用户的基本信息.
	
	```graphql
	user(login: String) {
	   avatarUrl
	   bio
	   email
	   id
	   location
	   login
	   name
	}
	```
	
	返回的数据格式为:
	
	```json
	{
	  "data": {
	    "user": {
	    	"avatarUrl": "xxx",
	    	"bio": "xxx",
			"email": "xxx",
			"id": "xxx",
			"location": "xxx",
			"login": "xxx",
			"name": "xxx"
	    }
	  }
	}
	```

+ 减少网络请求和数据冗余:
	
	使用 GraphQL 不会存在过度获取的情况. 由于客户端可以按需求选择所需要的数据字段, 可以减少数据传输的大小. 在 GraphQL 中, 如果设计的数据结构是从属的, 直接在查询语句中指定即可. 即使数据结构是独立的, 也可以在查询语句中指定上下文, 只需要一次网络请求就可以获得资源和子资源的数据 (例如上面提到的同时获取用户的信息包括用户名, 头像, email, 个性签名和其所属的 organizations). 
	
+ 文档总是最新: 

	GraphQL会把schema定义和相关的注释生成可视化的文档, 从而使得代码的变更, 直接就反映到最新的文档上, 避免RESTful中手工维护可能会造成代码, 文档不一致的问题. 

+ 参数类型强校验: 

	GraphQL提供了强类型的schema机制, 从而天然确保了参数类型的合法性. 
	
### GraphQL 的一些不足
+ 查询的复杂性:
	
	对于一次网络请求, 客户端并不知道后端的具体实现, 无论是 RESTful 还是 GraphQL, 不同的资源和字段仍然需要从一个数据源获取. 因此, 当一个查询需要很多层的嵌套字段时, 很有可能造成性能瓶颈. 

+ 缓存:

	在 RESTful 中, 缓存十分简单, 但是在 GraphQL 中实现会变得极其复杂. 在 RESTful 中我们通过 URL 访问资源, 因此可以在资源级别实现缓存, 因为资源使用 URL 作为其标识符. 在 GraphQL 中就复杂了, 因为即便它操作的是同一个实体, 每个查询都各不相同. 不过很多基于 GraphQL 的类库都提供了开箱即用的缓存机制 (后面将要提到的 [apollo-android](https://github.com/apollographql/apollo-android) 也有提供). 

+ RateLimit:

	在  RESTful 中, RateLimit 可以简单理解为单位时间内能够请求的资源的数量. 使用 RateLimit 可以防止非预期的请求造成后端系统压力过大而被拖垮. 但是在 GraphQL 中很难做到这一点, 因为任何的请求都可以是廉价或者昂贵的. 
	
## GraphQL on Android
在 	Android 平台上主要使用的是 [apollo-android](https://github.com/apollographql/apollo-android)： 

> Apollo-Android is a GraphQL compliant client that generates Java models from standard GraphQL queries.

### 前期准备
1. 安装 Apollo Codegen
	
	Apollo Codegen 是一个生成 API 代码或基于 GraphQL schema 和查询文档注解的工具. 
	
	```shell
	npm install -g apollo-codegen
	```
	
	注意 **`-g`** 一定要加上, 表示全局安装. 如果你没有 npm, 则需要安装 [Node.js](https://nodejs.org/en/). 
	
2. 下载 schema.json
	
	Apollo 生成代码时需要 `schema.json` 文件. Schema.json 是用来描述你的 GraphQL API, 所有字段和输入参数等信息的文件. 我们以 Github 为例, 首先我们需要一个 access token, 用于获取 schema.json 文件:
	
	1. 在 GitHub 网站上点击头像并打开 setting, 选择 `Developer settings` -> `Personal access tokens` -> `Generate new token`, 根据需要选择 scope . 
	2. access token 准备好后, 使用 apollo-codegen 下载 schema.json 文件:
	
		```shell
		apollo-codegen download-schema https://api.github.com/graphql --output schema.json --header "Authorization: Bearer <access-token>"
		```
	
		大概 1 MB 的 json 文件会被下载到命令执行的目录. 

3. 创建 Android 项目并添加网络权限:
	
	```xml
	<uses-permission android:name="android.permission.INTERNET" />
	```

4. 在 **项目** 的 `build.gradle` 文件中添加依赖以使用 Apollo 的 Gradle 插件:

	```gradle
	buildscript {
	  repositories {
	    jcenter()
	  }
	  dependencies {
	    classpath 'com.apollographql.apollo:apollo-gradle-plugin:x.y.z'
	  }
	}
	
	repositories {
	    jcenter()
	}
	```

5. 在 **app** module 的 `build.gradle` 文件中添加:

	```gradle
	apply plugin: 'com.apollographql.android'
	
	dependencies {
		implementation 'com.apollographql.apollo:apollo-runtime:x.y.z'
		// 可选, Cached Response
		// implementation 'com.apollographql.apollo:apollo-http-cache:x.y.z'
		// 可选, RxJava1 支持
		// implementation 'com.apollographql.apollo:apollo-rx-support:x.y.z'
		// 可选, RxJava2 支持
		// implementation 'com.apollographql.apollo:apollo-rx2-support:x.y.z'
	}
	```

	需要注意的是 Android plugin 必须在 Apollo plugin 之前应用. 另外如果你的项目中使用了 Kotlin, Apollo plugin 需要应用在 Kotlin plugin 之前. 
	
6. 在 `src/main` 下创建一个与 `java/res` 文件夹相同级别的文件夹, 可以任意命名, 我命名为 `graphql` 并创建了子文件夹, 将下载好的 `schema.json` 文件复制到这个文件夹.

	<div align="center">
		<img src="https://ws3.sinaimg.cn/large/006tNbRwgy1fyspydk2vxj306o08lmyb.jpg" />
	</div>

	然后在相同目录下创建一个 `.graphql` 为后缀的文件, 在文件内创建一个 GraphQL 查询: 
	
	```graphql
	query User($login: String!) {
	  user(login: $login) {
	   avatarUrl
	     bio
	     bioHTML
	     company
	     companyHTML
	     createdAt
	     databaseId
	     email
	     id
	     isBountyHunter
	     isCampusExpert
	     isDeveloperProgramMember
	     isHireable
	     isEmployee
	     isSiteAdmin
	     isViewer
	     location
	     login
	     name
	     resourcePath
	     updatedAt
	     url
	     viewerCanFollow
	     viewerIsFollowing
	     websiteUrl
	     repositories {
	       totalCount
	     }
	     followers {
	       totalCount
	     }
	      following {
	       totalCount
	     }
	     starredRepositories {
	       totalCount
	     }
	  }
	}
	```
	
	手写 GraphQL 查询还是有点难度的, 这里推荐使用 GitHub GraphQL API Explorer: https://developer.github.com/v4/explorer/ . 使用 Explorer 还可以直接查询文档和执行查询. 
	
	![](https://ws2.sinaimg.cn/large/006tNbRwgy1fysqm9z3plj31m50u0k8y.jpg)
	
7. rebuild 项目, apollo 会生成对应的代码. build 完成后, 我们可以在 `app/build/generated/source/apollo` 文件夹下查看生成的文件. 

### Hello World
Apollo 使用 [OkHttp](https://github.com/square/okhttp) 作为底层的网络客户端, 所以我们可以使用 headers, interceptors, authenticator 等. 

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(HttpLoggingInterceptor().setLevel(HttpLoggingInterceptor.Level.BODY))
    .authenticator { _, response ->
        response.request()
            .newBuilder()
            .addHeader("Authorization", "Bearer <YOUR ACCESS TOKEN>")
            .build()
    }
    .build()
```

创建 ApolloClient. ApolloClient 创建好之后, 搜索的网络请求都可以使用同一个 client .

```kotlin
val apolloClient = ApolloClient.builder()
    .okHttpClient(okHttpClient)
    .serverUrl("https://api.github.com/graphql")
    .build()
```

发起请求:

```kotlin
val call = apolloClient
    .query(UserQuery.builder().login(login).build())
    .httpCachePolicy(HttpCachePolicy.CACHE_FIRST)
    .watcher()
val disposable = Rx2Apollo.from(call)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ data ->
        Timber.d("data: ${data.data()}")
    }, {
        Timber.e(it, "disposable error: ${it.message}")
    })
```

Apollo 支持普通的回调和 RxJava. 如果使用普通的回调:

```kotlin
apolloClient
    .query(UserQuery.builder().login(login).build())
    .httpCachePolicy(HttpCachePolicy.CACHE_FIRST)
    .enqueue(object : ApolloCall.Callback<UserQuery.Data>() {

        override fun onFailure(e: ApolloException) {
            Timber.e(e, "disposable error: ${e.message}")
        }

        override fun onResponse(data: Response<UserQuery.Data>) {
            Timber.d("data: ${data.data()}")
        }

    })
```

## 写在最后
apollo-android 已经发布了 1.0.0 的 alpha 版本, 相信很快就会推出正式版. 如果你还没有了解过 GraphQL, 并且正好又在处理复杂的 RESTful API, 那么不妨试试 GraphQL 简化应用.

## 参考
+ https://qiita.com/ssoejima/items/0ae0cb97aabfcac11c6a
+ https://juejin.im/post/5a44a7876fb9a044fc450b13
+ https://developer.github.com/v4/
+ https://github.com/apollographql
+ https://segmentfault.com/a/1190000012878342