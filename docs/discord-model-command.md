# How the Discord `/model` Command Works in Hermes Agent

This report traces the full path of the Discord `/model` slash command shown in
the attached screenshot — from the native Discord slash command, through the
gateway's platform-agnostic command handler, to the Discord-specific dropdown
("Select") UI that lets a user change provider and model directly inside a
Discord message.

Screenshot context: running `/model` with no arguments produced an embed
titled **"⚙ Model Configuration"** showing the current model
(`deepseek/deepseek-v4-flash-0731`) and provider (`OpenRouter`), followed by a
`Choose a provider...` dropdown and a red `Cancel` button. That is the
`ModelPickerView` described below, in its first ("provider") stage.

## Reference documentation

- `website/docs/user-guide/messaging/discord.md`
  - [`## Interactive Model Picker`](../website/docs/user-guide/messaging/discord.md#interactive-model-picker)
    (lines 635–642) — the only doc section describing this feature:

    > Send `/model` with no arguments in a Discord channel to open a
    > dropdown-based model picker:
    > 1. **Provider selection** — a Select dropdown showing available
    >    providers (up to 25).
    > 2. **Model selection** — a second dropdown with models for the chosen
    >    provider (up to 25).
    >
    > The picker times out after 120 seconds. Only authorized users (those in
    > `DISCORD_ALLOWED_USERS`) can interact with it. If you know the model
    > name, type `/model <name>` directly.
  - [`## Slash Command Access Control`](../website/docs/user-guide/messaging/discord.md#slash-command-access-control)
    (lines 593–633) — how `allow_admin_from` / `user_allowed_commands` can gate
    who is even allowed to invoke `/model` as a slash command.

## End-to-end flow

```
Discord client (iOS app)
  │  user types /model, Discord's native slash-command UI sends an Interaction
  ▼
plugins/platforms/discord/adapter.py
  _register_slash_commands()              — registers "model" as a discord.app_commands tree command
    └─ slash_model(interaction, name="")
         └─ _run_simple_slash(interaction, "/model")
              ├─ _check_slash_authorization(interaction, "/model")   — user/role allowlist gate
              ├─ interaction.response.defer(ephemeral=True)
              ├─ _build_slash_event(interaction, "/model")           — builds a MessageEvent
              └─ self.handle_message(event)                         — hands off to the gateway core
  ▼
gateway/run.py  (canonical command dispatch table)
  if canonical == "model": return await self._handle_model_command(event)
  ▼
gateway/slash_commands.py
  _handle_model_command(event)
    ├─ parse_model_switch_args(raw_args)                — hermes_cli/model_switch.py
    ├─ reads current model/provider from config.yaml + any session override
    ├─ if no model name/provider was typed:
    │     adapter = self._adapter_for_source(source)
    │     has_picker = hasattr(type(adapter), "send_model_picker")
    │     providers = list_picker_providers(...)         — hermes_cli/model_switch.py
    │     await adapter.send_model_picker(
    │         chat_id, providers, current_model, current_provider,
    │         session_key, on_model_selected=_on_model_selected, metadata)
    └─ else: performs a direct switch_model() call (typed `/model <name>` path)
  ▼
plugins/platforms/discord/adapter.py
  send_model_picker(...)                   — builds the Discord Embed + ModelPickerView, sends it
  class ModelPickerView(discord.ui.View)    — the interactive dropdown UI (Select menus + buttons)
    ├─ _build_provider_select()            — Stage 1: "Choose a provider..." Select (screenshot)
    ├─ _on_provider_selected()             — Stage 2: edits message in place, builds model Select
    ├─ _build_model_select()
    ├─ _on_model_selected() → _expensive_warning_for() → _switch_selected_model()
    │     └─ calls back into on_model_selected(chat_id, model_id, provider_slug)
    │          which is the `_on_model_selected_scoped` closure built in
    │          gateway/slash_commands.py — this is what actually performs the
    │          model switch (config write, session override, cache eviction)
    ├─ _on_back() / _on_cancel()
    └─ on_timeout()                        — 120s View timeout, edits message to show "expired"
```

## 1. Discord slash command registration

**File:** `plugins/platforms/discord/adapter.py`, `_register_slash_commands()` (~line 5835)

```python
@tree.command(name="model", description="Show or change the model")
@discord.app_commands.describe(name="Model name (e.g. anthropic/claude-sonnet-4). Leave empty to see current.")
async def slash_model(interaction: discord.Interaction, name: str = ""):
    await self._run_simple_slash(interaction, f"/model {name}".strip())
```
(`plugins/platforms/discord/adapter.py:5850-5853`)

`/model` is registered as a native **Discord Application Command** (this is
what gives it Discord's autocomplete `/` menu entry, matching the doc's
"Native Slash Commands" section). It takes one optional string parameter,
`name`. In the screenshot, the user ran `/model` with the field empty.

`slash_model` just forwards to a shared helper, `_run_simple_slash`
(`plugins/platforms/discord/adapter.py:~5790-5834`), which:

1. Logs the invocation.
2. Runs `_check_slash_authorization(interaction, command_text)` — an
   authorization gate mirroring normal-message auth (`DISCORD_ALLOWED_USERS`,
   `DISCORD_ALLOWED_ROLES`, or the `allow_admin_from` /
   `user_allowed_commands` scheme documented in the doc's *Slash Command
   Access Control* section) (`plugins/platforms/discord/adapter.py:5257-5273`).
3. Defers the interaction (`interaction.response.defer(ephemeral=True)`) so
   Discord doesn't show "This interaction failed" while the gateway does work.
4. Builds a `MessageEvent` via `_build_slash_event(interaction, "/model")`
   (`plugins/platforms/discord/adapter.py:6382-...`), which normalizes the
   Discord interaction (DM vs. thread vs. guild channel, channel name, etc.)
   into the platform-agnostic event shape the rest of the gateway understands.
5. Calls `self.handle_message(event)` — this hands control to the
   **platform-agnostic gateway core**, exactly the same entry point normal
   chat messages use.

## 2. Gateway-side command dispatch

**File:** `gateway/run.py` (~line 17271)

```python
if canonical == "model":
    return await self._handle_model_command(event)
```

This is one arm of a large canonical-command dispatch table that routes every
slash command (`/reset`, `/status`, `/fast`, `/model`, …) to its handler,
regardless of which platform (Discord, Telegram, Slack, etc.) it came from.

## 3. The platform-agnostic `/model` handler

**File:** `gateway/slash_commands.py`, `_handle_model_command()` (line 1751)

This is the real brain of `/model`. Supported forms (from its docstring,
lines 1754-1761):

```
/model                              — interactive picker (Telegram/Discord) or text list
/model <name>                       — switch model (this session only)
/model <name> --once                — switch for the next turn only
/model <name> --session             — switch for this session only (explicit)
/model <name> --global              — switch and persist to config.yaml
/model <name> --provider <provider> — switch provider + model
/model --provider <provider>        — switch to provider, auto-detect model
```

Key steps for the no-argument case (what happened in the screenshot):

1. **Parse args** via `parse_model_switch_args()`
   (`hermes_cli/model_switch.py:849`) — a single shared parser used by every
   platform so `--once`/`--session`/`--global`/`--provider`/`--refresh` mean
   the same thing everywhere.
2. **Load current model/provider** from `config.yaml` (`model.default`,
   `model.provider`, `model.base_url`), then overlay any active **session
   override** (`self._session_model_overrides[session_key]`) — a per-session
   model switch that hasn't been persisted globally.
3. **No model/provider given** → try the interactive picker
   (`gateway/slash_commands.py:1853-1861`):
   ```python
   adapter = getattr(self, "_adapter_for_source")(source)
   has_picker = (
       adapter is not None
       and getattr(type(adapter), "send_model_picker", None) is not None
   )
   ```
   This is a **duck-typed capability check**: any platform adapter that
   implements a `send_model_picker` method gets the rich dropdown UI (today:
   Discord and Telegram). Platforms without it silently fall back to a
   plain-text model list (`gateway/slash_commands.py:2169+`).
4. **Build the provider list** via `list_picker_providers()`
   (`hermes_cli/model_switch.py:3921`), which wraps
   `list_authenticated_providers()` (`hermes_cli/model_switch.py:2570`):
   - Only includes providers that actually have credentials configured (an
     API key present) or are user-defined custom endpoints — you never see a
     provider you can't use.
   - For OpenRouter specifically, it replaces the curated static model list
     with a **live fetch** (`fetch_openrouter_models()`) filtered against the
     actual OpenRouter catalog, so the picker never offers a dead model ID.
   - Optionally prepends a synthetic "MoA" (mixture-of-agents) picker
     provider (`include_moa=True`).
   - Drops any provider whose resulting model list is empty (unless it's a
     custom endpoint where the user supplies their own models).
   - Capped to `max_models=50` per provider for the fetch; the Discord UI
     later re-caps to 25 per Discord's own Select-menu limit.
   - This all runs via `asyncio.to_thread(...)` because provider listing can
     fall through to a **synchronous** HTTP call on a cold cache, and the
     gateway must not block its event loop.
5. **Build an `_on_model_selected` closure** (`_on_model_selected_scoped`,
   `gateway/slash_commands.py:1892-2154`) that captures everything the actual
   switch will need (`self`, `session_key`, current model/provider/base_url/
   api_key, the profile "home" directory under multiplex). This closure *is*
   the callback the Discord UI invokes once the user finishes picking — see
   step 5 below for what it does.
6. **Hand off to the adapter**:
   ```python
   result = await adapter.send_model_picker(
       chat_id=source.chat_id,
       providers=providers,
       current_model=current_model,
       current_provider=current_provider,
       session_key=session_key,
       on_model_selected=_on_model_selected,
       metadata=metadata,
   )
   if result.success:
       return None  # Picker sent — adapter handles the response
   ```

## 4. Discord's picker UI — `send_model_picker` + `ModelPickerView`

**File:** `plugins/platforms/discord/adapter.py`

### `send_model_picker()` (line 7765)

This builds and sends the initial message the screenshot shows:

```python
embed = discord.Embed(
    title="⚙ Model Configuration",
    description=(
        f"Current model: `{current_model or 'unknown'}`\n"
        f"Provider: {provider_label}\n\n"
        f"Select a provider:"
    ),
    color=discord.Color.blue(),
)

view = ModelPickerView(
    providers=providers,
    current_model=current_model,
    current_provider=current_provider,
    session_key=session_key,
    on_model_selected=on_model_selected,
    allowed_user_ids=self._allowed_user_ids,
    allowed_role_ids=self._allowed_role_ids,
)

msg = await channel.send(embed=embed, view=view)
```

This is a **Discord embed + a `discord.ui.View`**, i.e. Discord's native
"Components" system (buttons and select menus attached to a message) — this
is exactly what produces the rounded card with the dropdown and the red
Cancel button in the iOS screenshot. It resolves the target channel from
`chat_id` (or a thread ID from `metadata`, if the command was run inside a
thread).

### `ModelPickerView` (`discord.ui.View` subclass, line 9154)

A **two-step drill-down UI** with a 120-second timeout
(`super().__init__(timeout=120)`), matching the doc's "times out after 120
seconds" claim.

State it carries:
- `providers`: the list built by `list_picker_providers()`.
- `current_model` / `current_provider`: for display only.
- `session_key`: identifies which gateway session the switch applies to.
- `on_model_selected`: the callback from `gateway/slash_commands.py` that
  performs the actual switch.
- `allowed_user_ids` / `allowed_role_ids`: **captured at send time** from the
  adapter's resolved `DISCORD_ALLOWED_USERS` / `DISCORD_ALLOWED_ROLES`, so
  every button/select click can be independently re-authorized.
- `resolved`: a guard flag so the view can't be used twice after a switch
  decision has been made.
- `_selected_provider`: set once stage 1 completes.
- `_pending_expensive_model`: used only if the "this model is unusually
  expensive" confirmation step triggers.

**Stage 1 — `_build_provider_select()`** (line 9191): This is exactly what's
in the screenshot.

```python
select = discord.ui.Select(
    placeholder="Choose a provider...",
    options=options[:25],
    custom_id="model_provider_select",
)
select.callback = self._on_provider_selected
self.add_item(select)

cancel_btn = discord.ui.Button(
    label="Cancel", style=discord.ButtonStyle.red, custom_id="model_cancel"
)
cancel_btn.callback = self._on_cancel
self.add_item(cancel_btn)
```

- One `discord.SelectOption` per provider, label `"{name} ({count} models)"`,
  truncated to Discord's 100-char field limit
  (`_DISCORD_SELECT_FIELD_LIMIT = 100`, line 86) via
  `_truncate_discord_component_text`.
- The currently-active provider gets `description="current"` on its option.
- Discord caps select menus at 25 options — `options[:25]` enforces that; if
  there are more providers than 25, only the first 25 are shown (a limitation
  also called out in the doc).
- A red `Cancel` button below the dropdown, matching the screenshot exactly.

**Stage 1 → Stage 2 — `_on_provider_selected()`** (line 9309): fires when the
user picks a provider from the dropdown.

1. Re-checks auth via `_check_auth()` → `_component_check_auth()`
   (line 8644) — even though the message is visible to everyone in the
   channel, only allowed users/roles can actually operate the dropdown; an
   unauthorized click gets an ephemeral `"You're not authorized~"` reply and
   the component state is unchanged.
2. Reads the chosen provider slug from `interaction.data["values"][0]`
   (Discord sends the selected option value(s) back in the interaction
   payload).
3. Calls `_build_model_select(provider_slug)` (line 9226) to swap the
   dropdown's options for that provider's models (again capped to 25, each
   labeled by the short model name with the full ID as the option value).
4. **Edits the original message in place** —
   `await interaction.response.edit_message(embed=..., view=self)` — rather
   than sending a new message. This is why the whole flow feels like one
   evolving card instead of a spam of new messages. If the provider has more
   than 25 models, the description gets a hint: *"N more available — type
   `/model <name>` directly"*.

**Stage 2 — model chosen — `_on_model_selected()`** (line 9383):

1. Same `resolved` + auth checks.
2. Reads `model_id` from `interaction.data["values"][0]`.
3. Checks `_expensive_warning_for(model_id)`
   (`hermes_cli/model_selection_guards.py`'s `combined_selection_warning`) —
   if the chosen model is flagged as unusually costly, the view swaps to a
   **confirmation stage** (`_build_expensive_confirm`, line 9274: "Switch
   anyway" / "Cancel" buttons) instead of switching immediately.
4. Otherwise calls `_switch_selected_model(interaction, model_id)`
   (line 9338), which:
   - Marks `self.resolved = True` and clears all components
     (`self.clear_items()`), so the picker can't be reused.
   - Immediately edits the message to show a transient
     **"⚙ Switching Model — Switching to `<model_id>`..."** embed with no
     view — visible, immediate feedback while the switch runs.
   - Calls `await self.on_model_selected(str(interaction.channel_id),
     model_id, self._selected_provider)` — this invokes the closure built
     back in `gateway/slash_commands.py` (`_on_model_selected_scoped`),
     which is where the **actual provider/model switch happens** (see
     section 5).
   - Once that returns confirmation text, edits the message again via
     `interaction.edit_original_response(...)` to a green
     **"⚙ Model Switched"** embed with the result text (e.g. the new model,
     provider label, and context-window size).

**Navigation:**
- `_on_back()` (line 9427) rebuilds the provider select and restores the
  original "Select a provider" embed — lets the user return to stage 1
  without restarting `/model`.
- `_on_cancel()` (line 9455) marks the view resolved, clears components, and
  edits the embed to a grey "Model selection cancelled." message — this is
  what the visible red **Cancel** button in the screenshot does at either
  stage (`model_cancel` in stage 1, `model_cancel2` in stage 2).
- `on_timeout()` (line 9467) fires automatically 120 seconds after the view
  was created (Discord's `discord.ui.View` timeout machinery) and edits the
  original message to a grey **"⏱ Selection expired — no model change."**
  embed with no components.

A very similar, simpler `ChoicePickerView` class (line 9484) is the generic
single-dropdown version used for flat-choice commands like `/reasoning` and
`/fast` — `/model` is the only one that needs the two-level
provider→model drill-down because providers can each have many models.

## 5. What actually performs the switch

The `on_model_selected` callback the `ModelPickerView` invokes is **not**
Discord-specific — it's the `_on_model_selected_scoped` closure defined in
`gateway/slash_commands.py:1892-2154`, inside `_handle_model_command()`. This
keeps all real model-switching logic platform-agnostic; Discord's adapter
only owns rendering and interaction plumbing. Key steps of that closure:

1. **Stale-code guard** — `_model_switch_skew_guard()`
   (`gateway/slash_commands.py:72`) refuses to switch models if the running
   gateway process's code is stale relative to disk (e.g. after a `git pull`
   without a restart), since the model-switch code path is the
   highest-risk trigger for hitting removed/renamed symbols.
2. **`switch_model()`** (`hermes_cli/model_switch.py:1439`) — run via
   `asyncio.to_thread` because it can synchronously hit models.dev for
   metadata on a cold cache. Returns a `SwitchResult` with the resolved
   model, provider, API key, base URL, API mode, provider label, and model
   metadata (context length, pricing, etc.).
3. **Context-window warnings** —
   `enrich_model_switch_warnings_for_gateway()` merges any "you're switching
   to a smaller context window" preflight-compression warning into the
   result text.
4. **Live cached agent update** — if this session has a warm in-memory agent
   (`self._agent_cache`), `cached_entry[0].switch_model(...)` swaps its model
   in place. If that in-place swap throws, the code **rolls back** cleanly:
   it does *not* persist the failed model, does *not* set a session override
   pointing at a broken model, and does *not* evict the still-good cached
   agent — a failed switch is a no-op rather than leaving the session broken.
5. **Session DB update** — `update_session_model()` so the dashboard/UI shows
   the new model immediately.
6. **Session override + note** — stores `self._session_model_overrides[session_key]`
   (model/provider/api_key/base_url/api_mode) and a one-turn `_pending_model_notes`
   message telling the agent it was just switched (so it can update its own
   self-identification in the next reply).
7. **Write-through persistence** — the non-secret parts of the override are
   written to the session store (`set_model_override`) so the picked model
   survives a gateway restart; the API key itself is never persisted this
   way.
8. **Cache eviction** — `self._evict_cached_agent(session_key)` so the next
   turn rebuilds a fresh agent from the override rather than trusting a stale
   cache signature.
9. **Optional global persistence** — unless `--session` was explicitly
   requested, the picked model is written back into `~/.hermes/config.yaml`
   (`model.default`, `model.provider`, `model.base_url`/`api_mode` as
   appropriate, with stale custom-endpoint credentials cleared via
   `clear_model_endpoint_credentials`), mirroring what a typed
   `/model <name> --global` would do (see #49066 referenced in the code
   comments — picker switches now persist the same way typed switches do).
10. **Confirmation text** — builds the final human-readable string (model
    name via `format_model_for_display`, provider label, resolved context
    length) that `_switch_selected_model()` puts into the green "Model
    Switched" embed.

## 6. Authorization model for the picker

Two independent gates protect `/model`:

- **Invoking the slash command at all**: `_check_slash_authorization()`
  (`plugins/platforms/discord/adapter.py:5257`) — standard Discord message
  auth (`DISCORD_ALLOWED_USERS` / `DISCORD_ALLOWED_ROLES`), plus, if
  configured, the `allow_admin_from` / `user_allowed_commands` scheme from
  the doc's *Slash Command Access Control* section (note: `/model` is one of
  the example commands explicitly listed as safe to expose to non-admin
  users there).
- **Operating the dropdowns/buttons afterward**: `_component_check_auth()`
  (`plugins/platforms/discord/adapter.py:8644`) — re-checked on *every*
  select/button interaction (`_on_provider_selected`, `_on_model_selected`,
  `_on_expensive_confirm`, `_on_back`, `_on_cancel`), independent of who can
  see the message. Semantics:
  1. `DISCORD_ALLOW_ALL_USERS` / `GATEWAY_ALLOW_ALL_USERS` → allow.
  2. User ID in `DISCORD_ALLOWED_USERS` (or global `GATEWAY_ALLOWED_USERS`)
     → allow.
  3. User has a role in `DISCORD_ALLOWED_ROLES` → allow (fails closed if the
     interaction has no resolvable `roles`, e.g. a DM).
  4. User approved via the pairing store (`hermes pairing approve`) → allow.
  5. Otherwise → reject with an ephemeral `"You're not authorized~"` message;
     the view's visible state is left untouched for the (still-authorized)
     original invoker.

This means the picker message is visible to the whole channel, but only
authorized users can actually drive it — matching the doc's "Only authorized
users ... can interact with it."

## Summary table

| Layer | File | Responsibility |
|---|---|---|
| Native Discord command | `plugins/platforms/discord/adapter.py:5850` | Registers `/model` as an Application Command; forwards to gateway |
| Slash auth + dispatch glue | `plugins/platforms/discord/adapter.py:_run_simple_slash`, `_check_slash_authorization`, `_build_slash_event` | Auth gate, defer, build `MessageEvent`, call `handle_message` |
| Canonical command router | `gateway/run.py:17271` | Routes `canonical == "model"` to the handler |
| Platform-agnostic logic | `gateway/slash_commands.py:_handle_model_command` (1751) | Parses args, loads current model, decides picker vs. text list vs. direct switch, builds the `on_model_selected` closure |
| Provider/model catalog | `hermes_cli/model_switch.py:list_picker_providers` (3921), `list_authenticated_providers` (2570) | Filters to providers with credentials, live-checks OpenRouter models |
| Discord UI rendering | `plugins/platforms/discord/adapter.py:send_model_picker` (7765) | Sends the embed + `ModelPickerView` |
| Interactive UI/state machine | `plugins/platforms/discord/adapter.py:ModelPickerView` (9154) | Provider Select → Model Select → (optional expensive-model confirm) → switch, with Back/Cancel/timeout |
| Component auth | `plugins/platforms/discord/adapter.py:_component_check_auth` (8644) | Re-authorizes every dropdown/button click |
| Actual model switch | `hermes_cli/model_switch.py:switch_model` (1439), via the closure in `gateway/slash_commands.py:1892` | Resolves credentials, updates cached agent, session DB, session override, optional config.yaml persistence |
| Tests | `tests/gateway/test_discord_model_picker.py` | Regression test asserting the "clear controls → switch → final edit" ordering |

## Related test coverage

`tests/gateway/test_discord_model_picker.py` exercises exactly the interaction
shown in the screenshot's next step (selecting a model): it drives
`ModelPickerView._on_model_selected` directly with a mocked `discord.Interaction`
and asserts the message is edited to *"⚙ Switching Model"* before the switch
callback runs, then to *"⚙ Model Switched"* after — i.e. the view always shows
"switching" feedback before the (potentially slow) provider switch completes,
never leaving the user staring at a stale dropdown.
