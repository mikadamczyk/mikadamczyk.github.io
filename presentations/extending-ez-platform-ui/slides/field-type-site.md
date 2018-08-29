## Poll Field Type
### site

Note: 
Now we will try to use our new Field Type on the site. We need to save users 
votes in a database. So we need an entity, a repository to save votes, a form 
and a controller.


### Creating a `PollVote` Entity Class
#### We need those properties to represent vote
- question
- answer
- fieldId
- contentId

```bash
bin/console doctrine:schema:update --dump-sql
```
Note:
PollVote class represents and stores the data for a single vote
```sql
mysql -u admin -p ezplatform -e "our-sql"
```


### Final class `PollVoteRepository`
Note:
To be honest we will not need this class in this step, but we can create it anyway.
Wi will use it later, in one of the next steps.


### `PollType` class

```php
public function buildForm(FormBuilderInterface $builder, array $options)
{
    $choices = array_filter($options['answers'], function($value) {
        return $value !== null;
    });
    ...
}
```

```php
public function configureOptions(OptionsResolver $resolver)
{
    ....
    $resolver->setRequired('answers');
}
```
Note:
Because we need create vote, we are going to need to build a form. Because we did not 
perform answer validation we need to be sure that we exclude empty values. And we need
a way to pass possible answers to our form. 


### `FormFactory`

```php
    public function createPollForm(
        PollVote $data,
        ?string $name = null,
        array $answers
    ): FormInterface {
        $name = $name ?: StringUtil::fqcnToBlockPrefix(PollType::class);
        $options = null !== $answers ? ['answers' => $answers] : [];

        return $this->formFactory->createNamed(
            $name,
            PollType::class,
            $data,
            $options
        );
    }
```
Note:
Because we will use the form in more then one place we will create a very simple 
FormFactory class. It will create form for us. We will create 'createPollForm'. 
It will receive data class, the name of the form and answers which we will set as an
one of the options.


### Create a controller
#### `PollController::voteAction()`
We need a new method in it to handle voting action.

Note: 
Here we need a simple controller action to handle our request. It will get 
Content id from request object. Then we will load a content and get 
field value based on fieldIdentifier. This will be our answers. Then we will 
create data class, the entity. We will set fieldId and contentId. Then we will 
handle the request and if the form is submitted we will save the vote and render 
Success template.


### `ParameterProvider`
provides additional parameters to a Field Type's view template.

```php
class ParameterProvider implements ParameterProviderInterface
{
    public function getViewParameters(Field $field)
    {
        $pollData = new PollVote();
        $pollData->setQuestion($field->value->question);
        $pollForm = $this->formFactory->createPollForm($pollData, null, $field->value->answers);

        return [
            'pollForm' => $pollForm->createView(),
        ];
    }
}
```
Note:
We need pass the Form to our view FieldType view template. If you need to inject 
something to FieldType template, this is the easiest way to do this.


#### `ParameterProvider` needs to be registered in the `ParameterProviderRegistry`
```yml
services:
    ...
    AppBundle\eZ\Publish\FieldType\Poll\ParameterProvider:
        tags:
            - {name: ezpublish.fieldType.parameterProvider, alias: ezpoll}
```
Note:
Of course after we create the ParameterProvider class we need to register it in the 
ParameterProviderRegistry. We do this by tag our service with 
`ezpublish.fieldType.parameterProvider` and add 'ezpoll' alias which is 
FieldType identifier.  


#### Create Field Type template for the site SiteAccess

```twig
{# Resources/views/ezpoll_view.html.twig #}
{% raw %}
{% extends "EzPublishCoreBundle::content_fields.html.twig" %}

{% block ezpoll_field %}
    ...
    {{ form_start(parameters.pollForm, {'action': path('ez_systems_poll_vote', {
        'contentId': content.id,
        'fieldDefIdentifier': field.fieldDefIdentifier
    }), 'method': 'POST'}) }}
    {{ form_widget(parameters.pollForm) }}
    ...
{% endblock %}
{% endraw %}
```
Note:
In short, such a template must: extend EzPublishCoreBundle::content_fields.html.twig
, define a dedicated Twig block for the type, named by convention <TypeIdentifier_field>, in this case, ezpoll_field
, be registered in parameters.


#### Register Field Type template for the site SiteAccess
```yml
# Resources/config/ez_field_templates.yml
system:
    site:
        field_templates:
            - {template: 'AppBundle::ezpoll_view.html.twig', priority: 0}
```
Note:
To make template available, we must register it in the system. We will use 
a separate template for previewing the Field in the Site.
