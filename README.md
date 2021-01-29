# Container
Laravel Container


## Introduction
A dependency injection container

## How to use
Your class struct

```
class ContainerConcreteStub
{
    //
}

interface IContainerContractStub
{
    //
}

class ContainerImplementationStub implements IContainerContractStub
{
    //
}

class ContainerImplementationStubTwo implements IContainerContractStub
{
    //
}

class ContainerDependentStub
{
    public $impl;

    public function __construct(IContainerContractStub $impl)
    {
        $this->impl = $impl;
    }
}

class ContainerNestedDependentStub
{
    public $inner;

    public function __construct(ContainerDependentStub $inner)
    {
        $this->inner = $inner;
    }
}

class ContainerDefaultValueStub
{
    public $stub;
    public $default;

    public function __construct(ContainerConcreteStub $stub, $default = 'taylor')
    {
        $this->stub    = $stub;
        $this->default = $default;
    }
}

class ContainerMixedPrimitiveStub
{
    public $first;
    public $last;
    public $stub;

    public function __construct($first, ContainerConcreteStub $stub, $last)
    {
        $this->stub  = $stub;
        $this->last  = $last;
        $this->first = $first;
    }
}

class ContainerInjectVariableStub
{
    public $something;

    public function __construct(ContainerConcreteStub $concrete, $something)
    {
        $this->something = $something;
    }
}

class ContainerInjectVariableStubWithInterfaceImplementation implements IContainerContractStub
{
    public $something;

    public function __construct(ContainerConcreteStub $concrete, $something)
    {
        $this->something = $something;
    }
}
```

Then you can manage your class by Container

