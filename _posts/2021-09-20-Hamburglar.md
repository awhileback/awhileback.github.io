---
layout: default
title: Hamburglar, Or.. How to get rid of CSS (mostly)
categories: [wagtail, css, grid]
---

<h1 align="center">{{ page.title }}</h1>
<br/>
<h3 align="center"> <i>&bull; {{ page.date | date: "%b %-d, %Y" }}</i></h3>
<br/>
<hr>

As web development passes its 30th anniversary, we can list some things that are ubiquitous in terms of website design.

1. Browsers render HTML elements, and its syntax is basically the same as it always has been.
2. Stylesheets control what the user sees in terms of layout, and they're still stored in plain old text files.
3. Databases provide dynamic content from the server side, while most of the advancement in terms of user-supplied content manipulation has become concentrated in javascript and its many frameworks.

Imagine for a moment you're explaining all of this to someone who has no idea how any of it works, let's say with a slideshow that includes pictures of servers, code blurbs, and all of that sort of thing.  If one of those three sections could leave them scratching their heads at the end of your lecture, wondering why this is done a certain way, I think it would likely be number two.

It's baffling that file-based UI and layout rules have made it this far. Even if you build a user interface completely in javascript and include stylesheeet particulars in your javascript code, it still winds up going through a build process that generates static layout rules, which live in a plain old text file. Assuming someone has hired a UI developer to design the front facing aspects of their app or site, the customer will be left with two choices: the plain text stylesheet option, or the javascript compiled option in which case the UI will be a mysterious black box unless the customer is capable of re-creating the developer's build process and tools, as well as learning how to edit the code. The CSS stylesheet file option at least gives the customer the option of editing the end result later, but it's still not in keeping with modern code collabortion standards.  If you have several people working on a file, how long before someone breaks something that is being used by someone else?  If you divide it all up into different files, then you have to agree on some sort of naming scheme to describe what the files all do, which presumes that a person in the future who had no hand at all in creating these files can discern the meaning of the names. On the other hand, with the compiled javascript option the customer probably wouldn't know where to begin if they wanted to edit what the UI developer built for them. 

All of this is a sort of weird musical chairs type exercise that dances around a seemingly obvious solution: the database that the site or app uses has field constraints, validation, and is connected via a back-end framework that allows a theoretically infinite amount of error handling code to be run, so why not put the UI definition in the database?

If we consider for a moment the assumption that the visual representation of a thing is poorly described by words, one could argue that text files with code and variables in them holding user interface particulars *was always wrong*. The tools to change this already exist in Wagtail/Django, although perhaps not explicitly designed for this purpose. ParentalKeys and Orderables allow Wagtail and Django to work with complex related data objects without first saving them to the database, and CSS grid layouts aren't as verbose as the old media-query CSS layouts that used to be required for layouts friendly to both mobile and desktop screens, so why can't you create a simple layout system that the user/editor can visually change, per page?

Lets take a simple variation of the "Page" class that ships with Wagtail and do just that: give users/editors a visual layout tool that they can use on their own without having to rely on an external library.

First give the user/editor some pre-defined common fit and finish choices in your main app's **models.py** , like so:

```python
    'CSS_ALIGN_CHOICES': (
        (None, 'Default'),
        ('center', 'Center'),
        ('stretch', 'Stretch'),
        ('start', 'Start'),
        ('end', 'End'),
    ),
    
    'CSS_FIT_CHOICES': (
        ('auto-fit', 'Auto-Fit'),
        ('auto-fill', 'Auto-Fill'),
    ),
```

Next let's define two orderable classes that define a CSS grid layout. One that's sort of free-form:
        
