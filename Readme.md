Technical Documentation: Authentication & Role-Based Routing in Phoenix (with LiveView)
Overview
This guide explains how to:
Add authentication to your Phoenix app
Add a role column to users
Implement role-based routing for /admin and /user pages using LiveView
Avoid common routing errors (especially with Phoenix 1.7+)
1. Generate Authentication
Run the following commands to scaffold authentication:
mix phx.gen.auth Accounts User users --live
mix deps.get
mix ecto.migrate
2. Add a role Column to Users
Generate a migration:
mix ecto.gen.migration add_role_to_users
Edit the generated migration file (in priv/repo/migrations/) to:
def change do
  alter table(:users) do
    add :role, :string, default: "user", null: false
  end
end           
Run the migration:
mix ecto.migrate
3. Update the User Schema
Edit lib/documentation_app/accounts/user.ex:
Add field :role, :string to the schema.
Update the registration_changeset/3 function to include :role and set a default:
def registration_changeset(user, attrs, opts \\ []) do
  user
  |> cast(attrs, [:email, :password, :role])
  |> validate_required([:email, :password])
  |> put_change(:role, Map.get(attrs, "role", "user"))
  |> validate_email(opts)
  |> validate_password(opts)
end
4. Update UserSessionController for Role-Based Redirects
Edit lib/documentation_app_web/controllers/user_session_controller.ex to:
Redirect users after login based on their role (admin or user)
Handle special login actions
Example:
defp redirect_user_by_role(conn, user) do
  case user.role do
    "admin" -> redirect(conn, to: ~p"/admin")
    "user" -> redirect(conn, to: ~p"/user")
    _ -> redirect(conn, to: ~p"/")
  end
end
5. Define LiveView Routes Correctly (Avoiding NoRouteError)
Important:
All LiveView routes (e.g., /admin, /user) must be inside a live_session block in your router.
Correct Example for Phoenix 1.7+:
scope "/", DocumentationAppWeb do
  pipe_through [:browser, :require_authenticated_user]

  live_session :require_authenticated_user,
    on_mount: [{DocumentationAppWeb.UserAuth, :ensure_authenticated}] do
    live "/admin", PageLive, :admin
    live "/user", PageLive, :user
    live "/users/settings", UserSettingsLive, :edit
    live "/users/settings/confirm_email/:token", UserSettingsLive, :confirm_email
  end
end
Do NOT define live routes outside of a live_session block or you will get Phoenix.Router.NoRouteError.
6. Implement the LiveView Page
Create or update lib/documentation_app_web/live/page_live.ex:
defmodule DocumentationAppWeb.PageLive do
  use DocumentationAppWeb, :live_view

  # Ensures current_user is available in assigns
  on_mount {DocumentationAppWeb.UserAuth, :mount_current_user}

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
7. Seed an Admin User
Edit priv/repo/seeds.exs:
alias DocumentationApp.Repo
alias DocumentationApp.Accounts.User

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
Run:
mix run priv/repo/seeds.exs
8. Troubleshooting: NoRouteError for LiveView
If you see:
Phoenix.Router.NoRouteError at GET /admin
no route found for GET /admin (DocumentationAppWeb.Router)
Solution:
Ensure your live routes are inside a live_session block in your router.
Restart your Phoenix server after making changes.
9. Final Checklist
[x] All LiveView routes are inside a live_session block.
[x] on_mount {DocumentationAppWeb.UserAuth, :mount_current_user} is used in LiveViews that need current_user.
[x] Role-based redirects and access control are implemented.
[x] Server is restarted after changes.
10. Additional Tips
Always restart your Phoenix server after adding new files or routes.
If you change the user schema or authentication logic, recompile and test login/logout flows.
Use the seeded admin user to test /admin access.
By following this documentation, future developers will avoid common pitfalls and errors, especially with LiveView routing in Phoenix 1.7+.
If you encounter any issues, check the router and LiveView setup as described above. - make it look better and more readable 
