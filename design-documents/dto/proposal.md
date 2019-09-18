# DTO Management proposal

The current proposal is working and implemented in a WIP branch: https://github.com/magento/magento2/pull/23265 .

## Current situation
DTOs in Magento2 must be manually created as PHP classes and immutables are not currently supported by `\Magento\Framework\Api\DataObjectHelper::populateWithArray` method which is widely used in the core.

Most of the time, DTOs are simply used to transport immutable information, such as an API request or an API resonse. In this case and because of implementation limitatsion, each DTO must also declare setter methods even if it supposed to not change. This requirement can lead to unexpected conditions and can introduce mutations where they should not be allowed.

Another problem with the manual DTO creation is represented by the possibility for a non experienced developer to add side effects to getter methods.

## Proposal
The aim of this proposal is to switch from a manual DTO creation to an XML declarative way and an autogeneration of DTO PHP classes.

Other than speeding up the developing process of new features, this approach could avoid most of the human errors such as typos or side effects in DTOs.

### XML Syntax

The XML declarative DTO definition should be base on the following model:

```
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Dto:etc/dto.xsd">

    <dto
            id="test-dto"
            mutable="false"
    >
        <class>Test\DtoGenerator\Dto\Test</class>
        <interface>Test\DtoGenerator\Api\TestInterface</interface>
        <property name="test1" type="string" nullable="true" />
        <property name="test2" type="string" optional="true" />
        <property name="test3" type="string" />
        <property name="test_abc" type="string" />
    </dto>
</config>

```

The class name and the interface name should be declaread as fully qualified class names to allow a valid code generation.

### DTO mutability

A DTO can be specified with a `mutable` attribute. Depending on its value, setters methods will be created or not.
In case of `mutable` = `false`, a PSR-7 like interface will be applied by creating a `with` method for each property. BY calling a `with` method, a new instance of the object will be returned with the updated information.

Example:
```
$dto = $dto
    ->withTest1('another value')
    ->withTest2('something different');
```

#### DTO mutators
Since the PSR-7 like approach can lead to an high memory consumption in case of several fields update, an autogenerated `Mutator` class is created to help such process.
Mutator classes follow the builder model to store all the changed properties and return an updated object only at the end of the mutation process.

Example:
```
/** @var \Test\DtoGenerator\Dto\TestMutator $mutator */
$dto = $mutator
	->withTest1('another value')
    ->withTest2('something different')
    ->mutate($dto);

```

In this last case, **only a new instance** is created instead of 2.

**NOTE:**
Developers shold be aware that mutating several properties of an immutable can mean that the object should be considered as a mutable. Maybe a warning in static testing should be considered.

### Implementation consequences introdcuing immutables
The current core implementation deeply relies on `\Magento\Framework\Api\DataObjectHelper::populateWithArray` that requires an empty object to be created before hydrating it.

Example:
```
$address = $this->addressFactory->create();
$this->dataObjectHelper->populateWithArray($address, $data, AddressInterface::class);
```

This approach is of course not compatible with an immutable DTO since the DTO information should be vailable at the moment of its creation.

The solution proposed is to add a new method to `\Magento\Framework\Api\DataObjectHelper::createFromArray` and deprecate `populateWithArray`.

**\Magento\Framework\Api\DataObjectHelper::createFromArray:**
```
public function createFromArray(array $data, string $type)
{
    if ($this->dtoConfig->isDto($type)) {
        return $this->dtoProcessor->createFromArray($data, $type);
    }

    // Compatibility mode
    $dataObject = $this->objectFactory->create($type, []);
    $this->populateWithArray($dataObject, $data, $type);

    return $dataObject;
}
```

This new method should be used by `ServiceInputProcessor` and used in the recursive operation of `\Magento\Framework\Api\DataObjectHelper::_setDataValues` allowing a full backward compatibility.

#### The hydration strategy

The `createFromArray` method should be supporting both mutable and immutable DTOs, so a values injection srategy should be defined to allow the compatibility.

The proposal is to create a class `\Magento\Framework\Dto\DtoProcessor\GetHydrationStrategy` able to define the attributes injection strategy depending on code reflection. In this way we know whenever an attribute should be injected via constructor or setter.

### The extension attributes problem in immutables

One of the most logical effects of introducing immutable DTOs is related to **Extension Attributes**. Since Extension Attributes are `setter` based by designed, they are not supposed to be immutable, but an immutable DTO cannot have a mutable part.

The proposal is to define a new interface `\Magento\Framework\Api\ImmutableExtensibleDataInterface` not defining setter methods while code generating the extensible data class.

The open point now is how to inject values inside the extension classes if no setters are provided and extension must be hydrated only after the DTO creation.

#### The extension attributes injectors

The proposal is to modify the `extension_attributes.xsd` by adding the following specifications:

```
<xs:complexType name="injectorType">
    <xs:attribute type="xs:string" name="code" use="required"/>
    <xs:attribute type="xs:string" name="type" use="required"/>
</xs:complexType>
<xs:complexType name="extension_attributesType">
    <xs:sequence>
        <xs:element type="attributeType" name="attribute" minOccurs="0" maxOccurs="unbounded"/>
        <xs:element type="injectorType" name="injector" minOccurs="0" maxOccurs="unbounded"/>
    </xs:sequence>
    <xs:attribute type="xs:string" name="for" use="required"/>
</xs:complexType>
```

