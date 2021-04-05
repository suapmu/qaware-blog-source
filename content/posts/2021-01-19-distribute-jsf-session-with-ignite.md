---
title: "Distribute Jsf Sessions With Ignite"
date: 2021-01-19
draft: true
author: "Susanne Apel"
tags: ["Ignite", "JSF", "Cloud", "Migration", "Session Distribution", "scaling"]
---

## Motivation

Within a resent project, we were working in a software migration project
insurance industry: applications formally running on an IBM Portal
server as JSF portlets now were targeted to run standalone in a
enterprise cloud environment.

This article is about the session handling of these applications. In the
old world, the sessions were sticky sessions, i.e. associated with a
running instance of the portal server. This means that every restart of
a portal server would result in a loss of session data. This was ok in
the old world since restarts and deployments were rare and mostly
management decisions. Running an application in a cloud environment,
rolling redeployments at least of un-changed software should not cause
problems. Otherwise, it would conflict with all resilience mechanisms a
cloud comes with.

Since, in an earlier, similar migration project, we made good
experiences with ignite

<!-- TODO: reference-->

the plan was to use ignite again for session distriubtion.

The present post describes the special challenges which were found in
the migrated JSF applications and why they were solved how.

<!-- todo: behalten? -->

In conclusion, there are a few more comments for future operations or
further future adjustments.

## Project-Setup and where observations are mede.

<!--todo: Aussage validieren-->

In the following, we will describe roughly in chronological order the
findings we had. We will also point out, which observations where a key
for a further understanding.

In order for you to understand the context of these observations, a  
few words about the project setup:

We did not only migrate a single application but a bunch of somehow
similar applications. The similarity was (among others) that all were  
JSF-applications build within the same time period and using same
(self-build) frameworks. Our software development team was responsible
for prviding central solutions for all applications where needed. The
development-teams of the applications were the ones migrating the
application with support from the central team.

In order to set the space for more technical findings, some more
technical information: All applications where migrated to Spring Boot
(we omit the reasons for that ) and

<!--todo: weg? -->

the central team was providing commonly used configuration via spring
boot starters.

<!--All non-migrated applications did have Spring already included. -->

## Step 0: Idea: Use Ignite's WebSessionFilter

