# Accessibility Examples from Our Codebase

This document contains real examples of accessibility issues found in our codebase along with solutions for each issue.

## 1. Semantic HTML Issues

### ❌ Problem: Using `<div>` with click handlers instead of proper interactive elements

```javascript
// In formio.module.ts
Array.from(
  document.getElementsByClassName('btn formcomponent drag-copy')
).forEach((element: Element) => {
  element.addEventListener('mousedown', onDrag);
  scrollingEventListeners.push({ element, onDrag });
});
```

**Why it's a problem:** This creates elements that are clickable with a mouse but not accessible via keyboard and don't communicate their interactive nature to screen readers.

### ✅ Solution: Use proper semantic elements

```javascript
// Better approach
Array.from(
  document.getElementsByClassName('btn formcomponent drag-copy')
).forEach((element: Element) => {
  // Make sure the element is a button or has proper role
  if (element.tagName !== 'BUTTON') {
    element.setAttribute('role', 'button');
    element.setAttribute('tabindex', '0');

    // Add keyboard event listener
    element.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        onDrag();
      }
    });
  }
  element.addEventListener('mousedown', onDrag);
  scrollingEventListeners.push({ element, onDrag });
});
```

## 2. Keyboard Navigation & Focus Management

### ❌ Problem: Menus that don't manage focus properly

```javascript
// In FormTable.vue
handleRightClick = (e: MouseEvent, item: SelectedInstanceRow): void => {
  document.querySelector('body')?.click();
  e.preventDefault();
  menu.value.row = item;
  menu.value.positionX =
    e.clientX + menu.value.minWidth > window.innerWidth
      ? e.pageX - menu.value.minWidth
      : e.pageX;
  menu.value.positionY =
    e.clientY + menu.value.minHeight > window.innerHeight
      ? e.pageY - menu.value.minHeight
      : e.pageY;
  menu.value.showMenu = true;
  // No focus management when the menu opens
};
```

**Why it's a problem:** When a menu or dialog opens, focus should move to it so keyboard users can navigate its contents.

### ✅ Solution: Properly manage focus when opening UI elements

```javascript
// Better approach
handleRightClick = (e: MouseEvent, item: SelectedInstanceRow): void => {
  document.querySelector('body')?.click();
  e.preventDefault();
  menu.value.row = item;
  menu.value.positionX = e.clientX + menu.value.minWidth > window.innerWidth ? e.pageX - menu.value.minWidth : e.pageX;
  menu.value.positionY = e.clientY + menu.value.minHeight > window.innerHeight ? e.pageY - menu.value.minHeight : e.pageY;
  menu.value.showMenu = true;

  // Move focus to the menu
  nextTick(() => {
    const menuElement = document.querySelector('.ellipsis-menu');
    if (menuElement) {
      const firstFocusable = menuElement.querySelector('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])');
      if (firstFocusable) {
        (firstFocusable as HTMLElement).focus();
      }
    }
  });
}
```

### ❌ Problem: Using positive `tabindex` values

```javascript
// Implied in our codebase - setting tabindex to positive values
element.setAttribute('tabindex', '2'); // This creates an unnatural tab order
```

**Why it's a problem:** Positive tabindex values create a custom tab order that overrides the natural document flow, making navigation unpredictable and harder to maintain.

### ✅ Solution: Stick to natural document flow or use `tabindex="0"`

```javascript
// Better approach
element.setAttribute('tabindex', '0'); // Makes the element focusable in the natural document order
```

## 3. ARIA and Accessible Names

### ❌ Problem: Adding ARIA attributes via DOM manipulation

```javascript
// In SearchableAssigneeSelector.vue
onMounted(() => {
  // ...
  autocompleteRef.value?.$el.children[0].children[0].setAttribute(
    'id',
    'assignee-dropdown-list'
  );
  autocompleteRef.value?.$el.children[0].children[0].children[2].children[1].children[0].setAttribute(
    'aria-label',
    t('vac_ui.workflow_ui.assignee_selector.inputs.external_user_name')
  );
  autocompleteRef.value?.$el.children[0].children[0].setAttribute(
    'aria-controls',
    'assignee-dropdown-list'
  );
});
```

**Why it's a problem:** This approach is fragile and depends on the exact DOM structure. It also adds accessibility as an afterthought rather than building it into the component design.

