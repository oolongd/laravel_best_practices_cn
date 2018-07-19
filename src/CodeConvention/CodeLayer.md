### 应用代码分层
  
我们在写应用里的代码时根据代码负责的不同任务讲其分为四大块`Controller`, `Service`, `Model`, `View`。

- `Model` 数据模型， 数据模型面向的是数据层，在这里我们只关心数据表的问题，在Model中应该只定义数据与对象映射相关的属性和方法如：表名、主键名、是否让laravel管理时间字段等属性，以及模型关联、查询作用域等方法。其他与数据表无关的业务逻辑都不应出现在这里。
- `Service` 业务逻辑层, 业务逻辑层负责对单一业务逻辑进行抽象，在Service中我们可以通过对一个或多个Model进行操作、或者调用第三方API来完成业务逻辑。
- `Controller` 控制器，面向的对象是一个完整的页面或者是接口，其主要职责是作为接收请求和发送响应的中控程序，通过调度项目中的Service、 Model等来完成请求、组合响应数据，并通过页面响应或者接口数据的形式将响应返回给客户端。
- `View` 视图， 负责渲染HTML响应，使用Laravel自带的blade模版引擎，并且应该尽量减少PHP代码。

总结：所以如果一个请求对应的业务逻辑相当复杂，我们可以通过Controller方法调用多个Service方法(单一逻辑)来完成这个复杂的逻辑，在Service方法中我们通过多个Model操作来实现更新、获取数据。通过这种原则来对复杂逻辑进行解耦。

我们通过看两个例子来更好的理解代码如何分层：

#### 代码示例 

##### Example 1: 应该在哪里写SQL操作 

如果一个查询在项目中只会用到一次，那么我们直接在Controller方法里写这个查询就很好， 举例来说你想查出所有的管理员用户:

```php
$admins = User::where('type', 'admin')->get();
```   

现在假设你需要在不止一个Controller方法中用到这个查询， 你可以在`UserService`类里包装对`User`模型的访问:

```php
class UserService
{
    public function getAlladmins()
    {
        return User::where('type', 'admin')->get();
    }
}
```

现在你可以在用到`UserService`的`Controller`中通过依赖注入`UserService`, 然后通过这个Service方法来获取所有管理员用户:
   
```php
//Controller
public function __construct(UserService $userService)
{
    $this->userService = $userService;
}
//Controller action
public function index()
{
    $admins = $this->userService->getAllAdmins();
}
```

最后，假设你还需要一个查询来计算管理员的数量， 可以在把这个查询封装到UserService的一个方法里：

```php
public function getAdminNum()
{
    return User::where('type', 'admin')->count();
}
```   	

这样写是OK的，但是你可能已经注意到了`User::where('type', 'admin')`这个片段在`getAllAdmins`这个查询里也有用到，所以我们可以用查询作用域来改进它（查询作用域也会让你的代码可读性变得更高）:
   
```php
//UserModel
public function scopeAdmins($query)
{
   return $query->where('type', 'admin');
}
```
       
然后在UserService里我们可以向下面这样重写它的查询:
   		
```php
//UserService
public function getAllAdmins()
{
    return User:admins()->get();
}

public function getAdminNum()
{
    return User::admins()->count();
}
```
   		
就像上面说的那样在Model中我们应该撇开业务逻辑，面向数据表进行抽象，只定义表相关的属性、模型关联和查询作用域， 具体怎么应用Model中定义的这些内容， 那是Controller层和Service层要关心的事情。
   	
##### Example2: 通过Service类提高代码复用率  

我们在做CMS系统管理内容时， 通常都会涉及到文章的上下线，置顶等操作。假设在系统中我们会用两个数据表分别存储文章和置顶, 在项目中对应两个Model: Article和TopArticle。

假设我们需要实现下线文章的功能，如果文章是置顶文章，还要把文章取消置顶，我们在ArticleService里包装这个操作:
   
```php
public function setArticleOffline(Article $article)
{
   if ($article->is_top == 1) {//如果是置顶文章需要将文章取消置顶
       $this->cancelTopArticle($article);
   }
   $article->status = Article::STATUS_OFFLINE;
   $article->offline_time = date('Y-m-d H:i:s');
   $article->save();

   return true;
}

/**
* 取消文章的置顶
* @param \App\Models\Article $article
*/
public function cancelTopArticle($article)
{
    if (TopArticle::specificArticle($article->id)->count()) {
        //删除置顶表里的记录(待上线的置顶文章上线后置顶表中才能有相应的记录)
        TopArticle::specificArticle($article->id)->delete();
    }
    //将文章的is_top字段置为0
    $article->is_top = 0;
    $article->save();

    return true;
}
```
    
在Controller的Action里我们只需要负责验证接收来的参数，查询文章对象传递给Service来做下线操作， 这里之所以把取消置顶也单独提取出来是因为通常CMS系统里还有单独的文章取消置顶功能。
    
```php
//ArticleController 
public function setArticleOff(Request $request, ArticleService $artService)
{
    ...//表单验证
    
    $article = Article::find($request->get('article_id'));
    $this->articleService->setArticleOffline($article);
    
    ...
}
```
   
除了上面讲到的主动下线文章的功能， 一般的文章系统里还会有定时上下线文章的功能， 因为我们在ArticleService里包装了下线文章的功能， 所以在设置定时任务时就可以再次调用setArticleOffline方法来完成文章下线的功能:
    
```php
//App\Console\Commands\ArticleCommand
public function __construct(ArticleService $articleService)
{
   $this->articleService = $articleService;
   parent::__construct();
}

public function handle()
{
   $operation = $this->argument('operation');
   switch ($operation) {
       case 'offline':
        $this->setOffline();
        break;
    default:
        break;
}

public function setOffline()
{
   ......//前置条件查出要下线的文章
   $this>articleService>setArticleOffline($article);
   ......    
}
```
       
上面两个例子简单说明了我们把代码分层后每个层里应该写什么类型的程序，以及代码分层后在可读性、耦合性、可测试性、维护成本等方面带来的收益。接下来会有专门章节来说明各个部分的规范和常用的最佳实践。