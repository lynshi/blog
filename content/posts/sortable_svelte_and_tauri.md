---
title: "Sortable + Svelte + Tauri: Getting a sortable list right in a cross-platform app"
description: >-
  AI might be better than a rubber duck
date: 2026-01-17T19:04:10-08:00
tags:
  - Tauri
  - Svelte
  - Sortable
---

[Sortable](https://github.com/SortableJS/Sortable) is a very nice JavaScript library for making HTML lists sortable via drag-and-drop,
and it's got a very clear and easy to use API. Unfortunately, I spent a very long time struggling to integrate it into my app, so now
that I've figured it out I might as well collect my findings into the blog post I wish I could have referenced instead.

It was quite hard to get working for two reasons. First, I'm using [Svelte](https://svelte.dev/) (version 5), which is a UI framework that very
much wants to solely manage the HTML. Second, I'm using [Tauri](https://tauri.app/) (version 2.0), a framework for building cross-platform 
applications; this adds complexity beyond a web app.

Before getting started, I searched for resources that would tell
me exactly how to integrate all three things together. Sadly, there were none. The closest I got, and what I started with, was this ["Svelte 5 and SortableJS" blog post](https://dev.to/jdgamble555/svelte-5-and-sortablejs-5h6j). Unfortunately, there were still several issues to iron out thanks to the Tauri variable.
Ultimately, I ended up with an [attachment-based](https://svelte.dev/docs/svelte/@attach) solution for an even more
\~modern\~ Svelte flavor[^0].

---
# Writing the attachment
The attachment itself is very simple. I created a function that takes `Sortable` options and returns an attachment that creates a `Sortable` with the provided options.
```javascript
import Sortable from "sortablejs";
import type { Attachment } from "svelte/attachments";

export const sortableList = (options: Sortable.Options): Attachment => {
  return (element: Element) => {
    const sortable = Sortable.create(element as HTMLElement, options);
    return () => {
      sortable.destroy();
    };
  };
};
```

Similarily to the "Svelte 5 and SortableJS" blog post, I provided a helper function for reordering, though I chose to return `null` if the
order remained the same to avoid unnecessary updates.
```javascript
export function reorder<T>(
  array: T[],
  evt: Sortable.SortableEvent,
): T[] | null {
  const newArray = [...$state.snapshot(array)] as T[];
  const { oldIndex, newIndex } = evt;

  if (oldIndex === undefined || newIndex === undefined) {
    return null;
  }
  if (newIndex === oldIndex) {
    return null;
  }

  const target = newArray[oldIndex];
  const increment = newIndex < oldIndex ? -1 : 1;

  for (let k = oldIndex; k !== newIndex; k += increment) {
    newArray[k] = newArray[k + increment];
  }
  newArray[newIndex] = target;
  return newArray;
}
```

The exact parameters in your `Sortable` options will vary, but the most important bit is to actually reorder your items!
```javascript
import { reorder } from "$lib/sortable.svelte";
const sortableOptions = {
  onUpdate(event: any) {
    const newList = reorder(groups, event);
    if (newList == null) {
      return;
    }

    // Placeholder for updating the order of items.
    updateOrdering(newList);
  }
}
```

Then, we need to create the list and attach `sortableList`. You can generate list items dynamically with `@each`, but I found that
it's crucial to wrap the list in a `#key` block to recreate the list and `Sortable` every time the items are 
reordered. If you do not do this, the item ordering will get out of sync between Svelte and Sortable and your list 
will not display in the correct order.
```svelte
<script>
  import { sortableList } from "$lib/sortable.svelte";
  
  // Your list of things
  const items = [
    {
      id: "item-id",
      text: "item-text",
    }
  ];
</script>

{#key items}
  <ul {@attach sortableList(sortableOptions)}>
    {#each items as item (item.id)}
      <li>{item.text}</li>
    {/each}
  </ul>
{/key}
```

---
# Making drag and drop work
From the Svelte side, everything looks great now. In fact, it will work perfectly if you throw it in a REPL[^1],
which is maddening because locally, in your Tauri app, you will not be able to drag any list items!

This is the first Tauri-introduced issue I encountered. The OS (i.e. Mac) drag-and-drop functionality was preventing the drag-and-drop event  from registering in the webview. To avoid this, I had to add `dragDropEnabled: false` to my `tauri.conf.json`:
```json
{
    "app": {
      "windows": [
        {
          "dragDropEnabled": false
        }
      ],
    }
}
```

Many thanks to [this Reddit post](https://www.reddit.com/r/tauri/comments/1hg85k7/using_vue_3_dragable_components_why_do_i_need_to/) for
providing the hint I needed to find this fix!

---
# AI to the rescue
At this point, I had a working sortable list with one big problem. While I could drag one item to reorder it, I could not consistently
reorder items consecutively. The vast majority of the time, after moving one item I would have to click anywhere else on the page to be
able to move another item. If I did not click elsewhere, clicking a sortable item would not start a drag event, and I would end up
highlighting the text instead.

And yes, "vast majority of the time" means that on occassion I would be able to reorder items consecutively. So at first glance, this looks 
like a race condition along the lines of the list being rendered before the sortable functionality is available. However, that hypothesis
is somewhat invalidated by the fact that when you are unable to reorder items consecutively, you cannot move the second item no matter how
long you wait after moving the first item.

In any event, I did the obvious thing which is to add a bunch of print statements and then try various things to ascertain the order of events.
When that failed, I had a few more hypotheses, but with my limited JavaScript knowledge I didn't know how to test them out.

Enter AI. At work, I'm a pretty big fan of [Claude Code](https://claude.com/product/claude-code) because it doesn't interrupt my stream of 
thought by injecting slop when I hit `Tab` to, literally, insert spaces. However, it's a lot easier to use a tool when you don't have to pay 
for it! Luckily, [Sourcegraph](https://sourcegraph.com/) makes a similar terminal-based agent that has a free, ad-supported mode: [Amp](https://sourcegraph.com/amp). The ads are not bothersome at all and the agent worked quite well. With Amp, I was able to iterate quickly on my 
hypotheses.

The most promising hypothesis I had was that dragging the first item hijacked pointer events in a way that required a "reset" (i.e. by
clicking elsewhere). I tried a few related fixes, such as injecting `pointerup` events after destroying the `Sortable`, to no avail. Along the
way, Amp suggested various code changes, one of which led to me printing `event.originalEvent` from the `event` received by `onUpdate`.

I noticed that the original event is actually a `DragEvent`. I'm no frontend developer, but this seems a little 
different from a pointer-based event. I responded to Amp with this:
> Dispatching pointerup doesn't help. Even though it's logged, I still need to click again to reactivate the sortable. I also noticed that event.originalEvent is a dragEvent with preventDefault: true.

With that, it produced an interesting code block:
```javascript
const sortable = Sortable.create(el, {
  ...options,
  forceFallback: true, // Force pointer events instead of drag events
});
```

From [Sortable docs](https://github.com/SortableJS/Sortable?tab=readme-ov-file#forcefallback-option), `forceFallback` changes how drag-and-drop
works to be compatible with non-HTML5 browsers, even on HTML5 browsers. Setting this option completely fixes the issue!

Unfortunately, what I can't tell you is why this works, and after bashing my head on this for far too long I lack the inclination to do further 
research. Amp reported that "HTML5 Drag and Drop has known quirks, especially in Electron-like environments like Tauri", but when
pressed to cite sources it deflected with "I was making educated guesses based on the symptoms you showed, not citing specific sources. I don't have concrete sources to back up those claims." Perhaps someone else can write an explanation as to why this is the fix 😅

Nonetheless, this is a solution I may not have been able to come up with without AI. There is basically no chance I would've read the Sortable
docs carefully enough to notice the `forceFallback` parameter, especially because it wasn't an option I was looking for.

On the other hand, AI
alone couldn't solve this either. Amp was only able to suggest a solution because I provided a prompt based on a hunch. Amusingly, once this is published, both humans and AI will be able to successfully use Sortable with Svelte and Tauri!

<!--- Footnotes -->
[^0]: To be honest, my AI assistant told me to do that while debugging. It didn't fix the immediate issue, so you probably don't _need_ to use `@attach` instead of `$effect`, but I'm not going back to rewrite it.
[^1]: Yes, that's how I first tried to debug.
