

A QtQuick framework for visualizing models
===========================================

---

Opinion #1
==========

Back in the QtWidgets days, if you had modelized data, you could tell
management:

* Yes, I can display it with Qt
* Yes, I can display it, but I need to implement a thing called
  QtWidgets QAbstractItemView

#### Now we have

* Yes, the ListView/GridView/TableView will work
* Sorry, Qt doesn't allow to display our data, implementing new views is
  expensive, let use something else.
* Lets use the Web stack, it can do everything

#### Worst still

If you had old, complex and battle tested models and a QtWidgets based GUI,
you are stuck with a *legacy* and non-mobile-friendly GUI stack **forever**.

I see this as a critical issue that prevents current/former customers to use
the Qt technologies again. It's bad for business, bad for consultants and only
profit the Web stack, because that's what they will switch to.

---

Why?
====

---

Historical notes
================

The old QtWidgets design scaled well because it was simple. You had a single
"delegate" instance with a QPainter function and the metadata necessary to
paint at the right position. The size hints were previously determined. Rather
than after the fact like QML.

QtQuick uses "declarative" widgets. They are QObjects and created by the engine
to turn the intent into a GUI visual. It also uses the GPU much more often than
QtWidgets and tracks many more buffers and objects sent between the CPU and GPU.
These implementation details make using a single instance painted multiple time
impractical.

QtQuick2.ListView and GridView sidestep this problem by ignoring it.
However it restricts them to the smallest possible subset of the model topology:
lists. While tables are quite well implemented in Qt 5.12, it took a decade and
now we have the two simplest model typologies. This wont scale to trees very
well.

---

Opinion #2
==========

The current upstream view implementation was "rushed" when QML was first
written as time-to-market was important to stay competitive. Those views at the
time targeted mobile and embedded and ignored the more complex desktop use
cases. This made sense from a business point of view and I would have taken
the same decision, but now there is a technical debt to repay.

The current view implementation do too many things each classes and they are
tangled in a web of low level private APIs.

This design is and was always faster to develop than taking an overly general
conceptual approach to solve the problem. However it isn't flexible enough to
be adapter to use cases that were not taken into account when the view was
coded.

---

# Why: Limitations of the current QtQuick.Views architecture

* Each class/component do too much, it causes much duplications across views
    * Optimizations are done in the higher layers instead of reusable low-level
      building blocks
    * Some views have slow corner cases, but they are in fact impossible to
      optimize due to implementation details.
* A lot of Qt5 QML components rely on "private" and "QML only" APIs
  like the QQuickAnchors and internal QtQuick "V4" engine data types
* Existing Qt customers codebase designed for QtWidgets don't fully map into
  the current QtQuick.Views
    * They might leave the Qt ecosystem if they can't re-use their past
      investment.
* It is "all or nothing". Either the views fit your use case or you can't use
  QML.
    * If you hit *any* limitation, the amount of work to ship QML based apps
      skyrockets.

---

Opinion #3
==========

A large and complex body of code written from scratch to handle modelizations
and all its corner cases is a better path forward than brute forcing every
topologies one by one.

I am not claiming the design described in this document is the only way or the
sanest one. But I claim it solves the limitations mentioned above.

---

What is this all about?
=======================

In the last year I designed a generic framework to implement custom views for
Qt models. Since September 1, I worked full time implementing it.

This work has been sponsored by BlueSystems. It is part of a larger body of work
allow existing Qt/KDE apps to be ported from QtWidgets to QtQuick without major
rewrites. This is important for BlueSystems projects such as Plasma Mobile and
other projects under NDA.

Since from its inception it duplicates a lot of components provided by QtQuick
itself, we should investigate what can be shared.

Even if this ends up a research project and is never upstreamed, there is
lessons to be learned and maybe some optimizations strategies can make their
way upstream.

---

Table of content
================

 * Goals and scope
 * General architecture
 * Public API components (adapters)
 * Private API state trackers
 * Optimization strategies
 * Status, demo, timeline, etc

---

Functional goals
================

---

# Functional goals

The goal is first are foremost to offer a framework to create custom model
views to be used in QtQuick2 applications.

* Support all model typologies supported by QtWidgets (and more)
* Restore the ability to write views for the existing models in C++
* Deliver fast performance even when using user defined views (without tons
  of complexity in the view code)
* Allow existing QtWidgets applications to be ported to QtQuick/Kirigami by
  leveraging their existing models as-is.
    * Potentially allow a migration path such as using raster delegates when
      it makes sense
* Support all technologies (MIME, drag and drop, etc) relevant for the desktop
* Make implementing new views as easy as it can get, hide all complexity
* Provide all the building blocks in the public API to create more views
  without unmaintainable amount of code duplication

---

# Non functional goals

* Do not use Qt private APIs or at least not require them
* Keep strict separation of concerns and small components
* Put hard-line limits on scope creep for each component.
    * If it can be split in half and still compile, then it will be.
    * If it adds classes whose sole purpose is to glue smaller ones, then so be
      it.
    * Cache fault and indirection related loss of performance are acceptable
      given they are irrelevant compared.
    * Micro optimization that cause 2 way coupling should not be implemented
  to QtQuick own overhead.
