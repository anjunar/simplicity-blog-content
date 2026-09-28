# Translating the Interface

Our blog can publish an article with formatted text and images. Now a reader
should be able to use its navigation in German, and an administrator should be
able to edit that same article with German controls.

These are interface translations. Switching from **Save post** to **Beitrag
speichern** must leave the title and document untouched. Translating those
editorial values needs its own model, which we will build in chapter 21.

This chapter adds English and German URLs, one message catalog and a language
switch. The [checkpoint guide](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/docs/translating-the-interface.md)
contains the runnable version and verification commands. We use scalajs-ui
1.0.13 from Maven Central; there is no database migration.

## Let the URL choose the language

Open `/de/posts/same-post` directly. The page should already use German, without
requiring a previous visit or a stored browser preference. Reloading it and
sharing its address should preserve that choice.

We initialize the runtime from the same URL that initializes the router.
Inside [BlogPage.compose](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala),
before composing either navigation or routed children:

```scala
import ui.core.i18n.I18nRuntime

val initialUrl = cursor.browserUrl.getOrElse("/")
val translations = I18nRuntime.managed(BlogI18n.config, initialUrl)
I18nRuntime.provide(translations)(using this)
```

`BlogI18n.config` declares English and German as supported locales and English
as the default. Providing the runtime makes it available throughout the
component tree, including the editor. We do not keep a second language value
in local storage.

The HTTP server must recognize these addresses too. `FrontendHandler` now
serves the existing browser shell for known routes under either `/en` or
`/de`. Account and editorial pages retain their private cache policy.
REST and image requests still use `/service/`; a language prefix belongs to
the page address, not the API.

Unknown locales and unknown routes return 404. A missing post at a known
route still loads the shell before REST reports the missing content.
Server-rendered page statuses belong to the later SSR chapter.

## Give each message a catalog entry

Earlier chapters already used the `i18n` macro for application labels. Those
English source messages now become the keys for German translations.

[BlogI18n](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogI18n.scala)
uses this helper. The excerpt shows two entries from its larger catalog:

```scala
import ui.core.i18n.{CatalogEntry, I18n, I18nLocale, MessageCatalog, RuntimeMessage, i18n}
import ui.editor.EditorMessages

val German: I18nLocale = I18nLocale("de")

private def de(message: RuntimeMessage, translation: String): CatalogEntry =
  I18n.entry(message.key).translations(German -> translation)

val catalog: MessageCatalog = MessageCatalog(
  de(i18n"Save post", "Beitrag speichern"),
  de(EditorMessages.editImage, "Bild bearbeiten")
)
```

The macro supplies the message identity and source information. We refer to
`message.key` rather than maintaining our own hashes or identifiers.
The resolver uses the English source when a German entry is missing.

The second entry illustrates a useful boundary: the editor publishes
`EditorMessages`. Its toolbar and dialogs use the same runtime as our
application. We translate those public messages in our catalog; the existing
`editor("content")` DSL and plugin configuration remain intact.

A catalog should contain whole messages. Splitting “Save”, “post” and punctuation
into separately translated pieces would impose English sentence structure on
every language.

## Keep values and bindings intact

Some messages contain data. Our list already knows how many rows it received
and the total number of matches. Inside
[PostListPage's composition](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostListPage.scala):

```scala
import ui.core.i18n.{I18n, i18n}
import ui.core.layout.TextComponent.text

text(i18n"Showing ${I18n.named("count", table.rows.size)} of ${I18n.named("total", table.size)} posts") {}
```

The German pattern is `"{count} von {total} Beiträgen"`. Translators can place
the named values where the sentence needs them. The catalog's sample values
identify the pattern; the rendered message supplies the actual numbers.
Tests check that every translation preserves its source placeholder names.

These patterns perform named substitution. They are not an ICU pluralization
language, so we choose wording that works for our count display.

The DSL accepts the macro directly: `text(i18n"Save post")`,
`button(i18n"Save post")` or, inside a form control,
`placeholder = i18n"Choose an author"`. These APIs use `TextValue` to resolve
the component's i18n context and maintain the reactive binding. The same applies
to `ariaLabel = i18n"Publication status"` and
`SelectOption("DRAFT", i18n"Draft")`. No explicit translation call is needed.

For messages selected by changing state, pass `state.map(...)` to the DSL and
return the appropriate `i18n` message from the mapping. `TextValue` follows both
the message property and the locale. The macro alone does not observe arbitrary
interpolated values: our list data is fixed for that routed page; changing data
would need to produce an updated message.

## Treat a language switch as navigation

The header keeps the current route while changing its locale prefix. A reader
on `/de?q=Scala#posts` moves to `/en?q=Scala#posts`, with the query and anchor
preserved. Browser Back returns to German.

The handler in `BlogPage.compose` uses the router's public path API:

```scala
import org.scalajs.dom
import ui.core.i18n.I18nLocale

def changeLanguage(next: I18nLocale): Unit =
  if (!languageNavigation.blocked.get && translations.locale.get != next) {
    val state = router.state.get
    val suffix = if (cursor.isBrowser)
      dom.window.location.search + dom.window.location.hash
    else state.search + state.hash
    router.navigate(router.localizedPath(state.path, next) + suffix)
  }
```

Using the current browser fragment matters for account links. Confirmation
and password-reset pages remove their secret token from the visible URL
after reading it. Restoring an earlier router hash could put that token back.

Navigation also replaces the routed component. An unsaved post, unfinished
search or partly entered password must not disappear when someone presses
the language button.

[LanguageNavigation](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/LanguageNavigation.scala)
therefore combines guards owned by the current forms. A form watches its
changed values and pending operations; disposal removes its guard and
subscriptions. While blocked, the language buttons are disabled and explain
that the user must finish or clear the form first.

After saving, the post can reopen in the other language with its saved values.
Token-bearing recovery pages stay guarded until the user leaves through their
normal completion link. This protection covers the language buttons; it is
not a general safeguard against reload or every possible navigation.

## Verify the complete interaction

Start at `/de`, open a post, switch to English, go Back and reload. Navigation
and controls should follow the URL while the article remains unchanged.
The browser also updates `html.lang` for assistive technology.

Then edit a title. Switching language should be unavailable until the edit is
saved or discarded. After saving and switching, reopen the image dialog:
its labels should be German and the document should retain its content.

The six new browser tests cover these transitions, account errors, recovery
tokens and error routes. Catalog tests check fallbacks, reactive messages and
placeholder names; server tests check localized routes and cache policies.

There are explicit limits to this checkpoint. Detailed backend errors and
built-in validation diagnostics retain their existing English strings, as do
transactional emails. The initial static HTML is still English until browser
boot; SSR will address that later.

We now have one interface in two languages. Next we will model translations
of the articles themselves, including how a reader gets a useful result when
a requested translation is missing.
