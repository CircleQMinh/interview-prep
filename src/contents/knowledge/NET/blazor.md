---
id: blazor
topic: Blazor
category: .NET
---

In Blazor, **hosting models** means the different ways a Blazor app can be **run and delivered**.

It answers questions like:

* Where does the UI code execute?
* How does the browser communicate with .NET?
* Where is the app logic running?

# Main Blazor hosting models

## 1. Blazor Server

In **Blazor Server**, the app runs on the **server**.

What happens:

* UI events from the browser are sent to the server
* The server processes them
* The updated UI is sent back to the browser over **SignalR**

So:

* .NET code runs on the server
* The browser only shows the UI and sends user actions

### Pros

* Very small download size
* Fast initial load
* Full access to server resources
* Easy to use secure server-side code and database access

### Cons

* Needs constant network connection
* More noticeable latency for distant users
* Server must keep connection/state for each client

---

## 2. Blazor WebAssembly

In **Blazor WebAssembly**, the app runs **inside the browser** using WebAssembly.

What happens:

* The browser downloads the app, .NET runtime, and assemblies
* The UI logic runs in the client browser
* API calls are usually made to a backend separately

So:

* .NET code runs in the browser
* Server is optional for UI rendering, but usually needed for data/API

### Pros

* Can reduce server load
* Better for interactive client-side apps
* Can keep working somewhat independently after loading
* Deployment can be simpler for static hosting

### Cons

* Larger initial download
* Slower first load compared to Server
* Limited direct access to server resources
* Usually needs API for secure database/business logic

---

## 3. Blazor Web App

This is the newer modern Blazor model in recent ASP.NET Core versions.

It supports multiple rendering approaches in one app, such as:

* **Static SSR** (server-side rendering)
* **Interactive Server**
* **Interactive WebAssembly**
* **Interactive Auto**

This means one app can choose how each page/component behaves.

### Why this matters

You are no longer forced to choose only one global style for the whole app.
Some pages can be server-rendered, while others can become interactive using Server or WebAssembly.

---

# Easy way to think about it

A hosting model is basically **where your Blazor code lives and runs**.

* **Blazor Server** → runs on the server
* **Blazor WebAssembly** → runs in the browser
* **Blazor Web App** → modern combined model that can use different render modes




# What a Blazor component is

A **component** is a reusable piece of UI.

In Blazor, a component usually lives in a **`.razor`** file and can contain:

* HTML markup
* C# code
* data binding
* event handling
* parameters
* other nested components

Example:

```razor
<h3>Hello @Name</h3>

@code {
    private string Name = "My Name";
}
```

---

# Important component basics

## 1. Components are built with `.razor` files

A `.razor` file is the main building block of Blazor UI.

Example:

```razor
<p>This is a component.</p>
```

If the file is named `MyComponent.razor`, you can use it like this:

```razor
<MyComponent />
```

---

## 2. Components can contain markup and C#

Blazor lets you mix HTML and C# in the same file.

Example:

```razor
<h3>Counter</h3>
<p>Current count: @count</p>

<button @onclick="Increment">Click me</button>

@code {
    private int count = 0;

    private void Increment()
    {
        count++;
    }
}
```

This is one of the most basic Blazor concepts.

---

## 3. Components can be reused

A component is meant to be reusable.

For example, you can create a `UserCard.razor` and use it in many places:

```razor
<UserCard />
<UserCard />
<UserCard />
```

Reusable UI is one of the main reasons components exist.

---

## 4. Parent can pass data to child using `[Parameter]`

A child component can receive values from a parent.

Child component:

```razor
<h3>Hello @Title</h3>

@code {
    [Parameter]
    public string Title { get; set; } = string.Empty;
}
```

Parent component:

```razor
<ChildComponent Title="Welcome" />
```

`[Parameter]` is one of the most important component basics.

---

## 5. Child can notify parent with `EventCallback`

