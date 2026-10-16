# UE 5.7.4 Linux Editor: Tooltip, Notification, and Menu Focus Fixes

Source fixes and launch settings for Unreal Editor input problems when running through **XWayland on a Wayland desktop**.

These changes were tested with **Unreal Engine 5.7.4, Fedora KDE Plasma on Wayland, and an NVIDIA RTX 3090**. They address the reproduced issues below; they are not a complete native-Wayland port or a guarantee for every compositor and dialog.

## Symptoms

- Hovering an asset until its tooltip appears makes selection require two clicks.
- Notification toasts steal focus from the editor.
- Multiple notifications cause severe lag as they repeatedly exchange focus.
- The first click outside a File/Edit menu does not dismiss it.

## Solution overview

| Problem | Change | Requires compilation? |
| --- | --- | --- |
| Tooltips and notifications taking keyboard focus | Make SDL window focusability respect Slate's activation policy | Yes |
| Notifications activating one another when raised | Temporarily disable SDL activation-on-raise while raising notifications | Yes |
| SDL discarding the click that restores focus from a menu | Set `SDL_MOUSE_FOCUS_CLICKTHROUGH=1` | No |
| Editor selecting native Wayland instead of XWayland | Launch with the SDL X11 backend explicitly selected | No |

The source changes must be built into the editor. Environment settings alone do not apply those changes to Epic's prebuilt editor.

## 1. Respect Slate's window activation policy

**File:**

```text
Engine/Source/Runtime/ApplicationCore/Private/Linux/LinuxWindow.cpp
```

Inside `FLinuxWindow::Initialize()`, find the existing declaration of `bShouldActivate`. Immediately after it, set the SDL focusability property:

```cpp
const bool bShouldActivate = Definition->ActivationPolicy != EWindowActivationPolicy::Never;

// Match Slate's activation policy at the native window level as well. Tooltips
// and notification toasts must not take keyboard focus when mapped or clicked.
// This does not prevent their buttons and hyperlinks from receiving mouse input.
SDL_SetBooleanProperty(Props, SDL_PROP_WINDOW_CREATE_FOCUSABLE_BOOLEAN, bShouldActivate);
```

Keep the rest of the function's window-type handling in place. This property must be set before `SDL_CreateWindowWithProperties()`.

### Why this helps

Slate creates tooltip and standard notification windows with:

```cpp
EWindowActivationPolicy::Never
```

The Linux window-creation code did not translate that policy into SDL's native window focusability. In the failing run, the logs showed the main editor losing keyboard focus immediately after the tooltip appeared, followed by the tooltip gaining focus.

This change makes those windows non-focusable without disabling mouse input. Notification buttons and hyperlinks remain interactive. Windows with an activation policy that allows focus, such as ordinary dialogs, remain focusable.

The initial tooltip-only experiment set this property to `false` inside the tooltip branch. The final implementation above generalizes it to respect Slate's activation policy for all Linux windows; the tooltip-only assignment is no longer needed.

**Observed result:** tooltips remained visible, and one click selected the hovered asset.

## 2. Raise notifications without activating them

**File:**

```text
Engine/Source/Runtime/ApplicationCore/Private/Linux/LinuxApplication.cpp
```

Update `FLinuxApplication::RaiseNotificationWindows()` so notification stacking does not request activation:

```cpp
void FLinuxApplication::RaiseNotificationWindows(const TSharedPtr<FLinuxWindow>& ParentWindow)
{
    // Raising toasts is a stacking operation, not an activation request. Otherwise
    // a focus-gained event can raise another toast, causing a focus/raise feedback loop.
    const bool bActivate = SDL_GetHintBoolean(SDL_HINT_WINDOW_ACTIVATE_WHEN_RAISED, true);
    SDL_SetHint(SDL_HINT_WINDOW_ACTIVATE_WHEN_RAISED, "0");

    // Raise notification windows above everything except for modal windows.
    for (int32 WindowIndex = 0; WindowIndex < NotificationWindows.Num(); ++WindowIndex)
    {
        TSharedRef<FLinuxWindow> NotificationWindow = NotificationWindows[WindowIndex];

        if (!ParentWindow->IsModalWindow())
        {
            SDL_RaiseWindow(NotificationWindow->GetHWnd());
        }
    }

    SDL_SetHint(SDL_HINT_WINDOW_ACTIVATE_WHEN_RAISED, bActivate ? "1" : "0");
}
```

### Why this helps

Unreal raises notification windows when windows gain focus. SDL's X11 backend can interpret raising a window as a request to activate it.

With multiple focusable notifications, the failing test produced this cycle:

```text
Notification gains focus
    -> Unreal raises notification windows
    -> Another notification gains focus
    -> Unreal raises notification windows again
    -> Repeat
```

The diagnostic trace recorded hundreds of thousands of focus events during the failing test. Preventing notification activation on both creation and raising addresses the two contributing paths.

**Observed result:** notification-induced lag disappeared, and assets and other editor controls remained clickable while a notification was visible.

## 3. Deliver clicks that restore focus

Set this environment variable before launching the editor:

```bash
export SDL_MOUSE_FOCUS_CLICKTHROUGH=1
```

SDL normally suppresses a click that also activates a window. When returning from a separate menu window to the editor, that can discard the click Unreal needs to process.

In the failing test, the first click outside the menu left the menu open. Enabling focus click-through corrected the behavior while retaining menu keyboard navigation.