* Get a stable-ish API before 2019
* Split things into smaller "building blocks" and assemble them as late as
  possible to maximize the future flexibility
* Have more Q_ASSERT in debug mode. Invalid models and mis-coded views should
  have undefined bahavior in release mode. Garbage in: garbage out. The model
  developers have to implement them properly.
    * The state machine should fail into their ERROR state when things go wrong

---

# Code pattern goals

* Use precise state tracking to (try to) optimize away uneeded operations
* "bloat" all the complexity in the lowest layers, never expose complexity in
  the public API even if doing so requires more complexity
* Have an highly modular "middle" layer on top of the optimizations and make it
  part of the public API
* Only ever use public APIs to build the "real" views

---

# Design process

* Solutions that try to fix everything have inherent "front loaded" complexity.
  It is an acceptable tradeoff.
* The problem was studied and broken down into:
   * Small chunks, represented here by the adapters
   * Transactions & life cycles, represented here by state trackers
* Design the public API to scale horizontally in complexity when developers
  implement custom views

---

Rationals: Why state machines?
==============================

State machine are like gears made out of Adamentium: They are hard to work with,
but once set, they are indestructible.

![Adamentium](wolverine-jackman-claws-onehand.jpg)

---

# Where state machine make sense?

* Life cycle management
* Viewport frame synchronization
* Cache invalidation events
* Transaction (commit/rollback)
* QtQuick "state:" property synchronization
* Sequence of events (drag&drop, scrolling, inertia)

Note that they should not be exposed in the public API. The public API
should be deeply Qt. State machines are not cute.

---
# Where state machine make sense?

Each state change should trigger state changes in related state machines.

![](clock.jpg)

This way the whole system should converge toward correctness.

---

Why build everything out of LEGO?
=================================

By making sure the top level views are just assembled adapters, the design
goal of making sure developers can always write custom views cannot regress.

![LEGO](lego.jpg)

---

# About speed (Q&A)

* All these tiny building blocks and indirection can make things slower: I
  know. If it becomes a bottleneck, then there is known code transformation
  to inline much of it. It makes the code less readable so I don't plan to do
  this until its a necessity
* CPU are good at branch prediction, your code would have been good on an
  in-order CPU but that era is long gone: I know. I made the choice of using
  this type of design to minimize the number of "if" (branch) in the code
  to make it readable and provable. The downside of reducing the number of
  branch in favor of vTables and callmap is that CPU sucks at them.
* The concept of converging toward correctness often cause many iterations
  of nonsense: True. This part of the design exist because I failed to make
  the paging system work. It was too much for me and progress was too slow.
  Once the framework is feature complete, this can be revisited without
  breaking the API.
* X or Y is slow: See the planned optimization list. The current state is
  fast enough to ship and faster than loading everything ahead of time. Future
  iterations can implement various optimizations.
* You have some API or mode but they don't work: Some code has been written for
  feature whose design is final. Yet, if I have no use for them and no tests,
  then the code is most probably incomplete. It's partially implemented where
  it was trivial to do so when I wrote the code around it. Everything regarding
  tables and exotic size hint strategies are prime examples.

---

# Why start from scratch instead of using Qt APIs.

All relevant APIs used to build the QtQuick2.ListView are private and having
applications deeply rely on private APIs was a non-starter. This framework
only go deep enough so private APIs can be avoided, not any deeper.

Not all components that were created, like the Flickable), were rewritten
because of problems in the original, but rather because it wasn't possible to
build this framework without going this deep.

---

Design: Architecture
======

---

# Design: Overview of the layers (1/3)

                  SOME RANDOM QML
    ===============================================
                       VIEWS
    ===============================================
                     ADAPTERS
    ===============================================
                  STATE TRACKERS

---

# Design: Overview of the layers (2/3)


                  SOME RANDOM QML <- For normal users/designers/programmers
    ===============================================
                       VIEWS      <- Convenient abstractions users/designers/programmers
    ===============================================
                     ADAPTERS     <- For developers who need custom views
    ===============================================
                  STATE TRACKERS  <- Fully internal state machines

---

# Design: Overview of the layers (3/3)

 * The views are "prepackaged" groups of adapters to provide a specific behavior
 * The adapters are flexible building blocks to assemble fast views
    * They encapsulate the optimizations

---

Design of the public API
========================

---

Design: Adapters: Overview
======

Core:

* ModelAdapter
* (Abstract)ItemAdapter
* SelectionAdapter
* DecorationAdapter
* ScrollAdapter
* ContextAdapter

Related:

* ViewBase
* Viewport
* TreeTraversalReflector

---

Design: Adapters: ModelAdapter
======

**Cardinality:** zero to many per view

**Owner:** The ViewBase

**Roles:**

* Attach a model to the other components
* Support replacing the models
* Owns the TreeTraversalReflector, the "core" class that tracks all model
  events

**Notes:**

* Multiple models per view are actually very useful
    * Use the selected item "selection delegate" as a model
    * Expose the ListView "categories" as a model
    * Expose the List/Table/Tree header as a model
    * Expose charts legend or axis as models

