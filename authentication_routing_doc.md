# 🔐 Authentication & Role-Based Routing in Phoenix (LiveView, Phoenix 1.7+)

## 📘 Overview

This guide explains how to:

- ✅ Add authentication to your Phoenix app
- ✅ Add a `role` column to users
- ✅ Implement **role-based routing** for `/admin` and `/user` pages using **LiveView**
- ✅ Avoid common Phoenix 1.7+ routing pitfalls

---

## 1. 🧪 Generate Authentication

```bash
mix phx.gen.auth Accounts User users --live
mix deps.get
mix ecto.migrate
```

---

## 2. 🏗️ Add a `role` Column to Users

```bash
mix ecto.gen.migration add_role_to_users
```

Update the generated migration:

```elixir
def change do
  alter table(:users) do
    add :role, :string, default: "user", null: false
  end
end
```

Then run:

```bash
mix ecto.migrate
```

---

## 3. 🧬 Update the User Schema

In `lib/your_app/accounts/user.ex`, add:

```elixir
schema "users" do
  field :role, :string
  ...
end
```

Update the registration changeset:

```elixir
def registration_changeset(user, attrs, opts \\ []) do
  user
  |> cast(attrs, [:email, :password, :role])
  |> validate_required([:email, :password])
  |> put_change(:role, Map.get(attrs, "role", "user"))
  |> validate_email(opts)
  |> validate_password(opts)
end
```

---

## 4. 🔀 Role-Based Redirects on Login

In `lib/your_app_web/controllers/user_session_controller.ex`:

```elixir
defp redirect_user_by_role(conn, user) do
  case user.role do
    "admin" -> redirect(conn, to: ~p"/admin")
    "user" -> redirect(conn, to: ~p"/user")
    _ -> redirect(conn, to: ~p"/")
  end
end
```

---

## 5. 🧭 Define LiveView Routes Properly

> ⚠️ All LiveView routes **must** be inside a `live_session` block.

✅ Example for Phoenix 1.7+:

```elixir
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
```

❌ Do **not** define live routes outside of `live_session`.

---

## 6. 💻 Implement the Page LiveView

Create `lib/your_app_web/live/page_live.ex`:

```elixir
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
        <h1>Welcome to the App</h1>
        <p>Please login or register to continue.</p>
        """
    end
  end
end
```

---

## 7. 👤 Seed an Admin User

Edit `priv/repo/seeds.exs`:

```elixir
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
```

Run:

```bash
mix run priv/repo/seeds.exs
```

---

## 8. 🚨 Troubleshooting: NoRouteError for LiveView

If you get:

```
Phoenix.Router.NoRouteError at GET /admin
```

### ✅ Solution:

- Ensure all live routes are in a `live_session` block
- Restart the Phoenix server

---

## 9. ✅ Final Checklist
[x] All LiveView routes are inside a live_session block.
[x] on_mount {DocumentationAppWeb.UserAuth, :mount_current_user} is used in LiveViews that need current_user.
[x] Role-based redirects and access control are implemented.
[x] Server is restarted after changes.

---

## 10. 💡 Tips

- Restart server after adding routes or files
- Test login/logout flow after schema changes
- Use your seeded admin to verify `/admin` access
- Avoid `{:halt, socket}` — use `push_redirect` or `redirect` properly

---

> 💬 Questions or feedback? PRs welcome!\
> ✨ Happy coding with Phoenix + LiveView!

