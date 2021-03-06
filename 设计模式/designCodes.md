## 创建型
### 单利模式
> 定义：一个类只允许创建一个对象或实例，这个对象在进程内是唯一。
>
> 线程内唯一：若想实现线程内单利对象唯一，可以通过hashmap方式，key为线程id,value为对象实例实现。
>
> 集群唯一：单利放在公共的存储空间内，进程在使用时去公共空间去取，为了保证唯一性，在使用时候加锁，使用结束后释放锁。

#### php
```
class Singleton
{
    private static $instance;

    private function __construct(){}
    private function __clone(){
        throw new \Exception('不允许克隆');
    }
    public function __wakeup()
    {
        throw new \Exception('不能反序列化');
    }

    public static function getInstance()
    {
        if (!self::$instance instanceof self) {
            self::$instance = new self();
        }

        return self::$instance;
    }

    //业务逻辑
    public function someBusinessLogic()
    {
        echo '业务逻辑';
    }
}

$instance = Singleton::getInstance();
var_dump($instance->someBusinessLogic());exit;
```

```
namespace RefactoringGuru\Singleton\RealWorld;

/**
 * If you need to support several types of Singletons in your app, you can
 * define the basic features of the Singleton in a base class, while moving the
 * actual business logic (like logging) to subclasses.
 */
class Singleton
{
    /**
     * The actual singleton's instance almost always resides inside a static
     * field. In this case, the static field is an array, where each subclass of
     * the Singleton stores its own instance.
     */
    private static $instances = [];

    /**
     * Singleton's constructor should not be public. However, it can't be
     * private either if we want to allow subclassing.
     */
    protected function __construct() { }

    /**
     * Cloning and unserialization are not permitted for singletons.
     */
    protected function __clone() { }

    public function __wakeup()
    {
        throw new \Exception("Cannot unserialize singleton");
    }

    /**
     * The method you use to get the Singleton's instance.
     */
    public static function getInstance()
    {
        $subclass = static::class;
        if (!isset(self::$instances[$subclass])) {
            // Note that here we use the "static" keyword instead of the actual
            // class name. In this context, the "static" keyword means "the name
            // of the current class". That detail is important because when the
            // method is called on the subclass, we want an instance of that
            // subclass to be created here.

            self::$instances[$subclass] = new static();
        }
        return self::$instances[$subclass];
    }
}

/**
 * The logging class is the most known and praised use of the Singleton pattern.
 * In most cases, you need a single logging object that writes to a single log
 * file (control over shared resource). You also need a convenient way to access
 * that instance from any context of your app (global access point).
 */
class Logger extends Singleton
{
    /**
     * A file pointer resource of the log file.
     */
    private $fileHandle;

    /**
     * Since the Singleton's constructor is called only once, just a single file
     * resource is opened at all times.
     *
     * Note, for the sake of simplicity, we open the console stream instead of
     * the actual file here.
     */
    protected function __construct()
    {
        $this->fileHandle = fopen('php://stdout', 'w');
    }

    /**
     * Write a log entry to the opened file resource.
     */
    public function writeLog(string $message): void
    {
        $date = date('Y-m-d');
        fwrite($this->fileHandle, "$date: $message\n");
    }

    /**
     * Just a handy shortcut to reduce the amount of code needed to log messages
     * from the client code.
     */
    public static function log(string $message): void
    {
        $logger = static::getInstance();
        $logger->writeLog($message);
    }
}

/**
 * Applying the Singleton pattern to the configuration storage is also a common
 * practice. Often you need to access application configurations from a lot of
 * different places of the program. Singleton gives you that comfort.
 */
class Config extends Singleton
{
    private $hashmap = [];

    public function getValue(string $key): string
    {
        return $this->hashmap[$key];
    }

    public function setValue(string $key, string $value): void
    {
        $this->hashmap[$key] = $value;
    }
}

/**
 * The client code.
 */
Logger::log("Started!");

// Compare values of Logger singleton.
$l1 = Logger::getInstance();
$l2 = Logger::getInstance();
if ($l1 === $l2) {
    Logger::log("Logger has a single instance.");
} else {
    Logger::log("Loggers are different.");
}

// Check how Config singleton saves data...
$config1 = Config::getInstance();
$login = "test_login";
$password = "test_password";
$config1->setValue("login", $login);
$config1->setValue("password", $password);
// ...and restores it.
$config2 = Config::getInstance();
if ($login == $config2->getValue("login") &&
    $password == $config2->getValue("password")
) {
    Logger::log("Config singleton also works fine.");
}

Logger::log("Finished!");
```

```
//单利与trait结合
trait Singleton
{
    private static $instance;

    static function getInstance(...$args)
    {
        if(!isset(self::$instance)){
            self::$instance = new static(...$args);
        }
        return self::$instance;
    }
}

class MySingleton
{
    use Singleton;
}

class ElseSingleton
{
    use Singleton;
}

$mySingleton = Mysingleton::getInstance();
$elseSingleton = ElseSingleton::getInstance();
var_dump($mySingleton, $elseSingleton);exit;
```

#### go
```
type single struct{}

var (
	once           sync.Once
	singleInstance *single
)

func getInstance() *single {
	if singleInstance == nil {
		once.Do(
			func() {
				singleInstance = &single{}
			})
	}

	return singleInstance
}

func main() {
	for i := 0; i < 30; i++ {
		go getInstance()
	}

	fmt.Scanln()
}

```

### 工厂模式
> **工厂方法**
>
> 定义：其在父类中提供一个创建对象的方法， 允许子类决定实例化对象(产品)的类型。

#### php
```
abstract class SocialNetworkPoster
{
    /**
     * The actual factory method. Note that it returns the abstract connector.
     * This lets subclasses return any concrete connectors without breaking the
     * superclass' contract.
     */
    abstract public function getSocialNetwork(): SocialNetworkConnector;

    /**
     * When the factory method is used inside the Creator's business logic, the
     * subclasses may alter the logic indirectly by returning different types of
     * the connector from the factory method.
     */
    public function post($content): void
    {
        // Call the factory method to create a Product object...
        $network = $this->getSocialNetwork();
        // ...then use it as you will.
        $network->logIn();
        $network->createPost($content);
        $network->logout();
    }
}

/**
 * This Concrete Creator supports Facebook. Remember that this class also
 * inherits the 'post' method from the parent class. Concrete Creators are the
 * classes that the Client actually uses.
 */
class FacebookPoster extends SocialNetworkPoster
{
    private $login, $password;

    public function __construct(string $login, string $password)
    {
        $this->login = $login;
        $this->password = $password;
    }

    public function getSocialNetwork(): SocialNetworkConnector
    {
        return new FacebookConnector($this->login, $this->password);
    }
}

/**
 * This Concrete Creator supports LinkedIn.
 */
class LinkedInPoster extends SocialNetworkPoster
{
    private $email, $password;

    public function __construct(string $email, string $password)
    {
        $this->email = $email;
        $this->password = $password;
    }

    public function getSocialNetwork(): SocialNetworkConnector
    {
        return new LinkedInConnector($this->email, $this->password);
    }
}

/**
 * The Product interface declares behaviors of various types of products.
 */
interface SocialNetworkConnector
{
    public function logIn(): void;

    public function logOut(): void;

    public function createPost($content): void;
}

/**
 * This Concrete Product implements the Facebook API.
 */
class FacebookConnector implements SocialNetworkConnector
{
    private $login, $password;

    public function __construct(string $login, string $password)
    {
        $this->login = $login;
        $this->password = $password;
    }

    public function logIn(): void
    {
        echo "Send HTTP API request to log in user $this->login with " .
            "password $this->password\n";
    }

    public function logOut(): void
    {
        echo "Send HTTP API request to log out user $this->login\n";
    }

    public function createPost($content): void
    {
        echo "Send HTTP API requests to create a post in Facebook timeline.\n";
    }
}

/**
 * This Concrete Product implements the LinkedIn API.
 */
class LinkedInConnector implements SocialNetworkConnector
{
    private $email, $password;

    public function __construct(string $email, string $password)
    {
        $this->email = $email;
        $this->password = $password;
    }

    public function logIn(): void
    {
        echo "Send HTTP API request to log in user $this->email with " .
            "password $this->password\n";
    }

    public function logOut(): void
    {
        echo "Send HTTP API request to log out user $this->email\n";
    }

    public function createPost($content): void
    {
        echo "Send HTTP API requests to create a post in LinkedIn timeline.\n";
    }
}

/**
 * The client code can work with any subclass of SocialNetworkPoster since it
 * doesn't depend on concrete classes.
 */
function clientCode(SocialNetworkPoster $creator)
{
    // ...
    $creator->post("Hello world!");
    $creator->post("I had a large hamburger this morning!");
    // ...
}

/**
 * During the initialization phase, the app can decide which social network it
 * wants to work with, create an object of the proper subclass, and pass it to
 * the client code.
 */
echo "Testing ConcreteCreator1:\n";
clientCode(new FacebookPoster("john_smith", "******"));
echo "\n\n";

echo "Testing ConcreteCreator2:\n";
clientCode(new LinkedInPoster("john_smith@example.com", "******"));
```

#### go
```
package design

import (
	"fmt"
)
//这种通过具体工厂struct去实例化具体的产品

//RuleConfigParser 产品接口
type RuleConfigParser interface {
	Parse(data []byte)
}

type jsonRuleConfigParser struct{}

func (j *jsonRuleConfigParser) Parse(data []byte) {
	fmt.Println("jsonRuleConfigParser")
	// panic("implement me")
}

type xmlRuleConfigParser struct{}

func (x *xmlRuleConfigParser) Parse(data []byte) {
	fmt.Println("xmlRuleConfigParser")
}

//工厂方法
type RuleConfigParserFactory interface {
	CreateParser() RuleConfigParser
}

type jsonRuleConfigParserFactory struct{}

func (jf *jsonRuleConfigParserFactory) CreateParser() RuleConfigParser {
	return &jsonRuleConfigParser{}
}

type xmlRuleConfigParserFactory struct{}

func (xf *xmlRuleConfigParserFactory) CreateParser() RuleConfigParser {
	return &xmlRuleConfigParser{}
}

func NewRuleConfigParserFactory(t string) (RuleConfigParserFactory, error) {
	if t == "json" {
		return &jsonRuleConfigParserFactory{}, nil
	}

	if t == "xml" {
		return &xmlRuleConfigParserFactory{}, nil
	}

	return nil, fmt.Errorf("无法处理")
}

//第二种方式
//RuleConfigParser 产品接口
type RuleConfigParser interface {
	Parse(data []byte)
}

//具体产品
type jsonRuleConfigParser struct{}

func (j *jsonRuleConfigParser) Parse(data []byte) {
	fmt.Println("jsonRuleConfigParser")
}

type xmlRuleConfigParser struct{}

func (x *xmlRuleConfigParser) Parse(data []byte) {
	fmt.Println("xmlRuleConfigParser")
}

func NewJsonRuleConfigParser() RuleConfigParser {
	return &jsonRuleConfigParser{}
}

func NewXmlRuleConfigParser() RuleConfigParser {
	return &xmlRuleConfigParser{}
}

//第二种其他实例

type iGun interface {
	setName(name string)
	setPower(power int)
	getName() string
	getPower() int
}

type gun struct {
	name  string
	power int
}

func (g *gun) setName(name string) {
	g.name = name
}

func (g *gun) getName() string {
	return g.name
}

func (g *gun) setPower(power int) {
	g.power = power
}

func (g *gun) getPower() int {
	return g.power
}

//具体产品
type ak47 struct {
	gun
}

func newAk47() iGun {
	return &ak47{
		gun: gun{
			name:  "AK47 gun",
			power: 4,
		},
	}
}

//具体产品
type musket struct {
	gun
}

func newMusket() iGun {
	return &musket{
		gun: gun{
			name:  "Musket gun",
			power: 1,
		},
	}
}

//工厂方法
func getGun(gunType string) (iGun, error) {
	if gunType == "ak47" {
		return newAk47(), nil
	}
	if gunType == "musket" {
		return newMusket(), nil
	}
	return nil, fmt.Errorf("Wrong gun type passed")
}


```

