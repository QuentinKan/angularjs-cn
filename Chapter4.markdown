#分析一个AngularJS应用程序

在第2章中, 我们已经讨论了一些AngularJS常用的功能, 然后在第3章讨论了该如何结构化开发应用程序. 现在, 我们不再继续深单个的技术点, 第4章将着眼于一个小的, 实际的应用程序进行讲解. 我们将从一个实际有效的应用程序中感受一下我们之前已经讨论过的(示例)所有的部分.

我们将每次介绍一部分, 然后讨论其中有趣和关联的部分, 而不是讨论完整应用程序的前端和核心, 最后在本章的后面我们会慢慢简历这个完整的应用程序.

##应用程序

Guthub是一个简单的食谱管理应用, 我们设计它用于存储我们超级没味的食谱, 同时展示AngularJS应用程序的各个不同的部分. 这个应用程序包含以下内容:

+ 一个两栏的布局
+ 在左侧有一个导航栏
+ 允许你创建新的食谱
+ 允许你浏览现有的食谱列表

主视图在左侧, 其变化依赖于URL, 或者食谱列表, 或者单个食谱的详情, 或者可添加新食谱的编辑功能, 或者编辑现有的食谱. 我们可以在图4-1中看到这个应用的一个截图:

![Guthub](figure/4-1.png)

Figure 4-1. Guthub: A simple recipe management application

这个完整的应用程序可以在我们的Github中的`chapter4/guthub`中得到.

##模型, 控制器和模板之间的关系

在我们深入应用程序之前, 让我们来花一两段文字来讨论以下如何将标题中的者三部分在应用程序中组织在一起工作, 同时来思考一下其中的每一部分.

`model`(模型)是真理. 只需要重复这句话几次. 整个应用程序显示什么, 如何显示在视图中, 保存什么, 等等一切都会受模型的影响. 因此你要额外花一些时间来思考你的模型, 对象的属性将是什么, 以及你打算如何从服务器获取并保存它. 视图将通过数据绑定的方式自动更新, 所以我们的焦点应该集中在模型上.

`controller`保存业务逻辑: 如何获取模型, 执行什么样的操作, 视图需要从模型中获取什么样的信息, 以及如何将模型转换为你所想要的. 验证职责, 使用调用服务器, 引导你的视图使用正确的数据, 大多数情况下所有的这些事情都属于控制器来处理.

最后, `template`代表你的模型将如何显示, 以及用户将如何与你的应用程序交互. 它主要约束以下几点:

+ 显示模型
+ 定义用户可以与你的应用程序交互的方式(点击, 文本输入等等)
+ 应用程序的样式, 并确定何时以及如何显示一些元素(显示或隐藏, hover等等)
+ 过滤和格式化数据(包括输入和输出)

要意识到在Angular中的模型-视图-控制器涉及模式中模板并不是必要的部分. 相关, 视图是模板获取执行被编译后的版本. 它是一个模板和模型的组合.

任何类型的业务逻辑和行为都不应该进入模板中; 这些信息应该被限制在控制器中. 保持模板的简单可以适当的分离关注点, 并且可以确保你只使用单元测试的情况下就能够测试大多数的代码. 而模板必须使用场景测试的方式来测试.

但是, 你可能会问, 在哪里操作DOM呢? DOM操作并不会真正进入到控制器和模板中. 它会存在于Angular的指令中(有时候也可以通过服务来处理, 这样可以避免重复的DOM操作代码). 我们会在我们的Github的示例文件中涵盖一个这样的例子.

废话少说, 让我们来深入探讨一下它们.

##模型

对于应用程序我们要保持模型非常简单. 这一有一个菜谱. 在整个完整的应用程序中, 它们是一个唯一的模型. 它是构建一切的基础.

每个菜谱都有下面的属性:

+ 一个用于保存到服务器的ID
+ 一个名称
+ 一个简短的描述
+ 一个烹饪说明
+ 是否是一个特色的菜谱
+ 一个成份数组, 每个成分的数量, 单位和名称

就是这样. 非常简单. 应用程序的中一切都基于这个简单的模型. 下面是一个让你食用的示例菜谱(如图4-1一样):

	{
		'id': '1',
		'title': 'Cookies',
		'description': 'Delicious. crisp on the outside, chewy' +
			' on the outside, oozing with chocolatey goodness' +
			' cookies. The best kind',
		'ingredients': [
			{
				'amount': '1',
				'amountUnits': 'packet',
				'ingredientName': 'Chips Ahoy'
			}
		],
		'instructions': '1. Go buy a packet of Chips Ahoy\n'+
			'2. Heat it up in an oven\n' +
			'3. Enjoy warm cookies\n' +
			'4. Learn how to bake cookies from somewhere else'
	}