---

Design: Adapters: (Abstract)ItemAdapter
======

**Cardinality:** one per `ModelAdapter`

**Owner:** The viewport

**Roles:**

* Map the QQuickItem delegate instance to the other components of this framework
* Have many virtual methods the view can implement to handle the life cycle
  events.

---

Design: Adapters: SelectionAdapter
======

**Cardinality:** currently one per model adapter, could be many per model adapter
   if judged to be relevant.

**Owner:** The `ModelAdapter`

**Roles:**

* Attach a QItemSelectionModel and map all events

---

Design: Adapters: DecorationAdapter (needs a new name)
======

**Cardinality:** Created in QML

**Owner:** The QML Item Delegate component

**Roles:**

* Convert the content of Qt::DecorationRole and eventually the
  background/foreground roles to something QML can consume

**Note:**

* This is the oldest part of the whole project and may need to be revisited to
  redefine its scope.

---

Design: Adapters: ContextAdapter
======

**Cardinality:** Many

**Owner:** Undefined on purpose to allow static instances

**Roles:**

* Add a set of Q_PROPERTY to a QQmlContext

**Note:**

* Does a lot of QtQuick magic to create new QMetaType at runtime
* The first generation is fast, but memory hungry, a second generation fixes
  this by using its built-in introspection to create smaller vtables based
  classes in memory.

---

Design: Adapters: ScrollAdapter
======

**Cardinality:** One per view

**Owner:** The view

**Roles:**

* Consume the internal size hint metadata to create a scrollbar
* Support the ListView categories as a scrollbar "table of content"

**Note:**

* The current implementation pre-date many of the components it should use. It
  works well enough for now, but will need an overhaul in future iterations.

---

Design: ~~Adapters:~~ Viewport
======

**Cardinality:** Officially one per `ModelAdapter`

**Owner:** The `ModelAdapter`

**Roles:**

* Track the edge of which part of the model is visible on screen
* Holds the size hint strategies
* Be the public API of the TreeTraversalReflector

---

Design: ~~Adapters:~~ TreeTraversalReflector
======

**Cardinality:** One per `Viewport`

**Owner:** The `Viewport`

**Roles:**

 * Create a projection on a cartesian plan (viewed by the viewport) of the model
 * Track all model events
 * Drive the state trackers using the model events

---

Design: ~~Adapters:~~ ViewBase
======

**Cardinality:** None to Many

**Owner:** Is a top level component. Is defined in QML

**Roles:**

 * Most abstract version of the model view
 * Allows multiple models and wierd cardinalities

**Notes:**

 * Note that some cardinalities are 0-N "because I can" and doing so makes
   keeping component coupling low easier. The `SingleModelViewBase` is an
   higher level API and fits more real-life use cases.

---

Design: ~~Adapters:~~ SingleModelViewBase
======

**Cardinality:** None to Many

**Owner:** Is a top level component. Is defined in QML

**Roles:**

 * Collapse uneeded abstractions and cardinalities into something closer to
   QAbstractItemView.

---

Design of the private API
=========================

---

State trackers: Overview
======

**Per QModelIndex:**

 * Model item (the rowsMoved transaction and the like)
 * Model index metadata (track which part of the model is "tracked")
 * View item (the delegate instance life cycle)
 * Context (the ever changing values of the roles)
 * Geometry (the relative and absolute position in the view)
 * Proximity (track if the nearby elements are fully loaded)

**Per View:**

 * Model (allow to cleanup and replace the QAbstractItemModel)
 * ViewEdge (track the visible and buffer area edges)
 * Viewport (allow to cleanup and replace the QAbstractItemModel)

---

State trackers: StateTracker::Index
======

**Purpose:**

 * Store transient indices during rowsMoved

**States:**

| | |
|-----------|------------------------------------------------------|
|NEW        |  Not part of a tree yet                              |
|NORMAL     |  There is a valid index and a parent node            |
|TRANSITION |  Between rowsAbouttoMove and rowsMoved (and removed) |
|ROOT       |  This is the root element                            |


**Actions:**

N/A

---

State trackers: StateTracker::ModelItem (1/2)
======

**Purpose:**

 * Handle the layered loading states, including pre-loading, edge buffer and
   re-usability pooling
 * React to model changes when they are relevant the viewport
 * Keep a projection of the "visible" part of the model:
    * As a 2D Cartesian plan *and* As a parent/children/sibling tree

---

State trackers: StateTracker::ModelItem (2/2)
======

**States:**

| | |
|-----------|------------------------------------------------------|
| NEW       | During creation, not part of the tree yet            |
| BUFFER    | Not in the viewport, but close                       |
| REMOVED   | Currently in a removal transaction                   |
| REACHABLE | The [grand]parent of visible indexes                 |
| VISIBLE   | The element is visible on screen                     |
| ERROR     | Something went wrong                                 |
| DANGLING  | Being destroyed                                      |
| MOVING    | Currently undergoing a move operation                |

**Actions:**