This is how child-to-parent communication usually works.

Child:

```razor
<button @onclick="NotifyParent">Click</button>

@code {
    [Parameter]
    public EventCallback OnClick { get; set; }

    private async Task NotifyParent()
    {
        await OnClick.InvokeAsync();
    }
}
```

Parent:

```razor
<ChildComponent OnClick="HandleClick" />

@code {
    private void HandleClick()
    {
        Console.WriteLine("Child clicked");
    }
}
```

---

## 6. Components can have private state

A component can store its own local data.

Example:

```razor
<p>Count: @count</p>

@code {
    private int count = 5;
}
```

When state changes, the component can re-render.

---

## 7. Event handling is built in

Blazor components handle UI events with directives like:

* `@onclick`
* `@onchange`
* `@oninput`

Example:

```razor
<input @oninput="HandleInput" />

@code {
    private void HandleInput(ChangeEventArgs e)
    {
        var value = e.Value?.ToString();
    }
}
```

---

## 8. Data binding with `@bind`

Blazor supports two-way binding.

Example:

```razor
<input @bind="Name" />
<p>You typed: @Name</p>

@code {
    private string Name = string.Empty;
}
```

This is very common in forms and input components.

---

## 9. Conditional rendering

You can show or hide UI based on conditions.

```razor
@if (isLoggedIn)
{
    <p>Welcome back</p>
}
else
{
    <p>Please log in</p>
}
```

---

## 10. Rendering lists

You often render collections with `@foreach`.

```razor
<ul>
@foreach (var item in items)
{
    <li>@item</li>
}
</ul>

@code {
    private List<string> items = new() { "A", "B", "C" };
}
```

---

## 11. Components can use other components

Blazor components are composable.

Example:

```razor
<PageTitle>Home</PageTitle>
<MyHeader />
<MyMenu />
<MyContent />
```

This means you can build larger pages from smaller components.

---

## 12. Code can be inline or code-behind

You can write logic inside `@code` or move it to a separate partial class.

Example:

```razor
@code {
    private string message = "Hello";
}
```

Or use:

* `MyComponent.razor`
* `MyComponent.razor.cs`

This is useful for cleaner structure in larger components.

---

## 13. Lifecycle matters for components

Every component has a lifecycle, such as:

* `OnInitialized`
* `OnInitializedAsync`
* `OnParametersSet`
* `OnAfterRender`

These are important because components often need to load data or react to parameter changes.

`OnInitialized` runs when the component is first initialized, after it has received its **initial parameters** from the parent. It is the synchronous initialization hook.
Use it for:

* setting default values
* initializing local fields
* doing quick synchronous setup that does not need `await`

`OnInitializedAsync` is the async version of initialization. It also runs when the component is first initialized, and it is used when startup work requires asynchronous operations such as calling an API or loading data from a service. When the async work finishes, the component refreshes.
Use it for:

* API calls
* database/service calls through injected services
* loading reference data when the component first starts

```c
    protected override async Task OnInitializedAsync()
    {
        await LoadAsync();
    }

    private async Task LoadAsync()
    {
        _products = (await Mediator.Send(new GetProductsRequest("",1,10))).Result;
    }
```
`OnParametersSet` runs after the component has received parameters from its parent **and the values have been assigned to the component’s parameter properties**. It is the synchronous hook for reacting to parameter input. ([Microsoft Learn][4])

This is the key difference from initialization:

* `OnInitialized*` → first-time setup
* `OnParametersSet*` → respond whenever parameters are supplied or updated

Example:

```razor
@code {
    [Parameter]
    public int ProductId { get; set; }

    private string message = string.Empty;

    protected override void OnParametersSet()
    {
        message = $"Current product id: {ProductId}";
    }
}
```

Use it for:

* recalculating derived values from parameters
* reacting when a parent passes new values
* handling route parameter changes
* resetting UI state when input parameters change

