---
layout: post
title: Modify a form in the Django admin panel based on the request
tags:
- Django
- Software
---

Sometimes it's tricky to customize the Django admin site. This particular issue plagued me for a good hour, and I wasn't able to find a solution online.

If you want to serve the admin forms differently based on the request & object, it's intuitive to override the `get_form` method.

```python
@admin.register(Something)
class SomethingAdmin(admin.ModelAdmin):
    form = SomethingForm
    fields = ('ownery', 'name')

    def get_form(self, request, obj=None, **kwargs):
        form = super(SomethingAdmin, self).get_form(request, obj=obj, **kwargs)
        # Do something here ...
        return form
```

Let's say you want to disable the `owner` field if the object already has an owner. Subclassing `forms.ModelForm` doesn't give you access to the form while rendering, so best to do it from the admin. The problem is that `get_form`, unintuitively, does not return a form but a form factory. So there's no `fields` attribute, only `base_fields`. If you change `base_fields`, you change *every* form, which is probably not what you want.

```python
# This has confusing results
def get_form(self, request, obj=None, **kwargs):
    form = super(SomethingAdmin, self).get_form(request, obj=obj, **kwargs)
    form.base_fields['owner'].disabled = obj is not None
    return form
```

Instead, you want to provide your own factory.

```python
@admin.register(Something)
class SomethingAdmin(admin.SomethingAdmin):
    form = SomethingForm
    # Note that if this attribute is removed the get_form method will fail
    fields = ('owner', 'name')

    def get_form(self, request, obj=None, **kwargs):
        """
        Modify the default factory to change form fields based on the request/object.
        """
        default_factory = super(SomethingAdmin, self).get_form(request, obj=obj, **kwargs)

        def factory(*args, **_kwargs):
            form = default_factory(*args, **_kwargs)
            return self.modify_form(form, request, obj, **_kwargs)

        return factory

    @staticmethod
    def modify_form(form, request, obj, **kwargs):
        """
        Disable the 'owner' field if it already exists.
        """
        form.fields['owner'].disabled = obj is not None and obj.owner is not None
        return form
```