下面我们将会看到如何基于这个简单的模型构建更复杂的UI特性.

##控制器, 指令和服务

现在我们终于可以得到这个让我们牙齿都咬到肉里面去的美食应用程序了. 首先, 我们来看看代码中的指令和服务, 以及讨论以下它们都是做什么的, 然后我们我们会看看这个应用程序需要的多个控制器.

###服务

	//this file is app/scripts/services/services.js

	var services = angular.module('guthub.services', ['ngResource']);

	services.factory('Recipe', ['$resource', function(){
		return $resource('/recipes/:id', {id: '@id'});
	}]);

	services.factory('MultiRecipeLoader', ['Recipe', '$q', function(Recipe, q){
		return function(){
			var delay = $.defer();
			Recipe.query(function(recipes){
				delay.resolve(recipes);
			}, function(){
				delay.reject('Unable to fetch recipes');
			});
			return delay.promise;
		};
	}]);

	services.factory('RecipeLoader', ['Recipe', '$route', '$q', function(Recipe, $route, $q){
		return function(){
			var delay = $q.defer();
			Recipe.get({id: $route.current.params.recipeId}, function(recipe){
				delay.resolve(recipe);
			}, function(){
				delay.reject('Unable to fetch recipe' + $route.current.params.recipeId);
			});
			return delay.promise;
		};
	}]);

首先让我们来看看我们的服务. 在33页的"使用模块组织依赖"小节中已经涉及到了服务相关的知识. 这里, 我们将会更深一点挖掘服务相关的信息.

在这个文件中, 我们实例化了三个AngularJS服务.

有一个菜谱服务, 它返回我们所调用的Angular Resource. 这些是RESETful资源, 它指向一个RESTful服务器. Angular Resource封装了低层的`$http`服务, 因此你可以在你的代码中只处理对象.

注意单独的那行代码 - `return $resource` - (当然, 依赖于`guthub.services`模型), 现在我们可以将`recipe`作为参数传递给任意的控制器中, 它将会注入到控制器中. 此外, 每个菜谱对象都内置的有以下几个方法:

+ Recipe.get()
+ Recipe.save()
+ Recipe.query()
+ Recipe.remove()
+ Recipe.delete()

> 如果你使用了`Recipe.delete`方法, 并且希望你的应用程序工作在IE中, 你应该像这样调用它: `Recipe[delete]()`. 这是因为在IE中`delete`是一个关键字.

对于上面的方法, 所有的查询众多都在一个单独的菜谱中进行; 默认情况下`query()`返回一个菜谱数组.

`return $resource`这行代码用于声明资源 - 也给了我们一些好东西:

1. 注意: URL中的id是指定的RESTFul资源. 它基本上是说, 当你进行任何查询时(`Recipe.get()`), 如果你给它传递一个id字段, 那么这个字段的值将被添加早URL的尾部.