Example scenario:
A page `/products/{id}` reuses the same component instance while `id` changes. In that case, logic tied to `id` belongs in `OnParametersSet` or `OnParametersSetAsync`, not `OnInitializedAsync`.

`OnAfterRender` runs after the component has rendered and the UI output has been applied. This hook is used for logic that must happen **after the rendered DOM exists**.

Typical use cases:

* JavaScript interop that needs actual DOM elements
* focusing an input after render
* measuring element size or position
* integrating third-party JavaScript widgets
Example:

```razor
@inject IJSRuntime JS

<input @ref="inputRef" />

@code {
    private ElementReference inputRef;

    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            // usually JS interop would go here via OnAfterRenderAsync
        }
    }
}
```

A simple way to remember them:

* **Need one-time setup?** → `OnInitialized` / `OnInitializedAsync`
* **Need to react to route or parent parameter changes?** → `OnParametersSet` / `OnParametersSetAsync`
* **Need browser DOM or JS?** → `OnAfterRender` / `OnAfterRenderAsync`
---

## 14. Components re-render when state changes

Blazor automatically re-renders components after many UI events.

For example, after button click:

```razor
<button @onclick="Increment">Add</button>
<p>@count</p>
```

When `count` changes, the UI updates.

Sometimes you may manually trigger rendering with `StateHasChanged()`.

---

## 15. Cascading values can pass data deeply

If many nested components need the same value, Blazor can pass it using cascading values instead of sending parameters manually through every level.

This is more advanced than normal `[Parameter]`, but still a common basic interview topic.

A parent provides a value using `CascadingValue`, and a descendant receives it with `[CascadingParameter]`.

### Parent

```razor
<CascadingValue Value="theme">
    <PageContent />
</CascadingValue>

@code {
    private string theme = "dark";
}
```

### Deep child

```razor
@code {
    [CascadingParameter]
    public string Theme { get; set; } = string.Empty;
}
```

Now the deep child can access `Theme` without every intermediate component passing it manually.

---

## When to use cascading values

They are useful for data shared by many nested components, such as:

* theme information
* authenticated user context
* form context
* layout-level settings
* shared UI state for a subtree


---


# 1. Forms in Blazor

A **form** is the UI that lets the user enter and submit data.

In Blazor, the most common form component is:

* `EditForm`

Example:

```razor
<EditForm Model="user" OnValidSubmit="HandleSubmit">
    <InputText @bind-Value="user.Name" />
    <button type="submit">Save</button>
</EditForm>

@code {
    private UserModel user = new();

    private void HandleSubmit()
    {
        // save logic
    }

    public class UserModel
    {
        public string Name { get; set; } = string.Empty;
    }
}
```

## What `EditForm` does

`EditForm` is a Blazor component that helps manage:

* model binding
* validation
* submit handling

It is different from a plain HTML `<form>` because it is integrated with Blazor’s component model and validation system.

---

# 2. Binding in Blazor

**Binding** means connecting a UI element to a C# value.

The most common syntax is:

* `@bind`
* `@bind-Value`

This creates **two-way binding**.

That means:

* when the user changes the input, the C# property updates
* when the C# property changes, the UI updates

Example:

```razor
<input @bind="name" />
<p>Hello @name</p>

@code {
    private string name = string.Empty;
}
```

If the user types `"My Name"`, the `name` variable becomes `"My Name"`.

---


## Two-way binding with Blazor input components

Blazor also provides built-in input components like:

* `InputText`
* `InputNumber`
* `InputDate`
* `InputSelect`
* `InputCheckbox`
* `InputTextArea`

Example:

```razor
<InputText @bind-Value="user.Email" />
<InputNumber @bind-Value="user.Age" />
<InputCheckbox @bind-Value="user.IsActive" />
```

These are preferred inside `EditForm` because they integrate better with validation.

---

## Why binding matters

Binding lets you avoid manually reading field values from the DOM.

