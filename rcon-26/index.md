---
title: Automating the Annoying
sub_title: Lessons from Extracting Loaders Directly from the DOM
event: RenderCon Kenya 2026
author: Ajith Kumar P M
theme:
  name: catppuccin-mocha

---
## Lot of skeletons
![](./skeleton-usage-1.jpg)
_An example UI skeleton_
<!--end_slide -->
## Design Principles
---

<!-- pause -->
### 1. The source should be the only thing you need to maintain

Skeletons should be generated from the source code rather than maintained separately.

<!-- pause -->
### 2. Generated skeletons should still be customizable

You should be able to adjust the generated result when it isn't exactly what you
want, without maintaining an entirely separate skeleton.

<!-- pause -->
### 3. The generated skeleton should represent the rendered UI

What matters is not just the source markup, but what the component actually looks
like when rendered.

<!-- pause -->
### 4. The generated result should actually be a skeleton

Meaningful content from the source should not leak into the generated skeleton.

<!-- end_slide -->
<!-- jump_to_middle -->
Stage 1: <span style="color:blue">No generated skeletons</span>
---

<!-- end_slide -->
## Building the skeleton 
---

A perfectly normal user card, shipping to production.
<!-- column_layout: [3, 2] -->

<!-- column: 0 -->
```tsx {3|7-8|9}
function UserCard({ user }) {
  return (
    <article className="user-card">
      <img className="avatar" src={user.avatar} alt={user.name} />

      <div className="content">
        <h3>{user.name}</h3>
        <p>{user.headline}</p>
        <button onClick={connect}>Connect</button>
      </div>
    </article>
  );
}
```
<!-- column: 1 -->
![](./user-card.png)

<!-- reset_layout -->
<!-- end_slide -->
## The skeleton we write instead
---
>  A skeleton made of the real markup would be read out as
> content — a `p` announced as text, a `button` announced as a control you can
> tab to. We can't keep the same semantic elements here.

<!-- column_layout: [3, 2] -->

<!-- column: 0 -->
```tsx

import "./skeleton-styles.css"
function UserCardSkeleton() {
  return (
    <div className="user-card-skeleton">
      <div className="avatar-skeleton" />

      <div className="content-skeleton">
        <div className="name-skeleton" />
        <div className="headline-skeleton" />
        <div className="connect-skeleton" />
      </div>
    </div>
  );
}
```
<!-- column: 1 -->
![](./user-card-skeleton.png)

<!-- reset_layout -->

<!-- end_slide -->
## Multi source of truth
---
Product ships one more action.

```diff
 function UserCard({ user }) {
   return (
     <article className="user-card">
       <img className="avatar" src={user.avatar} alt={user.name} />

       <div className="content">
         <h3>{user.name}</h3>
         <p>{user.headline}</p>
         <button onClick={connect}>Connect</button>
+        <button onClick={message}>Message</button>
       </div>
     </article>
   );
 }
```

<!-- end_slide -->
### And the skeleton has to follow it.
--- 

```diff
 function UserCardSkeleton() {
   return (
     <div className="user-card">
       <div className="avatar" />

       <div className="content">
         <div className="name" />
         <div className="headline" />
         <div className="connect" />
+        <div className="message" />
       </div>
     </div>
   );
 }
```

Two files, one line, and nothing that forces them to move together.

<!-- end_slide -->

<!-- jump_to_middle -->

<!-- alignment: center -->
Stage 2: <span style="color:green">Generating UI during the build</span>
--

<!-- end_slide -->
## Can we generate this from source alone?
---
```tsx {4-12|3,13}
function UserCardPage({ user }) {
  return (
    <Skeleton loading={user.isLoading} name="UserCardSkeleton">
      <article className="user-card">
        <img className="avatar" src={user.avatar} alt={user.name} />

        <div className="content">
          <h3>{user.name}</h3>
          <p>{user.headline}</p>
          <button onClick={connect}>Connect</button>
        </div>
      </article>
    </Skeleton>
  );
}
```

<!-- pause -->

```d2 +render
direction: right

Bundler Plugin: Walk source & find <Skeleton />
Generate: Generate & save skeleton
Runtime: Load generated skeleton while UI loads

Bundler Plugin -> Generate
Generate -> Runtime
```

<!-- end_slide -->

## Option 1: The skeleton
---
The skeleton has to represent the rendered UI, so nothing semantic survives. So we
walk the source, and swap every element we can't keep.

```tsx {1,3}
function Skeleton({ loading, children, name }) {
  if (!loading) return children;
  return magicallyLoadTheGeneratedSkeleton(name);
}
```