As written earlier, there was one problem to solve: How to distribute
sessions. It was decided to use Ignite and it's Web Session Clustering
capabilities  
(see
[new slim documentation](https://ignite.apache.org/use-cases/caching/web-session-clustering.html)  
and  
[old more verbose documentation](https://apacheignite-mix.readme.io/docs/web-session-clustering)
).

The way to activate web session clustering capabilities is integrating
Ingnite's
[WebSessionFilter](https://ignite.apache.org/releases/latest/javadoc/org/apache/ignite/cache/websession/WebSessionFilter.html)

<!--todo: kommt later wirklich? -->

within your application's `web.xml` (or any equivalent, see later).

The key idea of the filter is to wrap around your application invocation
(as filters do)  
and exchange your app-servers implementation / instance of the session
object  
with an ignite-specific one. The ignite implemantation reads  
the data from distributed Ignite cache and after the request is
executed, it writes the session data to the distributed Ignite cache.  
Therefore, the session is distributed.


## Step 1: Correct integragtion of Ingnite's WebSessionFilter

So the setup in all applications was to register the WebSessionFilter
for all Servlets of the application with session interaction, in
particular

<!--todo: link -->

for JSF's FacesServlet.

<!-- todo: starter? -->

In order for the session replacement mechanism to work, the filter and
other interaction have to be brought to the correct execution order.  
In principle, the WebSessionFilter is meant to run as early as possible
to do its replacement.

<!--### Filter-Reihenfolge und weitere beteiligte Akteure-->

### Spring's interaction with session handling

The applications use Spring's autowiring-capabilities for accessing
session scoped beans. Out-of-the-box, the session is no available in
threads not answering the original request. In order to make the session
available to other threads, Spring provides 3 alternatives which are
documented as being interchangable from a Spring perspective:

`RequestContextListener`, `RequestContextFilter` und
`DispatcherServlet`.

Within our setup, the `RequestContextListener` must be avoided in all
parts of the application: Beeing a listener, it is invoked before
filters are invoked. This means that Ignite's `WebSessionFilter` is
invoked _after_ the listener and Igite is to late to exchange the
session with its own implementation. Within our applications, we decided
to uses `RequestContextFilter` which seems natural to wrap around the
JSF-Servlet-invocation. Furthermore, since also `WebSessionFilter` is a
filter, we can control the filter order and therefore the order of both
filters with respect to each other.

<!--todo: ggf. mehr details aus: -->
<!--Siehe auch-->
<!--[Issue-Kommentar](https://github.developer.allianz.io/LEAP/portal-migration/issues/288#issuecomment-264474)-->

### filter registration in Tomcat impl: path vs. servlet

It is important to know, that filter registrations via a path / url
pattern are treated differently from registrations via servlet name
within the Tomcat implementation:

Looking at the source code, the method,
`org.apache.catalina.core.ApplicationFilterFactory#createFilterChain` is
responsible for creating the filter chain. It not only looks at the
`Order` explicitly given but also give higher priority to filters that
are registerd via a url pattern. All filters registered via a servlet
name follow afterwards, now again sorted by the `Ordery` attrbute.

This is the reason why we registered the `WebSessionFilter` via a url
pattern in all applications for all servlets (or the urls associated
with these servlets). This ensures that teh `WebSessionFilter` can be
invoked early enough within the filter chain.


### Dispatcher Types

Within a concrete application, it is useful to  
write down your filter chain and to think about the desired order of filters in  
all types of invocations:

Why do we emphasize "all types of invocations"? Within a servlet, you  
may hve different `javax.servlet.DispatcherType`s, e.g.  
 `REQUEST`, `FORWARD`, `INCLUDE`, `ASYNC`, `ERROR`.
 E.g. we had a welcome page filter that dispatched with a
`FORWARD` to another page.
 In this situation the following happens
 when handling a request:
 The request is of type `REQUEST` and all filters registered for this
 type are applied. When we reach our custom forward filter, the
 request no is o f dispatcher type `FORWARD` and a new filter  
 chain is computed. So there might be filter that are executed twice  
 (registered for both  `REQUEST`and `FORWARD` and in filter chain before  
 custom redirect filter. On the other hand, if you have a filter  
 running after teh redirect filter and you do not register it for `FORWARD`  
 type, it will not be invoked.

 In order to avoid the double execution you can wrap your filters with Spring's
 `org.springframework.web.filter.OncePerRequestFilter`. In order to
 avoid a missing invocation of a filter, try to have your custom redirect filter
 in the rear part of the filter chain.

## Prevent parallel requests using the same session

Because of the working mechanism of `WebSessionFilter` described earlier,
it is essential do to avoid concurrent / parallel requests that have  
an interaction with the session: assume you were in such a situation, the  
following could happen: Both requests read the "old" session, and  
both requests also update the session, i.e. they write an altered session  
to Ignite at the end of request processing. In this case, the request,  
that terminates last overwrites the changes done in the other request.  
In order to address this problem, we introduced some mechanisms:

### BufferingFilter

We added a `BufferingFilter`-implementation to the centrally provided
parts of the software in order to include it in all applications in the
migration scope. The goal of this filter is to send the response to the
user / browser just _after_ the session was written to ignite.

In order to explain why we introduced this filter, assume that the filer  
is not there. In this case, assume e.g. that a header or more parts of  
the web page are sent to the browser before the process updating the  
session within Ignite is finished. In this case, a ressource can be  
queried or even the user could initialize an action such that the resulting  
request reads the old session from Ignite. This would imply a concurrent  
request - and this should be avoided as described earlier.  
Now, when configuring the filter order such that  
the `BufferingFilter` to be executed before `WebSessionFilter`,  
all user interaction or ressource requesting can be started only after  
a successful session update to Ignite.

### No session interaction for ressource-requests

Make sure that you do not interact with the session in request accessing
ressources. In particular, when rendering a webpage in JSF,
there is a resource-request querying `/javax.faces.resource/jsf.js.xhtml`.

When processing the request, some standard rendering within the JSF  
framework happens, including creation of some beans. If in this transitiv
 process of creating beans there is a session scoped bean, your ressource
 request now has session interaction and can be a request concurrent
 with other requests. Avoid session interaction within ressource requests.

Another idea you might come along with is to exclude the ressource  
request from the `WebSessionFilter`. This isn't a good idea either:  
Since there is still a session interaction, the request will not pass  
`WebSessionFilter` but will hit the application server. Now the  
application server will create a new not-Ignite-managed web session  
and will sent a set-cookie-header with the new JSESSION-ID within the  
 response. This will overwrite the Ignite-managed JSESSION-ID within  
 the browser and the "real" session id will be lost.


##  Serialization and  DE-Serialization with Ignite and multiples of previously same object


<!-- todo: davon etwas übernehmen? -->
<!--siehe auch Kommentar an-->
<!--[diesem Ticket](https://github.developer.allianz.io/LEAP/portal-migration/issues/288#issuecomment-272193)-->

### Keep session small and (Ignite-) serializable ("session requires application / singleton")

You should not hold services within the session / within a session
scoped bean. Regardless of the specific implementation of the serialization
used, holding services (e.g. for database access) within the session
can potentially cause problems during (de)serialization. This does not  
only affect the direct fields of session scoped beans but also transitivly  
the field of fieldas of session scoped beans (and so on).  
Avoid also storing e.g. a whole application context within a session.  
This does indeed conflict with serialization. If you need the spring  
context within your business logic (for whatever reason), do retrieve it  
from somewhere else.

An alternative for accessing services from session scoped beans is  
using lazy getters.

In order to have a reasonable easy rule, within session scoped beans,  
avoid holding references to application scoped beans / singletons.  
After reading also the next paragraph, you will see that there are even
<!-- todo: check: wirklich der nächste? -->
more reasons why you should  avoid such references: the multiplication of objects
described there would imply that your singleton objects aren't singletons  
after deserialization but there would be man copyies of it.


### Avoid Multiplication of Objects during Ignite Objekt-Deserialization.

#### session-requires-session

It can happen that objects are multiplied after deserialization: We mean  
that there are more instances of same object  
compared to state before serialization.

**Description of the Problem**

```
                                   +----------------------+
                   contains        |                      |
                 XXXXXXXXXXXXXXXXXX>    Container Bean    +-----------------+
             XXXX                  |    session-scoped    |                 |
+-----------XX-----+               |                      |                 |
|                  |               +----------------------+                 |
|                  |                                                        | has field 
|   Session        |                                                        |
|                  |                                                        |
|                  |                                                        |
+------------X-----+                                          +-------------v-----------+
              X                                               |                         |
                X X                                           |    Inner Bean           |
                      X X  X  X  X  X  X  X  X  X  XXXXX    X >    session-scoped       |
                         contains                             |                         |
                                                              |                         |
                                                              +-------------------------+

```

A setup where an object multiplication can happen is depiced in the previous
picture. Let's describe it:
An instance, that is called Container Bean, contains a (non-transient)
field to an instance called Inner Bean, where Inner Bean is also
session scoped. After de-serialization, i.e. reading from Ignite,  
unfrotunatly, you willfind to instances of Inner Bean where from a  
coding perspecive, you would expect them to be the same.

This isn't a problem of all Serialization methods. E.g. Java doc of  
`ObjectOutputStream` says:
> Multiple references to a single object are encoded using a reference sharing mechanism so that graphs of objects can be restored to the same shape as when the original was written.

`ObjectInputStream` as the reading counterpart describes reference handling similarly.

Within Ignite and its serialization mechanism, it wasn't so easy to
avoid this problem. We in our case decided to refactor the
applications. In parts we did an automated refactoring with spoon.
<!--todo: reference. -->

The refactoring done in the application applied to the example given above
would be: Inner Bean is still contained in the session, but all  
references from other session scoped beans must be cut. If e.g. Container  
Bean needs Inner Bean for a task, it retrieves it (lazyly) from the  
session directly.

To state it more generally:

##### Refactoring Goal for Avoiding Object Multiplication in the case session-requires-session

Each session-scoped bean `A` is stored within the session on its own.
No other session-scoped bean `B` is allowed to have a (non-transient)
field holding a reference to `A`.
In addtion, trasitivly, no (non-transient) transitive
field of `B` is allowed to hold  
a (non-transient) reference to `A`.

If an access from `B` to `A` is required, this can happen with lazy-loading
and e.g. accessing the JSF-Context or Spring application context.



##### Possible Alternative

We have the impression, that the session sharing mechanisim in
[Spring Session](https://spring.io/projects/spring-session)
works better. We do not know a production-ready integration with
ignite. If you can use [Redis](https://redis.io/) instead of Ignite,
you can investigate if the object multiplication happens there as well.

<!--todo: auch übersetzen. - will ich den Abschnitt haben?  -->

## Ignite-Grid

At start, we did use Ignite in the embedded mode. This implies that
all data is hold within the running application(s).

Later, we switched to the Ignite standalone solution: We do deploy
a seperate ignite grid with at default 2 instances. The reason for the
switch was initially that...

Der Grund für die Umstellung war initial gewesen, dass wir immer wieder
ClassCastExceptions und ArrayIndexOutOfBounds Exceptions in der
JSF-Phase `RestoreView` gesehen haben. Eine mögliche Interpretation
davon ist (weitere
Gründe siehe unten), dass die Session, die gelesen wird und mit der die
View auf einer bestimmten Anwendungsseite wiederhergestellt werden soll,
zu alt ist.

Wir hatten keinen Reproducer für dieses Verhalten, bis wir die
Auffälligkeiten in einem Lasttest immer geballt gesehnen haben. Bei
Verlangsamung der Anwendung (z.B. durch Debugger) traten die Fehler
nicht mehr auf. Alles sprach für eine Race-Condition.

Wenn man sich vor Augen hält, dass die Datenhaltung in der Anwendung
erfolgt, erscheint das sehr plausibel: Für jeden Datensatz gibt es in
Ignite einen PrimaryNode. Und wenn jetzt der andere Node diesen
Datensatz verarbeitet, hat er eine viel höhere Latenz als der andere
Knoten. So kann potentiell aus einer zu alten Session gelesen werden.

Bei der Verwendung des Standalone-Ignite-Grids sind im gleichen Setup
die Probleme so nicht mehr aufgetaucht.

Weiter Vorteile durch das Ignite-Standalone-Grid:
* Bei einem Neustart der Anwendung gehen nicht mehr die
  Anwendungs-Zustand- Daten der Nutzer verloren
* vermutlich (nicht getestet) bessere Skalierbarkeit der Anwendung:
  * Hintergrund: Ignite muss bei vielen Ignite-Instanzen viele Daten
    synchron halten / verteilen. In der embedded-Variante müssen so
    viele Daten-Backups angelegt werden, wie es Anwendungsinstanzen
    gibt. In der Standalone- Lösung kann die Menge der
    Ignite-Grid-Instanzen gleich bleiben und dennoch (vermutlich) die
    Anwendung hochskaliert werden.
* beide Punkte zeigen eine geringere Komplexität im Betrieb der
  eigentlichen Anwendung


<!-- todo: hier weiter: der idmapper ist wichtig-->

## Custom JSF-ID-Mapper

<!--todo: in Abhängigkeit zum vorigen Abschintt auch übersetzen. -->
Auch nach dem Einsatz des Ignite-Grids kam es in anderen Konstellationen
vereinzelt zu ClassCastExceptions und ArrayIndexOutOfBounds.  Diese waren
sehr sporadisch und in gewissem Sinne schwer zu reproduzieren. Zu
beobachten war folgendes Fehlerbild: Tauchte der Fehler einmal auf,
tauchte er Nutzerübergreifend an der gleichen Stelle / auf der gleichen
Seite wieder auf. Durch einen Anwendungsneustart verschwand der Fehler
wieder.

#### About the JSF IdMapper

We found the reason for the described behaviour in
 `com.sun.faces.facelets.impl.IdMapper`.  
It is part of the JSF-Impl and its Java-Doc reads as:

> Used to provide aliases to Facelets generated unique IDs with tend to be
 womewhat long.

The ids generated differ in priciple depending whether partial state
saving or full-state-saving is configured in the JSF-application (see  
JSF config param `javax.faces.PARTIAL_STATE_SAVING`).
However, they have in common: as an input, they take a so-called Faclet-Id
uniquly describing an element creating during transforming xhtml templates  
to html. This Facelet-Id input is mapped to an html-id that is generated.  
Within the implementation for obtaining these new ids, you can see that  
an AtomicInteger is incremented when new ids must be provided.  
You can see this in the (shortend) implementation of  
`com.sun.faces.facelets.impl.IdMapper`s:

```java
/**
 * Used to provide aliases to Facelets generated unique IDs with tend to be
 * womewhat long.
 */
public class IdMapper {

    private Cache<String,String> idCache = new Cache<>(new IdGen());


    public String getAliasedId(String id) {
        return idCache.get(id);
    }

    private static final class IdGen implements Cache.Factory<String,String> {

        private AtomicInteger counter = new AtomicInteger(0);

        @Override
        public String newInstance(String arg) throws InterruptedException {

            return 't' + Integer.toString(counter.incrementAndGet());

        }

    }
}

```

In order to see the impact of this mechanism for id generation, it
is important to understand the lifetime, i.e. the scope of the IdMapper.  
Regardless of the Interaction with `FacesContext` you can see in the above
implementation of `IdMapper`, the scope of the `IdMapper` in fact is
the application and is not user specific, an is fix for a ressource-alias  
(e.g. the path to an xhtml-file). In particular, the IdMapper are  
not shared between to instances of the application. This means, that  
in principle, it is possible that Ids are generated differently on two  
instances of the application. Below, we give examples when this really
happens. For the situations we found this to happen we need  
a certain xhtml-structure and furthermore, the very first requests hitting
both applications have to diffeer.


<details>
<summary>Details about the fact that teh IdMapper has application scope</summary>
<p>

When the IdMapper is used, it is normally read by the static method
`com.sun.faces.facelets.impl.IdMapper#getMapper(FacesContext ctx)`  
depending on the FacesContext. This seems to
be specific to request or session.

However, the content within FacesContext is pretty much the same for
every FacesContext:

Setting the IdMapper into the FacesContext is done in
`com.sun.faces.facelets.impl.DefaultFacelet`.
Here, the IdMapper ist taken from
`com.sun.faces.facelets.impl.DefaultFaceletFactory#idMappers`.
This factory lives as long as the application lives.

</p>
</details>

#### Sketch for reproducing the JSF state errors

In order to reproduce the problematic situation, we need some knowledge
about tag handler:


##### Diffrence between Tag Handler and Component

It is important to know that the sometimes wide-spread xml tags
`c:forEach`, `c:if`, `c:set` and also more tags area so-called
tag handler. In particular, this results in the behaviour that a
conditional non-rendering of a tag excludes this tag from the component
tree. In this case, the component tree has one element less.


For more details about the difference between tag handlers and
components see
[Jsf c:foreach Vs ui:repeat](https://rogerkeays.com/jsf-c-foreach-vs-ui-repeat)

##### Sketch for Reproduction

Assuming the following constellation, we explain the problems
observed:

Assume, an xhtml-file (say `input.xhtml`) contains a form and also
(somewhere close to the top) a `c:if`, that is rendered only if some
value in some session-scoped beans says that the user did an invalid
input before. This value is supposed to be independent from  
validation done within the JSF lifecycle.

<!--todo: hier weiter. -->

Für das weitere Setup brauchen wir 2 frisch gestartete
Anwendungesinstanzen. Wir können uns eine Reproduktion in etwas so vorstellen:
* Der Nutzer ruft per Get `input.xhtml` auf. Er landet auf Instanz A. Auf
  Instanz A wird der IdMapper initialisiert. Das `c:if` bekommt noch
  keine Id zugeordnet.
* Der Nutzer füllt das Formular aus und sendet es per Post.
* Annahme: der Request landet jetzt auf Instanz B. Hier ist der IdMapper
  noch nicht initialisiert, in der RestoreView-Phase wird für
  `input.xhtml` erstmalig ein Faclet erzeugt. Durch die Veränderung in
  den Daten wird jetzt das `c:if` kondtionial gerendert und landet im
  Komponentenbaum. Dadurch bekommt es beim atomaren hochzählen der ids
  im `IdMapper` eine id, die auf Instanz A für das darauf folgende Element
  im Komponentenbaum verwendet wurde. Tatsächelich sind ab dem `c:if`
  alle Ids von Instanz B im Vergleich zu Instanz A um eins verschoben.

Wenn jetzt im JSF-Liefecycle der Komponenten-Zustand aus der Session auf die
Komponenten angewendet werden soll und das Mapping über die ids
geschieht, dann passen die Werte nicht mehr zusammen.

Im Partial-state-saving-mode von JSF funktionieren dann ClassCasts nicht
mehr oder Arrays mit Daten sind zu kurz. Das passt zu den beobachteten
Phänomenen.

Im full-state-saving modus kommt es zum Teil schon zuvor aufgrund eines
anderen Fehlers zu doppelten Ids.

#### Abhilfe

##### Id-Mapper

Wir haben das Problem gelöst, indem wir eine eigene
IdMapper-Implementierung hinterlegt haben. Mehr infos in
[Megatherium-Starter: Optional Change The Idmapper Working Mode](https://github.developer.allianz.io/LEAP/leap-megatherium-starter#optional-change-the-idmapper-working-mode)

Wichtig dabei ist zu beachten, dass die transitive maven-dependency
`AA-leap-jsf-patch` im Classpath _vor_ der jsf-impl eingebunden wird.

Im
Leap-Image wird der Classpath alphabetisch aufgebaut und der
Namensgebung steht der Patch weit vorne im Alphabet.

Für die lokale Ausführung mit maven muss die
`AA-leap-jsf-patch`-Dependency vor der JSF-Impl eingebunden werden.

Der Patch behebt die Probleme wie folgt: nach Aktivierung von
* `facelet_id`-Mapper: full- und partial-state-saving
* `mapped_id_with_host`-Mappper: nur für full-state-saving anwendbar.


##### Weiteres

Prinzipiell treten die Probleme vermutlich weniger auf, wenn weniger
Tag-Handler verwendet werden. Zusätzlich sollte es prinzipiell auch
möglich sein, alle Tags von Komponenten,
die ihren Zustand in der Session speichern,
mit eigenen html-ids zu versehen. Dieses Vorgehen ist fehleranfällig, da
bereits das Vergesssen einer Id erneut zu den Problemen führen kann

Zu berücksichtigende Komponenten wären dabei alle, die
`javax.faces.component.StateHolder#saveState` und
`javax.faces.component.StateHolder#restoreState` implementieren.

Eine (rechtlich nicht bindende) Lizenzbewertung findet sich hier:
[Mojarra-Id-Mapper-Lizenz-Bewertung](Mojarra-IdMapper-Lizenz-Bewertung)




