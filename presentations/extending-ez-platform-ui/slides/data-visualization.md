## Data visualization
### Creating a voting results list
Note:
We will create a list with all voting results. To do this we will need two 
new methods in PollController,   

### Bootstrap
We rely on Bootstrap's framework which makes eZ Platform UI easier to extend. 
It facilitates adapting and styling the interface to your needs.

Note: 
In our Admin UI v2 we are using Bootstrap in version 4.


### Add routing to list
```yml
ez_systems_poll_list:
    path:     /poll/list
    defaults: { _controller: AppBundle:Poll:list }

ez_systems_poll_show:
    path:     /poll/show/{fieldId}/{contentId}
    defaults: { _controller: AppBundle:Poll:show }
```


### Add two methods to a PollController
- `listAction(Request $request): Response`
- `showAction(Request $request): Response`


### Add a template
#### To easily customize the list view to the appearance of the Admin Panel it should extend:
```twig
{% raw %}
{% extends '@ezdesign/layout.html.twig' %}
{% endraw %}
```


### Some of the available blocks
```
{% raw %}
{% block body_class %}

{% block breadcrumbs %}

{% block page_title %}

{% block content %}
{% endraw %}
```
Note:
You can use those block to keep view the same as the rest of admin panel

### Pagination
#### To handle pagination we are using `WhiteOctoberPagerfantaBundle`, which is a great library for pagination.

[white-october/pagerfanta-bundle](https://github.com/whiteoctober/WhiteOctoberPagerfantaBundle)


```twig        
{% raw %}
{% if pager.haveToPaginate %}
   ...
   {{ pagerfanta(pager, 'ez') }}
{% endif %}
{% endraw %}
```
Note:
'ez' is the name of View to render Pagerfanta pagination
/vendor/ezsystems/ezplatform-admin-ui/src/bundle/View/EzPagerfantaView.php


### In the same way we can add detail view for the poll.