Can we get the skeleton from the source alone?

<!-- end_slide -->
## No You Cant!!
---

### The `img` becomes a `div`... at what size?

<!-- pause -->
A replaced element has intrinsic dimensions. A `div` doesn't — it collapses to
nothing until something gives it a size. Only the browser knows the real size,
and only *after* layout, which is the one thing we don't have while loading.

<!-- pause -->
### The `button` becomes a `div`... wearing whose CSS?

<!-- pause -->
`button { }` stops matching. So do `:hover`, `:focus`, `:active`, `::before`.
Inherited styles change too: a `div` takes the parent's `font` and `line-height`,
so the placeholder is a different size than the real control — a layout shift, at
the exact moment the user can least tolerate one.

<!-- pause -->
### And everything else the rendered tree gives away

`.card > button` · `:nth-child` · `[type=…]` · `[data-…]` · `:has()` · `::before`
· `svg`, `canvas`, `video` · `useEffect` state · portals · lazy components ·
media queries · CSS-in-JS that only exists at runtime

<!-- pause -->
> Only the browser which is going to render this component knows this information 

<!-- end_slide -->
<!-- jump_to_middle -->
Stage 3: <span style="color:blue">We need the browser<span>
---

<!-- end_slide -->

## Option 2: Little help from browser
---
We need a browser for the runtime information. So use one — once, at build time:

```d2 +render +width:100%
direction: down
app: |md
  **1 · App in a browser**
  the real `UserCard`, marked as a skeleton
|
info: |md
  **2 · Runtime information**
  boxes, `getComputedStyle`, the DOM tree
|
gen: |md
  **3 · `toSkeleton(runtimeInfo)`**
  elements and text become measured `div`s
|
file: |md
  **4 · React component** → **5 · `UserCardSkeleton.tsx`**
  written to the file system, ready to import
|
app -> info
info -> gen
gen -> file
```

<!-- pause -->
The developer imports `UserCardSkeleton` and swaps it in while loading.

<!-- end_slide -->
## What the developer writes
---

A marker, an import, and a switch. That's the whole change:

```tsx {1-2,5-7,10}
import { markAsSkull } from "skullmaster";
import { UserCardSkeleton } from "~/__generated";

function UserCardPage({ user, isLoading }) {
  if (isLoading) {
    return <UserCardSkeleton />;
  }

  return (
    <article {...markAsSkull("UserCard")} className="user-card">
      <img className="avatar" src={user.avatar} alt={user.name} />

      <div className="content">
        <h3>{user.name}</h3>
        <p>{user.headline}</p>
        <button onClick={connect}>Connect</button>
      </div>
    </article>
  );
}
```

<!-- pause -->
The skeleton is generated once, committed, and never touched by hand again.

<!-- end_slide -->
## First challenge: Chicken or egg
---

The first time you run this, the skeleton doesn't exist yet — so importing it by name
is an error. So we ship one generic component that resolves the name lazily:

```tsx {all,15}
import DefaultBone from "./skeletons/DefaultBone";
import { lazy } from "react";

const registry = {
  UserCard: lazy(() => import("./skeletons/UserCard")),
} as const;

type SkeletonProps = { name: keyof typeof registry | (string & {}) };

export default function Skeleton({ name }: SkeletonProps) {
  const Component = registry[name];

  if (!Component) return <DefaultBone />;

  return <Component />;
}
```

<!-- pause -->
The name is a string, not an import. Generate it later, register it later — and
until then every unknown name falls back to a generic bone instead of crashing.

<!-- end_slide -->
## The fix: one generic `Skeleton`
---

No generated file in sight. The component asks for a skeleton by name, and the
registry does the rest:

```tsx {1,5-6}
import { markAsSkull, Skeleton } from "skullmaster/react";

function UserCardPage({ user, isLoading }) {
  if (isLoading) {
    return <Skeleton name="UserCard" />;
  }

  return (
    <article {...markAsSkull("UserCard")} className="user-card">
      <img className="avatar" src={user.avatar} alt={user.name} />

      <div className="content">
        <h3>{user.name}</h3>
        <p>{user.headline}</p>
        <button onClick={connect}>Connect</button>
      </div>
    </article>
  );
}
```

<!-- pause -->
The same file works before and after the skeleton is generated. Nothing to rename,
nothing to un-import, nothing to remember in the review.

<!-- end_slide -->
## Generating the skeleton
---

The developer's side is done. Now we have to produce `skeletons/UserCard` and add
it to the registry.

