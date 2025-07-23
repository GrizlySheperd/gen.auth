Phoenix LiveView Multi-Role Authentication with phx.gen.auth

This guide outlines how to build a Phoenix LiveView project with two distinct roles: User and Admin, using phx.gen.auth for authentication and role-based access.

Goals

Use phx.gen.auth for both User and Admin authentication.

Restrict access so:

Users can only access user pages.

Admins can access both admin and user pages.

Use LiveView for dashboards.

1. Create a New Phoenix Project

mix phx.new multi_auth_demo --live
cd multi_auth_demo
mix deps.get

2. Generate Authentication for Users

mix phx.gen.auth Accounts User users
mix ecto.migrate

Context: Accounts

Schema: User

Table: users

3. Generate Authentication for Admins

mix phx.gen.auth Admins Admin admins
mix ecto.migrate

Context: Admins

Schema: Admin

Table: admins

4. Define Auth Pipelines in Router

Update lib/multi_auth_demo_web/router.ex:

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

5. Define Routes

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

6. Create LiveViews

mix phx.gen.live Web UserDashboard dashboards --no-schema
mix phx.gen.live Web AdminDashboard dashboards --no-schema

These generate lib/multi_auth_demo_web/live/user_dashboard_live.ex and admin_dashboard_live.ex

7. Enforce Access Control in LiveViews

UserDashboardLive

def mount(_params, _session, socket) do
  if socket.assigns[:current_user] || socket.assigns[:current_admin] do
    {:ok, socket}
  else
    {:halt, Phoenix.LiveView.redirect(socket, to: ~p"/users/log_in")}
  end
end

AdminDashboardLive

def mount(_params, _session, socket) do
  if socket.assigns[:current_admin] do
    {:ok, socket}
  else
    {:halt, Phoenix.LiveView.redirect(socket, to: ~p"/admin/log_in")}
  end
end

8. Optional: Add Role-Based Layouts

lib/multi_auth_demo_web/layouts/user.html.heex

lib/multi_auth_demo_web/layouts/admin.html.heex

These are automatically selected via put_root_layout in the router pipelines.

9. Admin Access to User Pages

In AdminAuth, add a fallback plug:

def maybe_authenticate_admin(conn, _opts) do
  MultiAuthDemoWeb.AdminAuth.fetch_current_admin(conn, [])
end

Use this in the admin route that points to user pages.

10. Access Matrix Summary

Route

Access

/user/dashboard

User ✅



Admin ✅

/admin/dashboard

Admin ✅



User ❌

✅ Conclusion

You now have a Phoenix LiveView app with two role-based auth flows using phx.gen.auth, each isolated but capable of shared access logic when desired.
