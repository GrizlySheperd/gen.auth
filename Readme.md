🔐 Authentication & Role-Based Routing in Phoenix (LiveView, Phoenix 1.7+)
Overview
This guide explains how to:

✅ Add authentication to your Phoenix app

✅ Add a role column to the users table

✅ Implement role-based routing for /admin and /user pages using LiveView

✅ Avoid common pitfalls (especially with Phoenix 1.7+ routing)

1. 🧪 Generate Authentication
Run the following to scaffold authentication:

mix phx.gen.auth Accounts User users --live
mix deps.get
mix ecto.migrate
2. 🏗️ Add a role Column to Users
Generate a migration:

mix ecto.gen.migration add_role_to_users
Edit the generated file in priv/repo/migrations/:

def change do
  alter table(:users) do
    add :role, :string, default: "user", null: false
  end
end
Then run:

mix ecto.migrate
3. 🧬 Update the User Schema
Update lib/your_app/accounts/user.ex:

Add to the schema:

schema "users" do
  field :role, :string
  ...
end
Modify registration_changeset/3:

def registration_changeset(user, attrs, opts \\ []) do
  user
  |> cast(attrs, [:email, :password, :role])
  |> validate_required([:email, :password])
  |> put_change(:role, Map.get(attrs, "role", "user"))
  |> validate_email(opts)
  |> validate_password(opts)
end
4. 🔀 Add Role-Based Redirects on Login
In lib/your_app_web/controllers/user_session_controller.ex, add:

defp redirect_user_by_role(conn, user) do
  case user.role do
    "admin" -> redirect(conn, to: ~p"/admin")
    "user" -> redirect(conn, to: ~p"/user")
    _ -> redirect(conn, to: ~p"/")
  end
end
Use it in the create/3 function.

5. 🧭 Define LiveView Routes (Correctly!)
⚠️ Important
All LiveView routes must be inside a live_session block.

✅ Correct usage for Phoenix 1.7+:

scope "/", YourAppWeb do
  pipe_through [:browser, :require_authenticated_user]

  live_session :require_authenticated_user,
    on_mount: [{YourAppWeb.UserAuth, :ensure_authenticated}] do

    live "/admin", PageLive, :admin
    live "/user", PageLive, :user

    live "/users/settings", UserSettingsLive, :edit
    live "/users/settings/confirm_email/:token", UserSettingsLive, :confirm_email
  end
end
❌ Do NOT define LiveView routes outside live_session — you'll get Phoenix.Router.NoRouteError.

6. 🧠 Implement the Page LiveView
Create lib/your_app_web/live/page_live.ex:

defmodule YourAppWeb.PageLive do
  use YourAppWeb, :live_view

  on_mount {YourAppWeb.UserAuth, :mount_current_user}

  def mount(_params, _session, socket), do: {:ok, socket}

  def render(assigns) do
    case assigns.live_action do
      :admin ->
        if assigns.current_user && assigns.current_user.role == "admin" do
          ~H"<h1>You are in the admin page</h1>"
        else
          ~H"<h1>Access denied</h1>"
        end

      :user ->
        if assigns.current_user && assigns.current_user.role == "user" do
          ~H"<h1>You are in the user page</h1>"
        else
          ~H"<h1>Access denied</h1>"
        end

      :home ->
        ~H"""
        <h1>Welcome to Documentation App</h1>
        <p>Please login or register to continue.</p>
        """
    end
  end
end
7. 👤 Seed an Admin User
In priv/repo/seeds.exs:

alias YourApp.Repo
alias YourApp.Accounts.User

unless Repo.get_by(User, email: "admin@gmail.com") do
  params = %{
    email: "admin@gmail.com",
    password: "eikaarylin1234"
  }

  %User{}
  |> User.registration_changeset(params)
  |> Ecto.Changeset.change(%{
    role: "admin",
    confirmed_at: DateTime.utc_now() |> DateTime.truncate(:second)
  })
  |> Repo.insert!()

  IO.puts("✅ Admin user created!")
end
Then run:

mix run priv/repo/seeds.exs
8. 🛠️ Troubleshooting: NoRouteError for LiveView
Error:

Phoenix.Router.NoRouteError at GET /admin
no route found for GET /admin (YourAppWeb.Router)
✅ Solution:

Ensure LiveView routes are inside a live_session

Restart Phoenix server after adding new routes or files

9. ✅ Final Checklist
 Live routes are inside a live_session

 on_mount {YourAppWeb.UserAuth, :mount_current_user} is used

 Role-based redirects and access control are working

 Server restarted after code changes

10. 💡 Tips
Restart your server after adding routes, migrations, or live components

Always test login/logout flow after schema changes

Use the seeded admin user to test /admin access

Avoid {:halt, socket} in LiveView; use push_redirect/2 or redirect/2 with {:ok, socket}