### <span style="color:#A1A1AA">Option 1: a headless browser</span>
Load the site in a headless browser, query the marked component with JS, save the
rendered HTML, update the registry file. `boneyard.js` works almost exactly like
this — we'll see at the end why we didn't take it.

<!-- pause -->
### <span style="color:#4ADE80">Option 2: a dev-only script ← we go this way</span>
Inject a development-only script. The developer opens the site, clicks the fully
rendered component, and we read the runtime information off that click.

<!-- pause -->
### Add the provider
In the entry file — `App.tsx`, `main.tsx` or `layout.tsx`:

```tsx {1,3}
import { Skullmaster } from "@skullmaster/react";

<Skullmaster />;
```

<!-- end_slide -->
## Selecting the component
---

![image:width:100%](./comp-select-optimized.gif)

<!-- pause -->
The added `Skullmaster` component highlights the marked component, and a single
click copies the rendered result — as in the video.

<!-- end_slide -->
## The raw HTML
---

The provider hands back the rendered markup of the marked component — the class
names we wrote, and nothing else:

```html
<article data-skullmaster="UserCard" class="user-card">
  <img class="avatar" src="https://example.com/avatar.png" alt="Ada Lovelace">
  <div class="content">
    <h3>Ada Lovelace</h3>
    <p>Mathematician</p>
    <button>Connect</button>
  </div>
</article>
```

<!-- pause -->
```d2 +render
direction: right

click: {
  label: "👆 User clicks card"
}

capture: {
  label: "Capture rendered HTML\n+ runtime information, and process it"
}


decision: {
  label: "🤔 What do we do with it?"
}

save: {
  label: "💾 Download"
}

send: {
  label: "☁️ Send via http"
}



click -> capture
capture -> decision

decision -> save
decision -> send
```
<!-- end_slide -->
<!-- jump_to_middle -->
Stage 4: <span style="color:blue">The Dev Server<span>
---



<!-- end_slide -->
## The server
---

```d2 +render
direction:down
Click: "User clicked on\ncomponent"
Server: "Skullmaster Dev Server\nlocalhost:8008"

Transform: "Transform HTML payload\n→ React component"
Skeleton: "skeletons/UserCardSkeleton.tsx\n\nNew React component"
Registry: "registry.tsx\n\nAdd newly saved component"


Click -> Server: "2. Send captured\nHTML payload"

Server -> Transform: "3. Process payload"
Transform -> Skeleton: "4. Save generated component"
Transform -> Registry: "5. Update registry.tsx"

```




<!-- end_slide -->
## How it works
---

```d2 +render +width:100%
direction: down
src: |md
  **1 · The developer**
  marks the component, adds the provider
|
click: |md
  **2 · A click in the browser**
  the provider highlights it and captures the HTML
|
send: |md
  **3 · `localhost:8008`**
  the payload is posted to `skullmaster serve`
|
done: |md
  **4 · `skeletons/UserCardSkeleton.tsx`**
  the HTML becomes a component, the registry gets it
|
src -> click -> send -> done
```

<!-- pause -->
One click from the developer. Everything after it is the tool.

<!-- end_slide -->
## The transformation
---

Two ways to turn the captured markup into a skeleton.

### <span style="color:#A1A1AA">Option 1: replace everything with `div`s</span>
Every element becomes a box, sized from `getBoundingClientRect()`:

```tsx
toSkeleton(root) {
  return replaceEveryChildWithABox(root, measuredFromTheBrowser);
}
```

A static picture of one screen size — render it wider and it is wrong. `boneyard.js`
runs in a headless browser, so it can check more sizes than we can.

<!-- pause -->
### <span style="color:#4ADE80">Option 2: keep the elements semantically the same</span>

```tsx
<img className="avatar" alt="" aria-hidden="true" />
<h3 className="title" aria-hidden="true" />
<button className="connect" disabled />
```

For some use cases that is a deal breaker, and it is fair. So the result still
behaves like a skeleton: we inject the ARIA attributes and strip the meaningful
information out of the source.

<!-- end_slide -->
## Transformation 1: Strip meaningful text
---

Every text node becomes a placeholder with the same approximate visual footprint.

```tsx
<span className="empty-set__text" data-text-node="true" data-depth="1">
  ██████ █████
</span>
```

The original content is gone. Names, prices, labels, and other meaningful strings never
reach the generated skeleton.

<!-- end_slide -->
## Transformation 2: Strip images
---