In this way, an `extension_attributes.xml` file may look like:

```
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Test\DtoGenerator\Api\TestInterface">
        <attribute code="test_attr" type="string" />
        <injector code="test_attr_injector" type="Test\DtoGenerator\Model\MyInjector" />
    </extension_attributes>
</config>

```

The class `Test\DtoGenerator\Model\MyInjector` must implement `\Magento\Framework\Api\ExtensionAttribute\InjectorProcessorInterface` and declare an `execute` method with the DTO data as array and return an array with the extension attributes values.

Unfortunately, at this stage, only an associative array can be used to manupulate the information, since the object should be created only after its dataset is defined.

Example:
```
<?php
declare(strict_types=1);

namespace Test\DtoGenerator\Model;

use Magento\Framework\Api\ExtensionAttribute\InjectorProcessorInterface;

class MyInjector implements InjectorProcessorInterface
{
    /**
     * Process object for injections
     *
     * @param string $type
     * @param array $objectData
     * @return array
     */
    public function execute(string $type, array $objectData): array
    {
        return ['test_attr' => 'Hello world'];
    }
}

```

The attributes extension injector mechanism could also be used in mutable DTOs or existing classes.

## Generated code examples

```
<?php
declare(strict_types=1);

namespace Test\DtoGenerator\Dto;

class Test implements \Test\DtoGenerator\Api\TestInterface
{
    /**
     * @var string
     */
    private $test1 = null;

    /**
     * @var string
     */
    private $test2 = null;

    /**
     * @var string
     */
    private $test3 = null;

    /**
     * @var string
     */
    private $testAbc = null;

    /**
     * @var \Test\DtoGenerator\Api\TestExtensionInterface
     */
    private $extensionAttributes = null;

    /**
     * @param string|null $test1
     * @param string $test2
     * @param string $test3
     * @param string $testAbc
     * @param \Test\DtoGenerator\Api\TestExtensionInterface|null $extensionAttributes
     */
    public function __construct(?string $test1, string $test3, string $testAbc, ?string $test2 = null, ?\Test\DtoGenerator\Api\TestExtensionInterface $extensionAttributes = null)
    {
        $this->test1 = $test1;
        $this->test2 = $test2;
        $this->test3 = $test3;
        $this->testAbc = $testAbc;
        $this->extensionAttributes = $extensionAttributes;
    }

    /**
     * @return string|null
     */
    public function getTest1() : ?string
    {
        return $this->test1;
    }

    /**
     * @return string|null
     */
    public function getTest2() : ?string
    {
        return $this->test2;
    }

    /**
     * @return string
     */
    public function getTest3() : string
    {
        return $this->test3;
    }

    /**
     * @return string
     */
    public function getTestAbc() : string
    {
        return $this->testAbc;
    }

    /**
     * @return \Test\DtoGenerator\Api\TestExtensionInterface|null
     */
    public function getExtensionAttributes() : ?\Test\DtoGenerator\Api\TestExtensionInterface
    {
        return $this->extensionAttributes;
    }

    /**
     * @param string|null $value
     * @return Test\DtoGenerator\Api\TestInterface
     */
    public function withTest1(?string $value) : \Test\DtoGenerator\Api\TestInterface
    {
        $dtoProcessor = \Magento\Framework\App\ObjectManager::getInstance()->get(\Magento\Framework\Dto\DtoProcessor::class);
        return $dtoProcessor->createUpdatedObjectFromArray($this, ['test1' => $value]);
    }

    /**
     * @param string $value
     * @return Test\DtoGenerator\Api\TestInterface
     */
    public function withTest2(string $value) : \Test\DtoGenerator\Api\TestInterface
    {
        $dtoProcessor = \Magento\Framework\App\ObjectManager::getInstance()->get(\Magento\Framework\Dto\DtoProcessor::class);
        return $dtoProcessor->createUpdatedObjectFromArray($this, ['test2' => $value]);
    }

    /**
     * @param string $value
     * @return Test\DtoGenerator\Api\TestInterface
     */
    public function withTest3(string $value) : \Test\DtoGenerator\Api\TestInterface
    {
        $dtoProcessor = \Magento\Framework\App\ObjectManager::getInstance()->get(\Magento\Framework\Dto\DtoProcessor::class);
        return $dtoProcessor->createUpdatedObjectFromArray($this, ['test3' => $value]);
    }

    /**
     * @param string $value
     * @return Test\DtoGenerator\Api\TestInterface
     */
    public function withTestAbc(string $value) : \Test\DtoGenerator\Api\TestInterface
    {
        $dtoProcessor = \Magento\Framework\App\ObjectManager::getInstance()->get(\Magento\Framework\Dto\DtoProcessor::class);
        return $dtoProcessor->createUpdatedObjectFromArray($this, ['testAbc' => $value]);
    }

    /**
     * @param \Test\DtoGenerator\Api\TestExtensionInterface|null $value
     * @return Test\DtoGenerator\Api\TestInterface
     */
    public function withExtensionAttributes(?\Test\DtoGenerator\Api\TestExtensionInterface $value) : \Test\DtoGenerator\Api\TestInterface
    {
        $dtoProcessor = \Magento\Framework\App\ObjectManager::getInstance()->get(\Magento\Framework\Dto\DtoProcessor::class);
        return $dtoProcessor->createUpdatedObjectFromArray($this, ['extensionAttributes' => $value]);
    }
}
```