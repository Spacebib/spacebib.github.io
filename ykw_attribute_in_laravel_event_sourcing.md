# Laravel-event-sourcing v5 中的注解

Command **AddCartItem**:

```php
use Spatie\EventSourcing\Commands\HandleBy;
use Spatie\EventSourcing\Commands\AggregateUuid;

#[HandleBy(CartAggregateRoot::class)]
class AddCartItem
{
    public function __construct(
        #[AggregateUuid] public string $cartUuid,
        public string $cartItemUuid,
    )
}
```

只要在 command 添加这样的注解，command bus 就会通过`CartAggregateRoot::retrieve($cartUuid)` 找到对应的 Aggregate Root 并执行接收 `AddCartItem` 的方法。

首先看 PHP 文档：

> 注解功能使得代码中的声明部分都可以添加结构化、机器可读的元数据， 注解的目标可以是类、方法、函数、参数、属性、类常量。 通过 反射 API 可在运行时获取注解所定义的元数据。 因此注解可以成为直接嵌入代码的配置式语言。

> 注解声明总是以 #[ 开头，以 ] 结尾来包围。 内部则是一个或以逗号包含的多个注解。 注解的参数是可以选的，以括号()包围。 注解的参数可以是字面值或者常量表达式。 它同时接受位置参数和命名参数两种语法。

通过反射 API 请求注解实例时，注解的名称会被解析到一个类，注解的参数则传入该类的构造器中。 因此每个注解都需要引入一个类。在这里将其称为 *注解类*。

注解类如果要限制指定注解的声明类型，可为 `#[Attribute]` 注解第一个参数传入字节位掩码设置。

```php
use Attribute;

// 限制 AggregateUuid Attribute 只可以注解在 属性 上
#[Attribute(Attribute::TARGET_PROPERTY)]
class AggregateUuid
{
}

// 限制 HandledBy Attribute 只可以注解在 类 上
#[Attribute(Attribute::TARGET_CLASS)]
class HandledBy
{
    // 注解的参数会传入注解类的构造方法中
    public function __construct(
        public string $handlerClass
    ) {
    }
}
```

再看 Laravel-event-sourcing 是如何确定 Aggregate Root class 和 aggregate uuid：

```php
private function resolveHandler(): void
{
    // 通过 command 类的 HandledBy 注解拿到对应 Handler Class
    $attribute = (new ReflectionClass($this->command))->getAttributes(HandledBy::class)[0] ?? null;

    if (! $attribute) {
        throw new CommandHandlerNotFound($this->command::class);
    }
    // 注解类 HandleBy 的属性：handlerClass
    $handlerClass = ($attribute->newInstance())->handlerClass;

    // 如果 Handler 是 AggregateRoot 的子类时，尝试解析对应的 AggregateRootCommandHandler
    if (is_subclass_of($handlerClass, AggregateRoot::class)) {
        $this->resolveHandlerForAggregateRoot($handlerClass);

        return;
    }

    $this->handler = app($handlerClass);
}

private function resolveHandlerForAggregateRoot(string $handlerClass): void
{
    $this->resolveAggregateUuid();

    $this->handler = new AggregateRootCommandHandler(
        $handlerClass,
        $this->aggregateUuid
    );
}

// 从 command 的构造函数中通过 AggregateUuid 注解找到代表 uuid 的属性
private function resolveAggregateUuid()
{
    $constructorParameters = (new ReflectionClass($this->command))->getConstructor()->getParameters();

    $uuidField = null;

    foreach ($constructorParameters as $constructorParameter) {
        $attribute = $constructorParameter->getAttributes(AggregateUuid::class);

        if (! count($attribute)) {
            continue;
        }

        $uuidField = $constructorParameter->getName();

        break;
    }

    if (! $uuidField) {
        throw new MissingAggregateUuid($this->command::class);
    }
    // 取到 aggregate uuid
    $this->aggregateUuid = $this->command->{$uuidField};
}
```

 `AggregateRootCommandHandler` 执行时是如何找到对应的方法的：

```php
/**
 * 通过反射找到第一个参数类型为对应的 command 的方法 
 *
 * @param object | string $object command
 * @param object $handler AggregateRoot 或 AggregatePartial, 会在这个类里面找对应的方法
 *
 * @return Collection 返回的是方法名的集合
 */
public static function find(object | string $object, object $handler): Collection
{
    $eventName = is_object($object)
        ? $object::class
        : $object;

    $handlerClass = new ReflectionClass($handler);

    return collect($handlerClass->getMethods(ReflectionMethod::IS_PUBLIC | ReflectionMethod::IS_PROTECTED))
        ->filter(function (ReflectionMethod $method) use ($eventName) {
            // 只看方法的第一个参数
            $parameter = $method->getParameters()[0] ?? null;

            if (! $parameter) {
                return false;
            }

            $type = $parameter->getType();

            if (! $type) {
                return false;
            }
      
            /** @var \ReflectionNamedType[] $types 第一个参数的类型 */
            $types = match ($type::class) {
                ReflectionUnionType::class => $type->getTypes(),
                ReflectionNamedType::class => [$type],
            };

            return collect($types)
                ->contains(fn (ReflectionNamedType $type) => $type->getName() === $eventName);
        })
        ->values()
        ->map(fn (ReflectionMethod $method) => $method->getName());
}
```
