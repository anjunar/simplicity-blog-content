# Ein Formular vom Anfang bis zum Ende entwickeln

Die API kann einen Artikel speichern. Nun soll das auch für Administratoren im Redaktionsbereich möglich sein.

Ein nützliches Formular benötigt mehr als Eingaben und einen Button. Es muss die Version senden, die der Editor tatsächlich geladen hat, die Feldfehler des Servers anzeigen und den eingegebenen Text beibehalten, während ein vorheriger Speichervorgang noch läuft.

Dieses Kapitel setzt den gesamten Ablauf um. Wir erstellen einen Entwurf, bearbeiten ihn, löschen ein optionales Feld und behandeln einen Versionskonflikt, ohne den lokalen Text zu verlieren.

## Beginnen Sie mit dem Schreibvertrag

Ausgangspunkt ist Kapitel 14:

```text
git switch --detach bb212a133dc20f9647ead13d5347be42e5829592
```

Der fertige Kapitelstand für dieses Kapitel ist:

```text
git switch --detach 9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b
sbt --server frontendAssets
sbt --server "application-backend/run"
```

Behalten Sie die vorhandene Entwicklungsdatenbank und den Administrator. Es ist keine Schemamigration oder ein Abhängigkeits-Upgrade erforderlich. Für lokales HTTP behalten Sie `BLOG_COOKIE_SECURE=false` bei. Melden Sie sich unter `/en/account` an und öffnen Sie den redaktionellen Arbeitsbereich.

Die [begleitende Anleitung](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/docs/building-a-form-from-end-to-end.md) enthält die Konfiguration und den vollständigen Walkthrough.

Der Arbeitsbereich bietet jetzt **New post**, wenn die Sammlung einen Create-Link bereitstellt. Eine Post-Vorschau bietet **Edit post**, wenn das Detail einen Update-Link enthält. Auch bei direkter Navigation werden diese Berechtigungen geprüft. Das Backend bleibt für die Autorisierung bei jeder Anfrage verantwortlich.

## Das spiegelnde Entity-Modell direkt binden

Wir haben bereits einen Frontend-BlogPost mit den gleichen veröffentlichten Feldern wie die JPA-Entität. Das Formular bindet sich an dieses Objekt.

Die Titel-Property trägt nun auch die Client-Constraint-Metadaten:

```scala
  @(NotBlank @field)()
  @(Size @field)(min = 3, max = 180)
  val title: Property[String] = Property("")
```

Dieser Auszug gehört zu `BlogPost.scala` im Frontend. `Property` wird aus `ui.core.state` importiert, `NotBlank` und `Size` aus `ui.forms.validators` sowie `field` aus `scala.annotation.meta`. Dies sind Frontend-Deskriptor-Annotationen, keine JPA-Annotationen.

Der Slug trägt die gleiche Länge und Kleinbuchstabenstrichmuster wie die Entität. Zusammenfassung ist auf 300 Zeichen und Inhalt auf 100000 begrenzt. Die Formularbibliothek liest diese Annotationen aus ihrem Modelldeskriptor.

Clientseitige Prüfungen geben sofortiges Feedback. Serverseitige Feldvalidierung findet immer noch in `applyChanges` des JSON-Mappers statt. Hibernate behält seine bestehende vollständige Validierung vor Persistenz bei. Wir fügen keinen Controller-Validierungspass hinzu.

Die beiden String-Felder, die im gelesenen Modell optional waren, bedürfen einiger Sorgfalt. Native Texteingaben binden `Property[String]`; sie wandeln kein `Option`-Objekt in Eingabetext um. Inhalt und Zusammenfassung verwenden daher nullbare String-Eigenschaften. Ihre Namen und JSON-Werttypen bleiben die gleichen wie auf der Entität.

Null ist immer noch wichtig. Eine öffentliche Liste lässt den Inhalt aus, während ein Entwurfsdetail eine leere Zeichenfolge haben kann. Der redaktionelle Detailloader stellt leeren Entwurfsinhalt wieder her; writeBody lehnt ein Modell ab, dessen Inhalt noch fehlt. Wir verwandeln eine teilweise geladene Listenzeile niemals in ein vollständiges Bearbeitungsformular.

## Den Formularbaum zusammenhalten

`PostEditorPage` enthält `form(post)` in seiner `compose`-Methode. Jedes Steuerelement nennt genau die Property, an die es bindet: `input("title")`, `input("slug")`, `textAreaInput("summary")` und `textAreaInput("content")`.