### 抽象工厂
> **抽象工厂**
>
> 定义：一个工厂创建一簇对象（多种），解决的是多维问题（文章中是二维）。工厂方法是抽象工厂的一种特例。抽象工厂是采用降维的思想来解决复杂问题。

#### php
```
interface TemplateFactory
{
    public function createTitleTemplate(): TitleTemplate;

    public function createPageTemplate(): PageTemplate;

    public function getRenderer(): TemplateRenderer;
}

/**
 * Each Concrete Factory corresponds to a specific variant (or family) of
 * products.
 *
 * This Concrete Factory creates Twig templates.
 */
class TwigTemplateFactory implements TemplateFactory
{
    public function createTitleTemplate(): TitleTemplate
    {
        return new TwigTitleTemplate();
    }

    public function createPageTemplate(): PageTemplate
    {
        return new TwigPageTemplate($this->createTitleTemplate());
    }

    public function getRenderer(): TemplateRenderer
    {
        return new TwigRenderer();
    }
}

/**
 * And this Concrete Factory creates PHPTemplate templates.
 */
class PHPTemplateFactory implements TemplateFactory
{
    public function createTitleTemplate(): TitleTemplate
    {
        return new PHPTemplateTitleTemplate();
    }

    public function createPageTemplate(): PageTemplate
    {
        return new PHPTemplatePageTemplate($this->createTitleTemplate());
    }

    public function getRenderer(): TemplateRenderer
    {
        return new PHPTemplateRenderer();
    }
}

/**
 * Each distinct product type should have a separate interface. All variants of
 * the product must follow the same interface.
 *
 * For instance, this Abstract Product interface describes the behavior of page
 * title templates.
 */
interface TitleTemplate
{
    public function getTemplateString(): string;
}

/**
 * This Concrete Product provides Twig page title templates.
 */
class TwigTitleTemplate implements TitleTemplate
{
    public function getTemplateString(): string
    {
        return "<h1>{{ title }}</h1>";
    }
}

/**
 * And this Concrete Product provides PHPTemplate page title templates.
 */
class PHPTemplateTitleTemplate implements TitleTemplate
{
    public function getTemplateString(): string
    {
        return "<h1><?= \$title; ?></h1>";
    }
}

/**
 * This is another Abstract Product type, which describes whole page templates.
 */
interface PageTemplate
{
    public function getTemplateString(): string;
}

/**
 * The page template uses the title sub-template, so we have to provide the way
 * to set it in the sub-template object. The abstract factory will link the page
 * template with a title template of the same variant.
 */
abstract class BasePageTemplate implements PageTemplate
{
    protected $titleTemplate;

    public function __construct(TitleTemplate $titleTemplate)
    {
        $this->titleTemplate = $titleTemplate;
    }
}

/**
 * The Twig variant of the whole page templates.
 */
class TwigPageTemplate extends BasePageTemplate
{
    public function getTemplateString(): string
    {
        $renderedTitle = $this->titleTemplate->getTemplateString();

        return <<<HTML
        <div class="page">
            $renderedTitle
            <article class="content">{{ content }}</article>
        </div>
        HTML;
    }
}

/**
 * The PHPTemplate variant of the whole page templates.
 */
class PHPTemplatePageTemplate extends BasePageTemplate
{
    public function getTemplateString(): string
    {
        $renderedTitle = $this->titleTemplate->getTemplateString();

        return <<<HTML
        <div class="page">
            $renderedTitle
            <article class="content"><?= \$content; ?></article>
        </div>
        HTML;
    }
}

/**
 * The renderer is responsible for converting a template string into the actual
 * HTML code. Each renderer behaves differently and expects its own type of
 * template strings passed to it. Baking templates with the factory let you pass
 * proper types of templates to proper renders.
 */
interface TemplateRenderer
{
    public function render(string $templateString, array $arguments = []): string;
}

/**
 * The renderer for Twig templates.
 */
class TwigRenderer implements TemplateRenderer
{
    public function render(string $templateString, array $arguments = []): string
    {
        return \Twig::render($templateString, $arguments);
    }
}

/**
 * The renderer for PHPTemplate templates. Note that this implementation is very
 * basic, if not crude. Using the `eval` function has many security
 * implications, so use it with caution in real projects.
 */
class PHPTemplateRenderer implements TemplateRenderer
{
    public function render(string $templateString, array $arguments = []): string
    {
        extract($arguments);

        ob_start();
        eval(' ?>' . $templateString . '<?php ');
        $result = ob_get_contents();
        ob_end_clean();

        return $result;
    }
}

/**
 * The client code. Note that it accepts the Abstract Factory class as the
 * parameter, which allows the client to work with any concrete factory type.
 */
class Page
{

    public $title;

    public $content;

    public function __construct($title, $content)
    {
        $this->title = $title;
        $this->content = $content;
    }

    // Here's how would you use the template further in real life. Note that the
    // page class does not depend on any concrete template classes.
    public function render(TemplateFactory $factory): string
    {
        $pageTemplate = $factory->createPageTemplate();

        $renderer = $factory->getRenderer();

        return $renderer->render($pageTemplate->getTemplateString(), [
            'title' => $this->title,
            'content' => $this->content
        ]);
    }
}

/**
 * Now, in other parts of the app, the client code can accept factory objects of
 * any type.
 */
$page = new Page('Sample page', 'This is the body.');

echo "Testing actual rendering with the PHPTemplate factory:\n";
echo $page->render(new PHPTemplateFactory());


// Uncomment the following if you have Twig installed.

// echo "Testing rendering with the Twig factory:\n"; echo $page->render(new
// TwigTemplateFactory());
```

#### go
```

type iSportsFactory interface {
	makeShoe() iShoe
	makeShirt() iShirt
}

func GetSportsFactory(brand string) (iSportsFactory, error) {
	if brand == "adidas" {
		return &adidas{}, nil
	}

	if brand == "nike" {
		return &nike{}, nil
	}

	return nil, fmt.Errorf("非法产品")
}

type adidas struct{}

func (a *adidas) makeShoe() iShoe {
	return &adidasShoe{
		shoe: shoe{
			logo: "adidas",
			size: 35,
		},
	}
}

func (a *adidas) makeShirt() iShirt {
	return &adidasShirt{
		shirt: shirt{
			logo: "adidas",
			size: 14,
		},
	}
}

type nike struct{}

func (n *nike) makeShoe() iShoe {
	return &nikeShoe{
		shoe: shoe{
			logo: "nike",
			size: 14,
		},
	}
}

func (n *nike) makeShirt() iShirt {
	return &nikeShirt{
		shirt: shirt{
			logo: "nike",
			size: 14,
		},
	}
}

//抽象产品
type iShoe interface {
	setLogo(logo string)
	setSize(size int)
	getLogo() string
	getSize() int
}

type shoe struct {
	logo string
	size int
}

func (s *shoe) setLogo(logo string) {
	s.logo = logo
}

func (s *shoe) getLogo() string {
	return s.logo
}

func (s *shoe) setSize(size int) {
	s.size = size
}

func (s *shoe) getSize() int {
	return s.size
}

//具体产品
type adidasShoe struct {
	shoe
}
type nikeShoe struct {
	shoe
}

type iShirt interface {
	setLogo(logo string)
	setSize(size int)
	getLogo() string
	getSize() int
}

type shirt struct {
	logo string
	size int
}

func (s *shirt) setLogo(logo string) {
	s.logo = logo
}

func (s *shirt) getLogo() string {
	return s.logo
}

func (s *shirt) setSize(size int) {
	s.size = size
}

func (s *shirt) getSize() int {
	return s.size
}

type adidasShirt struct {
	shirt
}

type nikeShirt struct {
	shirt
}

```

<font color="red">**TIP：**</font>
``` 简单工厂、工厂方法、抽象工厂 区分```
> 简单工厂没有子类，通过分支条件(if)实例化对象。工厂方法在父类中提供创建对象的方法，允许子类决定实例化对象。抽象工厂创建一系列相关或相互依赖的对象， 而无需指定其具体类。
> https://refactoringguru.cn/design-patterns/factory-comparison

### 建造者模式
> 定义：分步骤创建复杂对象。使用相同的代码生成不同类型和形式的对象。主要是通过建造类创建对象。

#### php
```
// 只有当产品较为复杂且需要详细配置时，使用生成器模式才有意义。下面的两个
// 产品尽管没有同样的接口，但却相互关联。
class Car is
    // 一辆汽车可能配备有 GPS 设备、行车电脑和几个座位。不同型号的汽车（
    // 运动型轿车、SUV 和敞篷车）可能会安装或启用不同的功能。

class Manual is
    // 用户使用手册应该根据汽车配置进行编制，并介绍汽车的所有功能。


// 生成器接口声明了创建产品对象不同部件的方法。
interface Builder is
    method reset()
    method setSeats(...)
    method setEngine(...)
    method setTripComputer(...)
    method setGPS(...)

// 具体生成器类将遵循生成器接口并提供生成步骤的具体实现。你的程序中可能会
// 有多个以不同方式实现的生成器变体。
class CarBuilder implements Builder is
    private field car:Car

    // 一个新的生成器实例必须包含一个在后续组装过程中使用的空产品对象。
    constructor CarBuilder() is
        this.reset()

    // reset（重置）方法可清除正在生成的对象。
    method reset() is
        this.car = new Car()

    // 所有生成步骤都会与同一个产品实例进行交互。
    method setSeats(...) is
        // 设置汽车座位的数量。

    method setEngine(...) is
        // 安装指定的引擎。

    method setTripComputer(...) is
        // 安装行车电脑。

    method setGPS(...) is
        // 安装全球定位系统。

    // 具体生成器需要自行提供获取结果的方法。这是因为不同类型的生成器可能
    // 会创建不遵循相同接口的、完全不同的产品。所以也就无法在生成器接口中
    // 声明这些方法（至少在静态类型的编程语言中是这样的）。
    //
    // 通常在生成器实例将结果返回给客户端后，它们应该做好生成另一个产品的
    // 准备。因此生成器实例通常会在 `getProduct（获取产品）`方法主体末尾
    // 调用重置方法。但是该行为并不是必需的，你也可让生成器等待客户端明确
    // 调用重置方法后再去处理之前的结果。
    method getProduct():Car is
        product = this.car
        this.reset()
        return product

// 生成器与其他创建型模式的不同之处在于：它让你能创建不遵循相同接口的产品。
class CarManualBuilder implements Builder is
    private field manual:Manual

    constructor CarManualBuilder() is
        this.reset()

    method reset() is
        this.manual = new Manual()

    method setSeats(...) is
        // 添加关于汽车座椅功能的文档。

    method setEngine(...) is
        // 添加关于引擎的介绍。

    method setTripComputer(...) is
        // 添加关于行车电脑的介绍。

    method setGPS(...) is
        // 添加关于 GPS 的介绍。

    method getProduct():Manual is
        // 返回使用手册并重置生成器。


// 主管只负责按照特定顺序执行生成步骤。其在根据特定步骤或配置来生成产品时
// 会很有帮助。由于客户端可以直接控制生成器，所以严格意义上来说，主管类并
// 不是必需的。
class Director is
    private field builder:Builder

    // 主管可同由客户端代码传递给自身的任何生成器实例进行交互。客户端可通
    // 过这种方式改变最新组装完毕的产品的最终类型。
    method setBuilder(builder:Builder)
        this.builder = builder

    // 主管可使用同样的生成步骤创建多个产品变体。
    method constructSportsCar(builder: Builder) is
        builder.reset()
        builder.setSeats(2)
        builder.setEngine(new SportEngine())
        builder.setTripComputer(true)
        builder.setGPS(true)

    method constructSUV(builder: Builder) is
        // ...


// 客户端代码会创建生成器对象并将其传递给主管，然后执行构造过程。最终结果
// 将需要从生成器对象中获取。
class Application is

    method makeCar() is
        director = new Director()

        CarBuilder builder = new CarBuilder()
        director.constructSportsCar(builder)
        Car car = builder.getProduct()

        CarManualBuilder builder = new CarManualBuilder()
        director.constructSportsCar(builder)

        // 最终产品通常需要从生成器对象中获取，因为主管不知晓具体生成器和
        // 产品的存在，也不会对其产生依赖。
        Manual manual = builder.getProduct()
```