### ✅ Solution: Use component props and built-in accessibility features

```vue
<!-- Better approach -->
<v-autocomplete
  ref="autocompleteRef"
  id="assignee-search"
  aria-controls="assignee-dropdown-list"
  :aria-label="
    t('vac_ui.workflow_ui.assignee_selector.inputs.external_user_name')
  "
  :list-props="{
    id: 'assignee-dropdown-list',
    role: 'listbox',
    'aria-label': 'Assignee List',
  }"
  <!--
  ...other
  props
  --
>
>
  <!-- ...content -->
</v-autocomplete>
```

### ❌ Problem: Missing ARIA attributes on interactive components

```vue
<!-- In various components where elements lack proper labels -->
<v-btn
  icon="close"
  size="small"
  class="cursor-pointer"
  @click="clearSelectedValue()"
>
</v-btn>
```

**Why it's a problem:** Screen reader users won't know the purpose of the button.

### ✅ Solution: Always include accessible names for interactive elements

```vue
<!-- Better approach -->
<v-btn
  icon="close"
  size="small"
  class="cursor-pointer"
  :aria-label="$t('common.close')"
  @click="clearSelectedValue()"
>
</v-btn>
```

## 4. Common Components & Patterns

### ❌ Problem: Inaccessible dialog implementations

```vue
<!-- Partial example from AssignmentsDialog.vue -->
<v-dialog v-model="showDialog" @click:outside="handleClickOutside">
  <!-- Content without proper ARIA roles or focus management -->
</v-dialog>
```

**Why it's a problem:** Dialogs need proper ARIA roles, focus management, and keyboard interactions to be accessible.

### ✅ Solution: Implement dialogs with proper accessibility features

```vue
<!-- Better approach -->
<v-dialog
  v-model="showDialog"
  @click:outside="handleClickOutside"
  role="dialog"
  aria-modal="true"
  :aria-labelledby="dialogTitleId"
  @after-enter="focusFirstElement"
  @before-leave="returnFocusToTrigger"
>
  <v-card>
    <v-card-title :id="dialogTitleId">{{ dialogTitle }}</v-card-title>
    <!-- Content -->
    <v-card-actions>
      <v-btn @click="handleClose" ref="closeButton">Close</v-btn>
    </v-card-actions>
  </v-card>
</v-dialog>
```

```javascript
// In component script
const previousFocus = ref<HTMLElement | null>(null);
const closeButton = ref<HTMLElement | null>(null);

const focusFirstElement = () => {
  previousFocus.value = document.activeElement as HTMLElement;
  closeButton.value?.focus();
};

const returnFocusToTrigger = () => {
  if (previousFocus.value) {
    previousFocus.value.focus();
  }
};
```

### ❌ Problem: Color contrast issues

```vue
<!-- In AllPages.vue -->
<v-card
  v-else-if="isFileHovered"
  class="d-inline-flex justify-center align-center justify-space-around px-4 py-2 bg-grey-darken-3 hover-small rounded-lg"
  @mousemove="setHovering"
  id="theCard"
  v-click-outside="clickOutside"
>
  <!-- Text that might have contrast issues against this background -->
  <span class="mr-2">{{ t('viewer.page') }}</span>
  <span>{{ fileStore.currentPage.index + 1 }}</span>
  <span class="mx-2">/</span>
  <span>{{ pageList.length }}</span>
</v-card>
```

**Why it's a problem:** Text may not have sufficient contrast against the background, making it hard to read for users with low vision.

### ✅ Solution: Ensure sufficient color contrast

```vue
<!-- Better approach -->
<v-card
  v-else-if="isFileHovered"
  class="d-inline-flex justify-center align-center justify-space-around px-4 py-2 bg-grey-darken-3 hover-small rounded-lg high-contrast-text"
  @mousemove="setHovering"
  id="theCard"
  v-click-outside="clickOutside"
>
  <!-- Text with guaranteed contrast -->
  <span class="mr-2">{{ t('viewer.page') }}</span>
  <span>{{ fileStore.currentPage.index + 1 }}</span>
  <span class="mx-2">/</span>
  <span>{{ pageList.length }}</span>
</v-card>
```

```css
/* Add to your CSS */
.high-contrast-text {
  color: white !important; /* Ensures light text on dark background */
  font-weight: 500; /* Slightly bolder text for better readability */
}
```