| | |
|----------|--------------------------------------------|
| POPULATE | Fetch the model content and fill the view  |
| DISABLE  | Disconnect the model tracking              |
| ENABLE   | Connect the pending model                  |
| RESET    | Remove the delegates but keep the trackers |
| FREE     | Free the whole tracking tree               |
| MOVE     | Try to fix the viewport with content       |
| TRIM     | Remove the elements until the edge is free |

---

State trackers: StateTracker::ViewItem (1/2)
======

**Purpose:**

 * Store transient indices during rowsMoved

---

State trackers: StateTracker::ViewItem (2/2)
======

**States:**

| | |
|----------|--------------------------------------------------------|
| POOLING  | Being currently removed from view                      |
| POOLED   | Not currently in use, either new or waiting for re-use |
| BUFFER   | Not currently on screen, pre-loaded for performance    |
| ACTIVE   | Visible                                                |
| FAILED   | Loading the item was attempted, but failed             |
| DANGLING | Pending deletion, invalid pointers                     |
| ERROR    | Something went wrong                                   |


**Actions:**

| | |
|--------------|--------------------------------------------|
| ATTACH       | Activate the element (do not sync it)      |
| ENTER_BUFFER | Sync all roles                             |
| ENTER_VIEW   | NOP (todo)                                 |
| UPDATE       | Reload the roles                           |
| MOVE         | Move to a new position                     |
| LEAVE_BUFFER | Stop keeping track of data changes         |
| DETACH       | Delete                                     |

---

State trackers: StateTracker::Context
======

**Purpose:**

 * Synchronize the QQmlContext (both way) and the different `ContextAdapter`s

**Roles:**

    !cpp
    enum Flags : int {
        UNUSED      = 0x0 << 0, /*!< This property was never used          */
        READ        = 0x1 << 0, /*!< If data() was ever called             */
        HAS_DATA    = 0x1 << 1, /*!< When the QVariant is valid            */
        TRIED_WRITE = 0x1 << 2, /*!< When setData returns false            */
        HAS_WRITTEN = 0x1 << 3, /*!< When setData returns true             */
        HAS_CHANGED = 0x1 << 4, /*!< When the value was queried many times */
        HAS_SUBSET  = 0x1 << 5, /*!< When dataChanged has a role list      */
        HAS_GLOBAL  = 0x1 << 6, /*!< When dataChanged has no role          */
        IS_ROLE     = 0x1 << 7, /*!< When the MetaProperty map to a model role */
    };

**Actions:**

| | |
|----------|--------------------------------------------|


---

State trackers: StateTracker::Geometry (1/2)
======

**Purpose:**

* Make sure the rectangle occupied by this QModelIndex is kept in sync
   * Manage the size
   * Manage the position
   * Manage the decorations around the item
---

State trackers: StateTracker::Geometry (2/2)
======

**States:**

| | |
|-----------|----------------------------------------|
| INIT      | The value has not been computed        |
| SIZE      | The size had been computed             |
| POSITION  | The position has been computed         |
| PENDING   | Has all information, but not assembled |
| VALID     | The geometry has been computed         |


**Actions:**

| | |
|----------|-----------------------------------------|
| MOVE     | When moved                              |
| RESIZE   | When the content size changes           |
| PLACE    | When setting the position               |
| RESET    | The delegate, layout changes, or pooled |
| MODIFY   | When the QModelIndex role changes       |
| DECORATE | When the decoration size changes        |
| VIEW     | When the geometry is accessed           |

---

State trackers: StateTracker::Proximity
======

**Purpose:**

 * Track if the next (on the Cardinal and tree projections) elements are loaded
 * Prevent holes with missing elements from being created

**States:**

| | |
|-----------|--------------------------------------------|
| UNKNOWN   | The information is not availablr           |
| LOADED    | The edges are valid                        |
| MOVED     | It was valid, but some elements moved      |
| UNLOADED  | It is known that some edges are not loaded |

**Actions:**

| | |
|----------|--------------------------------------------|

---

State trackers: ModelChange
======

**Purpose:**

 * Support having no model
 * Support replacing the model
 * Support resetting the model

**States:**

| | |
|-----------|--------------------------------------------------------------|
| NO_MODEL  |The model is not set, there is nothing to do                  |
| PAUSED    |The model is set, but the reflector is not listening          |
| POPULATED |The initial insertion has been done, it is ready for tracking |
| TRACKING  |The model is set and the reflector is listening to changes    |

**Actions:**

| | |
|----------|--------------------------------------------|
| POPULATE | Fetch the model content and fill the view  |
| DISABLE  | Disconnect the model tracking              |
| ENABLE   | Connect the pending model                  |
| RESET    | Remove the delegates but keep the trackers |
| FREE     | Free the whole tracking tree               |
| MOVE     | Try to fix the viewport with content       |
| TRIM     | Remove the elements until the edge is free |

---

State trackers: ViewEdge (WIP)
======

**Purpose:**

* Track the left/right/top/bottom edges of the loaded elements (from the model
  point of view)
    * Listed for changes to the StateTracker::Index events and update the edges
    * Make sure the edges are not corrupted by `rowsMoved` model events