Hier ist das vollständige Titelfeld aus diesem Formular:

```scala
        div {
          classes = "post-field"
          label { fieldLabel ?=> fieldLabel.setAttribute("for", "post-title"); text(i18n"Title") {} }
          val control = input("title") { fieldInput ?=>
            id = "post-title"
            fieldInput.setAttribute("aria-describedby", "post-title-errors")
          }
          control.addDisposable(control.invalid.observe(value => control.setAttribute("aria-invalid", value.toString)))
          paragraph { id = "post-title-errors"; classes = "field-error"; text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {} }
        }
```

`PostEditorPage` importiert die Layout-DSL, `Input.input`, `Form.form`, das `i18n`-Makro und `scala.scalajs.js`. Die vollständige Datei zeigt diese Importe und die umgebenden `compose`-, `render`- und `form`-Blöcke.

Das Label verweist auf die tatsächliche Eingabe. Der Fehlertext hat eine stabile ID und aria-invalid folgt dem Validierungsstatus des Steuerelements. Ein Bildschirmleser kann das Feld, sein Label und seinen Fehler zuordnen; der Browsertest lokalisiert es an diesem Label.

Der explizite Empfänger von `setAttribute` ist beabsichtigt. Eine Seitenkomponente hat auch diese Methode. Wenn Sie es ohne einen Empfänger in einem verschachtelten Block aufrufen, kann die äußere Komponente anstelle des beabsichtigten Labels oder Eingabefelds geändert werden.

Markup, Bindungen und Ereignisverhalten des Feldes bleiben in der Komposition zusammen. Es gibt keine Rendering-Helfer, die Teile der Form über Methoden verteilen.

## Überprüfen Sie die Bindung vor dem Senden

Das Formular verarbeitet das native Submit-Ereignis, so dass die Eingabe von Button und Tastatur den gleichen Pfad verwenden. Vor dem Speichern löscht es alte Serverfehler und führt zwei verschiedene Prüfungen durch:

```scala
            val bindings = mountedForm.validateBindings()
            val validation = mountedForm.validate()
            if (bindings.nonEmpty) actions.notice.set(BindingFailed)
            else if (validation.nonEmpty) actions.notice.set(Invalid)
```

Dies ist ein Auszug aus dem Submit-Handler von `PostEditorPage`. BindingFailed und Invalid sind SaveNotice-Werte, die in die Komponente importiert werden.

`validateBindings` erkennt ein Steuerelement, das nicht mit der benannten Modelleigenschaft verbunden ist. `validate` prüft die gebundenen Werte und macht deren Fehler sichtbar. Ein falsch geschriebener Feldname darf keine Eingabe erzeugen, die editierbar aussieht, aber das Modell niemals aktualisiert.

Erst nachdem beide Prüfungen erfolgreich sind, ruft der Handler die Save-Aktion auf. Die Speichern-Schaltfläche wird während einer Anforderung deaktiviert. Die Textsteuerelemente bleiben editierbar.

## Lassen Sie den Mapper die Nutzlast bauen

Der Scala.js-JSON-Mapper serialisiert geänderte Eigenschaften. Das gibt uns das teilweise Aktualisierungsformat aus Kapitel 14, ohne jedes Feld manuell in ein separates Anforderungsobjekt zu kopieren.

BlogPost.writeBody behandelt die wenigen Transportregeln rund um dieses Mapping:

```scala
  def writeBody(): js.Dynamic = {
    require(content.get != null, "Load a detail before editing")
    val body = JsonMapper.serialize(this)
    if (id.get.isEmpty) js.special.delete(body, "id")
    // The serializer emits dirty properties. A PATCH precondition is required even when clean.
    if (id.get.nonEmpty) body.updateDynamic("version")(version.get.toDouble)
    if (!js.isUndefined(body.summary) && body.summary.asInstanceOf[String] == "") body.updateDynamic("summary")(null)
    body
  }
```

JsonMapper kommt von ui.json; js wird von scala.scalajs importiert.

Ein vorhandener Beitrag muss immer seine gespeicherte Version senden, auch wenn sich diese Eigenschaft nicht geändert hat. Versionsnummer 0 ist gültig. Ein neuer Post lässt seine leere ID weg, so dass der Server eine generiert.

