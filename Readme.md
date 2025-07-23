# 🚀 Phoenix LiveView Multi-Role Authentication with `phx.gen.auth`

This guide outlines how to build a Phoenix LiveView project with two distinct roles: `User` and `Admin`, using `phx.gen.auth` for authentication and role-based access.

---

## 🎯 Goals

* ✅ Use `phx.gen.auth` for both `User` and `Admin` authentication
* ✅ Restrict access:

  * `Users` can only access user pages
  * `Admins` can access both admin and user pages
* ✅ Use LiveView for interactive dashboards

---

## 🧱 1. Create a New Phoenix Project

```bash
mix phx.new multi_auth_demo --live
cd multi_auth_demo
mix deps.get
```

---

## 🔐 2. Generate Authentication for Users

```bash
mix phx.gen.auth Accounts User users
mix ecto.migrate
```

* Context: `Accounts`
* Schema: `User`
* Table: `users`

---

## 🔐 3. Generate Authentication for Admins

```bash
mix phx.gen.auth Admins Admin admins
mix ecto.migrate
```

* Context: `Admins`
* Schema: `Admin`
* Table: `admins`

---

## 🛠 4. Define Auth Pipelines in Router

Edit `lib/multi_auth_demo_web/router.ex`:

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

## 🌐 5. Define Routes

```elixir
# User-only routes
scope "/", MultiAuthDemoWeb do
  pipe_through [:user_browser, :require_authenticated_user]
  live "/user/dashboard", UserDashboardLive
end

# Admin-only routes
scope "/admin", MultiAuthDemoWeb do
  pipe_through [:admin_browser, :require_authenticated_admin]
  live "/dashboard", AdminDashboardLive
end

# Admin accessing user dashboard
scope "/admin", MultiAuthDemoWeb do
  pipe_through [:admin_browser, :maybe_authenticate_admin]
  live "/user/dashboard", UserDashboardLive
end
```

---

## ⚡️ 6. Create LiveViews

```bash
mix phx.gen.live Web UserDashboard dashboards --no-schema
mix phx.gen.live Web AdminDashboard dashboards --no-schema
```

Edit the generated files under `lib/multi_auth_demo_web/live/` to add content.

---

## 🔒 7. Enforce Access Control in LiveViews

### `UserDashboardLive`

```elixir
def mount(_params, _session, socket) do
  if socket.assigns[:current_user] || socket.assigns[:current_admin] do
    {:ok, socket}
  else
    {:halt, Phoenix.LiveView.redirect(socket, to: ~p"/users/log_in")}
  end
end
```

### `AdminDashboardLive`

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

## 🎨 8. Optional: Add Role-Based Layouts

* `lib/multi_auth_demo_web/layouts/user.html.heex`
* `lib/multi_auth_demo_web/layouts/admin.html.heex`

These are configured via `put_root_layout` in router pipelines.

---

## 🧩 9. Admin Access to User Pages

In `AdminAuth`, add this plug:

```elixir
def maybe_authenticate_admin(conn, _opts) do
  MultiAuthDemoWeb.AdminAuth.fetch_current_admin(conn, [])
end
```

Then apply this in admin scopes that need access to user areas.

---

## 🧪 10. Access Matrix Summary

| Route              | User | Admin |
| ------------------ | ---- | ----- |
| `/user/dashboard`  | ✅    | ✅     |
| `/admin/dashboard` | ❌    | ✅     |

---

## ✅ Conclusion

You now have a Phoenix LiveView application with:

* Independent `User` and `Admin` authentication
* Proper access controls
* Role-aware dashboards using LiveView

Feel free to enhance this with dashboards, permissions, or API support!