#### go
```
package design

type iBuilder interface {
	setWindowType()
	setDoorType()
	setNumFloor()
	getHouse() house
}

func GetBuilder(builderType string) iBuilder {
	if builderType == "normal" {
		return &normalBuilder{}
	}

	if builderType == "igloo" {
		return &iglooBuilder{}
	}
	return nil
}

type normalBuilder struct {
	windowType string
	doorType   string
	floor      int
}

func newNormalBuilder() *normalBuilder {
	return &normalBuilder{}
}

func (b *normalBuilder) setWindowType() {
	b.windowType = "Wooden Window"
}

func (b *normalBuilder) setDoorType() {
	b.doorType = "Wooden Door"
}

func (b *normalBuilder) setNumFloor() {
	b.floor = 2
}

func (b *normalBuilder) getHouse() house {
	return house{
		doorType:   b.doorType,
		windowType: b.windowType,
		floor:      b.floor,
	}
}

type iglooBuilder struct {
	windowType string
	doorType   string
	floor      int
}

func newIglooBuilder() *iglooBuilder {
	return &iglooBuilder{}
}

func (b *iglooBuilder) setWindowType() {
	b.windowType = "Snow Window"
}

func (b *iglooBuilder) setDoorType() {
	b.doorType = "Snow Door"
}

func (b *iglooBuilder) setNumFloor() {
	b.floor = 1
}

func (b *iglooBuilder) getHouse() house {
	return house{
		doorType:   b.doorType,
		windowType: b.windowType,
		floor:      b.floor,
	}
}

type house struct {
	windowType string
	doorType   string
	floor      int
}

type director struct {
	builder iBuilder
}

func newDirector(b iBuilder) *director {
	return &director{
		builder: b,
	}
}

func (d *director) setBuilder(b iBuilder) {
	d.builder = b
}

func (d *director) buildHouse() house {
	d.builder.setDoorType()
	d.builder.setWindowType()
	d.builder.setNumFloor()
	return d.builder.getHouse()
}

/*normalBuilder := getBuilder("normal")
iglooBuilder := getBuilder("igloo")

director := newDirector(normalBuilder)
normalHouse := director.buildHouse()

fmt.Printf("Normal House Door Type: %s\n", normalHouse.doorType)
fmt.Printf("Normal House Window Type: %s\n", normalHouse.windowType)
fmt.Printf("Normal House Num Floor: %d\n", normalHouse.floor)

director.setBuilder(iglooBuilder)
iglooHouse := director.buildHouse()

fmt.Printf("\nIgloo House Door Type: %s\n", iglooHouse.doorType)
fmt.Printf("Igloo House Window Type: %s\n", iglooHouse.windowType)
fmt.Printf("Igloo House Num Floor: %d\n", iglooHouse.floor)*/

```

### 原型模式
> 定义：能够复制已有对象， 而又无需使代码依赖它们所属的类。

#### php
```
    class Person {
        private $name;
        private $age;

        public function __construct($name, $age) {
            $this->name = $name;
            $this->age = $age;
        }

        public function showMyself() {
            echo "我是{$this->name}, 年龄{$this->age}" . PHP_EOL;
        }

        public function coding() {
            echo "我是一个码农, 我在用双手改变世界以及我的发际线..." . PHP_EOL;
        }

        public function reading() {
            echo "阅读是我快乐, 我爱阅读胜过爱左手..." . PHP_EOL;
        }

        public function fallInLove() {
            echo "神鬼狐仙, 风月无边...";
        }

        public function copy() {
            return clone($this);
        }
    }


    // 代码测试
    $tony = new Person("Tony", 26);
    $tony->showMyself();
    $tony->coding();

    $tony1 = $tony->copy();
    $tony1->showMyself();
    $tony1->reading();

    $tony2 = $tony->copy();
    $tony2->showMyself();
    $tony2->fallInLove();
```

#### go
```
package design

import "fmt"

type Inode interface {
	print(string)
	clone() Inode
}

type File struct {
	name string
}

func (f *File) print(indentation string) {
	fmt.Println(indentation + f.name)
}

func (f *File) clone() Inode {
	return &File{name: f.name + "_clone"}
}

type Folder struct {
	children []Inode
	name     string
}

func (f *Folder) print(indentation string) {
	fmt.Println(indentation + f.name)
	for _, i := range f.children {
		i.print(indentation + indentation)
	}
}

func (f *Folder) clone() Inode {
	cloneFolder := &Folder{name: f.name + "_clone"}
	var tempChildren []Inode
	for _, i := range f.children {
		copy := i.clone()
		tempChildren = append(tempChildren, copy)
	}
	cloneFolder.children = tempChildren
	return cloneFolder
}

```

<font color="red">**TIP：**</font>
- 工厂模式与建造者区别
> **工厂模式**创建不同但是相关类型的对象(继承同一父类或接口)，由参数决定创建对象。**建造者模式**用来创建复杂对象，通过设置不同的可选参数，定制化创建不同的对象。(工厂模式是根据不同的条件生成不同Class的对象，构建者模式是根据不同参数生成一个class的不同对象)


## 结构型
### 代理模式
> 定义：在不改变原始类(或叫被代理类)代码情况下，通过引入代理类来给原始类附加功能(这里注意：<font color="red">**代理类是附加新的功能，装饰器是增强功能**</font>)。

#### 实例
```
// 代理模式的代码结构(下面的接口也可以替换成抽象类)
public interface IA {
  void f();
}
public class A impelements IA {
  public void f() { //... }
}
public class AProxy implements IA {
  private IA a;
  public AProxy(IA a) {
    this.a = a;
  }
  
  public void f() {
    // 新添加的代理逻辑
    a.f();
    // 新添加的代理逻辑
  }
}

// 装饰器模式的代码结构(下面的接口也可以替换成抽象类)
public interface IA {
  void f();
}
public class A implements IA {
  public void f() { //... }
}
public class ADecorator implements IA {
  private IA a;
  public ADecorator(IA a) {
    this.a = a;
  }
  
  public void f() {
    // 功能增强代码
    a.f();
    // 功能增强代码
  }
}

```

#### php
```
/**
 * Interface Subject 抽象主题角色
 *
 * 定义RealSubject和Proxy共同具备的东西
 */
interface Subject
{
    public function say();
    public function run();
}

/**
 * Class RealSubject 真正主题角色
 */
class RealSubject implements Subject
{
    // 姓名
    private $_name;

    /**
     * RealSubject constructor. 构造方法
     *
     * @param $name
     */
    public function __construct($name)
    {
        $this->_name = $name;
    }

    /**
     * 说话
     */
    public function say()
    {
        echo $this->_name."在说话<br>";
    }

    /**
     * 在跑步
     */
    public function run(){
        echo $this->_name."在跑步<br>";
    }
}

/**
 * Class Proxy 代理对象
 */
class Proxy implements Subject
{
    // 真实主题对象
    private $_realSubject = null;

    /**
     * Proxy constructor. 构造方法，依赖注入方式储存真实对象
     *
     * @param RealSubject|null $realSubject
     */
    public function __construct(RealSubject $realSubject = null)
    {
        if (empty($realSubject)) {
            $this->_realSubject = new RealSubject();
        } else {
            $this->_realSubject = $realSubject;
        }
    }

    /**
     * 调用说话方法
     */
    public function say()
    {
        $this->_realSubject->say();
    }

    /**
     * 调用跑步方法
     */
    public function run()
    {
        $this->_realSubject->run();
    }
}

/**
 * Class Client 本地测试
 */
class Client
{
    public static function test()
    {
        // 创建
        $subject = new RealSubject("张三");
        // 代理
        $proxy = new Proxy($subject);
        // 张三在说话
        $proxy->say();
        // 张三在跑步
        $proxy->run();
    }
}

// 测试
Client::test();
```

```
// 远程服务接口。
interface ThirdPartyTVLib is
    method listVideos()
    method getVideoInfo(id)
    method downloadVideo(id)

// 服务连接器的具体实现。该类的方法可以向腾讯视频请求信息。请求速度取决于
// 用户和腾讯视频的互联网连接情况。如果同时发送大量请求，即使所请求的信息
// 一模一样，程序的速度依然会减慢。
class ThirdPartyTVClass implements ThirdPartyTVLib is
    method listVideos() is
        // 向腾讯视频发送一个 API 请求。

    method getVideoInfo(id) is
        // 获取某个视频的元数据。

    method downloadVideo(id) is
        // 从腾讯视频下载一个视频文件。

// 为了节省网络带宽，我们可以将请求结果缓存下来并保存一段时间。但你可能无
// 法直接将这些代码放入服务类中。比如该类可能是第三方程序库的一部分或其签
// 名是`final（最终）`。因此我们会在一个实现了服务类接口的新代理类中放入
// 缓存代码。当代理类接收到真实请求后，才会将其委派给服务对象。
class CachedTVClass implements ThirdPartyTVLib is
    private field service: ThirdPartyTVLib
    private field listCache, videoCache
    field needReset

    constructor CachedTVClass(service: ThirdPartyTVLib) is
        this.service = service

    method listVideos() is
        if (listCache == null || needReset)
            listCache = service.listVideos()
        return listCache

    method getVideoInfo(id) is
        if (videoCache == null || needReset)
            videoCache = service.getVideoInfo(id)
        return videoCache

    method downloadVideo(id) is
        if (!downloadExists(id) || needReset)
            service.downloadVideo(id)

// 之前直接与服务对象交互的 GUI 类不需要改变，前提是它仅通过接口与服务对
// 象交互。我们可以安全地传递一个代理对象来代替真实服务对象，因为它们都实
// 现了相同的接口。
class TVManager is
    protected field service: ThirdPartyTVLib

    constructor TVManager(service: ThirdPartyTVLib) is
        this.service = service

    method renderVideoPage(id) is
        info = service.getVideoInfo(id)
        // 渲染视频页面。

    method renderListPanel() is
        list = service.listVideos()
        // 渲染视频缩略图列表。

    method reactOnUserInput() is
        renderVideoPage()
        renderListPanel()

// 程序可在运行时对代理进行配置。
class Application is
    method init() is
        aTVService = new ThirdPartyTVClass()
        aTVProxy = new CachedTVClass(aTVService)
        manager = new TVManager(aTVProxy)
        manager.reactOnUserInput()

```
#### go
```
//server server
type Server interface {
	HandleRequest(string, string) (int, string)
}

type nginx struct {
	application       *application
	maxAllowedRequest int
	rateLimiter       map[string]int
}

func NewNginxServer() *nginx {
	return &nginx{
		application:       &application{},
		maxAllowedRequest: 2,
		rateLimiter:       make(map[string]int),
	}
}

func (n *nginx) HandleRequest(url, method string) (int, string) {
	allowed := n.checkRateLimiting(url)
	if !allowed {
		return 403, "Not Allowed"
	}
	return n.application.HandleRequest(url, method)
}

func (n *nginx) checkRateLimiting(url string) bool {
	if n.rateLimiter[url] == 0 {
		n.rateLimiter[url] = 1
	}
	if n.rateLimiter[url] > n.maxAllowedRequest {
		return false
	}
	n.rateLimiter[url] = n.rateLimiter[url] + 1
	return true
}

type application struct {
}

func (a *application) HandleRequest(url, method string) (int, string) {
	if url == "/app/status" && method == "GET" {
		return 200, "Ok"
	}

	if url == "/create/user" && method == "POST" {
		return 201, "User Created"
	}

	return 404, "Not Ok"
}

```

<font color="red">**TIP：**</font>
- 代理模式、装饰器模式、适配器模式区别
> 代理和装饰器都采用了封装，不同的是：代理是对原始类添加新的功能，装饰器是对原始类的功能进行增强，并且这两个设计模式都隐藏对原始类的直接访问。
> 适配器模式能为被封装对象提供不同的接口， 代理模式能为对象提供相同的接口， 装饰模式则能为对象提供加强的接口。
> 装饰和代理有着相似的结构， 但是其意图却非常不同。 这两个模式的构建都基于组合原则， 也就是说一个对象应该将部分工作委派给另一个对象。**两者之间的不同之处在于代理通常自行管理其服务对象的生命周期， 而装饰的生成则总是由客户端进行控制**。

