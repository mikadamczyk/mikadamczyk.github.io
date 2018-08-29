## Custom permissions
### Roles, Policies and Limitations


A bundle can expose Policies via a `PolicyProvider` which can be added to `EzPublishCoreBundle`'s DIC extension.


### PollPolicyProvider
#### must implement `PolicyProviderInterface`
```php
// Security/PollPolicyProvider.php
use eZ\Bundle\EzPublishCoreBundle\DependencyInjection\Configuration\ConfigBuilderInterface;
use eZ\Bundle\EzPublishCoreBundle\DependencyInjection\Security\PolicyProvider\PolicyProviderInterface;

class PollPolicyProvider implements PolicyProviderInterface
{
    public function addPolicies(ConfigBuilderInterface $configBuilder)
    {
    }
}
```


### Poll Policy configuration hash
Policies configuration hash contains declared modules, functions and limitations
```php
    [
        'poll' => [
            'list' => null,
            'show' => ['Question'],
        ],
    ];
```


#### Integrating the `PolicyProvider` into `EzPublishCoreBundle`
```php
// AppBundle.php
public function build(ContainerBuilder $container)
{
    parent::build($container);

    // Retrieve "ezpublish" container extension.
    $eZExtension = $container->getExtension('ezpublish');

    // Add the policy provider.
    $eZExtension->addPolicyProvider(
        new PollPolicyProvider()
    );
}
```


Add policy check to the `PollVoteRepository` methods

```php
// Repository/PollVoteRepository.php
public function findAllOrderedByQuestion()
{
    if ($this->permissionResolver->hasAccess('poll', 'list') !== true) {
        throw new UnauthorizedException('poll', 'list');
    }
    ...
}
```


### Custom Limitation
#### provide the restrictions you can apply to a given access right to limit the right according to certain conditions
- `QuestionLimitation` represents the value
- `QuestionLimitationType` deals with the business logic


### `QuestionLimitation` class

```php
use eZ\Publish\API\Repository\Values\User\Limitation;

class QuestionLimitation extends Limitation
{
    public const QUESTION = 'Question';

    public function getIdentifier(): string
    {
        return self::QUESTION;
    }
}
```


### `QuestionLimitationType` 
```php
// Security/Limitation/QuestionLimitationType.php

use eZ\Publish\Core\Limitation\AbstractPersistenceLimitationType;
use eZ\Publish\SPI\Limitation\Type as SPILimitationTypeInterface;

class QuestionLimitationType 
    extends AbstractPersistenceLimitationType 
    implements SPILimitationTypeInterface {
    
    }
```


#### `acceptValue()` 
Accepts a Limitation value and checks for structural validity. Makes sure 
`LimitationValue` object and `LimitationValue->limitationValues` is of correct type.


#### `validate()` 
Makes sure `LimitationValue->limitationValues` is valid according to valueSchema().


#### `buildValue()` 
Create the Limitation Value.


#### `evaluate()`
Evaluate permission against content & target(placement/parent/assignment).

```php
public function evaluate(
    APILimitationValue $value, 
    APIUserReference $currentUser,
    ValueObject $object, 
    array $targets = nullss)
{
    ...
    return in_array($object->contentInfo->id, $value->limitationValues);
}
```


#### Limitations need to be implemented as Limitation types and declared as services identified with ezpublish.limitationType tag. 

```yml
# Resources/config/services.yml
    AppBundle\Security\Limitation\QuestionLimitationType:
        arguments: ["@ezpublish.api.persistence_handler"]
        tags:
            - {name: ezpublish.limitationType, alias: Question }
            
# Name provided in the hash for each Limitation is the same 
# value set in the alias attribute in the service tag.

```


### Integrating custom Limitation types with the UI

```php
use eZ\Publish\API\Repository\Values\User\Limitation;
use Symfony\Component\Form\FormInterface;

// Provide support for editing custom policies in Platform UI
interface LimitationFormMapperInterface
{
    public function mapLimitationForm(FormInterface $form, Limitation $data);

    public function getFormTemplate();

    public function filterLimitationValues(Limitation $limitation);
}
```


#### Register the service in DIC with the `ez.limitation.formMapper` tag 

```yml
# Resources/config/services.yml
    AppBundle\Security\Limitation\Mapper\QuestionLimitationFormMapper:
        calls:
            - [setFormTemplate, ["EzSystemsRepositoryFormsBundle:Limitation:base_limitation_values.html.twig"]]
        tags:
            - { name: ez.limitation.formMapper, limitationType: Question }
```
set the limitationType attribute to the Limitation type's identifier and set Twig template to use to render the limitation form


We are extending `MultipleSelectionBasedMapper` to choose multiple polls
```php
// Security/Limitation/Mapper/QuestionLimitationFormMapper.php
class QuestionLimitationFormMapper extends MultipleSelectionBasedMapper implements LimitationValueMapperInterface
{
}
```


`getSelectionChoices()` returns value choices to display, as expected by the "choices" option from Choice field.


To provide human-readable names of the custom Limitation values, we will implement `LimitationValueMapperInterface`
```php
interface LimitationValueMapperInterface
{
    /**
     * Map the limitation values, in order to pass them 
     * as context of limitation value rendering.
     *
     * @param Limitation $limitation
     * @return mixed[]
     */
    public function mapLimitationValue(Limitation $limitation);
}
```


register the service in DIC with the `ez.limitation.valueMapper tag`  
and set the `limitationType` attribute to Limitation type's identifier
```yml
# Resources/config/services.yml
    AppBundle\Security\Limitation\Mapper\QuestionLimitationFormMapper:
        ...
        tags:
            - { name: ez.limitation.valueMapper, limitationType: Question }
```


override the way of rendering custom Limitation values in the role view
```twig
// Resources/views/Limitation/question_limitation_value.html.twig
{% raw %}
{% block ez_limitation_question_value %}
    <span>{{ values|join(', ') }}</span>
{% endblock %}
{% endraw %}

```


Add the template to the configuration
```yml
# Resources/config/ez_field_templates.yml
ezpublish:
    system:
        admin_group:
            ...
            limitation_value_templates:
                - { template: AppBundle:Limitation:question_limitation_value.html.twig, priority: 0 }
```