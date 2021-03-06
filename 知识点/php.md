#### php深拷贝和浅拷贝
> 浅拷贝相当于一个对象的引用，一个对象属性修改，另一个也跟着变化。
> 深拷贝完全拷贝，两个独立的对象，互不影响。

- 实例

```
class A
{
    public $abc = 'abc';
    public $obj = null;

    public function __construct()
    {
        $this->obj = new Tmp;
    }


    //实现对象属性深拷贝方式，第二种方式是序列号和反序列化
    public function __clone()
    {
        $this->obj = clone $this->obj;
    }

    //第二种方式 序列化和反序列化
    // $b = serialize($a);
    // $b = unserialize($b);
}

class Tmp
{
    public $tmpa = '&&';
}

$a = new A;
$b = clone $a; //深拷贝，普通属性是深拷贝，但是对于对象属性是浅拷贝。除非添加魔术方法__clone()或序列化

$b->abc = 'test';
$b->obj->tmpa = '||';

echo $a->abc; //不受影响
echo "\r\n";
echo $b->abc;

//对象属性浅拷贝，两个都变化
echo $b->obj->tmpa;
echo $a->obj->tmpa;


//                  end                  //

$b = $a; //浅拷贝
echo $a->abc;
echo "\r\n";
echo $b->abc;
echo $b->obj->tmpa;

//修改某一个对象值另一个对象相应变化，包含对象属性
$b->abc = 'test';
$b->obj->tmpa = '||';

echo '======';
echo $a->abc;
echo "\r\n";
echo $b->abc;

echo $b->obj->tmpa;
echo $a->obj->tmpa;
```

#### array_filter(**$child != $t**)函数
```
//定义：遍历 array 数组中的每个值，并将每个值传递给 callback 回调函数。 如果 callback 回调函数返回 true，则将 array 数组中的当前值返回到结果 array 数组中。

$arr = ['a'=> 'aa', 'b' => 'bb', 'c' => 'cc'];
$t = 'aa';
var_dump(array_filter($arr, function ($child) use ($t) {
    return $child != $t; //这里可以过滤调arr数组中指定的值。在使用中常用的方式是利用返回为true进而保存到arr中，利用!反义来过滤arr中的值也是一个不错的方式。
}));

```