### 适配器模式
> 定义：它将原本不兼容的不能一起工作的类，经过适配，让它们可以一起工作。

#### 实例结构
```
// 类适配器: 基于继承
public interface ITarget {
  void f1();
  void f2();
  void fc();
}

public class Adaptee {
  public void fa() { //... }
  public void fb() { //... }
  public void fc() { //... }
}

public class Adaptor extends Adaptee implements ITarget {
  public void f1() {
    super.fa();
  }
  
  public void f2() {
    //...重新实现f2()...
  }
  
  // 这里fc()不需要实现，直接继承自Adaptee，这是跟对象适配器最大的不同点
}

// 对象适配器：基于组合
public interface ITarget {
  void f1();
  void f2();
  void fc();
}

public class Adaptee {
  public void fa() { //... }
  public void fb() { //... }
  public void fc() { //... }
}

public class Adaptor implements ITarget {
  private Adaptee adaptee;
  
  public Adaptor(Adaptee adaptee) {
    this.adaptee = adaptee;
  }
  
  public void f1() {
    adaptee.fa(); //委托给Adaptee
  }
  
  public void f2() {
    //...重新实现f2()...
  }
  
  public void fc() {
    adaptee.fc();
  }
}

```

#### php
```
/**
 * 类适配器模式
 * @author guisu
 * 
 */
 
/**
 * 目标角色
 * @version 2.0
 */
interface Target {
 
  /**
   * 源类的方法：这个方法将来有可能继续改进
   */
  public function hello();
 
  /**
   * 目标点
   */
  public function world();
}
 
/**
 * 源角色：被适配的角色
 */
class Adaptee {
 /**
   * 源类含有的方法
   */
  public function world() {
    echo ' world <br />';
  }
 
  /**
   * 加入新的方法,之前有hello这个方法
   */
  public function greet() {
    echo ' Greet ';
  }
}
 
/**
 * 类适配器角色
 */
class Adapter implements Target {
 
 private $_adaptee;
 /**
  * construct
  *
  * @param Adaptee $adaptee
  */
  public function __construct(Adaptee $adaptee) {
    $this->_adaptee = $adaptee;
  }
 
  /**
   * 源类中没有world方法，在此补充
   */
  //主要是这里 适配新方法greet
  public function hello() {
    $this->_adaptee->greet();
  }
 
  /**
   * 源类中没有world方法，在此补充
   */
  public function world() {
    $this->_adaptee->world();
  }
}

class Client {
 
  /**
   * Main program.
   */
  public static function main() {
   $adaptee = new Adaptee();
    $adapter = new Adapter($adaptee);
    $adapter->hello();
    $adapter->world();
  }
}
Client::main(); 

```

#### go
```
package design

import "fmt"

type Computer interface {
	InsertIntoLightningPort()
}

type Mac struct{}

func (m *Mac) InsertIntoLightningPort() {
	fmt.Println("mac机器")
}

type Window struct{}

func (w *Window) insertIntoUSBPort() {
	fmt.Println("window机器")
}

//适配器
type WindowsAdapter struct {
	WindowMachine *Window
}

func (w *WindowsAdapter) InsertIntoLightningPort() {
	fmt.Println("适配器")
	w.WindowMachine.insertIntoUSBPort()
}


func main() {
	mac := &design.Mac{}
	mac.InsertIntoLightningPort()

	win := &design.Window{}
	winAdapter := &design.WindowsAdapter{
		WindowMachine: win,
	}
	winAdapter.InsertIntoLightningPort()
}

```

<font color="red">**TIP：**</font>
- 代理、桥接、装饰器、适配器 4 种设计模式的区别
  
> **代理模式：** 代理模式在不改变原始类接口的条件下，为原始类定义一个代理类，主要目的是控制访问，而非加强功能，这是它跟装饰器模式最大的不同。
> **桥接模式：** 桥接模式的目的是将接口部分和实现部分分离，从而让它们可以较为容易、也相对独立地加以改变。
> **装饰器模式：** 装饰者模式在不改变原始类接口的情况下，对原始类功能进行增强，并且支持多个装饰器的嵌套使用。
> **适配器模式：** 适配器模式是一种事后的补救策略。适配器提供跟原始类不同的接口，而代理模式、装饰器模式提供的都是跟原始类相同的接口。

### 装饰器模式
> 定义：通过将对象放入包含行为的特殊封装对象中来为原对象绑定新的行为。

#### 实例
```
// 装饰可以改变组件接口所定义的操作。
interface DataSource is
    method writeData(data)
    method readData():data

// 具体组件提供操作的默认实现。这些类在程序中可能会有几个变体。
class FileDataSource implements DataSource is
    constructor FileDataSource(filename) { ... }

    method writeData(data) is
        // 将数据写入文件。

    method readData():data is
        // 从文件读取数据。

// 装饰基类和其他组件遵循相同的接口。该类的主要任务是定义所有具体装饰的封
// 装接口。封装的默认实现代码中可能会包含一个保存被封装组件的成员变量，并
// 且负责对其进行初始化。
class DataSourceDecorator implements DataSource is
    protected field wrappee: DataSource

    constructor DataSourceDecorator(source: DataSource) is
        wrappee = source

    // 装饰基类会直接将所有工作分派给被封装组件。具体装饰中则可以新增一些
    // 额外的行为。
    method writeData(data) is
        wrappee.writeData(data)

    // 具体装饰可调用其父类的操作实现，而不是直接调用被封装对象。这种方式
    // 可简化装饰类的扩展工作。
    method readData():data is
        return wrappee.readData()

// 具体装饰必须在被封装对象上调用方法，不过也可以自行在结果中添加一些内容。
// 装饰必须在调用封装对象之前或之后执行额外的行为。
class EncryptionDecorator extends DataSourceDecorator is
    method writeData(data) is
        // 1. 对传递数据进行加密。
        // 2. 将加密后数据传递给被封装对象 writeData（写入数据）方法。

    method readData():data is
        // 1. 通过被封装对象的 readData（读取数据）方法获取数据。
        // 2. 如果数据被加密就尝试解密。
        // 3. 返回结果。

// 你可以将对象封装在多层装饰中。
class CompressionDecorator extends DataSourceDecorator is
    method writeData(data) is
        // 1. 压缩传递数据。
        // 2. 将压缩后数据传递给被封装对象 writeData（写入数据）方法。

    method readData():data is
        // 1. 通过被封装对象的 readData（读取数据）方法获取数据。
        // 2. 如果数据被压缩就尝试解压。
        // 3. 返回结果。


// 选项 1：装饰组件的简单示例
class Application is
    method dumbUsageExample() is
        source = new FileDataSource("somefile.dat")
        source.writeData(salaryRecords)
        // 已将明码数据写入目标文件。

        source = new CompressionDecorator(source)
        source.writeData(salaryRecords)
        // 已将压缩数据写入目标文件。

        source = new EncryptionDecorator(source)
        // 源变量中现在包含：
        // Encryption > Compression > FileDataSource
        source.writeData(salaryRecords)
        // 已将压缩且加密的数据写入目标文件。


// 选项 2：客户端使用外部数据源。SalaryManager（工资管理器）对象并不关心
// 数据如何存储。它们会与提前配置好的数据源进行交互，数据源则是通过程序配
// 置器获取的。
class SalaryManager is
    field source: DataSource

    constructor SalaryManager(source: DataSource) { ... }

    method load() is
        return source.readData()

    method save() is
        source.writeData(salaryRecords)
    // ...其他有用的方法...


// 程序可在运行时根据配置或环境组装不同的装饰堆桟。
class ApplicationConfigurator is
    method configurationExample() is
        source = new FileDataSource("salary.dat")
        if (enabledEncryption)
            source = new EncryptionDecorator(source)
        if (enabledCompression)
            source = new CompressionDecorator(source)

        logger = new SalaryManager(source)
        salary = logger.load()
    // ...
```

#### php
```
/**
 * The Component interface declares a filtering method that must be implemented
 * by all concrete components and decorators.
 */
interface InputFormat
{
    public function formatText(string $text): string;
}

/**
 * The Concrete Component is a core element of decoration. It contains the
 * original text, as is, without any filtering or formatting.
 */
class TextInput implements InputFormat
{
    public function formatText(string $text): string
    {
        return $text;
    }
}

/**
 * The base Decorator class doesn't contain any real filtering or formatting
 * logic. Its main purpose is to implement the basic decoration infrastructure:
 * a field for storing a wrapped component or another decorator and the basic
 * formatting method that delegates the work to the wrapped object. The real
 * formatting job is done by subclasses.
 */
class TextFormat implements InputFormat
{
    /**
     * @var InputFormat
     */
    protected $inputFormat;

    public function __construct(InputFormat $inputFormat)
    {
        $this->inputFormat = $inputFormat;
    }

    /**
     * Decorator delegates all work to a wrapped component.
     */
    public function formatText(string $text): string
    {
        return $this->inputFormat->formatText($text);
    }
}

/**
 * This Concrete Decorator strips out all HTML tags from the given text.
 */
class PlainTextFilter extends TextFormat
{
    public function formatText(string $text): string
    {
        $text = parent::formatText($text);
        return strip_tags($text);
    }
}

/**
 * This Concrete Decorator strips only dangerous HTML tags and attributes that
 * may lead to an XSS vulnerability.
 */
class DangerousHTMLTagsFilter extends TextFormat
{
    private $dangerousTagPatterns = [
        "|<script.*?>([\s\S]*)?</script>|i", // ...
    ];

    private $dangerousAttributes = [
        "onclick", "onkeypress", // ...
    ];


    public function formatText(string $text): string
    {
        $text = parent::formatText($text);

        foreach ($this->dangerousTagPatterns as $pattern) {
            $text = preg_replace($pattern, '', $text);
        }

        foreach ($this->dangerousAttributes as $attribute) {
            $text = preg_replace_callback('|<(.*?)>|', function ($matches) use ($attribute) {
                $result = preg_replace("|$attribute=|i", '', $matches[1]);
                return "<" . $result . ">";
            }, $text);
        }

        return $text;
    }
}

/**
 * This Concrete Decorator provides a rudimentary Markdown → HTML conversion.
 */
class MarkdownFormat extends TextFormat
{
    public function formatText(string $text): string
    {
        $text = parent::formatText($text);

        // Format block elements.
        $chunks = preg_split('|\n\n|', $text);
        foreach ($chunks as &$chunk) {
            // Format headers.
            if (preg_match('|^#+|', $chunk)) {
                $chunk = preg_replace_callback('|^(#+)(.*?)$|', function ($matches) {
                    $h = strlen($matches[1]);
                    return "<h$h>" . trim($matches[2]) . "</h$h>";
                }, $chunk);
            } // Format paragraphs.
            else {
                $chunk = "<p>$chunk</p>";
            }
        }
        $text = implode("\n\n", $chunks);

        // Format inline elements.
        $text = preg_replace("|__(.*?)__|", '<strong>$1</strong>', $text);
        $text = preg_replace("|\*\*(.*?)\*\*|", '<strong>$1</strong>', $text);
        $text = preg_replace("|_(.*?)_|", '<em>$1</em>', $text);
        $text = preg_replace("|\*(.*?)\*|", '<em>$1</em>', $text);

        return $text;
    }
}


/**
 * The client code might be a part of a real website, which renders user-
 * generated content. Since it works with formatters through the Component
 * interface, it doesn't care whether it gets a simple component object or a
 * decorated one.
 */
function displayCommentAsAWebsite(InputFormat $format, string $text)
{
    // ..

    echo $format->formatText($text);

    // ..
}

/**
 * Input formatters are very handy when dealing with user-generated content.
 * Displaying such content "as is" could be very dangerous, especially when
 * anonymous users can generate it (e.g. comments). Your website is not only
 * risking getting tons of spammy links but may also be exposed to XSS attacks.
 */
$dangerousComment = <<<HERE
Hello! Nice blog post!
Please visit my <a href='http://www.iwillhackyou.com'>homepage</a>.
<script src="http://www.iwillhackyou.com/script.js">
  performXSSAttack();
</script>
HERE;

/**
 * Naive comment rendering (unsafe).
 */
$naiveInput = new TextInput();
echo "Website renders comments without filtering (unsafe):\n";
displayCommentAsAWebsite($naiveInput, $dangerousComment);
echo "\n\n\n";

/**
 * Filtered comment rendering (safe).
 */
$filteredInput = new PlainTextFilter($naiveInput);
echo "Website renders comments after stripping all tags (safe):\n";
displayCommentAsAWebsite($filteredInput, $dangerousComment);
echo "\n\n\n";


/**
 * Decorator allows stacking multiple input formats to get fine-grained control
 * over the rendered content.
 */
$dangerousForumPost = <<<HERE
# Welcome

This is my first post on this **gorgeous** forum.

<script src="http://www.iwillhackyou.com/script.js">
  performXSSAttack();
</script>
HERE;

/**
 * Naive post rendering (unsafe, no formatting).
 */
$naiveInput = new TextInput();
echo "Website renders a forum post without filtering and formatting (unsafe, ugly):\n";
displayCommentAsAWebsite($naiveInput, $dangerousForumPost);
echo "\n\n\n";

/**
 * Markdown formatter + filtering dangerous tags (safe, pretty).
 */
$text = new TextInput();
$markdown = new MarkdownFormat($text);
$filteredInput = new DangerousHTMLTagsFilter($markdown);
echo "Website renders a forum post after translating markdown markup" .
    " and filtering some dangerous HTML tags and attributes (safe, pretty):\n";
displayCommentAsAWebsite($filteredInput, $dangerousForumPost);
echo "\n\n\n";
```