Eine entfernte Zusammenfassung wird als JSON-`null` gesendet. Leere Inhalte bleiben eine leere Zeichenfolge: Ein leerer Entwurf ist erlaubt. Eine ausgelassene Zusammenfassung oder Inhaltseigenschaft bleibt ausgelassen. Diese Fälle dürfen nicht in einer pauschalen Regel für leere Werte zusammenfallen.

`status` und `publishedAt` tragen `JsonIgnore(deserializable = true)`. Wir lesen sie aus den Antworten, aber die Veröffentlichung verwendet immer noch die dedizierten Befehle.

Wenn Sie beispielsweise nur den Titel der Version Null ändern, entsteht ein Teilkörper mit Titel, Identitäts-/Typ-Metadaten und Version. Es sendet nicht den gesamten Beitrag erneut oder überschreibt seine unberührte Zusammenfassung.

## Die Eingabe vor dem asynchronen Warten festhalten

Das Speichern erfordert ein CSRF-Token aus der aktuellen Sitzung. Dieser Lookup ist asynchron, und der Editor tippt möglicherweise weiter, während er ausgeführt wird.

PostEditorActions erfasst somit sowohl die Vergleichswerte als auch die JSON-Nutzlast sofort:

```scala
      val submitted = post.snapshot
      // Serialize now, before session lookup; later keystrokes belong to the next save.
      val body = post.writeBody()
```

PostSnapshot ist eine kleine, unveränderliche Aufzeichnung der vier editierbaren Textwerte. Es ist ein interner Vergleichszustand. Die REST-Nutzlast kommt immer noch von BlogPost über JsonMapper.

EditorialService überprüft dann die Beziehung, die Methode und das Ziel des gelieferten Links, bevor die Sitzung nachgeschlagen wird. Es sendet POST zum Erstellen und PATCH zum Update mit dem CSRF-Header. Der vorhandene ApiLink-Schutz beschränkt das Ziel auf denselben Ursprung und einen Pfad unter `/service/` dieser Anwendung.

Die aktualisierte Detailhülle des Servers liefert die gespeicherten Werte, die Version und den nächsten Satz von Links.

## Die Speicherantwort übernehmen, ohne neue Eingaben zu verlieren

Betrachten Sie diese Sequenz:

1. Die bearbeitende Person reicht den Titel "Erste Revision" ein.
2. Bevor die Antwort eintrifft, geben sie "Zweite Revision" ein.
3. Der Server gibt die gespeicherte "Erste Revision" mit Version 1 zurück.

Das Ersetzen des Formularmodells durch diese Antwort würde "Zweite Revision" löschen. Das vollständige Beibehalten des alten Modells würde die neue Version und alle anderen anerkannten Serverwerte verlieren.

BlogPost.mergeSaved vergleicht jedes aktuelle Feld mit seinem eingereichten Wert:

```scala
  def mergeSaved(saved: BlogPost, submitted: PostSnapshot): Unit = {
    val before = submitted.values
    editableFields.zip(saved.editableFields).zipWithIndex.foreach { case ((current, fresh), index) =>
      val unchangedSinceSubmit = current.get == before(index)
      current.setDefault(fresh.get)
      if (unchangedSinceSubmit) current.set(fresh.get)
    }
    id.set(saved.id.get)
    id.setDefault(saved.id.get)
    version.set(saved.version.get)
    version.setDefault(saved.version.get)
    status.set(saved.status.get)
    publishedAt.set(saved.publishedAt.get)
  }
```

editableFields und PostSnapshot.values verwenden die gleiche Reihenfolge: Slug, Titel, Inhalt und Zusammenfassung. Ein Feld, das immer noch seinem eingereichten Wert entspricht, kann den Serverwert akzeptieren. Ein geändertes Feld behält die neuere Eingabe des Editors.

Der Default-Wert jedes Felds wird auf den bestätigten Wert gesetzt. Das ist wichtig, weil der JSON Mapper den Standard verwendet, um zu entscheiden, welche Property geändert wurde. Die nächste Anfrage sendet die verbleibenden Änderungen gegenüber Version 1.

Die Aktion ersetzt auch die Antwortlinks. Wenn ein Update nicht mehr angeboten wird, wird kein weiteres Save angeboten.

Auch beim Anlegen wird die Antwort so übernommen. Die zurückgegebene ID wird übernommen, auch wenn die Eingabe während der POST fortgesetzt wird. Der nächste Speicher verwendet ein Update, so dass er nicht versehentlich einen anderen Beitrag erstellen kann. Sobald keine neueren Änderungen mehr vorliegen, ersetzt der Router die New-Post-URL durch die Edit-URL des gespeicherten Posts.

