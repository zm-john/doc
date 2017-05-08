# laravel 请求到响应分析
1. public\index.php 入口


2. bootstrap\app.php
实例化 Application（容器）
	1. 注册系统事件服务
	2. 注册系统路由服务


3. public/index.php
实例化：App\Http\Kernel
获取 request： `Illuminate\Http\Request::capture()`
调用 `App\Http\Kernel handle` 方法

4. 调用 `App\Http\Kernel` 的 `sendRequestThroughRouter`
5. 调用 `App\Http\Kernel` 的 `bootstrap` 方法
…

### 服务注册、启动
利用 App\Http\Kernel $bootstrappers 属性实例化`Illuminate\Foundation\Bootstrap\RegisterProviders` 并调用 bootstrap 方法，此时将操作转移到  `Illuminate\Foundation\Application` 的registerConfiguredProviders 方法。

该方法利用 `Illuminate\Foundation\ProviderRepository` 加载 config/app.php 中的 providers。

`App\Providers\RouteServiceProvider` 也就是在此时注册到了`Illuminate\Foundation\Application`。

利用 App\Http\Kernel $bootstrappers 属性实例化`Illuminate\Foundation\Bootstrap\BootProviders`  并调用 boot 方法，此时将操作转移到 `Illuminate\Foundation\Application` 的 boot 方法。依次调用 config/app.php 中的 providers 实例的 boot 方法

`App\Providers\RouteServiceProvider` 继承的 `Illuminate\Foundation\Support\Providers\RouteServiceProvider`  ，boot 方法中调用 loadRoutes 方法，
loadRoutes 调用子类的 map 方法，从而将 routes文件夹下的路由文件引入，通过 Route Facade 注册(get、post … 等)路由，此时路由已经生成

### 管道线 （Illuminate\Pipeline\Pipeline@then 分析）
```
$array = [
    "Barryvdh\Cors\HandleCors",
    "Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode",
    "Dingo\Api\Http\Middleware\Request"
];

$slice = function ($stack, $pipe) {
    return function ($passable) use ($stack, $pipe) {
        try {
            $slice = parent::getSlice();
            $callable = $slice($stack, $pipe);

            return $callable($passable);
        } catch (Exception $e) {
            return $this->handleException($passable, $e);
        } catch (Throwable $e) {
            return $this->handleException($passable, new FatalThrowableError($e));
        }
    };
};

$first = function ($passable) use ($destination) {
    try {
        return $destination($passable);
    } catch (Exception $e) {
        return $this->handleException($passable, $e);
    } catch (Throwable $e) {
        return $this->handleException($passable, new FatalThrowableError($e));
    }
};

$destination = function ($request) {
    $this->app->instance('request', $request);

    return $this->router->dispatch($request);
};

array_reduce($array, $slice, $first);
```

#### array_reduce 第一次执行：
```

$slice($first, "Barryvdh\Cors\HandleCors")
得到结果：
    $firststack = function ($passable) use ($stack, $pipe) {
        try {
            $slice = parent::getSlice();
            $callable = $slice($stack, $pipe);

            return $callable($passable);
        } catch (Exception $e) {
            return $this->handleException($passable, $e);
        } catch (Throwable $e) {
            return $this->handleException($passable, new FatalThrowableError($e));
        }
    };
    此时返回闭包中 use 的 $stack, $pipe 的值如下：
    $stack = $first;
    $pipe = "Barryvdh\Cors\HandleCors";

```

#### array_reduce 第二次执行：
```

$slice($firststack, "Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode")
得到结果：
    $secondstack = function ($passable) use ($stack, $pipe) {
        try {
            $slice = parent::getSlice();
            $callable = $slice($stack, $pipe);

            return $callable($passable);
        } catch (Exception $e) {
            return $this->handleException($passable, $e);
        } catch (Throwable $e) {
            return $this->handleException($passable, new FatalThrowableError($e));
        }
    };
    此时返回闭包中 use 的 $stack, $pipe 的值如下：
    $stack = $firststack;
    $pipe = "Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode";
```

#### array_reduce 第三次执行：
```

$slice($secondstack, "Dingo\Api\Http\Middleware\Request")
得到结果：
    $thirdstack = function ($passable) use ($stack, $pipe) {
        try {
            $slice = parent::getSlice();
            $callable = $slice($stack, $pipe);

            return $callable($passable);
        } catch (Exception $e) {
            return $this->handleException($passable, $e);
        } catch (Throwable $e) {
            return $this->handleException($passable, new FatalThrowableError($e));
        }
    };
    此时返回闭包中 use 的 $stack, $pipe 的值如下：
    $stack = $secondstack;
    $pipe = "Dingo\Api\Http\Middleware\Request";
```