**States:**

TODO (it currently uses imperative logic)

**Actions:**

TODO (it currently uses imperative logic)

**Notes:**

I wont try to make a real state tracker out of this until paging is implemented
(I will come back to this below).

---

State trackers: Viewport (WIP)
======

**Purpose:**

* Track if the scrollbar is needed
* Track when the header and footer widgets are visible
* Track when the model is too small to fill the view

**States:**

| | |
|----------|-------------------------------------------------------|
| UNFILLED | There is less items that the space available          |
| ANCHORED | Some items are out of view, but it's at the beginning |
| SCROLLED | It's scrolled to a random point                       |
| AT_END   | It's at the end of the items                          |
| ERROR    | Something went wrong                                  |

**Actions:**

| | |
|----------|--------------------------------------------|
| INSERTION    | |
| REMOVAL      | |
| MOVE         | |
| RESET_SCROLL | |
| SCROLL       | |

**Notes:**

I wont try to make a real state tracker out of this until paging is implemented
(I will come back to this below).

---


Optimizations strategies
========================

---

Optimizations strategies: Context management
============================================

---

Optimizations strategies: Context management (1/5)
==================================================

* Model provide roles
* Calling setPropery on each delegate instance context doesn't scale
* QtQuick has some internal tricks to optimize this. They are using private
  APIs so I had to come up with my own.
* QtQuick views also set some more properties such as `rowCount` and `index`
    * To make it possible to implement views using the public API, such
      extensions need to be user definable
    * Some properties are static, some read-only and some read/write/notify
* The model give `QVariant`s, QML consume `QJSValue`s

---

Optimizations strategies: Context management (2/5)
==================================================

* Gather all the context extensions
    * The model `roleNames` set is implemented as an extension
* Build a large vTable like what `automoc` does
* Build a new large `QMetaType` using `QMetaObjectBuilder`
* Create a property map to convert roles ID to property ID
* Create an introspection map to gather statistics on what's used
* Set the `contextObject` of the `QQmlContext`
* Keep a cache of the QJsValues using the property ID as key for fast access
    * (Future iteration) use a cache the size of the used property count, not
      the total property count, it wastes memory.
* (Future iteration) Once the used subset of properties used by QML is known

---

Optimizations strategies: Context management (3/5)
==================================================

      .-- [QObject property ID]
      |    .--- [Ext ID map]                             .--[RoleID map]
      |    |   .--- [Usage flags]                        |
      |    |   |                                         |    [ Cache ]
     \/   \/  \/ [MetaObject vTable]                     \/       \/
    [0 ] [0] [0] |----------------|------------------|        |  NULL |
    [1 ] [0] [0] | 1  | <--.      |                  |        |  NULL |
    [2 ] [0] [1] | 2  |    |      |      QObject     |        |  "Hi" |
    [3 ] [0] [0] | 3  | Notify ID |     Internals    |        | NULL  |
    [4 ] [0] [0] | 4  |           |                  |        | NULL  |
    [5 ] [ ] [1] | 5  |           |------------------| |42  | |  true |
    [6 ] [1] [3] | 6  |           |                  | |1337| |"elite"|
    [7 ] [1] [0] | 7  |           | Role properties  | |0   | |  NULL |
    [8 ] [1] [0] | 8  | QObject   |                  | |333 | |  NULL |
    [9 ] [ ] [0] | 9  |           |------------------|        |  NULL |
    [10] [2] [0] | 10 | Internal  | Default context  |        |  NULL |
    [11] [2] [0] | 11 |           | extension        |        |  NULL |
    [12] [ ] [0] | 12 | Property  |------------------|        |  NULL |
    [13] [3] [1] | 13 |           | View defined     |        |  123  |
    [14] [3] [0] | 14 |  vTable   | context extension|        |  NULL |
    [15] [3] [0] | 15 |           |         1        |        |  NULL |
    [16] [ ] [0] | 16 |           |------------------|        |  NULL |
    [17] [4] [0] | 17 |           | View defined     |        |  NULL |
    [18] [4] [0] | 18 |           | context extension|        |  NULL |
    [19] [4] [0] | 19 |           |        2         |        |  NULL |
    [20] [ ] [0] | 20 |           |------------------|        |  NULL |
    [21] [5] [0] | 21 |           |       ...        |        |  NULL |
    [22] [ ] [0] |----------------|------------------|        |  NULL |

    |_______________________ Global __________________________|   /\
                                               [Per QModelIndex]--*

---

Optimizations strategies: Context management (4/5)
==================================================

Read:

* A QQuickItem query a property
* The QObject::qt_metacall get the first `id`
* Check the cache and returns the value if it exists
* It uses the extension map to get the extension ID
* It uses the extension offset to get the relative property ID
* It calls the extension with the relative ID
* Update the cache and go back to step 3

Write:

* Same, but calls a different extension method

Notify:

 * Convert the relative ID to the absolute one
 * Use the metacall using the notify ID map

---

Optimizations strategies: Context management (5/5)
==================================================

