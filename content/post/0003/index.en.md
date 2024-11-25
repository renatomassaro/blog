---
title: "Common pitfalls with Repo.transaction in Elixir"
date: 2024-11-24T20:09:31-03:00
categories: [programming]
tags: [Elixir]
image: cover.webp
---

This quick post highlights three common mistakes I often see with `Repo.transaction`. At the end, I’ve included a suggestion to help you and your team avoid these pitfalls.

## Gotcha 1: It may commit when you don't expect it to

This one is a big deal and so easy to miss: `Repo.transaction/2` will **always** commit, except when:

- An exception is raised; or
- `Repo.rollback/1` is called.

This is clearly stated in the [docs](https://hexdocs.pm/ecto/Ecto.Repo.html#c:transaction/2):

> If an Elixir exception occurs the transaction will be rolled back and the exception will bubble up from the transaction function. If no exception occurs, the transaction is committed when the function returns. A transaction can be explicitly rolled back by calling `rollback/1`.

And yet, it's surprisingly easy to overlook. I have missed it in the past and every now and then I spot this during code review, too.

### Examples

In the examples below, imagine `update_settings/2` returned an error tuple generated in the application layer. The block *will* commit and the changes made by `update_user/1` will persist even though `update_settings/2` has failed.

```elixir
# Example 1:
Repo.transaction(fn ->
  with {:ok, user} <- update_user(user_params),
       {:ok, settings} <- update_settings(user, settings_param) do
    %{user: user, settings: settings}
  end
end)

# Example 2:
Repo.transaction(fn ->
  {:ok, user} = update_user(user_params)

  case update_settings(user, settings_param) do
    {:ok, settings} ->
      settings

    {:error, reason} ->
      Logger.error("Failed to update settings: #{inspect reason}")
      {:error, reason}
  end
end)
```

For the example above, we are considering the error tuple was generated in the *application layer*. This means that the request never actually hit the database. As an example, maybe the Ecto changeset validations failed.

If there was an actual error from the database (say, due to a failed constraint check), then Ecto will rollback and raise an exception when another database operation is performed within the same transaction.


#### Make sure *all* branches are covered

In the example below, we do call `Ecto.rollback/1`, but there is at least one code path where `Ecto.rollback` is not called even though database operations were performed.

```elixir
Repo.transaction(fn ->
  with {:ok, user} <- update_user(user_params),
       # Suppose `has_permission_to_update_settings?/1` returned false
       true <- has_permission_to_update_settings?(user),
       {:ok, settings} <- update_settings(user, settings_param) do
    %{user: user, settings: settings}
  else
    {:error, reason} ->
      Ecto.rollback(reason)

    false ->
      # Code that reaches this branch never rolls back!
      Logger.error("User does not have permission to update settings")
      {:error, :no_permissions}
  end
end)
```

#### Experimenting locally

Curious about whether something commits? Open an IEx shell with access to a `Repo` and try the following:

```
iex(1)> Logger.configure(level: :debug)
:ok

iex(2)> Repo.transaction(fn -> :error end)
[debug] QUERY OK db=0.4ms idle=1939.9ms
begin []
[debug] QUERY OK db=0.2ms
commit []
{:ok, :error}

iex(3)> Repo.transaction(fn -> Repo.rollback(:error_updating_settings) end)
[debug] QUERY OK db=0.4ms idle=1795.7ms
begin []
[debug] QUERY OK db=0.2ms
rollback []
{:error, :error_updating_settings}

```

By executing the command with the `:debug` Logger level enabled, you'll see every query Ecto makes, including the commits/rollbacks. For a more accurate example, perform actual database operations -- as mentioned above, errors at the database layer will change the outcome of the function.

## Gotcha 2: It may not return what you are expecting

Both `Repo.transaction/2` and `Repo.rollback/1` have a slightly counterintuitive return type: it wraps the return type of the function into an `:ok` or `:error` tuple:

```
iex(1)> Repo.transaction(fn -> :it_commits end)
[debug] QUERY OK db=0.3ms idle=1941.2ms
begin []
[debug] QUERY OK db=0.2ms
commit []
{:ok, :it_commits}

iex(2)> Repo.transaction(fn -> Repo.rollback(:rollback) end)
[debug] QUERY OK db=0.4ms idle=1937.4ms
begin []
[debug] QUERY OK db=0.2ms
rollback []
{:error, :rollback}
```

This is less likely to go unnoticed because developers usually identify and adjust the return value during testing (manual or automatic).

It's worth pointing out this is also clearly stated in the docs:

> **Repo.transaction/2**: A successful transaction returns the value returned by the function wrapped in a tuple as `{:ok, value}`.<br/><br/>
> **Repo.rollback/1**: The transaction will return the value given as `{:error, value}`.

## Gotcha 3: Transactions are per-process

From the [docs](https://hexdocs.pm/ecto/Ecto.Repo.html#c:transaction/2-working-with-processes):

> The transaction is per process. A separate process started inside a transaction won't be part of the same transaction and will use a separate connection altogether.

This is a substantially less common gotcha, since most of the time when handling web requests we only need the one process serving the user. However, precisely *because* it's less common, this is particularly easy to miss when the exception does happen! So watch out if you are starting Tasks or hitting GenServers that perform database operations from their respective processes while wrapped in a transaction.

## Avoiding these gotchas from hitting prod

There really are only two ways: code reviews and tests.

It's not the purpose of this post to convince you or your company to adopt best practices -- but if you aren't doing code review or testing, I highly recommend you start doing so.

### Multi: an interesting alternative

Ecto provides [Multi](https://hexdocs.pm/ecto/Ecto.Multi.html), a "data structure for grouping multiple Repo operations" in its own words.

I have found Multi to have several advantages over passing a function to `Repo.transaction/2`, but for the scope of this post I'd like to focus on one: safety.

Because Multi uses pipes as first-class citizen, any time an error tuple (or unexpected return) shows up, the logic short-circuits to the `Repo.rollback/1` call, which is done automatically by Multi. In that sense, it's actually quite similar to a `with` block.

This is what our earlier examples would look like using Multi:

```elixir
Multi.new()
|> Multi.run(:update_user, fn _repo, _changes ->
  update_user(user_params)
end)
|> Multi.run(:update_settings, fn _repo, %{update_user: user} ->
  update_settings(user, settings_param)
end)
|> Repo.transaction()
```

Notice that the example above is inherently safe. If any operation returns an error tuple, it will go straight to the end. If an unexpected result is returned by the functions (say, `nil` instead of an ok/error tuple), an exception is raised. It's substantially harder (or outright impossible) to hit gotchas #1 and #2. Naturally, gotcha #3 still applies to Multi.

If you never used Multi, adopting it may be difficult. It has an unfamiliar syntax. I, too, had a similar objection when I first saw it. After 1 year of reluctance, I tried it. The first couple of weeks were difficult, but once it "clicked" I really started using it everywhere. The thing is, it will only "click" for you if you start actually using it.