#### go
```
package design

type Pizza interface {
	GetPrice() int
}

type VeggeMania struct {
}

func (v *VeggeMania) GetPrice() int {
	return 15
}

//具体装饰
type TomatoTopping struct {
	pizza Pizza
}

func (t *TomatoTopping) GetPrice() int {
	pizzaPrice := t.pizza.GetPrice()
	return pizzaPrice + 10
}

type CheeseTopping struct {
	pizza Pizza
}

func (c *CheeseTopping) GetPrice() int {
	pizzaPrice := c.pizza.GetPrice()
	return pizzaPrice + 20
}

func NewTomatoTopping(p Pizza) *TomatoTopping {
	return &TomatoTopping{pizza: p}
}

func NewCheeseTopping(p Pizza) *CheeseTopping {
	return &CheeseTopping{pizza: p}
}

```

### 门面模式
> 定义：门面模式为子系统提供一组统一的接口，定义一组高层接口让子系统更易用。

#### php
```
class Facade
{
    protected $subsystem1;

    protected $subsystem2;

    /**
     * Depending on your application's needs, you can provide the Facade with
     * existing subsystem objects or force the Facade to create them on its own.
     */
    public function __construct(
        Subsystem1 $subsystem1 = null,
        Subsystem2 $subsystem2 = null
    ) {
        $this->subsystem1 = $subsystem1 ?: new Subsystem1();
        $this->subsystem2 = $subsystem2 ?: new Subsystem2();
    }

    /**
     * The Facade's methods are convenient shortcuts to the sophisticated
     * functionality of the subsystems. However, clients get only to a fraction
     * of a subsystem's capabilities.
     */
    public function operation(): string
    {
        $result = "Facade initializes subsystems:\n";
        $result .= $this->subsystem1->operation1();
        $result .= $this->subsystem2->operation1();
        $result .= "Facade orders subsystems to perform the action:\n";
        $result .= $this->subsystem1->operationN();
        $result .= $this->subsystem2->operationZ();

        return $result;
    }
}

/**
 * The Subsystem can accept requests either from the facade or client directly.
 * In any case, to the Subsystem, the Facade is yet another client, and it's not
 * a part of the Subsystem.
 */
class Subsystem1
{
    public function operation1(): string
    {
        return "Subsystem1: Ready!\n";
    }

    // ...

    public function operationN(): string
    {
        return "Subsystem1: Go!\n";
    }
}

/**
 * Some facades can work with multiple subsystems at the same time.
 */
class Subsystem2
{
    public function operation1(): string
    {
        return "Subsystem2: Get ready!\n";
    }

    // ...

    public function operationZ(): string
    {
        return "Subsystem2: Fire!\n";
    }
}

/**
 * The client code works with complex subsystems through a simple interface
 * provided by the Facade. When a facade manages the lifecycle of the subsystem,
 * the client might not even know about the existence of the subsystem. This
 * approach lets you keep the complexity under control.
 */
function clientCode(Facade $facade)
{
    // ...

    echo $facade->operation();

    // ...
}

/**
 * The client code may have some of the subsystem's objects already created. In
 * this case, it might be worthwhile to initialize the Facade with these objects
 * instead of letting the Facade create new instances.
 */
$subsystem1 = new Subsystem1();
$subsystem2 = new Subsystem2();
$facade = new Facade($subsystem1, $subsystem2);
clientCode($facade);
```

#### go
```
package design

import "fmt"

type Facade struct {
	subsystem1 *subsystem1
	subsystem2 *subsystem2
}

func NewFacade() *Facade {
	return &Facade{
		subsystem1: newSubsystem1(),
		subsystem2: newSubsystem2(),
	}
}

func (f *Facade) Operation() {
	f.subsystem1.myName()
	f.subsystem2.myName()
}

type subsystem1 struct {
}

func (s *subsystem1) myName() {
	fmt.Println("subsystem1")
}

func newSubsystem1() *subsystem1 {
	return &subsystem1{}
}

type subsystem2 struct {
}

func (s *subsystem2) myName() {
	fmt.Println("subsystem2")
}
func newSubsystem2() *subsystem2 {
	return &subsystem2{}
}

```

```
package design

import "fmt"

//外观 组合复杂子系统
type walletFacade struct {
	account      *account
	securityCode *securityCode
}

func NewWalletFacade(accountId string, code int) *walletFacade {
	fmt.Println("创建账号")
	w := &walletFacade{
		account:      newAccount(accountId),
		securityCode: newSecurityCode(code),
	}

	fmt.Println("创建结束")
	return w
}

func (w *walletFacade) AddMoneyToWallet(accountID string, securityCode int) error {
	err := w.account.checkAccount(accountID)
	if err != nil {
		return err
	}
	err = w.securityCode.checkCode(securityCode)
	if err != nil {
		return err
	}

	return nil
}

//子系统一
type account struct {
	name string
}

func newAccount(accountName string) *account {
	return &account{
		name: accountName,
	}
}

func (a *account) checkAccount(accountName string) error {
	if a.name != accountName {
		return fmt.Errorf("Account Name is incorrect")
	}
	fmt.Println("Account Verified")
	return nil
}

//子系统二
type securityCode struct {
	code int
}

func newSecurityCode(code int) *securityCode {
	return &securityCode{
		code: code,
	}
}

func (s *securityCode) checkCode(incomingCode int) error {
	if s.code != incomingCode {
		return fmt.Errorf("Security Code is incorrect")
	}
	fmt.Println("SecurityCode Verified")
	return nil
}

```

<font color="red">**TIP：**</font>
- 适配器和门面设计区别
> 适配器是做接口转换，解决原接口和目标接口不匹配问题。门面模式做接口整合，解决是多接口调用带来的问题。

### 桥接模式
> “一个类存在两个（或多个）独立变化的维度，我们通过组合的方式，让这两个（或多个）维度可以独立进行扩展。”通过组合关系来替代继承关系，避免继承层次的指数级爆炸。
> 将抽象和实现解耦，让它们可以独立的变化

#### 实例
```
// “抽象部分”定义了两个类层次结构中“控制”部分的接口。它管理着一个指向“实
// 现部分”层次结构中对象的引用，并会将所有真实工作委派给该对象。
class RemoteControl is
    protected field device: Device
    constructor RemoteControl(device: Device) is
        this.device = device
    method togglePower() is
        if (device.isEnabled()) then
            device.disable()
        else
            device.enable()
    method volumeDown() is
        device.setVolume(device.getVolume() - 10)
    method volumeUp() is
        device.setVolume(device.getVolume() + 10)
    method channelDown() is
        device.setChannel(device.getChannel() - 1)
    method channelUp() is
        device.setChannel(device.getChannel() + 1)


// 你可以独立于设备类的方式从抽象层中扩展类。
class AdvancedRemoteControl extends RemoteControl is
    method mute() is
        device.setVolume(0)


// “实现部分”接口声明了在所有具体实现类中通用的方法。它不需要与抽象接口相
// 匹配。实际上，这两个接口可以完全不一样。通常实现接口只提供原语操作，而
// 抽象接口则会基于这些操作定义较高层次的操作。
interface Device is
    method isEnabled()
    method enable()
    method disable()
    method getVolume()
    method setVolume(percent)
    method getChannel()
    method setChannel(channel)


// 所有设备都遵循相同的接口。
class Tv implements Device is
    // ...

class Radio implements Device is
    // ...


// 客户端代码中的某个位置。
tv = new Tv()
remote = new RemoteControl(tv)
remote.togglePower()

radio = new Radio()
remote = new AdvancedRemoteControl(radio)
```

#### php
```
/**
 * The Abstraction defines the interface for the "control" part of the two class
 * hierarchies. It maintains a reference to an object of the Implementation
 * hierarchy and delegates all of the real work to this object.
 */
class Abstraction
{
    /**
     * @var Implementation
     */
    protected $implementation;

    public function __construct(Implementation $implementation)
    {
        $this->implementation = $implementation;
    }

    public function operation(): string
    {
        return "Abstraction: Base operation with:\n" .
            $this->implementation->operationImplementation();
    }
}

/**
 * You can extend the Abstraction without changing the Implementation classes.
 */
class ExtendedAbstraction extends Abstraction
{
    public function operation(): string
    {
        return "ExtendedAbstraction: Extended operation with:\n" .
            $this->implementation->operationImplementation();
    }
}

/**
 * The Implementation defines the interface for all implementation classes. It
 * doesn't have to match the Abstraction's interface. In fact, the two
 * interfaces can be entirely different. Typically the Implementation interface
 * provides only primitive operations, while the Abstraction defines higher-
 * level operations based on those primitives.
 */
interface Implementation
{
    public function operationImplementation(): string;
}

/**
 * Each Concrete Implementation corresponds to a specific platform and
 * implements the Implementation interface using that platform's API.
 */
class ConcreteImplementationA implements Implementation
{
    public function operationImplementation(): string
    {
        return "ConcreteImplementationA: Here's the result on the platform A.\n";
    }
}

class ConcreteImplementationB implements Implementation
{
    public function operationImplementation(): string
    {
        return "ConcreteImplementationB: Here's the result on the platform B.\n";
    }
}

/**
 * Except for the initialization phase, where an Abstraction object gets linked
 * with a specific Implementation object, the client code should only depend on
 * the Abstraction class. This way the client code can support any abstraction-
 * implementation combination.
 */
function clientCode(Abstraction $abstraction)
{
    // ...

    echo $abstraction->operation();

    // ...
}

/**
 * The client code should be able to work with any pre-configured abstraction-
 * implementation combination.
 */
$implementation = new ConcreteImplementationA();
$abstraction = new Abstraction($implementation);
clientCode($abstraction);

echo "\n";

$implementation = new ConcreteImplementationB();
$abstraction = new ExtendedAbstraction($implementation);
clientCode($abstraction);
```