Pros:

 * Every map but the roleID->PropId is a direct memory access away, no lookup
 * The flags allows to eventually merge the branch that optimize memory usage
 * The caching prevents most calls to `QAbstractItemModel::data()`

Cons:

* QAbstractItemModel::roleNames needs to be static
* It creates some QMetaType noise in GammaRay
* The context extensions needs to be added before the first delegate is created
* The extensions work using relative integer IDs instead of property names.
    * Maybe we could go meta and use Q_PROPERTY

---

Optimizations strategies: Pools
===============================

---

Optimizations strategies: Pools
===============================

* Updating a delegate is much faster than creating a new one
* The StateTracker::ModelItem can "retire" out of view elements into a pools
    * Right now the code to re-use them has been deleted because even with "cpp"
      ownership, it still caused crashes and I didn't want to lose time fixing
      this.
* There is also a "nearly visible" element buffer like in QtQuick.ListView
* It eventually needs many pools if multi-delegate support is added

---

Optimizations strategies: Size (and position) hints
===================================================

---

Optimizations strategies: Size (and position) hints
===================================================

Constraints (1/2):

* Knowing where a QModelIndex is displayed and its size ahead of time is not
  possible if the size is determined after the delegate instance
  `Component.onCompleted` is emited.
* Knowing the position and size is required to implement a deterministic
  scrollbar.
* Some QModelIndex may result in failed or no delegate instance
* Some QModelIndex may produce different sizes
* When using trees, the position is based on a 2D "table" projection of the
  tree
* The viewport size and ratio can change and affect the content
* Some metadata necessary to compute the size are constant or have known update
  life cycle but are expensive to acquire.
    * Some other are cheaper to recompute than to cache
* Some information necessary to get the size may exist at different time

---

Optimizations strategies: Size (and position) hints
===================================================

Constraints (2/2):

* Using different adapters and view capabilities requires different metadata
  at different time
* Sometime, a QModelIndex "position" in the model has no relation with its
  position in the view
* Some QModelIndex, such as ones used in QTreeView, have widget decorations
  that are not part of the delegate and also have dynamic sizes
* Some axis can have different size constraint (fixed columns vs. varying
  height)
* Some information can change when "nearby" QModelIndex change
* Some model have to be used by multiple views
    * Some other are tied
* The programmer and designer need to be able to implement those hints when
  necessary
    * They need to be deterministic and device agnostic)

---

Optimizations strategies: Size (and position) hints
===================================================

|   Name   |                        Description                             |
| -------- | -------------------------------------------------------------- |
| AOT      | Load everything ahead of time, doesn't scale but very reliable |
| JIT      | Do not try to compute the total size, scrollbars wont work     |
| UNIFORM  | Assume all elements have the same size, scales well when true  |
| PROXY    | Use a QSizeHintProxyModel, require work by all developers      |
| ROLE     | Use one of the QAbstractItemModel role as size                 |
| DELEGATE | Assume the view re-implemented ::sizeHint is correct           |
| STATIC   | Have a C++ function that return a size per column              |

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Ahead of time (AOT)

**Automatic:** yes

**Description:** Load every single delegates to make sure all metadata exists

**Usecase:**

* When there is a scrollbar and no other way to get the metadata

**Pros:**

* It is simple and always works

**Cons:**

* Horrible thing to do
* Doesn't scale past ~40 elements on low end mobile/embedded

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Just In Time (JIT)

**Automatic:** yes (and default strategy)

**Description:** Load each element after the other until the view is filled

**Usecase:**

* When there is no scrollbar

**Pros:**

* Doesn't need to know the size and position before loading
* Can "scroll" to random QModelIndex by flushing the loaded elements

**Cons:**

* Doesn't support a scrollbar
* Doesn't support contentHeight, contentY, etc

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Uniform (semi TODO)

**Automatic:** yes

**Description:** Load one element, get the size and assume everything else
will have the same size.

**Usecase:**

* Normal models
* Large models
* Mostly collapsed TreeViews

**Pros:**

* Every easy to compute the position and size of an element
* The scrollbar/contentHeight/contentY "just work"

**Cons:**

* Require each QModelIndex with non-collapsed children to be "registered" in
  an internal structure (just like `QtWidgets.QTreeView`)
* Doesn't support delgates with different size

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** ROLE (TODO)

**Automatic:** yes

**Description:** Use a defined model role (default: Qt::SizeHintRole) to get
the size

**Usecase:**

* Models which need scrollbars

**Pros:**

* The logic is done is C++

**Cons:**

* It ties the model with the view and that's evil (can be implemented as a
  proxy too, see the last strategy)
* Still need to query everything to get he full picture

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Delegate (TODO)

**Automatic:** no

**Description:** Add a special expression or callback property to the
 delegate and use that after loading the first one.

**Usecase:**

* When no C++ is involved

**Pros:**

* Quick to implement in QML+JavaScript
* Simple given the delegate has by definition access to the necessary font
  metrics

**Cons:**

* It's a hack

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Static (Used in `QtQuick.TableView` as `columnWidthProvider`, currently
 not implemented on my side)

**Automatic:** no

**Description:** Add a function either on the C++ model or just a random JS
 function to get a value and use it.