```python
    class CssGridLayout(Orderable):
        page = ParentalKey('Page', on_delete=models.CASCADE, related_name='css_grid_layout')
        layout_id = models.CharField(
            max_length=25,
            blank=False,
            null=False
            help_text=_('The ID of the container div.'),
        )
        grid_align = models.CharField(
            max_length=25,
            blank=True
            null=True,
            choices=CSS_ALIGN_CHOICES,
            default=None,
            help_text=_('Align (align vertically) items in this grid? If so, which way...')
        )
        grid_justify = models.CharField(
            max_length=25,
            blank=True,
            null=True,
            choices=CSS_ALIGN_CHOICES,
            default=None,
            help_text=_('Justify (align horiztontally) items in this grid? If so, which way...')
        )
        grid_css = models.TextField(
            blank=False,
            null=False,
            default=' ',
            help_text=_('The grid css code. Click "CSS Grid Tool" to draw one.'),
        )
        panels = [
            MultiFieldPanel(
              [
                 FieldPanel('layout_id'),
                 FieldPanel('grid_align'),
                 FieldPanel('grid_justify'),
                 FieldPanel('grid_css'),
              ],
              _('CSS Grid')
            )
         ]
```

... and another which holds all the data required for a basic responsive css grid row:

```python
    class CssGridRow(Orderable):
        page = ParentalKey('Page', on_delete=models.CASCADE, related_name='css_grid_row')
        row_id = models.CharField(
            max_length=25,
            blank=False,
            null=False,
            help_text=_('The ID of the container div.'),
        )
        grid_align = models.CharField(
            max_length=25,
            blank=True,
            null=True,
            choices=CSS_ALIGN_CHOICES,
            default=None,
            help_text=_('Align (align vertically) items in this grid? If so, which way...')
        )
        grid_justify = models.CharField(
            max_length=25,
            blank=True,
            null=True,
            choices=CSS_ALIGN_CHOICES,
            default=None,
            help_text=_('Justify (align horiztontally) items in this grid? If so, which way...')
        )
        grid_autofit = models.CharField(
            max_length=25,
            blank=False,
            null=False,
            choices=CSS_FIT_CHOICES
            help_text=_('Fit adds more elements, Fill expands the size of the elements.')
        )
        grid_minimum = models.SmallIntegerField(
            blank=False,
            null=False,
            help_text=_('Pixel size cells cannot be LESS than. Use with autofit.')
        )
        panels = [
            MultiFieldPanel(
              [
                 FieldPanel('row_id'),
                 FieldPanel('grid_align'),
                 FieldPanel('grid_justify'),
                 FieldPanel('grid_autofit'),
                 FieldPanel('grid_minimum'),
              ],
              _('CSS Grid Responsive Row')
            )
         ]
```

In your page class, include the two Orderable classes together in a MultiFieldPanel, like so.

```python
    class MyPage(Page):
        ...
        content_panels = Page.content_panels + [
            MultiFieldPanel(
                [
                    InlinePanel('css_grid_row', label='Grid row'),
                    InlinePanel('css_grid_layout', label='Grid layout'),
                ],
                heading=_('CSS grids'),
                classname='collapsible'
            )
        ]
```

