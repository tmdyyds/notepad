#### IoC容器核心解析(laravel容器)
```
class Container
{
    protected $bindings = [];

    public function bind($abstract, $concrete = null, $shared = false)
    {
        if (! $concrete instanceof Closure) {
            $concrete = $this->getClosure($abstract, $concrete);
        }
        $this->bindings[$abstract] = compact('concrete', 'shared');
    }

    public function getClosure($abstract, $concrete)
    {
        return function ($container) use ($abstract, $concrete) {
            // var_dump($this, $abstract, $concrete);exit;
            if ($abstract == $concrete) {
                return $container->build($concrete);
            }

            return $container->make($concrete);
        };
    }

    public function make($abstract)
    {
        //第一次 $app->make('t');这一步返回的是bind时的Closure函数
        //第二次 $container->make($concrete);调用
        //第三次 resolveClass调用
        $concrete = $this->getConcrete($abstract);
        if ($this->isBuildable($concrete, $abstract)) {
            $object = $this->build($concrete);
        } else {
            $object = $this->make($concrete);
        }

        return $object;
    }

    //通过反射解析类依赖
    public function build($concrete)
    {
        //匿名直接实例化
        if ($concrete instanceof Closure) {
            return $concrete($this);
        }

        $reflector = new ReflectionClass($concrete);
        if (! $reflector->isInstantiable()) {
            echo "[$concrete] 无法实例化";
        }

        $constructor = $reflector->getConstructor();
        if (is_null($constructor)) {
            return new $concrete;
        }

        $dependencies = $constructor->getParameters();
        $instances = $this->resolveDependencies($dependencies);
        return $reflector->newInstanceArgs($instances);
    }

    protected function resolveDependencies(array $dependencies)
    {
        $results = [];
        foreach ($dependencies as $dependency) {
            $dp = $dependency->getClass();

            if ($dp == null){
                $results = null;
            } else {
                //resolveClass个人认为 这个函数是解决依赖的关键
                $results[] = $this->resolveClass($dependency);
            }
        }

        return $results;
    }
    protected function resolveClass(ReflectionParameter $parameter)
    {
        //这里就是返回实例化的对象
        return $this->make($parameter->getClass()->name);
    }

    protected function isBuildable($concrete, $abstract)
    {
        return $concrete === $abstract || $concrete instanceof Closure;
    }

    protected function getConcrete($abstract)
    {
        if (isset($this->bindings[$abstract])) {
            return $this->bindings[$abstract]['concrete'];
        }

        return $abstract;
    }
}

class Traeller
{
    protected $tool;

    public function __construct(Visit $v)
    {
        $this->tool = $v;
    }

    public function go()
    {
        $this->tool->go();
    }
}

interface Visit
{
    public function go();
}

class v1 implements Visit
{
    public function go()
    {
        echo 'v1';
    }
}

class v2 implements Visit
{
    public function go()
    {
        echo 'v2';
    }
}

$app = new Container;
$app->bind('Visit', 'v1');
$app->bind('t', 'Traeller');

$t = $app->make('t');
$t->go();

```