**Usecase:**

* Be compatible with QtQuick.TableView

**Pros:**

* None, really, it doesn't even support the height

**Cons:**

* It doesn't even support the height
* There is better strategies described above and below

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Proxy variant 1 (implemented by this framework)

**Automatic:** no

**Description:** Create a special proxy model in QML that has access to the
 framework internal APIs so it can generate the size hints and be used
 internally

**Usecase:**

* It's a clean way to emulate the old QtWidgets ::sizeHint() method of the
  delegate

**Pros:**

* It seems rather a good compromise
* Support all view features
* Re-use the context optimizations introduced earlier
* Uses JavaScript expressions and feel very natural in QML

**Cons:**

* Requires access to private APIs
* Proxy models are not the most efficient models
* Either need to upstream my context optimizations or refactor the QtQuick own
  context optimizations to be mutualized between the proxy and the view.

---

Demonstration
=============

---

    !js
    model: KQuickView.SizeHintProxyModel {
        id: proxyModel

        /*invalidationRoles: [
            "object",
            "unreadTextMessageCount",
            "isRecording",
            "hasActiveVideo",
        ]*/

        constants: ({
            fmh: fontMetrics.height,
            fmh2: fontMetrics.height,
        })

        function getRowCount(obj) {
            var activeCM = obj ? obj.activeContactMethod : null

            return 2 + ((obj != null) && (obj.hasActiveCall
                || obj.unreadTextMessageCount > 0
                || (activeCM && activeCM.isRecording)
                || (activeCM && activeCM.hasActiveVideo)
            ) ? 1 : 0)
        }

        widthHint: recentView.width
        heightHint: (proxyModel.getRowCount(object)*2+1)*fmh + 13

        sourceModel: PeersTimelineModel
    }

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Proxy variant 2

**Automatic:** no

**Description:** A proxy model that just sets Qt::SizeHintRole using in C++

**Usecase:**

* It's the "role" strategy but without coupling the "real" model with the view

**Pros:**

* It's the "role" strategy but without coupling the "real" model with the view

**Cons:**

* Same as the "role" strategy


---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Proxy variant 3 (see link below)

**Automatic:** no

**Description:** A proxy model implemented in QML based on the design of the
link below

**Usecase:**

* Same as above

**Pros:**

* No C++ required
* Can use declarative objects to build rules that can then be used from C++
  without round trips in QML

**Cons:**

* It's more complex to use than variant one
* It feels a bit less native than "normal" QML expressions
* If it uses RegEx, then it's error prone, slow and limited to string roles

**Notes:**

* https://github.com/oKcerG/SortFilterProxyModel

---

Optimizations strategies: Size (and position) hints
===================================================

**Name:** Proxy variant 4

**Automatic:** no

**Description:** Create an implementation that supports variant 1, 2 and 3 and
 let the developer decides what fit their use case best

**Usecase:**

* Same as above

**Pros:**

* More choice
* Maybe a bit more future proof than putting all eggs in the same basket

**Cons:**

* It's more complex to implement
* Either need to upstream my context optimizations or refactor the QtQuick own
  context optimizations to be mutualized between the proxy and the view.

**Notes:**

* https://github.com/oKcerG/SortFilterProxyModel

---

Optimizations strategies: Other
===============================

---

Optimizations strategies: Other
===============================

Here's a slighly outdated version of the planned/possible optimizations. As
mentionned in the "status" section below, there is a lot of half-implemented
work in progress optimizations that are not planned for the first optimization
iteration but have some hooks and code paths already in place with
`Q_ASSERT(false); //TODO` and `#if 0`.

* [X] Fully decouple the QModelIndex traversal code vs. the GUI elements
* [X] Do not create QQuickItem for invisible elements
* [ ] "really" delete discarded elements without using QtQuick2 garbage collector
* [ ] Have a loaded elements buffer around the visible ones
* [ ] A a recyclable cache of discarded elements
    * [ ] Allow global recycling
    * [ ] Allow recycling per tree depth level only
    * [ ] Support different delegates per depth level without a
         `QtQuick2.Loader` with their own cache pool.

---

Optimizations strategies: Other
===============================

* [X] Lazy load the internal tree representation
    * [X] Stop parsing after the elements stop being visible
    * [ ] Only load the depth chain, not its content (unless visible)
    * [ ] Free memory of invisible QPersistentModelIndex tracker
    * [ ] Load many TreeTraversalItems at once to mitigate the overhead
        * [ ] Have a state machine for TreeTraversalItemsRange to mutualize range
            operations and move the code away from
            TreeTraversalItems::m_fStateMachine
* [X] Have an optimized QML context manager (requires Qt private APIs)
    * [X] Do not refresh the widget when unused roles are dataChanged
    * [X] Only set the used roles in the context instead of all of them
    * [X] Do not call QAbstractItemModel::data when it hasn't changed

---

Optimizations strategies: Other
===============================

* [X] Support the `QSizeHintsProxyModel` (needs a better name) to compute
    * [X] Have a default UniformRowHeight mode to bypass all this
        the geometry without actually creating QQuickItem just to know it...
    * [ ] Use the sizehint information to be able to load from a random QPoint
            without loading from a model edge.
    * [ ] Allow to discard