Images are replaced with empty generated placeholder graphics
```tsx
   <img
        data-depth="1"
        className="h-48 w-full object-cover"
        data-skull-btlr="0"
        data-skull-btrr="0"
        data-skull-bbrr="0"
        data-skull-bblr="0"
        data-visual-significance="0.20"
        alt=""
        src="data:image/svg+xml,%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.....%3E"
        data-image-skeleton="true"
      />
```


<!-- end_slide -->
## Transformation 3: Strip interactivity
---

Interactive elements remain in the generated tree, but their behavior is removed and
the element is hidden from assistive technology.

```tsx
<a
  data-depth="2"
  className="interactive-link"
  data-skull-btlr="0"
  data-skull-btrr="0"
  data-skull-bbrr="0"
  data-skull-bblr="0"
  data-visual-significance="0.20"
  data-skeleton-interactive="true"
  aria-hidden="true"
  tabIndex={-1}
>
  ████ ████
</a>
```


<!-- end_slide -->
## Transformation 4: Strip form interaction
---

Inputs keep their geometry and type, but become inert skeleton elements.

```tsx
<input
  data-depth="3"
  className="interactive-input"
  type="text"
  data-skull-btlr="0"
  data-skull-btrr="0"
  data-skull-bbrr="0"
  data-skull-bblr="0"
  data-visual-significance="0.46"
  data-skeleton-interactive="true"
  aria-hidden="true"
  tabIndex={-1}
  autoComplete="off"
  data-1p-ignore="true"
  data-lpignore="true"
  data-bwignore="true"
  data-protonpass-ignore="true"
  form="none"
/>
```


<!-- end_slide -->
## Transformation 5: Mark the skeleton root
---

The generated root is explicitly identified as a loading state.

```tsx
<div
  data-depth="0"
  className="card interactive-card empty-set__skeleton"
  data-skullmaster="AnchorLinks"
  role="status"
  aria-live="polite"
  aria-busy="true"
>
  ...
</div>
```


<!-- end_slide -->
## Transformation 7: Remove insignificant elements
---

Layout-only elements that do not contribute meaningful visual structure can be removed.

```tsx
// Source
<div className="wrapper">
  <div className="layout-helper">
    <span>{user.name}</span>
  </div>
</div>

// Generated skeleton
<div data-depth="0">
  <span data-text-node="true" data-depth="2">
    ██████ █████
  </span>
</div>
```

The generator keeps the visual structure that matters and drops elements that render to
nothing or contribute no meaningful visual information.

<!-- end_slide -->
## Automatic generation
---

![image:width:100%](./skullmaster-demo.gif)


<!-- end_slide -->
<!-- jump_to_middle -->
Stage 5: <span style="color:blue">Customization<span>
---

<!-- end_slide -->
## Customizability
---

Of course we could go and edit the generated files. But that breaks the single source
of truth — and the fix dies the next time we generate. So the correction goes into the
source, where it changes no styles in production: only the way the skeleton is generated.

<!-- pause -->
### <span style="color:#4ADE80">Data attributes to the rescue</span>

They ride along in the DOM for free, and the generator is the only thing that reads them.

<!-- pause -->
### `data-depth="-1"` — transparent, but still laid out

Set it on elements that should stay invisible while keeping their place in the layout.
Remove it, or set another depth, on elements that should render a visible bone.

<!-- pause -->
### `data-skip-skull` — leave the whole subtree out

Add it to the root of anything that should not be generated at all. SkullMaster ignores
that element and every one of its descendants.

```html
<div data-skip-skull>
  <!-- this subtree never reaches the skeleton -->
</div>
```

<!-- end_slide -->
## Customizability, type safe
---

The same corrections, without touching markup by hand. The attributes had to be typed
into the DOM; these are typed for you.

<!-- pause -->
### `markAsSkull(name, tweaks?)`

Registers the component for skeleton generation, and takes the tweaks for the top level
element:

```tsx {1}
<section {...markAsSkull("Hero", { isTransparent: true })}>...</section>
```

<!-- pause -->
### `tweakForSkull(tweaks?)`

The same tweaks, applied to a child of an element that is already registered:

```tsx {1}
<fieldset {...tweakForSkull({ hideSubTree: true })}>...</fieldset>
```

<!-- pause -->
Nothing here changes production styling — the tweaks only change what ends up in the
generated skeleton.

<!-- end_slide -->
## Thanks!
---
<!-- column_layout: [1] -->
<!-- column: 0 -->
<!-- jump_to_middle -->
<!-- no_footer -->

# Thanks!

<span style="color:#a8df8e">https://github.com/ajth-in/skullmaster</span>