也就是说, 调用`Recipe.get{id: 15})将会请求/recipe/15.

2. 那第二个对象是什么呢? {id: @id}吗? 是的, 正如他们所说的, 一行代码可能需要一千行解释, 那么让我们举一个简单的例子.

比方说我们有一个recipe对象, 其中存储了必要的信息, 并且包含一个id.

然后, 我们只需要像下面这样做就可以保存它:

	//Assuming existingRecipeObj has all the necessary fields,
	//including id(say 13)
	var recipe = new Recipe(existingRecipeObj);
	recipe.$save();

这将会触发一个POST请求到`/recipe/13`.

`@id`用于告诉它, 这里的id字段取自它的对象中同时用于作为id参数. 这是一个附加的便利操作, 可以节省几行代码.

在`apps/scripts/services/services.js`中有两个其他的服务. 它们两个都是加载器(Loaders); 一个用于加载单独的食谱(RecipeLoader), 另一个用于加载所有的食谱(MultiRecipeLoader). 这在我们连接到我们的路由时使用. 在核心上, 它们两个表现得非常相似. 这两个服务如下:

1. 创建一个`$q`延迟(deferred)对象(它们是AngularJS的promises, 用于链接异步函数).
2. 创建一个服务器调用.
3. 在服务器返回值时resolve延迟对象.
4. 通过使用AngularJS的路由机制返回promise.

> **AngularJS中的Promises**
>
> 一个promise就是一个在未来某个时刻处理返回对象或者填充它的接口(基本上都是异步行为). 从核心上讲, 一个promise就是一个带有`then()`函数(方法)的对象.
>
>让我们使用一个例子来展示它的优势, 假设我们需要获取一个用户的当前配置:

	var currentProfile = null;
	var username = 'something';

	fetchServerConfig(function(){
		fetchUserProfiles(serverConfig.USER_PROFILES, username, 
			function(profiles){
				currentProfile = profiles.currentProfile;	
		});	
	});

> 对于这种做法这里有一些问题:
>
> 1. 对于最后产生的代码, 缩进是一个噩梦, 特别是如果你要链接多个调用时.
> 
> 2. 在回调和函数之间错误报告的功能有丢失的倾向, 除非你在每一步中处理它们.
>
> 3. 对于你想使用`currentProfile`做什么, 你必须在内层回调中封装其逻辑, 无论是直接的方式还是使用一个单独分离的函数.
>
> Promises解决了这些问题. 在我们进入它是如何解决这些问题之前, 先让我们来看看一个使用promise对同一问题的实现.

	var currentProfile = fetchServerConfig().then(function(serverConfig){
		return fetchUserProfiles(serverConfig.USER_PROFILES, username);
	}).then(function{
		return profiles.currentProfile;
	}, function(error){
		// Handle errors in either fetchServerConfig or
		// fetchUserProfile here
	});

> 注意其优势:
>
> 1. 你可以链接函数调用, 因此你不会产生缩进带来的噩梦.
>
> 2. 你可以放心前一个函数调用会在下一个函数调用之前完成.
>
> 3. 每一个`then()`调用都要两个参数(这两个参数都是函数). 第一个参数是成功的操作的回调函数, 第二个参数是错误处理的函数.
> 4. 在链接中发生错误的情况下, 错误信息会通过错误处理器传播到应用程序的其他部分. 因此, 任何回调函数的错误都可以在尾部被处理.
>
> 你会问, 那什么是`resolve`和`reject`呢? 是的, `deferred`在AngularJS中是一种创建promises的方式. 调用`resolve`来满足promise(调用成功时的处理函数), 同时调用`reject`来处理promise在调用错误处理器时的事情.

当我们链接到路由时, 我们会再次回到这里.

###指令

我们现在可以转移到即将用在我们应用程序的指令上来. 在这个应用程序中将有两个指令:

`butterbar`

这个指令将在路由发生改变并且页面仍然还在加载信息时处理显示和隐藏任务. 它将连接路由变化机制, 基于页面的状态来自动控制显示或者隐藏是否使用哪个标签.

`focus`

这个`focus`指令用于确保指定的文本域(或者元素)拥有焦点.

让我们来看一下代码:

	// This file is app/scripts/directives/directives.js

	var directive = angular.module('guthub.directives', []);

	directives.directive('butterbar', ['$rootScope', function($rootScope){
		return {
			link: function(scope, element attrs){
				element.addClass('hide');

				$rootScope.$on('$routeChangeStart', function(){
					element.removeClass('hide');
				});

				$routeScope.$on('$routeChangeSuccess', function(){
					element.addClass('hide');
				});
			}
		};
	}]);

	directives.dirctive('focus',function(){
		return {
			link: function(scope, element, attrs){
				element[0].focus();
			}
		};
	});

上面所述的指令返回一个对象带有一个单一的属性, link. 我们将在第六章深入讨论你可以如何创建你自己的指令, 但是现在, 你应该知道下面的所有事情:

1. 指令通过两个步骤处理. 在第一步中(编译阶段), 所有的指令都被附加到一个被查找到的DOM元素上, 然后处理它. 任何DOM操作否发生在编译阶段(步骤中). 在这个阶段结束时, 生成一个连接函数.

2. 在第二步中, 连接阶段(我们之前使用的阶段), 生成前面的DOM模板并连接到作用域. 同样的, 任何观察者或者监听器都要根据需要添加, 在作用域和元素之前返回一个活动(双向)绑定. 因此, 任何关联到作用域的事情都发生在连接阶段.

那么在我们指令中发生了什么呢? 让我们去看一看, 好吗?

`butterbar`指令可以像下面这样使用:

	<div butterbar>My loading text...</div>

它基于前面隐藏的元素, 然后添加两个监听器到根作用域中. 当每次一个路由开始改变时, 它就显示该元素(通过改变它的class[className]), 每次路由成功改变并完成时, 它再一次隐藏`butterbar`.

另一个有趣的事情是注意我们是如何注入`$rootScopr`到指令中的. 所有的指令都直接挂接到AngularJS的依赖注入系统, 因此你可以注入你的服务和其他任何需要的东西到其中.

最后需要注意的是处理元素的API. 使用jQuery的人会很高兴, 因为他直到使用的是一个类似jQuery的语法(addClass, removeClass). AngularJS实现了一个调用jQuery的自己, 因此, 对于任何AngularJS项目来说, jQuery都是一个可选的依赖项. 如果你最终在你的项目中使用完整的jQuery库, 你应该直到它使用的是它自己内置的jQlite实现.

第二个指令(focus)简单得多. 它只是在当前元素上调用`focus()`方法. 你可以用过在任何input元素上添加`focus`属性来调用它, 就像这样:

	<input type="text" focus></input>

当页面加载时, 元素将立即获得焦点.

###控制器

随着指令和服务的覆盖, 我们终于可以进入控制器部分了, 我们有五个控制器. 所有的这些控制器都在一个单独的文件中(`app/scripts/controllers/controllers.js`), 但是我们会一个个来了解它们. 让我们来看第一个控制器, 这是一个列表控制器, 负责显示系统中所有的食谱列表.

	app.controller('ListCtrl', ['scope', 'recipes', function($scope, recipes){
		$scope.recipes = recipes;
	}]);

注意列表控制器中最重要的一个事情: 在这个控制器中, 它并没有连接到服务器和获取是食谱. 相反, 它只是使用已经取得的食谱列表. 你可能不知道它是如何工作的. 你可能会使用路由一节来回答, 因为它有一个我们之前看到`MultiRecipeLoader`. 你只需要在脑海里记住它.

在我们提到的列表控制器下, 其他的控制器都与之非常相似, 但我们仍然会逐步指出它们有趣的地方:

	app.controller('ViewCtrl', ['$scope', '$location', 'recipe', function($scope, $location, recipe){
			$scope.recipe = recipe;

			$scope.edit = function(){
				$location.path('/edit/' + recipe.id);
			};
	}]);

这个视图控制器中有趣的部分是其编辑函数公开在作用域中. 而不是显示和隐藏字段或者其他类似的东西, 这个控制器依赖于AngularJS来处理繁重的任务(你应该这么做)! 这个编辑函数简单的改变URL并跳转到编辑食谱的部分, 你可以看见, AngularJS并没有处理剩下的工作. AngularJS识别已经改变的URL并加载响应的视图(这是与我们编辑模式中相同的食谱部分). 来看一看!

接下来, 让我们来看看编辑控制器:

	app.controller('EditCtrl', ['$scope', '$location', 'recipe', function($scope, $location, recipe){
		$scope.recipe = recipe;

		$scope.save = function(){
			$scope.recipe.$save(function(recipe){
				$location.path('/view/' + recipe.id);
			});
		};

		$scope.remove = function(){
			delete $scope.recipe;
			$location.path('/');
		};
	}]);

那么在这个暴露在作用域中的编辑控制器中新的`save`和`remove`方法有什么.

那么你希望作用域内的`save`函数做什么. 它保存当前食谱, 并且一旦保存好, 它就在屏幕中将用户重定向到相同的食谱. 回调函数是非常有用的, 一旦你完成任务的情况下执行或者处理一些行为.

有两种方式可以在这里保存食谱. 一种是如代码所示, 通过执行$scope.recipe.$save()方法. 这只是可能, 因为`recipe`十一个通过开头部分的RecipeLoader返回的资源对象.

另外, 你可以像这样来保存食谱:

	Recipe.save(recipe);

`remove`函数也是很简单的, 在这里它会从作用域中移除食谱, 同时将用户重定向到主菜单页. 请注意, 它并没有真正的从我们的服务器上删除它, 尽管它很再做出额外的调用.

接下来, 我们来看看New控制器:

	app.controller('NewCtrl', ['$scope', '$location', 'Recipe', function($scope, $location, Recipe){
		$scope.recipe = new Recipe({
			ingredents: [{}]
		});

		$scope.save = function(){
			$scope.recipe.$save(function(recipe){
				$location.path('/view/' + recipe.id);
			});
		};
	}]);

New控制器几乎与Edit控制器完全一样. 实际上, 你可以结合两个控制器作为一个单一的控制器来做一个练习. 唯一的主要不同是New控制器会在第一步创建一个新的食谱(这也是一个资源, 因此它也有一个`save`函数).其他的一切都保持不变.

最后, 我们还有一个Ingredients控制器. 这是一个特殊的控制器, 在我们深入了解它为什么或者如何特殊之前, 先来看一看它:

	app.controller('Ingredients', ['$scope', function($scope){
		$scope.addIngredients = function(){
			var ingredients = $scope.recipe.ingredients;
			ingredients[ingredients.length] = {};
		};

		$scope.removeIngredient = function(index) {
			$scope.recipe.ingredients.splice(index, 1);
		};
	}]);

到目前为止, 我们看到的所有其他控制器斗鱼UI视图上的相关部分联系着. 但是这个Ingredients控制器是特殊的. 它是一个子控制器, 用于在编辑页面封装特定的恭喜而不需要在外层(父级)来处理. 有趣的是要注意, 由于它是一个字控制器, 继承自作用域中的父控制器(在这里就是Edit/New控制器). 因此, 它可以访问来自父控制器的`$scope.recipe`.

这个控制器本身并没有什么有趣或者独特的地方. 它只是添加一个新的成份到现有的食谱成份数组中, 或者从食谱的成份列表中删除一个特定的成份.

那么现在, 我们就来完成最后的控制器. 唯一的JavaScript代码块展示了如何设置路由:

	// This file is app/scripts/controllers/controllers.js

	var app = angular.module('guthub', ['guthub.directives', 'guthub.services']);

	app.config(['$routeProvider', function($routeProvider){
		$routeProvider.
			when('/', {
				controller: 'ListCtrl',
				resolve: {
					recipes: function(MultiRecipeLoader) {
						return MultiRecipeLoader();
					}
				},
				templateUrl: '/views/list.html'
			}).when('/edit/:recipeId', {
				controller: 'EditCtrl',
				resolve: {
					recipe: function(RecipeLoader){
						return RecipeLoader();
					}
				},
				templateUrl: '/views/recipeForm.html'
			}).when('/view/:recipeId', {
				controller: 'ViewCtrl',
				resolve: {
					recipe: function(RecipeLoader){
						return RecipeLoader();
					}
				},
				templateUrl: '/views/viewRecipe.html'
			}).when('/new', {
					controller: 'NewCtrl',
					templateUrl: '/views/recipeForm.html'
			}).otherwise({redirectTo: '/'});
	}]);

正如我们所承诺的, 我们终于到了解析函数使用的地方. 前面的代码设置Guthub AngularJS模块, 路由以及参与应用程序的模板.

它挂接到我们已经创建的指令和服务上, 然后指定不同的路由指向应用程序的不同地方.

对于每个路由, 我们指定了URL, 它备份控制器, 加载模板, 以及最后(可选的)提供了一个`resolve`对象.

这个`resolve`对象会告诉AngularJS, 每个resolve键需要在确定路由正确时才能显示给用户. 对我们来说, 我们想要加载所有的食谱或者个别的配方, 并确保在显示页面之前服务器要响应我们. 因此, 我们要告诉路由提供者我们的食谱, 然后再告诉他如何来取得它.

这个环节中我们在第一节中定义了两个服务, 分别时`MultiRecipeLoader`和`RecipeLoader`. 如果`resolve`函数返回一个AngularJS promise, 然后AngularJS会智能在获得它之前等待promise解决问题. 这意味着它将会等待到服务器响应.

然后将返回结果传递到构造函数中作为参数(与来自对象字段的参数的名称一起作为参数).

最后, `otherwise`函数表示当没有路由匹配时重定向到默认URL.

> 你可能会注意到Edit和New控制器两个路由通向同一个模板URL, `views/recipeForm.html`. 这里发生了什么呢? 我们复用了编辑模板. 依赖于相关的控制器, 将不同的元素显示在编辑食谱模板中.

完成这些工作之后, 现在我们可以聚焦到模板部分, 来看看控制器如何挂接到它们之上, 以及如何管理现实给最终用户的内容.

