# Tenzu Setup & Debugging Guide

This guide documents the critical configuration issues, permission bugs, and certificate setup steps required to successfully install and run Tenzu on a new machine from a fresh GitHub clone without encountering the auth, redirect, and TLS issues again.

---

## 1. CORS & Security (Local Development Settings)

By default, Tenzu expects a secure HTTPS production environment with specific CORS settings. For local development, these settings prevent the frontend from communicating with the backend.

### Backend CORS settings
Modify the development environment settings in `tenzu-back/.../dev.env` to allow HTTP connections and bypass strict cookie validation:
- Disable `SECURE_SSL_REDIRECT` or set it to `False`.
- Update `CORS_ALLOWED_ORIGINS` to include `http://localhost:4200` (or the local frontend URL).
- Set `SESSION_COOKIE_SECURE = False` and `CSRF_COOKIE_SECURE = False` so browser cookies work without TLS.

### Frontend API URL Configuration
Modify `tenzu-front/src/assets/configs/config.json` to route through HTTP instead of HTTPS if running local frontend:
```json
{
  "api": "http://localhost:8000"
}
```

---

## 2. Superuser Role Serialization Bug (Backend Fix)

### Symptom
Superusers (like `admin`) are redirected to the homepage `/` when trying to view any Kanban board or project detail page. The page appears briefly and then disappears.

### Root Cause
In `tenzu-back/src/permissions/__init__.py`, the `check_permissions` function checks `if user.is_superuser` and exits early before authorization components run. This skips setting the `user.project_role` dynamic attribute, causing the API serializer to return `userRole: null`. The frontend sees a null role and triggers a redirect, clearing state.

### Code Fix
Replace the `check_permissions` function in `tenzu-back/src/permissions/__init__.py` with the following:

```python
async def check_permissions(
    permissions: PermissionComponent,
    user: "AnyUser",
    obj: object = None,
) -> None:
    # Run the authorization check to trigger side-effects like setting
    # getattr(user, 'project_role') / getattr(user, 'workspace_role') via IsMember or HasPermission components.
    authorized = await permissions.is_authorized(user=user, obj=obj)
    
    if user.is_authenticated and user.is_superuser:
        return

    if not authorized:
        raise permissions.error
```

---

## 3. Creating and Seeding Admin Credentials & Memberships

If seed scripts do not automatically grant memberships or if you cannot log in, run the following commands to create the admin user and assign Owner permissions to all projects and workspaces.

### 1. Create Admin User
Inside the running backend container:
```bash
docker exec -it tenzu-dev-tenzu-back-1 python ./manage.py createsuperuser --username admin --email admin@example.com
```

### 2. Grant Owner Memberships
If the admin user cannot view projects or is missing workspace links, execute the following Django shell script to assign them as the "Owner" of all workspaces and projects:

```python
# Save this as seed_admin.py and run via:
# docker exec -i tenzu-dev-tenzu-back-1 python manage.py shell < seed_admin.py

import django
django.setup()
from users.models import User
from projects.projects.models import Project
from projects.memberships.models import ProjectMembership
from workspaces.workspaces.models import Workspace
from workspaces.memberships.models import WorkspaceMembership
from memberships.models import Role

user = User.objects.get(username="admin")

# Assign Workspace Owner Roles
for ws in Workspace.objects.all():
    role = ws.roles.filter(is_owner=True).first()
    if role:
        WorkspaceMembership.objects.get_or_create(
            user=user,
            workspace=ws,
            defaults={"role": role}
        )

# Assign Project Owner Roles
for project in Project.objects.all():
    role = project.roles.filter(is_owner=True).first()
    if role:
        ProjectMembership.objects.get_or_create(
            user=user,
            project=project,
            defaults={"role": role}
        )

print("Successfully seeded admin memberships!")
```

---

## 4. WebSocket & Caddy TLS issues (If using Caddy proxy)

If you configure the frontend to talk to Caddy over HTTPS, you may encounter certificate validation errors because of local self-signed certificates.
- Ensure you run `mkcert -install` on the host machine to trust local certificates.
- If using Caddy's auto-generated TLS, ensure Caddy is exposing port `80` and `443` correctly in `docker-compose.yml`, and your local `/etc/hosts` file resolves `tenzu.local` to `127.0.0.1`.
- If certificate errors persist, running the frontend locally pointed to `http://localhost:8000` (bypassing Caddy) resolves CORS and TLS problems.
