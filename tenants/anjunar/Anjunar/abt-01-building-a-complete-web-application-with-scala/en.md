This series builds a blog from an empty repository to an application running on its own domain. We will use Scala on the server and in the browser, PostgreSQL for persistence, and server-side rendering to deliver readable pages before the browser takes over interactivity.

The companion project is called `anjunar-blog-tutorial`. Each implementation article will extend that same project and explain how to run and check the result.

This first article establishes what we are building and how the parts fit together. The next article starts the project.

## What the finished application will do

Visitors will be able to read posts, search for them, and filter them by tags. Users will be able to register and sign in. Administrators will have an editor for writing, updating, and publishing posts.

A post will have a title, a stable URL, a publication state, and content containing formatted text, code blocks, and images. Drafts will remain private. Publishing a post will make it available on the public site.

The interface and posts will support English and German, with English as the primary language. We will handle interface translations separately from translated articles: a translated Save button and a German version of a post are different kinds of data.

The application will also need to survive beyond a development session. That means database migrations, a deployable package, HTTPS, backups, and enough logging to investigate a failed request.

The technical reference is Anjunar Stack. The tutorial will build a single site, so its data model and request handling will have no tenant IDs or tenant context.

## The stack and each part's job

Scala is used in two execution environments. The backend runs on the JVM. Scala.js compiles the frontend to JavaScript.

The main pieces have these responsibilities:

| Component | Responsibility |
| --- | --- |
| sbt | Define modules and dependencies, compile the code, run tests, and assemble the application. |
| Undertow | Accept HTTP requests and host the application. |
| RESTEasy | Route REST requests to annotated resource methods and produce HTTP responses. |
| Weld | Create and connect application components through CDI dependency injection. |
| Hibernate and PostgreSQL | Map persistent entities and store the application's data. |
| Agroal and Narayana | Manage database connections and transaction boundaries. |
| Scala.js UI | Build the interface, bind reactive state, handle forms, and connect routes to components. |
| GraalJS | Execute the frontend's server-rendering bundle inside the JVM process. |

We will also use Anjunar's JSON mapper and Hibernate DDL Manager. The mapper connects the entity schema, access rules, and JSON representation. The DDL manager handles database schema evolution from the entity mappings.

These are dependencies of the application. We will obtain their published versions from Maven Central. Each chapter will introduce the configuration needed for the part we are adding.

The frontend will produce two JavaScript bundles: one for rendering on the server and one for the browser. Both will use the same UI components, with separate entry points for their execution environments.

## Following a reader's request

Consider a published post titled *My First Post*. A visitor opens its public URL.

1. **Undertow receives the page request.** The application selects the route and starts rendering it with GraalJS.
2. **The page loads its data through the REST API.** The endpoint finds the post in PostgreSQL and checks whether the visitor may read it.
3. **The API returns the public representation.** The response includes the fields needed for this view. An entity graph describes the selected fields, while access rules govern what may be exposed.
4. **The frontend renders the page on the server.** The result contains the article's HTML, page metadata, and the initial application state needed by the browser.
5. **The browser takes over.** It loads its JavaScript bundle and hydrates the existing HTML, connecting the interface to its state and event handlers.

This flow gives us a concrete requirement to test: the initial HTML must contain the article. Once the browser starts, it must agree with the server about the selected route, language, and data.

An unpublished post follows a different path. The API must enforce its visibility before content reaches the public page. That decision belongs on the server even when the interface also hides unavailable actions.

## Following an editor's save

Writing a post adds another set of responsibilities.

The editor loads a frontend model that mirrors the fields exposed by the API. Form controls bind to that model. When the author changes the title and presses Save, the client sends the edited data and the version it originally loaded.

The server then needs to establish who is making the request, whether they may edit this post, and which fields they may change. A `PreparedChange` holds proposed changes until the controller has checked the relevant permissions and business conditions.

After applying the permitted changes, the server validates the resulting state and completes the transaction. A successful response carries the saved values and current version back to the editor.

Failures need equally specific behavior:

- An invalid title produces a field error the form can display.
- An unauthorized edit is rejected by the server.
- A save based on an outdated version produces a conflict instead of silently overwriting another edit.

These cases will be part of the implementation and its tests. They determine whether an editor can trust the application with their work.

## Keeping the pieces aligned

A post appears in several places: the database mapping, the entity schema, the REST response, the frontend model, and the form. Adding a field means following it through that chain.

For example, when we introduce a publication state, we will define its stored value, decide who may change it, include it in the appropriate API response, and bind it in the editor. We will also check that changing it affects public visibility correctly.

We will organize the code around those responsibilities. Feature modules will contain domain logic, REST endpoints, and frontend models and actions. Shared infrastructure will belong to platform modules. The application will assemble them into the running site.

The initial project only needs one backend module. We will add module boundaries as their responsibilities become concrete.

## The route through the series

Our first milestone is small: start the server locally, store a post in PostgreSQL, and retrieve it through REST.

From there, we will build the reader's interface, connect it to the API, and add accounts, permissions, and editing. Search, images, structured content, and translations will extend that working application.

We will then add server-side rendering and hydration, complete the public pages with metadata and discovery feeds, and package the application for deployment.

Each implementation article will identify the files being changed, show the relevant code, and provide a command or request that demonstrates the result. Published articles will be tied to specific commits or tags so their examples remain reproducible as the project grows.

The next article begins with the development tools, `build.sbt`, and the first backend module. Its goal is straightforward: start the application and receive an HTTP response.