Without binding, you would have to handle events and extract values yourself.

With binding, Blazor keeps UI and data synchronized for you.

---

# 3. Validation in Blazor

**Validation** means checking whether the user input is acceptable.

Examples:

* required field must not be empty
* email must be valid format
* age must be within a range
* password length must be long enough

Blazor commonly uses **Data Annotations** for validation.

Example model:

```csharp
using System.ComponentModel.DataAnnotations;

public class RegisterModel
{
    [Required]
    public string FirstName { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Range(18, 100)]
    public int Age { get; set; }
}
```

Then use it in a form:

```razor
<EditForm Model="model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div>
        <label>First Name</label>
        <InputText @bind-Value="model.FirstName" />
        <ValidationMessage For="@(() => model.FirstName)" />
    </div>

    <div>
        <label>Email</label>
        <InputText @bind-Value="model.Email" />
        <ValidationMessage For="@(() => model.Email)" />
    </div>

    <div>
        <label>Age</label>
        <InputNumber @bind-Value="model.Age" />
        <ValidationMessage For="@(() => model.Age)" />
    </div>

    <button type="submit">Submit</button>
</EditForm>

@code {
    private RegisterModel model = new();

    private void HandleSubmit()
    {
        // only runs if valid
    }
}
```

---

# 4. Important validation components

## `DataAnnotationsValidator`

This enables validation using attributes like:

* `[Required]`
* `[StringLength]`
* `[EmailAddress]`
* `[Range]`

Example:

```razor
<DataAnnotationsValidator />
```

Without it, annotation-based validation will not work in the form.

---

## `ValidationSummary`

Shows a summary of all validation errors.

```razor
<ValidationSummary />
```

Useful for showing all form errors together.

---

## `ValidationMessage`

Shows the error message for one specific field.

```razor
<ValidationMessage For="@(() => model.Email)" />
```

This is often better UX because the message appears next to the field.

---

# 5. Form submission in Blazor

Blazor supports different submit callbacks.

## `OnValidSubmit`

Runs only when the form is valid.

```razor
<EditForm Model="model" OnValidSubmit="HandleSubmit">
```

## `OnInvalidSubmit`

Runs when the form is invalid.

```razor
<EditForm Model="model" OnInvalidSubmit="HandleInvalid">
```

## `OnSubmit`

Runs regardless of validity.

```razor
<EditForm Model="model" OnSubmit="HandleAnySubmit">
```

Common practice:

* use `OnValidSubmit` for normal create/update forms
* use `OnInvalidSubmit` if you want custom invalid behavior

---

# 6. Difference between HTML form and `EditForm`

## Plain HTML form

```razor
<form>
    <input />
</form>
```

This is just standard HTML.

## Blazor `EditForm`

```razor
<EditForm Model="model">
    <InputText @bind-Value="model.Name" />
</EditForm>
```

This adds Blazor-aware features:

* model binding
* validation support
* Blazor submit events
* validation messages

# 8. Why use Blazor input components instead of raw HTML inputs

You can bind raw HTML inputs:

```razor
<input @bind="name" />
```

But Blazor input components are usually better inside forms because they:

* integrate with `EditForm`
* work better with validation
* connect with `EditContext`



# How routing is defined

In Blazor, a component becomes a routable page by using the `@page` directive.

Example:

```razor
@page "/products"

<h3>Products Page</h3>
```

This means when the user visits `/products`, this component is displayed.

---

# Example of basic routing

## Home page

```razor
@page "/"

<h3>Home</h3>
```

## About page

```razor
@page "/about"

<h3>About</h3>
```

Now:

* `/` shows Home
* `/about` shows About

---

# Route parameters

Blazor routing can capture values from the URL.

Example:

```razor
@page "/products/{id:int}"

<h3>Product Id: @Id</h3>

@code {
    [Parameter]
    public int Id { get; set; }
}
```

