# Django Users and Groups - Access Control Patterns

## Overview

Django provides several standard approaches for connecting users to groups and defining rights in different contexts. This document outlines the patterns from basic to advanced.

---

## 1. Django's Built-in Authorization System

### Model Permissions (Automatic)

Django automatically creates these permissions for each model:
- `add_modelname` - add permission
- `change_modelname` - change permission  
- `delete_modelname` - delete permission
- `view_modelname` - view permission (Django 2.1+)

```python
# In views or business logic
if user.has_perm('myapp.change_article'):
    # allow editing
```

### Custom Permissions

Define in your model's `Meta` class:

```python
class Article(models.Model):
    class Meta:
        permissions = [
            ("can_publish", "Can publish articles"),
            ("can_feature", "Can feature articles"),
        ]
```

### Groups & User Assignment

```python
# Assign permissions to groups
editors_group = Group.objects.get(name='Editors')
permission = Permission.objects.get(codename='can_publish')
editors_group.permissions.add(permission)

# Add user to group
user.groups.add(editors_group)
```

---

## 2. Context-Specific Rights

Django's built-in system is **model-level** only. For **context-specific** permissions (e.g., "this user can edit THIS article", or "rights vary by organization/project"), you need additional patterns.

### Option A: django-guardian (Object-Level Permissions)

```python
# Per-object permissions
from guardian.shortcuts import assign_perm

# User can only edit specific article
assign_perm('change_article', user, article)

# Check in views
if user.has_perm('myapp.change_article', article):
    # allowed
```

### Option B: Custom Context Field + Logic

```python
# Add context field to your permission model
class UserContextPermission(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    permission = models.CharField(max_length=100)
    context_type = models.CharField(max_length=50)  # 'org', 'project', etc.
    context_id = models.PositiveIntegerField()
    allowed = models.BooleanField(default=True)

# Usage
def has_context_perm(user, perm, context_type, context_id):
    try:
        cp = UserContextPermission.objects.get(
            user=user, 
            permission=perm, 
            context_type=context_type,
            context_id=context_id
        )
        return cp.allowed
    except UserContextPermission.DoesNotExist:
        return False
```

### Option C: Role-Based Context

```python
class Role(models.Model):
    name = models.CharField(max_length=50)
    permissions = models.ManyToManyField(Permission)

class UserRole(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    role = models.ForeignKey(Role, on_delete=models.CASCADE)
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)
```

---

## 3. Recommendation

For most applications:

1. **Start with Django's built-in permissions** for basic CRUD
2. **Use django-guardian** if you need per-object permissions
3. **For complex context hierarchies**, build a custom `UserContextPermission` model that links users/groups to permission sets with context fields

---

## 4. Next Steps / Questions

- What specific context hierarchy do you need? (multi-tenant, organization/project, draft/published workflow?)
- Do you need row-level permissions or just context-based role assignment?
- Are there inheritance patterns? (e.g., org-level permissions cascade to projects)
