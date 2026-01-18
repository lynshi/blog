---
title: "Sortable + Svelte + Tauri: Getting a sortable list right in a cross-platform app"
description: >-
  AI might be better than a rubber duck
date: 2026-01-17T19:04:10-08:00
draft: true
tags:
  - Tauri
  - Svelte
  - Sortable
---

[Sortable](https://github.com/SortableJS/Sortable) is a very nice JavaScript library for making HTML lists sortable via drag-and-drop,
and it's got a very clear and easy to use API. Unfortunately, I spent a very long time struggling to integrate it into my app, so now
that I've figured it out I might as well collect my findings into the blog post I wish I could have referenced instead.

It was quite hard to get working for two reasons. First, I'm using [Svelte](https://svelte.dev/), which is a UI framework that very
much wants to solely manage the HTML. Second, I'm using [Tauri](https://tauri.app/), a framework for building cross-platform applications;
this adds complexity beyond what you would expect in a web app. Before getting started, I searched for resources that would tell me
exactly how to integrate all three things together. Sadly, there were none.

The closest I got, and what I started with, was this [`Svelte 5 and SortableJS` blog post](https://dev.to/jdgamble555/svelte-5-and-sortablejs-5h6j). Unfortunately, there were still several issues to iron out thanks to the Tauri variable.
Ultimately, I ended up with an [attachment-based](https://svelte.dev/docs/svelte/@attach) solution for an even more ~modern~ flavor (to be honest, my AI assistant told me to do that while trying to help me debug - it didn't fix the issue, so you probably don't _need_ to use `@attach`, but I'm not going back to rewrite it).

---
# Writing the attachment
The attachment itself is very simple. I created a function that takes `Sortable` options and returns an attachment that creates a `Sortable`.
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

Similarily to the `Svelte 5 and SortableJS` blog post, I provided a helper function for reordering, though I chose to return `null` if the
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
```svelte
<script>
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
</script>
```

Then, we need to create the list and attach `sortableList`. You can generate list items dynamically with `@each`, but I found that
it's crucial to wrap the list in a `#key` block to recreate the list every time the items are reordered. If you do not do this, the
item ordering will get out of sync between Svelte and Sortable, and your list will not display the correct order.
```svelte
<script>
  import { sortableList } from "$lib/sortable.svelte";
  
  const items = [
    {
      id: "item-id",
      text: "item-text",
    }
  ]; // Your list of things
</script>

{#key items}
  <ul {@attach sortableList(sortableOptions)}>
    {#each items as item (item.id)}
      <li>{item.text}</li>
    {/each}
  </ul>
{/key}
```
