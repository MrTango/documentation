(interfaces)=

# Interfaces

## Introduction

Interfaces define what methods and attributes an object provides.
Plone extensively uses interfaces to define APIs between different subsystems.
They provide a more consistent and declarative way to define contracts between components than duck typing alone.

An interface defines the shape of a hole where different pieces fit.
The shape of the piece is defined by the interface, but the implementation details like color, material, etc. can vary.
This is a core concept in Zope's Component Architecture (ZCA), which Plone builds upon.

```{seealso}
-   [zope.interface documentation](https://zopeinterface.readthedocs.io/en/latest/)
-   [Zope Component Architecture documentation](https://zopecomponent.readthedocs.io/en/latest/)
```


## Common Interfaces

Some interfaces are commonly used throughout Plone.
A typical use case is specifying a context directive for a view, which determines where the view is available (for example, for which content types).

`zope.interface.Interface`
:   Base class for all interfaces.
    Also used as a wildcard when registering views, meaning the view applies to every object.

`Products.CMFCore.interfaces.IContentish`
:   All *content* items on the site.
    In the site root, this interface excludes Zope objects like `acl_users` (the user folder) and `portal_skins`, which might otherwise appear when iterating through root content.

`Products.CMFCore.interfaces.IFolderish`
:   All *folders* in the site.

`Products.CMFCore.interfaces.ISiteRoot`
:   The Plone site root object (CMF level).

`plone.base.interfaces.IPloneSiteRoot`
:   The Plone site root object (Plone-specific interface).

`plone.base.interfaces.INavigationRoot`
:   Navigation top object—where breadcrumbs are anchored.
    On multilingual sites, this is the top-level folder for the current language.

`plone.dexterity.interfaces.IDexterityContent`
:   All Dexterity content items.

`plone.dexterity.interfaces.IDexterityContainer`
:   All Dexterity folderish content items.


## Implementing Interfaces

Use the `@implementer()` decorator from `zope.interface` to declare that a class implements one or more interfaces.

```python
from zope.interface import implementer
from zope.interface import Interface


class IMyBehavior(Interface):
    """Marker interface for my custom behavior."""


class ILocalSyncedContent(Interface):
    """Marker interface for locally synced content."""


@implementer(IMyBehavior, ILocalSyncedContent)
class MyContent:
    """A content class implementing multiple interfaces."""
    pass
```

### Removing Parent Class Interface Implementations

Use `@implementer_only()` to redeclare all inherited interface implementations.
This is useful when you want to make interface bindings more accurate, for example with `z3c.form` widgets.

```python
from zope.interface import implementer_only
from zope.interface import Interface


class IAddressWidget(Interface):
    """Interface for address widget."""


@implementer_only(IAddressWidget)
class AddressWidget(BaseWidget):
    """Widget that only provides IAddressWidget, not parent interfaces."""
    pass
```


## Checking Whether an Object Provides an Interface

### Using `providedBy()`

In Python, you can check if an object provides an interface:

```python
from yourpackage.interfaces import IMyInterface

if IMyInterface.providedBy(obj):
    # obj provides the interface
    do_something()
else:
    # obj does not provide the interface
    handle_other_case()
```

### Using `plone_interface_info` in Templates

In page templates, you can use the `plone_interface_info` helper view:

```xml
<div tal:define="iinfo context/@@plone_interface_info">
    <span tal:condition="python:iinfo.provides('your.package.interfaces.IMyInterface')">
        Content for objects providing IMyInterface.
    </span>
</div>
```

```{seealso}
[plone.app.layout interface info source](https://github.com/plone/plone.app.layout/blob/master/plone/app/layout/globals/interface.py)
```

### Interface Resolution Order

Interface Resolution Order (IRO) is the list of interfaces provided by an object (directly or implemented by a class), sorted by priority.
Interfaces are evaluated from index zero (highest priority) to the last index (lowest priority).

You can access this information for debugging purposes:

```python
# Get the interface resolution order for an object
iro = obj.__provides__.__iro__
for interface in iro:
    print(interface)
```

```{note}
Since adapter factories are dynamic (adapter interfaces are not hardcoded on the object), the object can still adapt to interfaces not listed in `__iro__`.
```


## Getting an Interface's String Identifier

The interface identifier is stored in the `__identifier__` attribute:

```python
from zope.interface import Interface


class IFoo(Interface):
    """Example interface."""


# Returns 'yourpackage.interfaces.IFoo'
interface_id = IFoo.__identifier__
```

```{note}
This attribute does not respect import aliasing.
For example, `plone.app.contenttypes.interfaces.IDocument.__identifier__` returns `plone.app.contenttypes.interfaces.document.IDocument`.
```


## Getting an Interface Class by Its String Identifier

Use the `zope.dottedname` package to resolve dotted names to Python objects:

```python
from zope.interface import Interface
from zope.dottedname.resolve import resolve


class IFoo(Interface):
    """Example interface."""


# Get the identifier
interface_id = IFoo.__identifier__

# Resolve it back to the class
interface_class = resolve(interface_id)

assert IFoo is interface_class
```


## Applying Interfaces to Multiple Content Types

You can apply marker interfaces to content types to create a common denominator for otherwise unrelated classes.

Example use cases:

-   Assign a viewlet to a set of particular content types.
-   Enable certain behavior on specific content types.
-   Create a common registration point for adapters or views.

```{note}
A marker interface is needed only when you need to create a common denominator for several otherwise unrelated classes.
You can use an existing class or interface as a context without explicitly creating a marker interface.
Places accepting `zope.interface.Interface` as a context usually accept a normal Python class as well (`isinstance()` behavior).
```

You can assign a marker interface to several classes in ZCML using a `<class>` declaration.
Here we assign `IFeaturedContent` to Document and News Item content types:

```xml
<configure
    xmlns="http://namespaces.zope.org/zope"
    xmlns:five="http://namespaces.zope.org/five">

    <!-- Apply marker interface to Document -->
    <class class="plone.app.contenttypes.content.Document">
        <implements interface=".interfaces.IFeaturedContent" />
    </class>

    <!-- Apply marker interface to News Item -->
    <class class="plone.app.contenttypes.content.NewsItem">
        <implements interface=".interfaces.IFeaturedContent" />
    </class>

</configure>
```

Then you can register a view for these content types only:

```python
from Products.Five import BrowserView
from .interfaces import IFeaturedContent


class FeaturedView(BrowserView):
    """View available only for content with IFeaturedContent."""
    pass
```

```xml
<browser:page
    for=".interfaces.IFeaturedContent"
    name="featured-view"
    class=".views.FeaturedView"
    template="templates/featured.pt"
    permission="zope2.View"
    />
```


## Dynamic Marker Interfaces

Zope allows you to dynamically turn interfaces on and off on any content object through the Zope Management Interface (ZMI).
Browse to any object and visit the {guilabel}`Interfaces` tab.

Marker interfaces might need to be explicitly declared using ZCML so that Zope can find them:

```xml
<interface interface="your.package.interfaces.IPromotedContent" />
```

```{note}
The interface dotted name must refer directly to the interface class and not to an import from another module, such as `__init__.py`.
```

### Setting Dynamic Marker Interfaces Programmatically

Use `zope.interface.alsoProvides()` to add a marker interface to an object:

```python
from zope.interface import alsoProvides
from your.package.interfaces import IFeaturedContent

# Add the marker interface to the content object
alsoProvides(context, IFeaturedContent)
```

```{note}
This marking persists with the object; it is not temporary.
A persistent object (such as a content item) stores a reference to the interface class in its `__provides__` attribute.
This attribute is serialized and loaded by ZODB like any other reference to a class.
```

To remove a marker interface from an object, use `noLongerProvides()`:

```python
from zope.interface import noLongerProvides
from your.package.interfaces import IFeaturedContent

# Remove the marker interface from the content object
noLongerProvides(context, IFeaturedContent)
```

You can also use `directlyProvides()` to set the complete list of directly provided interfaces, replacing any existing ones:

```python
from zope.interface import directlyProvides
from your.package.interfaces import IFeaturedContent
from your.package.interfaces import IPromotedContent

# Set exactly these interfaces (replaces existing directly provided interfaces)
directlyProvides(context, IFeaturedContent, IPromotedContent)
```


## Tagged Values

Tagged values are arbitrary metadata you can attach to `zope.interface.Interface` subclasses.
For example, the `plone.autoform` package uses them to set form widget hints for `zope.schema` field declarations.

### Setting Tagged Values

```python
from zope.interface import Interface
from zope.interface import taggedValue
from zope import schema


class IMyContent(Interface):
    """Content type interface with tagged values."""

    taggedValue('custom_metadata', {'key': 'value'})

    title = schema.TextLine(title='Title')
```

### Accessing Tagged Values

```python
# Get a tagged value from an interface
value = IMyContent.getTaggedValue('custom_metadata')

# Get a tagged value with a default if not set
value = IMyContent.queryTaggedValue('custom_metadata', default=None)
```

```{seealso}
-   [plone.autoform documentation](https://pypi.org/project/plone.autoform/) for form hints using tagged values
-   [plone.supermodel documentation](https://pypi.org/project/plone.supermodel/) for XML-based schema definitions
```


## Schema Interfaces

In Plone, interfaces are often used to define content type schemas using `zope.schema` fields.
These schema interfaces define both the data model and can be used for form generation.

```python
from zope.interface import Interface
from zope import schema


class IBookContent(Interface):
    """Schema interface for a book content type."""

    title = schema.TextLine(
        title='Title',
        required=True,
    )

    description = schema.Text(
        title='Description',
        required=False,
    )

    isbn = schema.TextLine(
        title='ISBN',
        required=True,
    )

    publication_date = schema.Date(
        title='Publication Date',
        required=False,
    )
```

```{seealso}
-   {doc}`/backend/fields`
-   {doc}`/backend/behaviors`
```


## Interface Inheritance

Interfaces can inherit from other interfaces, creating a hierarchy:

```python
from zope.interface import Interface
from zope import schema


class IBasicContent(Interface):
    """Base interface with common fields."""

    title = schema.TextLine(title='Title')
    description = schema.Text(title='Description')


class INewsContent(IBasicContent):
    """Extended interface for news content."""

    publication_date = schema.Datetime(title='Publication Date')
    author = schema.TextLine(title='Author')
```

Objects implementing `INewsContent` will also be considered as implementing `IBasicContent`:

```python
from zope.interface import implementer


@implementer(INewsContent)
class NewsItem:
    pass


news = NewsItem()

# Both return True
INewsContent.providedBy(news)  # True
IBasicContent.providedBy(news)  # True
```


## Best Practices

1.  **Define interfaces in a separate module.**
    Create an `interfaces.py` file in your package to keep interface definitions organized.

2.  **Use marker interfaces sparingly.**
    Only create marker interfaces when you need to group unrelated classes.
    If classes share a common base, use that instead.

3.  **Document your interfaces.**
    Add docstrings to interfaces and their methods to describe the expected behavior.

4.  **Use schema interfaces for content types.**
    Define your content type fields in schema interfaces for better integration with Plone's form machinery.

5.  **Prefer `alsoProvides()` over `directlyProvides()`.**
    When adding marker interfaces dynamically, `alsoProvides()` is safer as it preserves existing interfaces.

```{seealso}
-   {doc}`adapters`
-   {doc}`utilities`
-   {doc}`/backend/behaviors`
```