As the first Orderable class suggests, there's a button in this example implementation that launches a visual CSS grid editor.  I have implemented this as a plain Django view that launches the visual editor as a template.  I'm using [this one](https://github.com/sdras/cssgridgenerator){:target="_blank" rel="noopener"} with some [slight modifications](https://github.com/awhileback/cssgridgenerator){:target="_blank" rel="noopener"} to make the colors look Wagtail admin'ish, provide some extra hints for basic usage, and slighly modify the defaults.  As the forked repo's readme suggests, simply download the code from the repo and run:

```
    yarn install
    yarn run build --nomaps
```

... then you should have a single file containing all of the app's compiled javascript code named "output.html" in your dist folder, which we'll use in the Wagtail editor.

To add the button to Wagtail's editor pages, first add a view for the CSS editor in your main app's **views.py** file, like so:

```python
    def csstool(request):
        return render(
            request,
            'wagtailadmin/pages/csstool/output.html'
        )
```

As the view suggests you'll need to put the template in that location, so in your main app's templates folder make the folder structure listed in the above view and put the output.html file in it:

```
    myapp
      templates
        wagtailadmin
          pages
            csstool
              output.html
```

Then, add a url for the view you just created, in your main app's **urls.py** (must be ABOVE the wagtail urls include):

```python
    from .views import csstool
        
    urlpatterns = [
       ...
         # css grid tool
         re_path(r'^csstool/$', csstool, name='css_tool'),
         # wagtail urls
         path("", include(wagtail_urls)),
```

Now add the button to the Wagtail admin for the page edit and page create templates. You'll need to add the relevant blocks to a sub-template in your main app's templates folder like so:

```
    myapp
      templates
        wagtailadmin
          pages
            edit.html
```

For **create.html** and **edit.html**, you'll need to copy over the block that contains the button you want to put the "CSS Grid Tool" button next to.  In my case I put it next to the "view live" button that's present on the **edit.html** template:

```html
    {% raw %}{% extends wagtailadmin/pages/edit.html %}
    {% load wagtailadmin_tags %}
    {% load i18n %}
    {% load l10n %}

    {% block content %}
    <div id="comments"></div>
    {% page_permissions page as page_perms %}
    <header class="merged tab-merged">
        {% explorer_breadcrumb page %}

        <div class="row">
            <h1 class="left col9 header-title" title="{% blocktrans with
                title=page.get_admin_display_title 
                page_type=content_type.model_class.get_verbose_name %}
                Editing {{ page_type }}{% endblocktrans %}">
                {{ page.get_admin_display_title }}
            </h1>

            <div class="right col3">
              {% include "wagtailadmin/pages/_page_view_live_tag.html" with page=page_for_status %}
            </div>
            <div class="right col3">
              {% include "wagtailadmin/pages/_css_grid_editor_button.html" with page=page_for_status %}
            </div>
        </div>
      ....
    {% endblock %}{% endraw %}
```

Similarly, for the **create.html** template place the button in the same location:

```html
     myapp
      templates
        wagtailadmin
          pages
            create.html

    {% raw %}{% extends "wagtailadmin/page/create.html" %}
    {% load wagtailadmin_tags %}
    {% load i18n %}
    {% load l10n %}
    {% block content %}
    <div id="comments"></div>
    <header class="merged tab-merged">
        {% explorer_breadcrumb parent_page include_self=1 trailing_arrow=True %}

        <div class="row">
            <div class="left col9 header-title">
                <h1>{% trans 'New' %} <span>{{ content_type.model_class.get_verbose_name }}</span></h1>
            </div>
            <div class="right col3">
              {% include "wagtailadmin/pages/_css_grid_editor_button.html" with page=page_for_status %}
            </div>
        </div>
        ....
    {% endblock %}{% endraw %}
```

As the template includes suggest there is also a template file for the new button itself, so create **\_css_grid_editor_button.html** as well, like so:

```html
     myapp
      templates
        wagtailadmin
          pages
            _css_grid_editor_button.html

    {% raw %}{% load i18n wagtailadmin_tags wagtailcore_tags %}
    {% wagtail_site as current_site %}
    <a href="{{ current_site.root_url }}/csstool/" 
        target="_blank" class="button button-nostroke" 
        title="{% trans 'CSS Grid Tool' %}" style="margin:2em;">
        {% icon name="link-external" class_name="initial" %}
        {% trans "Css Grid Tool" %}{% endraw %}
    </a>
```

Now you'll need to add loops to the wagtail page rendering template's head section to render the dynamically created styles.  Assuming your page template is:

```html
    myapp
      templates
        pages
          web_page.html
                    
    {% raw %}{% load static myapp_tags i18n 
        wagtailcore_tags wagtailimages_tags 
        wagtailsettings_tags wagtailuserbar %}
    {% wagtail_site as current_site %}
    {% get_settings %}
    {% get_current_language as LANGUAGE_CODE %}


    <!DOCTYPE html>
    <html lang="{{ LANGUAGE_CODE }}">
    {% endif %}
    <head>
    ...

    {% block inline_styles %}
    {% if self.css_grid_layout %}
    {% for grid in self.css_grid_layout.all %}
    <style>
    {% with grid.layout_id as gridid %}
    {% with grid.grid_align as gridalign %}
    {% with grid.grid_justify as gridjustify %}
    {{ grid.grid_css|cssid:gridid|cssalign:gridalign|cssjustify:gridjustify }}
    {% if request.is_preview %}
    #{{ gridid }} .parent-grid [class|="div"] { 
        border-color: pink; 
        border-style: dotted; 
        position: relative; 
        z-index: 999999; 
        border-width: 3px;
    }
    {% endif %}
    {% endwith %}
    {% endwith %}
    {% endwith %}
    </style>
    {% endfor %}
    {% endif %}
    {% if self.css_grid_layout %}
    {% for grid in self.css_grid_row.all %}
    <style>
    {% with grid.row_id as gridid %}
    {% with grid.grid_align as gridalign %}
    {% with grid.grid_justify as gridjustify %}
    #{{ gridid }} .parent-grid { 
    display: grid;
    grid-template-columns: repeat({% if grid.grid_autofit == 'auto-fit' %}
        auto-fit{% elif grid.grid_autofit == 'auto-fill' %}
        auto-fill{% endif %}, minmax({{ grid.grid_minimum }}px, 1fr));
    }
    {% if request.is_preview %}
    #{{ gridid }} .parent-grid [class|="div"] { 
        border-color: pink; 
        border-style: dotted; 
        position: relative; 
        z-index: 999999; 
        border-width: 3px;
    }
    {% endif %}
    {% endwith %}
    {% endwith %}
    {% endwith %}
    </style>
    {% endfor %}
    {% endif %}
    {% endblock %}
        ...
    </head>
    <body>
        ...
        {% block content %}
        {% block content_body %}
                {% include_block self.body %}
        {% endblock %}
        {% endblock %}
        </body>
    </html>{% endraw %}
```

Let's step through what this template is doing.  Firstly, check to see if there are any **self.css_grid_layout** entries in the page model instance. If so, loop over **self.css_grid_layout.all** naming the iterations "grid" and for each "grid" define the relevant field values as variables so that we can pass them through some template filters that apply some extra output to the code generated by the grid tool we created in the previous step.

If you played with the grid tool at all before installing it you'll notice that it spits out position-relative css code with css classes defined as **.parent-grid** and if you defined any divs within the grid you created, those are defined as **.div-1**, **.div-2** and so on.  This won't work out of the box, because our users/editors will be able to create multiple grids per page, that's one of the primary benefits of Orderables: the ability to dynamically make more than one of a set of fields.  So we will prefix each defined grid layout and grid row with the css ID that the user/editor must provide in the required **layout_id** and **row_id** fields.

Django does not inherently allow the passing of multiple function arguments to a template filter, normally you would just pass the value from the object you're working on as **value**.  We can overcome that limitation by explicitly declaring the values we need to pass along using the ["with" Django template tag](https://docs.djangoproject.com/en/stable/ref/templates/builtins/#with){:target="_blank" rel="noopener"} as named variables, though.  After the **with** statements above, we will pass our field that contains all of the css code the css gui tool created to three filters, **cssid**, **cssalign**, and **cssjustify**.

Open up your template tags file for your main app, and let's make some filters.

```python
    myapp
      templatetags
        pages
         myapp_tags.py
                 
    @register.filter
    def cssid(value, gridid):
        new_value = value.replace('.parent-grid', '#' + gridid + ' .parent-grid')
        return new_value


    @register.filter
    def cssalign(value, gridalign):
        if gridalign:
            align = 'align-items: ' + gridalign + ';' + '\r\n}'
            new_value = value.replace('}\r\n', align, 1)
        else:
            new_value = value
        return new_value

    @register.filter
    def cssjustify(value, gridjustify):
        if gridjustify:
            justify = 'justify-items: ' + gridjustify + ';' + '\r\n}\r\n'
            new_value = value.replace('}\r\n', justify, 1)
        else:
            new_value = value
        return new_value
```

First, **cssid** finds all instances of **.parent-grid**, which was generated by the css grid gui tool, and prefixes it with the parent div ID that the user/editor specified in the **layout_id** or **row_id** fields.  By declaring **with layout_id** or **with row__id** *as* **gridid**, the variable will have the same name whether we're looping over a grid layout or a grid row, so we can use the same template filter for both of them (and any future Orderable grid models you might create, as well).

Next, **cssalign** first checks to see if an alignment property was specified, and if the passed **gridalign** variable is not None, it builds the "align-items" css definition and replaces the closing css curly bracket from the passed CSS code with the "align-items" statement before the closing bracket.

Lastly, **cssjustify** does the exact same thing as **cssalign** for the "justify-items" variable, if it was provided.  If in either case **gridalign** or **gridjustify** are None, the filters simply return the original value, so that it may be rendered in the template unmodified.

And finally, for each grid defined, the template checks **request.is_preview** to see if the requested page is an admin preview. If it is, the template adds a secondary css definition for each grid that declares a dotted-line border in a bright-pink color for grid elements, with a z-index waayyyy on top of any other elements.  The wildcard statement **{% raw %}[class\|="div"]{% endraw %}** will match any css class that starts with "div-" so that it will catch all of the numbered divs that users/editors may define within a grid. This extra css on preview pages will give editors a nice wireframe overlay showing a visual representation of their grid layout so they can debug any errors in their content placement.

With all of this in place you are ready to rock and roll!  Create a new page, and you should see the "CSS Grid Tool" button linked at the top of the form, if you click it you'll be presented with a visual css grid editor in a new browser tab, in which you can specify some rows, columns, optionally some "gutter" padding between cells, all in a nice visual tool.  While you're in there, draw three random divs that span multiple cells in the grid you created.  When you're done, hit the "give me the code" button and copy your css code.

Back in the Wagtail admin, click the button to create a new grid layout, and paste the css code you generated into the textarea field, and give the **layout_id** field a name of "header" for testing purposes.

Then, if you add a simple **HtmlBlock** to the StreamField body for your page, and add the following HTML code in an HTML block:

```html
    <div id="header">
    <div class="parent-grid">
    <div class="div-1">test</div>
    <div class="div-2">test</div>
    <div class="div-3">test</div>
    </div>
    </div>
```

Hit the page preview, and you should be greeted with your "test" divs laid out in the grid that you drew in the gui, with nice pink dotted lines around them showing their placement on the page.  If you go back and repeat the process but create a "grid row" instead of a "grid layout" you will have a responsive grid row that line breaks as the screen size shrinks, at the points you specified in the form.

{:refdef: style="text-align: center;"}
![grids](/assets/images/001_1.jpg)
{: refdef}

At this point hopefully you can see the potential here, so all you have left to do to delete all of the layout related css files from your site is to add a block that contains a grid div id field, a grid div class field that gives the editor/user a list of choices of numbered divs to pick from (div-1, div-2, div-3 and so on up to an arbitrary number you think is plenty), adding that block to your other blocks and making them [StructBlocks](https://docs.wagtail.io/en/latest/advanced_topics/customisation/streamfield_blocks.html){:target="_blank" rel="noopener"}, and... of course... the long, arduous task of editing all of your block templates to include the divs with the proper variable names.  Or perhaps you want to implement the ability to let them make grid layouts as [Snippets](https://docs.wagtail.io/en/latest/topics/snippets.html){:target="_blank" rel="noopener"}, which is arguably a better place for all of this functionality to live I suppose. Perhaps this all inspires you to go all-out and *completely* eliminate static CSS from your site, and implement dynamic CSS rules for all of your StreamField blocks (hint, hint).

On the upside, once you're done with your layout generating vision, you can make artists into CSS developers, as they probably should always have been, if they only had the proper tools. Layers (which may be selectively overlapped), grids to specify content placement, and auto-flow settings are concepts they should already be familiar with since they're also present in print publication and graphic design tools. If you care to give them any advice, I would recommend that they delete all of those "hamburger" menus which people have forever tried to pass as responsive mobile menus, and add a blurb to their style guide that no one is to ever use one again. [CSS-Tricks has a great guide](https://css-tricks.com/look-ma-no-media-queries-responsive-layouts-using-css-grid/){:target="_blank" rel="noopener"} on how to create responsive header menus with css grid layouts to get them started on their way to creating real mobile-sized menus, rather than hamburger-sized menus.  They should find the guide pretty easy, since they don't have to write any code to make such a menu, rather just fill out some forms in their Wagtail admin.