If the URL is:

```text
/products/15
```

then `Id` becomes `15`.

---

# Why this matters

Routing lets you build pages like:

* `/login`
* `/register`
* `/users/5`
* `/orders/100/details`
* `/products/edit/8`

without manually checking the browser URL yourself.

---

# Navigation in Blazor

Routing works together with navigation.

You can move between pages using:

## `NavLink`

Used for clickable navigation links.

```razor
<NavLink href="/about">About</NavLink>
```

## `NavigationManager`

Used for programmatic navigation.

```razor
@inject NavigationManager Nav

<button @onclick="GoToHome">Go</button>

@code {
    private void GoToHome()
    {
        Nav.NavigateTo("/");
    }
}
```

---

# Routing in app structure

Blazor has a router component that looks at the current URL and finds the matching component.

In practice, you usually do not write the routing engine yourself.
You mainly work with:

* `@page`
* `[Parameter]`
* `NavLink`
* `NavigationManager`

---

# Multiple routes in one component

A component can support more than one route.

Example:

```razor
@page "/report"
@page "/reports"

<h3>Reports</h3>
```

Now both URLs work.

---

# Optional and typed route parameters

Blazor route parameters can be typed.

Example:

```razor
@page "/employee/{id:int}"
```

This means:

* route must contain `id`
* `id` must be an integer

Other constraints help validate the URL.

Optional route parameters in Blazor use a ? in the @page route template.
```razor
@page "/products/{id?}"
```

---

# Query string vs route parameter

These are different.

## Route parameter

Inside the path:

```text
/products/10
```

## Query string

After `?`:

```text
/products?id=10
```

Blazor commonly uses route parameters with `@page`, but query string values can also be read when needed.

---


# DI in a Blazor component

You usually inject services using `@inject`.

Example:

```razor
@inject WeatherService WeatherService

<h3>Weather</h3>

@code {
    protected override void OnInitialized()
    {
        var data = WeatherService.GetForecast();
    }
}
```

This means Blazor provides an instance of `WeatherService` to the component.

---


# Types of state in Blazor

## 1. Local component state

State stored inside one component.

Example:

```razor
@code {
    private bool isLoading = true;
    private string searchText = string.Empty;
}
```

This is good for UI that only matters inside that component.

---

## 2. Shared state

State that multiple components need.

Examples:

* current logged-in user
* shopping cart
* selected language
* theme
* page filters shared across components

This is often stored in a **state container service**.

---

# State container pattern

A common Blazor pattern is to create a service that stores shared state.

Example:

```csharp
public class AppState
{
    public string CurrentTheme { get; private set; } = "light";

    public event Action? OnChange;

    public void SetTheme(string theme)
    {
        CurrentTheme = theme;
        NotifyStateChanged();
    }

    private void NotifyStateChanged()
    {
        OnChange?.Invoke();
    }
}
```

Register it:

```csharp
builder.Services.AddSingleton<AppState>();
```

Use it in a component:

```razor
@inject AppState AppState
@implements IDisposable

<p>Theme: @AppState.CurrentTheme</p>

<button @onclick='() => AppState.SetTheme("dark")'>Dark</button>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }

    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
}
```

This lets multiple components react to the same shared state.

---

# Why JS interop exists

Blazor lets you build UI in C#, but the browser still has many features and libraries that are JavaScript-based, such as:

* DOM APIs
* browser storage APIs
* clipboard
* charts/editors/date pickers
* existing JS libraries

So JS interop is the bridge between Blazor and browser/JS functionality. In WebAssembly, .NET runs in the browser sandbox and can interop with JavaScript there; in Blazor Server, interop calls also cross the network. 

# The two directions

## 1. Call JavaScript from C#
```html
<script>
  window.convertArray = (win1251Array) => {
    var win1251decoder = new TextDecoder('windows-1251');
    var bytes = new Uint8Array(win1251Array);
    var decodedArray = win1251decoder.decode(bytes);
    return decodedArray;
  };
</script>

```