#### array_reduce 执行结束返回 `$thirdstack 闭包`

#### 最后执行 `$thirdstack($request)`:
```

function ($passable) use ($stack, $pipe) {
    try {
        $slice = parent::getSlice();
        $callable = $slice($stack, $pipe);

        return $callable($passable);
    } catch (Exception $e) {
        return $this->handleException($passable, $e);
    } catch (Throwable $e) {
        return $this->handleException($passable, new FatalThrowableError($e));
    }
};

第一步：
    $slice = parent::getSlice(); // Illuminate\Pipeline\Pipeline@getSlice
    此时 $slice = function ($stack, $pipe) {
        return function ($passable) use ($stack, $pipe) {
            if ($pipe instanceof Closure) {
                return $pipe($passable, $stack);
            } elseif (! is_object($pipe)) {
                list($name, $parameters) = $this->parsePipeString($pipe);

                $pipe = $this->getContainer()->make($name);

                $parameters = array_merge([$passable, $stack], $parameters);
            } else {
                $parameters = [$passable, $stack];
            }

            return $pipe->{$this->method}(...$parameters);
        };
    };

第二步：
    $callable = $slice($stack, $pipe);
    此时 $callable = function ($passable) use ($stack, $pipe) {
            if ($pipe instanceof Closure) {
                return $pipe($passable, $stack);
            } elseif (! is_object($pipe)) {
                list($name, $parameters) = $this->parsePipeString($pipe);

                $pipe = $this->getContainer()->make($name);

                $parameters = array_merge([$passable, $stack], $parameters);
            } else {
                $parameters = [$passable, $stack];
            }

            return $pipe->{$this->method}(...$parameters);
        };
    };
    此时返回闭包中 use 的 $stack, $pipe 的值如下（array_reduce 第三次执行的返回闭包 use 的值）：
    $stack = $secondstack;
    $pipe = "Dingo\Api\Http\Middleware\Request";

第三步：
    $callable($passable);
    $stack = $secondstack;
    $pipe = "Dingo\Api\Http\Middleware\Request";
    结果：
        $pipe 不是一个对象因此对应的条件分支对应的操作：
            list($name, $parameters) = $this->parsePipeString($pipe);
            // $name = "Dingo\Api\Http\Middleware\Request"
            // $parameters = []
            $pipe = $this->getContainer()->make($name);
            // $pipe = app(Dingo\Api\Http\Middleware\Request::class)
            $parameters = array_merge([$passable, $stack], $parameters);
            // $paramters = [$passable, $stack];
            $pipe->{$this->method}(...$parameters)
            // $this->method = 'handle'
            // $pipe->{$this->method}(...$parameters) 相当于 $pipe->handle($passable, $stack)
            // 等价于 app(Dingo\Api\Http\Middleware\Request::class)->handle($passable, $stack)

            // handle 方法追踪
                app(Dingo\Api\Http\Middleware\Request::class)->handle($passable, $stack)
                    return $next($request) 相当于 return $stack($passable)
                $stack 此时就是 $secondstack
                相当于 app(Dingo\Api\Http\Middleware\Request::class)->handle($passable, $secondstack)
                    return $secondstack($passable)
                回头看看 $sencodestack 闭包
                    $sencodestack 执行的步骤参考 $thirdstack 执行
                    return app(Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode::class)->handle($passable, $firststack)
                同理 $firststack 执行
                    return app(Barryvdh\Cors\HandleCors::class)->handle($passable, $first)
                同理 $first 执行, 此时所有的中间件已经处理完毕，最后进行路由匹配，到达控制器
                    return $destination($passable)

            // handle 方法总结
                middleware -> middleware -> middleware -> ... -> middleware -> controller
                return middleware->handle(
                    return $middleware->handle(
                        return $middleware->hanlde(
                            return controller@method
                        )
                    )
                )
            // middleware handle 可以终结 $next($request), 直接返回 response
```

### 总结

请求到响应过程:  
实例化 `Application` 并加载基本的 `EventServiceProvider` 和 `RoutingServiceProvider`  

实例化 `App\Http\Kernel`  

provider 处理: 实例化 `config/app.php` 中的 `providers`, 先调用 `provider` 的 `register`方法, 在调用 `provider` 的 `boot`方法  

中间件处理: 实例化 `App\Http\Kernel` 中的 `middleware` 属性  

路由匹配: `Illuminate\Routing\Router` 的 `dispatch` 方法负责匹配路由并返回响应  

请求结束