* [ ] Support TreeView collapsed elements without loading the children.
* [ ] Optimize the flickable to "disable itself" when `interactive` is false
* [ ] Cache the Cartesian navigation results and use the state machine to
      discard invalid ones.
* [ ] Batch detaching items instead of looping to prevent the feedback loop from
      being ran for nothing and "your neighbor just changed" being executed for
      items about to be removed.
* [ ] Resume using pages for the geometry state tracker
    * [ ] Implement the geometry relative to the page
    * [ ] Move whole pages at once
    * [ ] Dismiss whole pages at once
    * [ ] Balance the size pages using runtime introspection

---

Optimizations strategies: Other
===============================

* [ ] Batch slotRowsInserted to share the linked list insertion and siblings
      "move yourself, you have a new neighbor" events
* [ ] Support conditional delegates based on a QQmlScriptString instead of
      using conditional QtQuick2.Loader to keep pools of reusable delegates
* [ ] Cache the Cartesian navigation results and use the state machine to
      discard invalid ones.
* [ ] Allow to load from the bottom up and from the right
* [ ] Avoid loading the children of "tail"/"leaf" items when in raster mode,
      its wasted memory
* [ ] Add an option to compute the totalsize (ei: for the scrollbar) in a
      thread if the model is reentrant
* [ ] Support "load more"
* [ ] Allow the state machine code to be inlined (stop using vtables) if it
      ever reaches the top 10 bottleneck (I doubt it ever will)

---

Optimizations strategies: Other
===============================

* [ ] Support Qt::ItemNeverHasChildren Qt::InitialSortOrderRole Qt::SizeHintRole
* [ ] For non-Cartesian models where QRectF is known, implement tiling.

---



Status
======

---

Status: views
=============

* [ ] Cartesian views
    * [X] ListView with Sections
    * [/] TreeView with collapsing
    * [/] ComboBox
    * [ ] FlameGraph
    * [ ] GridView
    * [ ] Some Excel charts
    * [ ] CalendarView zoom from years single day
    * [ ] Timeline(tree)/Gantt
* [ ] Radial and 2.5D/3D
    * [ ] HierarchyView
    * [ ] RadialListView
    * [ ] FileLight
* Non item based
    * [ ] Some Excel line graphs
* [ ] Flow views
    * [ ] Node graph
    * [ ] Pipeline
    * [ ] UML style class/database diagrams
* Random bad ideas
    * PowerPoint slides using models, because why not
    * Some lost souls with too much time might want to attempt HTML rendering
      on top of the XML model (that would be beyond pointless, but "possible")
---

Limitations
===========

* At first I attempted to implement paging and batching to vastly reduce the
  CPU overhead. It caused the complexity to skyrocket and I deleted it.
    * It is still the "way to go", but only once everything else is **very**
      mature and unit tested. This should not affect the public API
    * I also removed the code that delayed refreshing the view in an idle
      main loop event because it was impossible to debug problems across event
      loop iterations.
* The "x" axis has been ignored. No tables for now.
    * QtQuick.TableView works fine
    * It is designed to be added later, but isn't present right now
    * I am 1 month late and I have no use for it right now
* The Cartesian projection is assumed to be true in wayyyy too many places,
  this needs to be cleaned up or it might make some views impossible to
  implement.
* There is no special APIs to implement more complex animations (yet?) beyond
  standard QML ones.

---

Not implemented yet
===================

* Better support for smart pointers
* Raster delegate
* Any views or features I have no use for
* All the micro optimizations that go beyond the performance threshold I need
  to ship products using these views.
* Delegate recycling has been gutted for now until I have time to test it
  properly
* Multiple delegate per model based on conditions
* All viewport projections except the Cartesian one
    * It needs to be a new adapter since custom views may need to define it

---

Work in progress
================

* Some of the state tracker have more Q_ASSERT and qDebug than lines of "real"
  code
* With the optimizations turned on, there is rendering issues
* There is tests but not full coverage
* Limited real world testing beside the chat app and the integration tests
* A lot of what I said "has code" but has not been tested because I am not there
  yet (and don't need it in the near term).
* The code style when I began was for the project I was working on
    * The code style at the end is closer to Qt
    * I need to run astyle or something to make it uniform again

---

Time table
==========

* Oct-Nov 2017: First version which load every single QQuickItem ahead of time
  and is updated for every changes (milestone 1, released, shipped)
* Sep 2018: Complete the user API (milestone 2, released)
* Oct-Nov 2018: Implement enough optimizations to ship Ring-KDE 3.1 on mobile
  (milestone 3, stabilizing, in-progress)

---

Demo #1: LibRingQt / Ring-KDE / Plasma Mobile dialer
====================================================

A modular phone and secure communication stack.

---

Demo #2: A model based node editor
==================================

* Uses an older QGraphicsView based version.
* It is one of the ancestor of this framework.
* Should be rather easy to port to the newer stack once the Cartesian
  limitations mentioned before get addressed.

---