```csharp
@page "/call-js-1"
@inject IJSRuntime JS

<PageTitle>Call JS 1</PageTitle>

<h1>Call JS Example 1</h1>

<p>
    <button @onclick="ConvertArray">Convert Array</button>
</p>

<p>
    @text
</p>

<p>
    Quote ©2005 <a href="https://www.uphe.com">Universal Pictures</a>: 
    <a href="https://www.uphe.com/movies/serenity-2005">Serenity</a><br>
    <a href="https://www.imdb.com/name/nm0472710/">David Krumholtz on IMDB</a>
</p>

@code {
    private MarkupString text;

    private uint[] quoteArray = 
        new uint[]
        {
            60, 101, 109, 62, 67, 97, 110, 39, 116, 32, 115, 116, 111, 112, 32,
            116, 104, 101, 32, 115, 105, 103, 110, 97, 108, 44, 32, 77, 97,
            108, 46, 60, 47, 101, 109, 62, 32, 45, 32, 77, 114, 46, 32, 85, 110,
            105, 118, 101, 114, 115, 101, 10, 10,
        };

    private async Task ConvertArray() => 
        text = new(await JS.InvokeAsync<string>("convertArray", quoteArray));
}

```

* `InvokeVoidAsync(...)` → call JS that returns nothing
* `InvokeAsync<T>(...)` → call JS and get a result back

## 2. Call .NET from JavaScript

```csharp
@page "/call-dotnet-1"
@implements IAsyncDisposable
@inject IJSRuntime JS

<PageTitle>Call .NET 1</PageTitle>

<h1>Call .NET Example 1</h1>

<p>
    <button id="btn">Trigger .NET static method</button>
</p>

<p>
    See the result in the developer tools console.
</p>

export function returnArrayAsync() {
  DotNet.invokeMethodAsync('BlazorSample', 'ReturnArrayAsync')
    .then(data => {
      console.log(data);
    });
}

export function addHandlers() {
  const btn = document.getElementById("btn");
  btn.addEventListener("click", returnArrayAsync);
}

@code {
    private IJSObjectReference? module;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            module = await JS.InvokeAsync<IJSObjectReference>("import",
                "./Components/Pages/CallDotnet1.razor.js");

            await module.InvokeVoidAsync("addHandlers");
        }
    }

    [JSInvokable]
    public static Task<int[]> ReturnArrayAsync() =>
        Task.FromResult(new int[] { 11, 12, 13 });

    async ValueTask IAsyncDisposable.DisposeAsync()
    {
        if (module is not null)
        {
            try
            {
                await module.DisposeAsync();
            }
            catch (JSDisconnectedException)
            {
            }
        }
    }
}
```
# Performance considerations

JS interop is not free. Microsoft notes there is overhead because:

* calls are asynchronous
* values are serialized
* in server-side Blazor, calls also traverse the network 

So good practice is:

* avoid many tiny interop calls in loops
* batch work when possible
* avoid unnecessary round trips
* be extra careful in Blazor Server, where latency is more noticeable
# Blazor Server vs WebAssembly for JS interop

The concept is the same in both, but the cost profile differs:

* **Blazor WebAssembly**: .NET and JS both run in the browser, so there’s no server round trip for the interop call itself
* **Blazor Server**: interop calls go through the server-client connection, so latency matters more 

# Common real-world use cases

JS interop is commonly used for:

* focusing or scrolling elements
* reading/writing `localStorage` or `sessionStorage`
* file download helpers
* clipboard access
* chart libraries
* rich text editors
* maps
* browser APIs not directly exposed by Blazor

These are all examples of places where Blazor often needs a JS bridge because the browser ecosystem is still JS-centric. The official docs broadly frame JS interop as the way to access JS functionality from Blazor and vice versa.