```

/**
 * The Abstraction.
 */
abstract class Page
{
    /**
     * @var Renderer
     */
    protected $renderer;

    /**
     * The Abstraction is usually initialized with one of the Implementation
     * objects.
     */
    public function __construct(Renderer $renderer)
    {
        $this->renderer = $renderer;
    }

    /**
     * The Bridge pattern allows replacing the attached Implementation object
     * dynamically.
     */
    public function changeRenderer(Renderer $renderer): void
    {
        $this->renderer = $renderer;
    }

    /**
     * The "view" behavior stays abstract since it can only be provided by
     * Concrete Abstraction classes.
     */
    abstract public function view(): string;
}

/**
 * This Concrete Abstraction represents a simple page.
 */
class SimplePage extends Page
{
    protected $title;
    protected $content;

    public function __construct(Renderer $renderer, string $title, string $content)
    {
        parent::__construct($renderer);
        $this->title = $title;
        $this->content = $content;
    }

    public function view(): string
    {
        return $this->renderer->renderParts([
            $this->renderer->renderHeader(),
            $this->renderer->renderTitle($this->title),
            $this->renderer->renderTextBlock($this->content),
            $this->renderer->renderFooter()
        ]);
    }
}

/**
 * This Concrete Abstraction represents a more complex page.
 */
class ProductPage extends Page
{
    protected $product;

    public function __construct(Renderer $renderer, Product $product)
    {
        parent::__construct($renderer);
        $this->product = $product;
    }

    public function view(): string
    {
        return $this->renderer->renderParts([
            $this->renderer->renderHeader(),
            $this->renderer->renderTitle($this->product->getTitle()),
            $this->renderer->renderTextBlock($this->product->getDescription()),
            $this->renderer->renderImage($this->product->getImage()),
            $this->renderer->renderLink("/cart/add/" . $this->product->getId(), "Add to cart"),
            $this->renderer->renderFooter()
        ]);
    }
}

/**
 * A helper class for the ProductPage class.
 */
class Product
{
    private $id, $title, $description, $image, $price;

    public function __construct(
        string $id,
        string $title,
        string $description,
        string $image,
        float $price
    ) {
        $this->id = $id;
        $this->title = $title;
        $this->description = $description;
        $this->image = $image;
        $this->price = $price;
    }

    public function getId(): string { return $this->id; }

    public function getTitle(): string { return $this->title; }

    public function getDescription(): string { return $this->description; }

    public function getImage(): string { return $this->image; }

    public function getPrice(): float { return $this->price; }
}


/**
 * The Implementation declares a set of "real", "under-the-hood", "platform"
 * methods.
 *
 * In this case, the Implementation lists rendering methods that can be used to
 * compose any web page. Different Abstractions may use different methods of the
 * Implementation.
 */
interface Renderer
{
    public function renderTitle(string $title): string;

    public function renderTextBlock(string $text): string;

    public function renderImage(string $url): string;

    public function renderLink(string $url, string $title): string;

    public function renderHeader(): string;

    public function renderFooter(): string;

    public function renderParts(array $parts): string;
}

/**
 * This Concrete Implementation renders a web page as HTML.
 */
class HTMLRenderer implements Renderer
{
    public function renderTitle(string $title): string
    {
        return "<h1>$title</h1>";
    }

    public function renderTextBlock(string $text): string
    {
        return "<div class='text'>$text</div>";
    }

    public function renderImage(string $url): string
    {
        return "<img src='$url'>";
    }

    public function renderLink(string $url, string $title): string
    {
        return "<a href='$url'>$title</a>";
    }

    public function renderHeader(): string
    {
        return "<html><body>";
    }

    public function renderFooter(): string
    {
        return "</body></html>";
    }

    public function renderParts(array $parts): string
    {
        return implode("\n", $parts);
    }
}

/**
 * This Concrete Implementation renders a web page as JSON strings.
 */
class JsonRenderer implements Renderer
{
    public function renderTitle(string $title): string
    {
        return '"title": "' . $title . '"';
    }

    public function renderTextBlock(string $text): string
    {
        return '"text": "' . $text . '"';
    }

    public function renderImage(string $url): string
    {
        return '"img": "' . $url . '"';
    }

    public function renderLink(string $url, string $title): string
    {
        return '"link": {"href": "' . $url . '", "title": "' . $title . '"}';
    }

    public function renderHeader(): string
    {
        return '';
    }

    public function renderFooter(): string
    {
        return '';
    }

    public function renderParts(array $parts): string
    {
        return "{\n" . implode(",\n", array_filter($parts)) . "\n}";
    }
}

/**
 * The client code usually deals only with the Abstraction objects.
 */
function clientCode(Page $page)
{
    // ...

    echo $page->view();

    // ...
}

/**
 * The client code can be executed with any pre-configured combination of the
 * Abstraction+Implementation.
 */
$HTMLRenderer = new HTMLRenderer();
$JSONRenderer = new JsonRenderer();

$page = new SimplePage($HTMLRenderer, "Home", "Welcome to our website!");
echo "HTML view of a simple content page:\n";
clientCode($page);
echo "\n\n";

/**
 * The Abstraction can change the linked Implementation at runtime if needed.
 */
$page->changeRenderer($JSONRenderer);
echo "JSON view of a simple content page, rendered with the same client code:\n";
clientCode($page);
echo "\n\n";


$product = new Product("123", "Star Wars, episode1",
    "A long time ago in a galaxy far, far away...",
    "/images/star-wars.jpeg", 39.95);

$page = new ProductPage($HTMLRenderer, $product);
echo "HTML view of a product page, same client code:\n";
clientCode($page);
echo "\n\n";

$page->changeRenderer($JSONRenderer);
echo "JSON view of a simple content page, with the same client code:\n";
clientCode($page);
```

```
<?php
//抽象化角色
abstract class MiPhone{
    protected $_audio;      //存放音频软件对象
    abstract function output();
    public function __construct(Audio $audio){
        $this->_audio = $audio;
    }
}
//具体手机
class Mix extends MiPhone{
    //语音输出功能
    public function output(){
        $this->_audio->output();
    }
}
class Note extends MiPhone{
    public function output(){
        $this->_audio->output();
    }
}
//实现化角色 功能实现者
abstract class Audio{
    abstract function output();
}
//具体音频实现者 -骨传导音频输出
class Osteophony extends Audio{
    public function output(){
        echo "骨传导输出的声音-----哈哈".PHP_EOL;
    }
}
//普通音频输出---声筒输出
class Cylinder extends Audio{
    public function output(){
        echo "声筒输出的声音-----呵呵".PHP_EOL;
    }
}

//让小米mix和小米note输出声音
$mix = new Mix(new Osteophony);
$mix->output();
$note = new Note(new Cylinder);
$note->output();
```

#### go

```
package design

import "fmt"

//Computer 抽象层 包含具体实施层的引用
type Computers interface {
	Print()
	SetPrinter(printer)
}

type Macs struct {
	printer printer
}

func (m *Macs) Print() {
	fmt.Println("MAC")
	m.printer.printFile()
}

func (m *Macs) SetPrinter(p printer) {
	m.printer = p
}

type Windows struct {
	printer printer
}

func (w *Windows) Print() {
	fmt.Println("Print request for windows")
	w.printer.printFile()
}

func (w *Windows) SetPrinter(p printer) {
	w.printer = p
}

//具体实施的接口
type printer interface {
	printFile()
}

type Epson struct {
}

func (p *Epson) printFile() {
	fmt.Println("Printing by a EPSON Printer")
}

type Hp struct {
}

func (p *Hp) printFile() {
	fmt.Println("Printing by a HP Printer")
}

```

### 组合模式
> 使用它将对象组合成树状结构， 并且能像使用独立对象一样使用它们。

#### php
```
<?php
abstract class FormElement
{
    /**
     * We can anticipate that all DOM elements require these 3 fields.
     */
    protected $name;
    protected $title;
    protected $data;

    public function __construct(string $name, string $title)
    {
        $this->name = $name;
        $this->title = $title;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setData($data): void
    {
        $this->data = $data;
    }

    public function getData(): array
    {
        return $this->data;
    }

    /**
     * Each concrete DOM element must provide its rendering implementation, but
     * we can safely assume that all of them are returning strings.
     */
    abstract public function render(): string;
}

/**
 * This is a Leaf component. Like all the Leaves, it can't have any children.
 */
class Input extends FormElement
{
    private $type;

    public function __construct(string $name, string $title, string $type)
    {
        parent::__construct($name, $title);
        $this->type = $type;
    }

    /**
     * Since Leaf components don't have any children that may handle the bulk of
     * the work for them, usually it is the Leaves who do the most of the heavy-
     * lifting within the Composite pattern.
     */
    public function render(): string
    {
        return "<label for=\"{$this->name}\">{$this->title}</label>\n" .
            "<input name=\"{$this->name}\" type=\"{$this->type}\" value=\"{$this->data}\">\n";
    }
}

/**
 * The base Composite class implements the infrastructure for managing child
 * objects, reused by all Concrete Composites.
 */
abstract class FieldComposite extends FormElement
{
    /**
     * @var FormElement[]
     */
    protected $fields = [];

    /**
     * The methods for adding/removing sub-objects.
     */
    public function add(FormElement $field): void
    {
        $name = $field->getName();
        $this->fields[$name] = $field;
    }

    public function remove(FormElement $component): void
    {
        $this->fields = array_filter($this->fields, function ($child) use ($component) {
            return $child != $component;
        });
    }

    /**
     * Whereas a Leaf's method just does the job, the Composite's method almost
     * always has to take its sub-objects into account.
     *
     * In this case, the composite can accept structured data.
     *
     * @param array $data
     */
    public function setData($data): void
    {
        foreach ($this->fields as $name => $field) {
            if (isset($data[$name])) {
                $field->setData($data[$name]);
            }
        }
    }

    /**
     * The same logic applies to the getter. It returns the structured data of
     * the composite itself (if any) and all the children data.
     */
    public function getData(): array
    {
        $data = [];

        foreach ($this->fields as $name => $field) {
            $data[$name] = $field->getData();
        }

        return $data;
    }

    /**
     * The base implementation of the Composite's rendering simply combines
     * results of all children. Concrete Composites will be able to reuse this
     * implementation in their real rendering implementations.
     */
    public function render(): string
    {
        $output = "";

        foreach ($this->fields as $name => $field) {
            $output .= $field->render();
        }

        return $output;
    }
}

/**
 * The fieldset element is a Concrete Composite.
 */
class Fieldset extends FieldComposite
{
    public function render(): string
    {
        // Note how the combined rendering result of children is incorporated
        // into the fieldset tag.
        $output = parent::render();

        return "<fieldset><legend>{$this->title}</legend>\n$output</fieldset>\n";
    }
}

/**
 * And so is the form element.
 */
class Form extends FieldComposite
{
    protected $url;

    public function __construct(string $name, string $title, string $url)
    {
        parent::__construct($name, $title);
        $this->url = $url;
    }

    public function render(): string
    {
        $output = parent::render();
        return "<form action=\"{$this->url}\">\n<h3>{$this->title}</h3>\n$output</form>\n";
    }
}

/**
 * The client code gets a convenient interface for building complex tree
 * structures.
 */
function getProductForm(): FormElement
{
    $form = new Form('product', "Add product", "/product/add");
    $form->add(new Input('name', "Name", 'text'));
    $form->add(new Input('description', "Description", 'text'));

    $picture = new Fieldset('photo', "Product photo");
    $picture->add(new Input('caption', "Caption", 'text'));
    $picture->add(new Input('image', "Image", 'file'));
    $form->add($picture);

    return $form;
}

/**
 * The form structure can be filled with data from various sources. The Client
 * doesn't have to traverse through all form fields to assign data to various
 * fields since the form itself can handle that.
 */
function loadProductData(FormElement $form)
{
    $data = [
        'name' => 'Apple MacBook',
        'description' => 'A decent laptop.',
        'photo' => [
            'caption' => 'Front photo.',
            'image' => 'photo1.png',
        ],
    ];

    $form->setData($data);
}

/**
 * The client code can work with form elements using the abstract interface.
 * This way, it doesn't matter whether the client works with a simple component
 * or a complex composite tree.
 */
function renderProduct(FormElement $form)
{
    // ..

    echo $form->render();

    // ..
}

$form = getProductForm();
loadProductData($form);
renderProduct($form);
```