This setting also allows a click that restores focus to perform its normal UI action. It does not make menus non-focusable.

**Observed result:** the first outside click dismissed the menu, and selection and menu navigation worked normally.

## 4. Use an XWayland launcher

Create `RunEditorXWayland.sh` in the engine source root, alongside `Setup.sh` and the `Engine` directory:

```bash
#!/usr/bin/env bash
set -euo pipefail

ENGINE_ROOT="$(dirname -- "$(readlink -f -- "${BASH_SOURCE[0]}")")"

export SDL_VIDEO_DRIVER=x11
export SDL_VIDEODRIVER=x11

# Deliver the click that restores focus from a separate popup menu window.
export SDL_MOUSE_FOCUS_CLICKTHROUGH="${SDL_MOUSE_FOCUS_CLICKTHROUGH:-1}"

# Show the project browser instead of automatically reopening the last project.
# Keep explicit project arguments first so Unreal recognizes them correctly.
exec "$ENGINE_ROOT/Engine/Binaries/Linux/UnrealEditor" "$@" -norecentproject
```

Make it executable:

```bash
chmod +x RunEditorXWayland.sh
```

Open the project browser:

```bash
./RunEditorXWayland.sh
```

Or open a project directly:

```bash
./RunEditorXWayland.sh "/path/to/MyProject.uproject"
```

Projects opened by the browser normally inherit the launch environment. Confirm the backend in the project's startup log:

```text
Using SDL video driver 'x11'
```

If it reports `wayland`, that run is using native Wayland and is not the configuration tested here. The Plasma desktop itself remains on Wayland when Unreal uses XWayland; an X11 desktop login is not required.

`-norecentproject` prevents automatic reopening of the most recent project when no project path is supplied. Explicit project arguments still open the requested project.

The launcher sets these variables only for the editor and its child processes, rather than changing the entire desktop environment.

## 5. Build the source changes

Run the following from the UE 5.7.4 source root. The source dependencies must already be downloaded by `Setup.sh`, and the editor should be closed while rebuilding its modules.

For an initial source build, build the shader worker and editor:

```bash
./Engine/Build/BatchFiles/Linux/Build.sh \
    ShaderCompileWorker Linux Development \
    -NoDebugInfo -MaxParallelActions=4

./Engine/Build/BatchFiles/Linux/Build.sh \
    UnrealEditor Linux Development \
    -NoDebugInfo -MaxParallelActions=4
```

These commands use Epic's bundled toolchain. `-NoDebugInfo` reduces disk usage at the cost of debugging information. Adjust parallelism for the available memory and CPU resources.

For subsequent source changes, repeat the editor build command with the same settings. UnrealBuildTool reuses existing outputs. Tooltip and notification input fixes do not normally require rebuilding `ShaderCompileWorker` or invalidating project shader caches.

On the test machine, the later application-module change needed one compilation action and one link action, completing in about nine seconds. That timing is an observation, not a guarantee for other machines or changes.

Restart using `RunEditorXWayland.sh` after the build succeeds.

## 6. Validate with tooltips and notifications enabled

The test project initially contained overrides disabling both tooltips and notifications. Remove those disabling overrides or set the following in the project's `Config/DefaultEngine.ini`:

```ini
[ConsoleVariables]
Slate.EnableTooltips=1
Slate.bAllowNotifications=1
```

If a `[ConsoleVariables]` section already exists, update its entries rather than adding conflicting values.

Useful diagnostic launch arguments are:

```bash
./RunEditorXWayland.sh "/path/to/MyProject.uproject" \
    -LogCmds="LogLinuxWindowEvent Verbose,LogLinuxWindowType Verbose" \
    -ExecCmds="Slate.EnableTooltips,Slate.bAllowNotifications"
```

The `-ExecCmds` entries above query the current values; they do not change them. These are diagnostic arguments and are not required for normal use.

### Interactive checks

1. Confirm that the startup log identifies the source-built editor and the `x11` SDL backend.
2. Hover an asset until its tooltip appears, then select it with one click.
3. Repeat across several assets and toolbar tooltips.
4. Show a notification and check that editor controls remain responsive while it is visible.
5. Test notification buttons and overlapping notifications; watch for focus cycling or lag.
6. Open File/Edit menus, click outside them, and check that the first click dismisses the menu.
7. Check menu items, submenus, arrow-key navigation, and Escape.
8. Test the dialogs relevant to your workflow separately.

### Optional local notification test command

For the investigation, a custom command was added to:

```text
Engine/Source/Developer/MessageLog/Private/MessageTesting.cpp
```

In the modified build, enter this in the editor console or Cmd field:

```text
Slate.TestNotification
```

It creates a notification after three seconds, includes a Dismiss button, and automatically expires after 60 seconds. Running it twice tests overlapping notifications.

**This command was added locally. It is not a stock UE 5.7.4 command, and applying only the two application-core fixes above does not add it.** It is a diagnostic convenience, not a requirement for the fixes.

## Scope and limitations

- The tooltip selection, notification focus/lag, and menu outside-click issues passed interactive testing on the described setup.
- Native Wayland was tested separately and had broader input problems. These changes do not establish native-Wayland compatibility.
- General dialog behavior and every compositor configuration have not been exhaustively validated.
- Notifications remain Unreal Slate notifications; they are not rerouted to the operating system's notification service.
- No compositor fork or permanent desktop-wide window rule is required by this approach.
