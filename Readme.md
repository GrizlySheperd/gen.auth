# Phoenix LiveView Multi-Role Authentication Guide

This document walks through setting up a Phoenix LiveView project with two separate roles: `User` and `Admin`. Each role has its own authentication flow, and access control is enforced so that only Admins can access Admin pages, while both Admins and Users can access User pages.

---

## Overview

* Authentication using `phx.gen.auth` for both roles
* Role-based access control to LiveView pages
* Layout switching based on role

---

## 1. Create the Project

```bash
mix phx.new multi_auth_demo --live
cd multi_auth_demo
mix deps.get
```

---

## 2. Add Authentication for Users

```bash
mix phx.gen.auth Accounts User users
mix ecto.migrate
```

This generates LiveView-friendly registration, login, settings, and password reset for users.

---

## 3. Add Authentication for Admins

```bash
mix phx.gen.auth Admins Admin admins
mix ecto.migrate
```

This creates a separate admin authentication context with its own schema and table.

---

## 4. Define Role-Based Pipelines

In `lib/multi_auth_demo_web/router.ex`:

```elixir
pipeline :user_browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, {MultiAuthDemoWeb.Layouts, :user}
  plug :protect_from_forgery
  plug :put_secure_browser_headers
  plug MultiAuthDemoWeb.UserAuth
end

pipeline :admin_browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, {MultiAuthDemoWeb.Layouts, :admin}
  plug :protect_from_forgery
  plug :put_secure_browser_headers
  plug MultiAuthDemoWeb.AdminAuth
end
```

---

## 5. Define Routes by Role

```elixir
scope "/", MultiAuthDemoWeb do
  pipe_through [:user_browser, :require_authenticated_user]
  live "/user/dashboard", UserDashboardLive
end

scope "/admin", MultiAuthDemoWeb do
  pipe_through [:admin_browser, :require_authenticated_admin]
  live "/dashboard", AdminDashboardLive
end

scope "/admin", MultiAuthDemoWeb do
  pipe_through [:admin_browser, :maybe_authenticate_admin]
  live "/user/dashboard", UserDashboardLive
end
```

---

## 6. Create LiveViews

```bash
mix phx.gen.live Web UserDashboard dashboards --no-schema
mix phx.gen.live Web AdminDashboard dashboards --no-schema
```

Edit the generated files under `lib/multi_auth_demo_web/live/`.

---

## 7. Secure the Mount Lifecycle

In `UserDashboardLive`:

```elixir
def mount(_params, _session, socket) do
  if socket.assigns[:current_user] || socket.assigns[:current_admin] do
    {:ok, socket}
  else
    {:halt, Phoenix.LiveView.redirect(socket, to: ~p"/users/log_in")}
  end
end
```

In `AdminDashboardLive`:

```elixir
def mount(_params, _session, socket) do
  if socket.assigns[:current_admin] do
    {:ok, socket}
  else
    {:halt, Phoenix.LiveView.redirect(socket, to: ~p"/admin/log_in")}
  end
end
```

---

## 8. Custom Layouts for Roles

Create these files:

* `lib/multi_auth_demo_web/layouts/user.html.heex`
* `lib/multi_auth_demo_web/layouts/admin.html.heex`

These are automatically chosen based on the router pipeline used.

---

## 9. Admin Access to User Pages

In `AdminAuth`, define:

```elixir
def maybe_authenticate_admin(conn, _opts) do
  MultiAuthDemoWeb.AdminAuth.fetch_current_admin(conn, [])
end
```

This allows access to user areas for signed-in admins.

---

## 10. Access Permissions Summary

| Route              | User Access | Admin Access |
| ------------------ | ----------- | ------------ |
| `/user/dashboard`  | Yes         | Yes          |
| `/admin/dashboard` | No          | Yes          |

---

## Conclusion

This setup allows for a clean separation between regular users and administrative users while still allowing shared access where appropriate. It also fully leverages Phoenix LiveView and `phx.gen.auth` to keep your code maintainable and secure.