## Setzen Sie Fehler, wo der Editor auf sie reagieren kann

Kapitel 14 bietet bereits typisierte ProblemDetails. Das Formular konvertiert übereinstimmende Feldfehler in die Fehlerantwort der Formularbibliothek:

```scala
        mountedForm.addDisposable(actions.errors.observe(values =>
          mountedForm.setErrorResponses(values.map(value => ErrorResponse(value.message, value.path)))))
```

ErrorResponse wird von ui.forms importiert. Ein Pfad wie `["slug"]` löst sich zum Slug-Feld auf. Ein Fehler auf Entity-Ebene wie publicationConsistent hat keine eigene Eingabe, so dass seine Nachricht in der Benachrichtigung auf Formularebene erscheint.

Fehler gehören auch zu einer bestimmten Einreichung. Wenn der Titel korrigiert wurde, während die Anforderung anhängig war, darf ein Fehler über den alten Titel den neuen Wert nicht als ungültig kennzeichnen. PostEditorActions vergleicht die eingereichten und aktuellen Werte, bevor ein Feldfehler angehängt wird.

Ein Slug-Konflikt ist korrigierbar: Zeigen Sie den Slug-Fehler und erlauben Sie ein weiteres Speichern. Eine veraltete Version braucht eine andere Antwort. Bewahren Sie alle lokalen Eingaben auf, deaktivieren Sie das Speichern und bieten Sie **Discard my edits and reload**. Diese Aktion ersetzt ausdrücklich das lokale Formular durch den aktuellen Serverzustand.

Authentifizierungsfehler und unsichere Netzwerk- / Serverergebnisse behalten auch den Text. Sie stoppen blinde Versuche. Insbesondere könnte ein nicht bestätigter Anlegevorgang bereits committet sein; eine automatische Wiederholung könnte einen zweiten Artikel erzeugen.

## Respektieren Sie die Lebensdauer der Komponente

Eine Antwort kann eintreffen, nachdem der Editor weg navigiert hat.

Die Seite registriert `actions.dispose` über `addDisposable`. Beim Entsorgen werden Modellbeobachter entfernt und verhindert, dass spätere Reaktionen den Zustand ändern oder in die verlassene Form zurückkehren.

Es wird keine Anfrage rückgängig gemacht, die bereits den Server erreicht hat. Dieses Kapitel hat keine automatische Speicherung, Offline-Entwurfsspeicherung oder automatische Konfliktverschmelzung. Beim Wegnavigieren oder Neuladen gehen ungespeicherte Eingaben verloren. Kopieren Sie benötigten Text, bevor Sie bei einem Konflikt Ihre Änderungen ausdrücklich verwerfen.

## Versuchen Sie den kompletten Workflow

Öffnen Sie Editorial, wählen Sie New post und geben Sie einen Titel, Slug, Zusammenfassung und Inhalt ein. Speichern, ändern Sie dann den Titel und löschen Sie die Zusammenfassung. Speichern Sie erneut und laden Sie die Seite neu. Der Titel sollte bestehen bleiben und die Zusammenfassung sollte leer bleiben.

Öffnen Sie für einen Konflikt die Bearbeitungsroute des gleichen Beitrags in zwei Registerkarten. Speichern Sie im ersten Tab. Geben Sie in der zweiten einen anderen Titel ein und speichern Sie seine ältere Version. Das Formular sollte Ihren Text behalten, den Konflikt erklären und das Speichern deaktivieren, bis Sie entscheiden, wie Sie fortfahren möchten.

Der automatisierte Datenbank-Workflow führt diese Operationen gegen die reale API aus. Die kontrollierten Browsertests verzögern außerdem Antworten, ändern Berechtigungen, liefern Feldfehler zurück und prüfen das mobile Layout.

```text
sbt --server "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=forms
```

Das Formularprojekt verwendet den dedizierten Testadministrator, die migrierte Datenbank und die im Begleithandbuch beschriebene psql-Konfiguration. Es entfernt danach seinen eigenen erstellten Post.

Die Überprüfung für dieses Kapitel besteht 21 Scala.js-Tests, 18 betroffene Backend-Integrationstests und alle 49 Browsertests. Die Browser-Gesamtzahl umfasst 42 kontrollierte Contract-Tests und sieben echte Abläufe.

Als nächstes werden wir die wachsende Postliste einfacher machen, mit Suche, Filterung und Paginierung zu navigieren.