```
abstract class Filesystem
{
    protected $name;

    public function __construct($name)
    {
        $this->name = $name;
    }

    public abstract function getName();
    public abstract function getSize();
}


class Dir extends Filesystem
{
    private $filesystems = [];

    // 组合对象必须实现添加方法。因为传入参数规定为Filesystem类型，
    // 所以目录和文件都能添加
    public function add(Filesystem $filesystem)
    {
        $key = array_search($filesystem, $this->filesystems);
        if ($key === false) {
            $this->filesystems[] = $filesystem;
        }
    }

    // 组合对象必须实现移除方法
    public function remove(Filesystem $filesystem)
    {
        $key = array_search($filesystem, $this->filesystems);
        if ($key !== false) {
            unset($this->filesystems[$key]);
        }
    }

    public function getName()
    {
        return '目录：' . $this->name;
    }

    public function getSize()
    {
        $size = 0;
        foreach ($this->filesystems as $filesystem) {
            $size += $filesystem->getSize();
        }

        return $size;
    }
}


class TextFile extends Filesystem
{
    public function getName()
    {
        return '文本文件：' . $this->name;
    }

    public function getSize()
    {
        return 10;
    }
}

/**
 * 独立对象2：图片文件类
 */
class ImageFile extends Filesystem
{
    public function getName()
    {
        return '图片：' . $this->name;
    }

    public function getSize()
    {
        return 100;
    }
}

/**
 * 独立对象：视频文件类
 */
class VideoFile extends Filesystem
{
    public function getName()
    {
        return '视频：'. $this->name;
    }

    public function getSize()
    {
        return 200;
    }
}


$dir = new Dir('home');
$dir->add(new TextFile('text1.txt'));
$dir->add(new ImageFile('bg1.png'));
$dir->add(new VideoFile('film1.mp4'));

// 在home下创建子目录source
$subDir = new Dir('source');
$dir->add($subDir);

// 创建一个text2.txt，并放到子目录source中
$text2 = new TextFile('text2.txt');
$subDir->add($text2);

// 打印信息
echo $text2->getName(), '-->', $text2->getSize();
echo '<br />';
echo $subDir->getName(), ' --> ',$subDir->getSize();
echo '<br />';
echo $dir->getName(), ' --> ', $dir->getSize();
```

#### go
```
```

## 行为型

### 观察者模式
> 在对象之间定义一个一对多的依赖，当一个对象状态改变的时候，所有依赖的对象都会收到通知。

#### 实例
```

public interface Subject {
  void registerObserver(Observer observer);
  void removeObserver(Observer observer);
  void notifyObservers(Message message);
}

public interface Observer {
  void update(Message message);
}

public class ConcreteSubject implements Subject {
  private List<Observer> observers = new ArrayList<Observer>();

  @Override
  public void registerObserver(Observer observer) {
    observers.add(observer);
  }

  @Override
  public void removeObserver(Observer observer) {
    observers.remove(observer);
  }

  @Override
  public void notifyObservers(Message message) {
    for (Observer observer : observers) {
      observer.update(message);
    }
  }

}

public class ConcreteObserverOne implements Observer {
  @Override
  public void update(Message message) {
    //TODO: 获取消息通知，执行自己的逻辑...
    System.out.println("ConcreteObserverOne is notified.");
  }
}

public class ConcreteObserverTwo implements Observer {
  @Override
  public void update(Message message) {
    //TODO: 获取消息通知，执行自己的逻辑...
    System.out.println("ConcreteObserverTwo is notified.");
  }
}

public class Demo {
  public static void main(String[] args) {
    ConcreteSubject subject = new ConcreteSubject();
    subject.registerObserver(new ConcreteObserverOne());
    subject.registerObserver(new ConcreteObserverTwo());
    subject.notifyObservers(new Message());
  }
}
```

#### php
```
class MyObserver1 implements SplObserver {
    public function update(SplSubject $subject) {
        echo __CLASS__ . ' - ' . $subject->getName();
    }
}

class MyObserver2 implements SplObserver {
    public function update(SplSubject $subject) {
        echo __CLASS__ . ' - ' . $subject->getName();
    }
}

class MySubject implements SplSubject {
    private $_observers;
    private $_name;

    public function __construct($name) {
        $this->_observers = new SplObjectStorage();
        $this->_name = $name;
    }

    public function attach(SplObserver $observer) {
        $this->_observers->attach($observer);
    }

    public function detach(SplObserver $observer) {
        $this->_observers->detach($observer);
    }

    public function notify() {
        foreach ($this->_observers as $observer) {
            $observer->update($this);
        }
    }

    public function getName() {
        return $this->_name;
    }
}

$observer1 = new MyObserver1();
$observer2 = new MyObserver2();

$subject = new MySubject("test");

$subject->attach($observer1);
$subject->attach($observer2);
$subject->notify();

```

#### go
```
package design

import (
	"fmt"
	"os"
)

// Subject 主题 也叫被观察者
type Subject interface {
	RegisterObserver(ob observer)
	DeleteObserver(ob observer)
	NotifyAll()
}

//具体主题
type item struct {
	observers []observer
	name      string
	inStock   bool
}

func NewItem(name string) *item {
	return &item{
		name: name,
	}
}

func (i *item) RegisterObserver(o observer) {
	i.observers = append(i.observers, o)
}

func (i *item) DeleteObserver(o observer) {
	i.observers = removeFromSlice(i.observers, o)
}

func (i *item) NotifyAll() {
	i.inStock = true
	i.nofity()
}

func (i *item) nofity() {
	for _, ob := range i.observers {
		ob.Update(i.name)
	}
}

func removeFromSlice(obList []observer, delOb observer) []observer {
	for i, ob := range obList {
		fmt.Println(i, ob)
		os.Exit(0)
	}

	return obList
}

//观察者
type observer interface {
	Update(string)
}

//Customer 具体观察者
type customer struct {
	id string
}

func NewCustomer(id string) *customer {
	return &customer{
		id: id,
	}
}

func (c *customer) Update(name string) {
	fmt.Printf("顾客邮箱 %s, 顾客姓名 %s \n", c.id, name)
}

func (c *customer) GetId() string {
	return c.id
}


func main() {
	n := design.NewItem("鸿星尔克")

	c1 := design.NewCustomer("wangbin1@163.com")
	c2 := design.NewCustomer("wangbin2@163.com")

	n.RegisterObserver(c1)
	n.RegisterObserver(c2)

	n.NotifyAll()
}
```

### 策略模式
> 定义一族算法，将每个算法封装起来，让它们可以互相替换。使算法的变化独立与使用它们的客户端。( **所有策略基于同一个接口** )

#### 实例
```
public interface Strategy {
  void algorithmInterface();
}

public class ConcreteStrategyA implements Strategy {
  @Override
  public void  algorithmInterface() {
    //具体的算法...
  }
}

public class ConcreteStrategyB implements Strategy {
  @Override
  public void  algorithmInterface() {
    //具体的算法...
  }
}
```

```
//策略与工厂模式结合
public class StrategyFactory {
  public static Strategy getStrategy(String type) {
    if (type == null || type.isEmpty()) {
      throw new IllegalArgumentException("type should not be empty.");
    }

    if (type.equals("A")) {
      return new ConcreteStrategyA();
    } else if (type.equals("B")) {
      return new ConcreteStrategyB();
    }

    return null;
  }
}
```

#### php
```
interface ApiStrategy
{
    public function apiQequest();
}

class ConcreteStrategyA implements ApiStrategy
{
    public function apiQequest()
    {
        echo 'A request';
    }
}

class ConcreteStrategyB implements ApiStrategy
{
    public function apiQequest()
    {
        echo 'B request';
    }
}

class ApiStrategyFactory
{
    public $typeMapClass = [];

    public function __construct()
    {
        //这里可以利用反射获取类或把A、B映射类放在配置文件中
        $this->typeMapClass = [
            'A' => new ConcreteStrategyA,
            'B' => new ConcreteStrategyB,
        ];
    }
    public function getStrategy(string $type)
    {
        return $this->typeMapClass[$type];
    }
}

$f = new ApiStrategyFactory;
$f->getStrategy('B')->apiQequest();
```

#### go
```
package design

import "fmt"

type evictionAlgo interface {
	evict(c *cache)
}

type Lru struct {
}

func (l *Lru) evict(c *cache) {
	fmt.Println("lru")
}

type Lfu struct {
}

func (l *Lfu) evict(c *cache) {
	fmt.Println("lfu")
}

type cache struct {
	storage      map[string]string
	evictionAlgo evictionAlgo
	capacity     int
	maxCapacity  int
}

func InitCache(e evictionAlgo) *cache {
	storage := make(map[string]string)
	return &cache{
		storage:      storage,
		evictionAlgo: e,
		capacity:     0,
		maxCapacity:  2,
	}
}

func (c *cache) SetEvictionAlgo(e evictionAlgo) {
	c.evictionAlgo = e
}

func (c *cache) Add(key, value string) {
	if c.capacity == c.maxCapacity {
		c.Evict()
	}

	c.capacity++
	c.storage[key] = value
}

func (c *cache) Get(key string) {
	delete(c.storage, key)
}

func (c *cache) Evict() {
	c.evictionAlgo.evict(c)
	c.capacity--
}
```

<font color="red">**TIP：**</font>
```
工厂模式解耦对象的创建。
观察者解耦观察者和被观察者。
策略模式解耦的是策略的定义、创建、使用。
```

### 模板模式
> 在父类中定义算法框架，运行子类在不修改结构的情况下重写算法的特定步骤。

#### 实例
```

public abstract class AbstractClass {
  public final void templateMethod() {
    //...
    method1();
    //...
    method2();
    //...
  }
  
  protected abstract void method1();
  protected abstract void method2();
}

public class ConcreteClass1 extends AbstractClass {
  @Override
  protected void method1() {
    //...
  }
  
  @Override
  protected void method2() {
    //...
  }
}

public class ConcreteClass2 extends AbstractClass {
  @Override
  protected void method1() {
    //...
  }
  
  @Override
  protected void method2() {
    //...
  }
}

AbstractClass demo = ConcreteClass1();
demo.templateMethod();
```


#### php
```

/**
 * The Abstract Class defines the template method and declares all its steps.
 */
abstract class SocialNetwork
{
    protected $username;

    protected $password;

    public function __construct(string $username, string $password)
    {
        $this->username = $username;
        $this->password = $password;
    }

    /**
     * The actual template method calls abstract steps in a specific order. A
     * subclass may implement all of the steps, allowing this method to actually
     * post something to a social network.
     */
    public function post(string $message): bool
    {
        // Authenticate before posting. Every network uses a different
        // authentication method.
        if ($this->logIn($this->username, $this->password)) {
            // Send the post data. All networks have different APIs.
            $result = $this->sendData($message);
            // ...
            $this->logOut();

            return $result;
        }

        return false;
    }

    /**
     * The steps are declared abstract to force the subclasses to implement them
     * all.
     */
    abstract public function logIn(string $userName, string $password): bool;

    abstract public function sendData(string $message): bool;

    abstract public function logOut(): void;
}

/**
 * This Concrete Class implements the Facebook API (all right, it pretends to).
 */
class Facebook extends SocialNetwork
{
    public function logIn(string $userName, string $password): bool
    {
        echo "\nChecking user's credentials...\n";
        echo "Name: " . $this->username . "\n";
        echo "Password: " . str_repeat("*", strlen($this->password)) . "\n";

        simulateNetworkLatency();

        echo "\n\nFacebook: '" . $this->username . "' has logged in successfully.\n";

        return true;
    }

    public function sendData(string $message): bool
    {
        echo "Facebook: '" . $this->username . "' has posted '" . $message . "'.\n";

        return true;
    }

    public function logOut(): void
    {
        echo "Facebook: '" . $this->username . "' has been logged out.\n";
    }
}

/**
 * This Concrete Class implements the Twitter API.
 */
class Twitter extends SocialNetwork
{
    public function logIn(string $userName, string $password): bool
    {
        echo "\nChecking user's credentials...\n";
        echo "Name: " . $this->username . "\n";
        echo "Password: " . str_repeat("*", strlen($this->password)) . "\n";

        simulateNetworkLatency();

        echo "\n\nTwitter: '" . $this->username . "' has logged in successfully.\n";

        return true;
    }

    public function sendData(string $message): bool
    {
        echo "Twitter: '" . $this->username . "' has posted '" . $message . "'.\n";

        return true;
    }

    public function logOut(): void
    {
        echo "Twitter: '" . $this->username . "' has been logged out.\n";
    }
}

/**
 * A little helper function that makes waiting times feel real.
 */
function simulateNetworkLatency()
{
    $i = 0;
    while ($i < 5) {
        echo ".";
        sleep(1);
        $i++;
    }
}

/**
 * The client code.
 */
echo "Username: \n";
$username = readline();
echo "Password: \n";
$password = readline();
echo "Message: \n";
$message = readline();

echo "\nChoose the social network to post the message:\n" .
    "1 - Facebook\n" .
    "2 - Twitter\n";
$choice = readline();

// Now, let's create a proper social network object and send the message.
if ($choice == 1) {
    $network = new Facebook($username, $password);
} elseif ($choice == 2) {
    $network = new Twitter($username, $password);
} else {
    die("Sorry, I'm not sure what you mean by that.\n");
}
$network->post($message);
```