```
class ContainerTest extends TestCase
{
    protected function tearDown(): void
    {
        Container::setInstance(null);
    }

    public function testContainerSingleton()
    {
        $container = Container::setInstance(new Container);

        $this->assertSame($container, Container::getInstance());

        Container::setInstance(null);

        $container2 = Container::getInstance();

        $this->assertInstanceOf(Container::class, $container2);
        $this->assertNotSame($container, $container2);
    }

    public function testClosureResolution()
    {
        $container = new Container;
        $container->bind('name', function () {
            return 'Taylor';
        });
        $this->assertSame('Taylor', $container->make('name'));
    }

    public function testSharedClosureResolution()
    {
        $container = new Container;
        $container->singleton('class', function () {
            return new stdClass;
        });
        $firstInstantiation  = $container->make('class');
        $secondInstantiation = $container->make('class');
        $this->assertSame($firstInstantiation, $secondInstantiation);
    }

    public function testAutoConcreteResolution()
    {
        $container = new Container;
        $this->assertInstanceOf(ContainerConcreteStub::class, $container->make(ContainerConcreteStub::class));
    }

    public function testSharedConcreteResolution()
    {
        $container = new Container;
        $container->singleton(ContainerConcreteStub::class);

        $var1 = $container->make(ContainerConcreteStub::class);
        $var2 = $container->make(ContainerConcreteStub::class);
        $this->assertSame($var1, $var2);
    }

    public function testAbstractToConcreteResolution()
    {
        $container = new Container;
        $container->bind(IContainerContractStub::class, ContainerImplementationStub::class);
        $class = $container->make(ContainerDependentStub::class);
        $this->assertInstanceOf(ContainerImplementationStub::class, $class->impl);
    }

    public function testNestedDependencyResolution()
    {
        $container = new Container;
        $container->bind(IContainerContractStub::class, ContainerImplementationStub::class);
        $class = $container->make(ContainerNestedDependentStub::class);
        $this->assertInstanceOf(ContainerDependentStub::class, $class->inner);
        $this->assertInstanceOf(ContainerImplementationStub::class, $class->inner->impl);
    }

    public function testContainerIsPassedToResolvers()
    {
        $container = new Container;
        $container->bind('something', function ($c) {
            return $c;
        });
        $c = $container->make('something');
        $this->assertSame($c, $container);
    }


    public function testBindingsCanBeOverridden()
    {
        $container = new Container;
        $container->bind('foo', function () {
            return 'bar';
        });
        $container->bind('foo', function () {
            return 'baz';
        });
        $this->assertSame('baz', $container->make('foo'));
    }

    public function testBindingAnInstanceReturnsTheInstance()
    {
        $container = new Container;

        $bound    = new stdClass;
        $resolved = $container->instance('foo', $bound);

        $this->assertSame($bound, $resolved);
    }

    public function testResolutionOfDefaultParameters()
    {
        $container = new Container;
        $instance  = $container->make(ContainerDefaultValueStub::class);
        $this->assertInstanceOf(ContainerConcreteStub::class, $instance->stub);
        $this->assertSame('taylor', $instance->default);
    }


    public function testInternalClassWithDefaultParameters()
    {
        $container = new Container;
        $container->make(ContainerMixedPrimitiveStub::class, ['first' => 9, 'last' => 9]);
    }

    public function testForgetInstanceForgetsInstance()
    {
        $container             = new Container;
        $containerConcreteStub = new ContainerConcreteStub;
        $container->instance(ContainerConcreteStub::class, $containerConcreteStub);
        $this->assertTrue($container->isShared(ContainerConcreteStub::class));
        $container->forgetInstance(ContainerConcreteStub::class);
        $this->assertFalse($container->isShared(ContainerConcreteStub::class));
    }

    public function testForgetInstancesForgetsAllInstances()
    {
        $container              = new Container;
        $containerConcreteStub1 = new ContainerConcreteStub;
        $containerConcreteStub2 = new ContainerConcreteStub;
        $containerConcreteStub3 = new ContainerConcreteStub;
        $container->instance('Instance1', $containerConcreteStub1);
        $container->instance('Instance2', $containerConcreteStub2);
        $container->instance('Instance3', $containerConcreteStub3);
        $this->assertTrue($container->isShared('Instance1'));
        $this->assertTrue($container->isShared('Instance2'));
        $this->assertTrue($container->isShared('Instance3'));
        $container->forgetInstances();
        $this->assertFalse($container->isShared('Instance1'));
        $this->assertFalse($container->isShared('Instance2'));
        $this->assertFalse($container->isShared('Instance3'));
    }

    public function testResolvedResolvesAliasToBindingNameBeforeChecking()
    {
        $container = new Container;
        $container->bind('ConcreteStub', function () {
            return new ContainerConcreteStub;
        }, true);

        $this->assertFalse($container->resolved('ConcreteStub'));

        $container->make('ConcreteStub');

        $this->assertTrue($container->resolved('ConcreteStub'));
    }

    public function testResolvingWithArrayOfParameters()
    {
        $container = new Container;
        $instance  = $container->make(ContainerDefaultValueStub::class, ['default' => 'adam']);
        $this->assertSame('adam', $instance->default);

        $instance = $container->make(ContainerDefaultValueStub::class);
        $this->assertSame('taylor', $instance->default);

        $container->bind('foo', function ($app, $config) {
            return $config;
        });

        $this->assertEquals([1, 2, 3], $container->make('foo', [1, 2, 3]));
    }

    public function testResolvingWithUsingAnInterface()
    {
        $container = new Container;
        $container->bind(IContainerContractStub::class, ContainerInjectVariableStubWithInterfaceImplementation::class);
        $instance = $container->make(IContainerContractStub::class, ['something' => 'laurence']);
        $this->assertSame('laurence', $instance->something);
    }

    public function testNestedParameterOverride()
    {
        $container = new Container;
        $container->bind('foo', function ($app, $config) {
            return $app->make('bar', ['name' => 'Taylor']);
        });
        $container->bind('bar', function ($app, $config) {
            return $config;
        });

        $this->assertEquals(['name' => 'Taylor'], $container->make('foo', ['something']));
    }

    public function testNestedParametersAreResetForFreshMake()
    {
        $container = new Container;

        $container->bind('foo', function ($app, $config) {
            return $app->make('bar');
        });

        $container->bind('bar', function ($app, $config) {
            return $config;
        });

        $this->assertEquals([], $container->make('foo', ['something']));
    }

    public function testSingletonBindingsNotRespectedWithMakeParameters()
    {
        $container = new Container;

        $container->singleton('foo', function ($app, $config) {
            return $config;
        });

        $this->assertEquals(['name' => 'taylor'], $container->make('foo', ['name' => 'taylor']));
        $this->assertEquals(['name' => 'abigail'], $container->make('foo', ['name' => 'abigail']));

    }

    public function testCanBuildWithoutParameterStackWithNoConstructors()
    {
        $container = new Container;
        $this->assertInstanceOf(ContainerConcreteStub::class, $container->build(ContainerConcreteStub::class));
    }

    public function testCanBuildWithoutParameterStackWithConstructors()
    {
        $container = new Container;
        $container->bind(IContainerContractStub::class, ContainerImplementationStub::class);
        $this->assertInstanceOf(ContainerDependentStub::class, $container->build(ContainerDependentStub::class));
    }

    public function testContainerKnowsEntry()
    {
        $container = new Container;
        $container->bind(IContainerContractStub::class, ContainerImplementationStub::class);
        $this->assertTrue($container->bound(IContainerContractStub::class));
    }

    public function testContainerCanBindAnyWord()
    {
        $container = new Container;
        $container->bind('Taylor', stdClass::class);
        $this->assertInstanceOf(stdClass::class, $container->make('Taylor'));
    }

    public function testContainerCanResolveClasses()
    {
        $container = new Container;
        $class     = $container->make(ContainerConcreteStub::class);

        $this->assertInstanceOf(ContainerConcreteStub::class, $class);
    }
}

```