#### go
```
package design

import "fmt"

type dataMiner interface {
	GetData()
	OpenFile()
}

type Datam struct {
	DataMiner dataMiner
}

func (d *Datam) ReadData() {
	//打开逻辑
	d.DataMiner.OpenFile()
	//读取逻辑
	d.DataMiner.GetData()

	fmt.Println("read data")
}

type Csv struct {
}

func (c *Csv) OpenFile() {
	fmt.Println("csv openfile")
}

func (c *Csv) GetData() {
	fmt.Println("csv getdata")
}

type Pdf struct {
}

func (p *Pdf) OpenFile() {
	fmt.Println("pdf openfile")
}

func (p *Pdf) GetData() {
	fmt.Println("pdf getdata")
}


func main() {
	csv := &design.Csv{}
	d := &design.Datam{
		DataMiner: csv,
	}
	d.ReadData()
}
```

<font color="red">**TIP：**</font>
```
模板方法基于继承机制： 它允许你通过扩展子类中的部分内容来改变部分算法。 策略模式基于组合机制： 你可以通过对相应行为提供不同的策略来改变对象的部分行为。
```




### 状态模式
> 让你能在一个对象的内部状态变化时改变其行为， 使其看上去就像改变了自身所属的类一样。
> 
> 状态模式一般用来实现状态机，而状态机常用在游戏、工作流引擎等系统开发中。不过，状态机的实现方式有多种，除了状态模式，比较常用的还有分支逻辑法和查表法。
> 
> 状态机有 3 个组成部分：状态（State）、事件（Event）、动作（Action）。其中，事件也称为转移条件（Transition Condition）。事件触发状态的转移及动作的执行。不过，动作不是必须的，也可能只转移状态，不执行任何动作。

#### php
```
<?php
class Context
{
    private $state;

    public function __construct(State $state)
    {
        $this->transitionTo($state);
    }

    /**
     * The Context allows changing the State object at runtime.
     */
    public function transitionTo(State $state): void
    {
        echo "类: Transition to " . get_class($state) . ".\n";
        $this->state = $state;
        $this->state->setContext($this);
    }

    /**
     * The Context delegates part of its behavior to the current State object.
     */
    public function request1(): void
    {
        $this->state->handle1();
    }

    public function request2(): void
    {
        $this->state->handle2();
    }
}

/**
 * The base State class declares methods that all Concrete State should
 * implement and also provides a backreference to the Context object, associated
 * with the State. This backreference can be used by States to transition the
 * Context to another State.
 */
abstract class State
{
    /**
     * @var Context
     */
    protected $context;

    public function setContext(Context $context)
    {
        $this->context = $context;
    }

    abstract public function handle1(): void;

    abstract public function handle2(): void;
}

/**
 * Concrete States implement various behaviors, associated with a state of the
 * Context.
 */
class ConcreteStateA extends State
{
    public function handle1(): void
    {
        echo "老王.\n";
        $this->context->transitionTo(new ConcreteStateB());
    }

    public function handle2(): void
    {
        echo "ConcreteStateA handles request2.\n";
    }
}

class ConcreteStateB extends State
{
    public function handle1(): void
    {
        echo "ConcreteStateB handles request1.\n";
    }

    public function handle2(): void
    {
        echo "ConcreteStateB handles request2.\n";
        echo "ConcreteStateB wants to change the state of the context.\n";
        $this->context->transitionTo(new ConcreteStateA());
    }
}

/**
 * The client code.
 */
$context = new Context(new ConcreteStateA());
$context->request1();
$context->request2();
```

```

//状态接口
abstract class IMario
{
    public $marioStateMachine;

    public function __construct($marioStateMachine)
    {
        $this->marioStateMachine = $marioStateMachine;
    }

    //事件
    abstract public function obtainMushRoom(); //获得蘑菇房
/*    abstract public function obtainCape(); //获得斗篷
    abstract public function obtainFireFlower(); //获得火焰
    abstract public function meetMonster(); //遇到怪物*/
}

//小马里奥状态
class SmallMario extends IMario
{
    public function obtainMushRoom()
    {
        $this->stateMachineActive(); //这一步 可以优化
        $this->marioStateMachine->setScore($this->marioStateMachine->getScore() + 100);
    }

    private function stateMachineActive()
    {
        //事件转义
        $this->marioStateMachine->setCurrentState(new SuperMario($this->marioStateMachine));
    }

    //....obtainCape obtainFireFlower meetMonster其他方法与之类似
}

// 省略CapeMario、FireMario类...

class SuperMario extends IMario
{
    public function obtainMushRoom()
    {
        exit('666');
    }
}


class MarioStateMachine
{
    private $statue;
    private $score;

    public function __construct()
    {
        //初始化
        $this->statue = new SmallMario($this);
    }

    public function setCurrentState(IMario $imario)
    {
        $this->statue = $imario;
    }

    public function getScore()
    {
        return intval($this->score);
    }

    public function setScore(int $score)
    {
        $this->score = $score;
    }

    public function obtainMushRoom()
    {
        $this->statue->obtainMushRoom();
    }
}

$n = new MarioStateMachine;
$n->obtainMushRoom();
var_dump($n->getScore());
```

#### go
```
package design

import "fmt"

type vendingMachine struct {
	hasItem       state
	itemRequested state
	hasMoney      state
	noItem        state

	currentState state

	itemCount int
	itemPrice int
}

func newVendingMachine(itemCount, itemPrice int) *vendingMachine {
	v := &vendingMachine{
		itemCount: itemCount,
		itemPrice: itemPrice,
	}
	hasItemState := &hasItemState{
		vendingMachine: v,
	}
	itemRequestedState := &itemRequestedState{
		vendingMachine: v,
	}
	hasMoneyState := &hasMoneyState{
		vendingMachine: v,
	}
	noItemState := &noItemState{
		vendingMachine: v,
	}

	v.setState(hasItemState)
	v.hasItem = hasItemState
	v.itemRequested = itemRequestedState
	v.hasMoney = hasMoneyState
	v.noItem = noItemState
	return v
}

func (v *vendingMachine) requestItem() error {
	return v.currentState.requestItem()
}

func (v *vendingMachine) addItem(count int) error {
	return v.currentState.addItem(count)
}

func (v *vendingMachine) insertMoney(money int) error {
	return v.currentState.insertMoney(money)
}

func (v *vendingMachine) dispenseItem() error {
	return v.currentState.dispenseItem()
}

func (v *vendingMachine) setState(s state) {
	v.currentState = s
}

func (v *vendingMachine) incrementItemCount(count int) {
	fmt.Printf("Adding %d items\n", count)
	v.itemCount = v.itemCount + count
}

type state interface {
	addItem(int) error
	requestItem() error
	insertMoney(money int) error
	dispenseItem() error
}

type noItemState struct {
	vendingMachine *vendingMachine
}

func (i *noItemState) requestItem() error {
	return fmt.Errorf("Item out of stock")
}

func (i *noItemState) addItem(count int) error {
	i.vendingMachine.incrementItemCount(count)
	i.vendingMachine.setState(i.vendingMachine.hasItem)
	return nil
}

func (i *noItemState) insertMoney(money int) error {
	return fmt.Errorf("Item out of stock")
}
func (i *noItemState) dispenseItem() error {
	return fmt.Errorf("Item out of stock")
}

type hasItemState struct {
	vendingMachine *vendingMachine
}

func (i *hasItemState) requestItem() error {
	if i.vendingMachine.itemCount == 0 {
		i.vendingMachine.setState(i.vendingMachine.noItem)
		return fmt.Errorf("No item present")
	}
	fmt.Printf("Item requestd\n")
	i.vendingMachine.setState(i.vendingMachine.itemRequested)
	return nil
}

func (i *hasItemState) addItem(count int) error {
	fmt.Printf("%d items added\n", count)
	i.vendingMachine.incrementItemCount(count)
	return nil
}

func (i *hasItemState) insertMoney(money int) error {
	return fmt.Errorf("Please select item first")
}
func (i *hasItemState) dispenseItem() error {
	return fmt.Errorf("Please select item first")
}

type itemRequestedState struct {
	vendingMachine *vendingMachine
}

func (i *itemRequestedState) requestItem() error {
	return fmt.Errorf("Item already requested")
}

func (i *itemRequestedState) addItem(count int) error {
	return fmt.Errorf("Item Dispense in progress")
}

func (i *itemRequestedState) insertMoney(money int) error {
	if money < i.vendingMachine.itemPrice {
		fmt.Errorf("Inserted money is less. Please insert %d", i.vendingMachine.itemPrice)
	}
	fmt.Println("Money entered is ok")
	i.vendingMachine.setState(i.vendingMachine.hasMoney)
	return nil
}
func (i *itemRequestedState) dispenseItem() error {
	return fmt.Errorf("Please insert money first")
}

type hasMoneyState struct {
	vendingMachine *vendingMachine
}

func (i *hasMoneyState) requestItem() error {
	return fmt.Errorf("Item dispense in progress")
}

func (i *hasMoneyState) addItem(count int) error {
	return fmt.Errorf("Item dispense in progress")
}

func (i *hasMoneyState) insertMoney(money int) error {
	return fmt.Errorf("Item out of stock")
}
func (i *hasMoneyState) dispenseItem() error {
	fmt.Println("Dispensing Item")
	i.vendingMachine.itemCount = i.vendingMachine.itemCount - 1
	if i.vendingMachine.itemCount == 0 {
		i.vendingMachine.setState(i.vendingMachine.noItem)
	} else {
		i.vendingMachine.setState(i.vendingMachine.hasItem)
	}
	return nil
}


//
func main() {
    vendingMachine := newVendingMachine(1, 10)

    err := vendingMachine.requestItem()
    if err != nil {
        log.Fatalf(err.Error())
    }

    err = vendingMachine.insertMoney(10)
    if err != nil {
        log.Fatalf(err.Error())
    }

    err = vendingMachine.dispenseItem()
    if err != nil {
        log.Fatalf(err.Error())
    }

    fmt.Println()

    err = vendingMachine.addItem(2)
    if err != nil {
        log.Fatalf(err.Error())
    }

    fmt.Println()

    err = vendingMachine.requestItem()
    if err != nil {
        log.Fatalf(err.Error())
    }

    err = vendingMachine.insertMoney(10)
    if err != nil {
        log.Fatalf(err.Error())
    }

    err = vendingMachine.dispenseItem()
    if err != nil {
        log.Fatalf(err.Error())
    }
}
```

## 设计模式
  - 创建、行为、结构
  > 创建型模式解决对象的创建问题。结构型模式解决类或对象组合或组装问题。行为型模式解决类或对象之间的交互问题。
  > 
  > 设计模式要干的事情就是解耦。创建型模式是将创建和使用代码解耦，结构型模式是将不同功能代码解耦，行为型模式是将不同的行为代码解耦，具体到观察者模式，它是将观察者和被观察者代码解耦

## 推荐
> https://refactoringguru.cn/design-patterns/singleton
>
> 