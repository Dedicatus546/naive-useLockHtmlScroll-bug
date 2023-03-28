# naive-uselockhtmlscroll-bug

a bug with naive ui utils composable function `uselockhtmlscroll` .

it seems that when parameter passed in `uselockhtmlscroll` is `false`, and when it change to `true`, it can't lock html scroll.

but when parameter is `true`, and when it change to `false`, it can work well.

there are two components which use `uselockhtmlscroll` -- `BodyWrapper(Modal)` and `DrawerBodyWrapper`. it looks like that they work well. i think the reason is that they are wrapped by component `VLazyTeleport` .

```jsx
// Modal/src/Modal.tsx
<VLazyTeleport show={this.show}>
  <NModalBodyWrapper show={this.show}></NModalBodyWrapper>
</VLazyTeleport>
```

when the parameter `show` is `false`, the slot of `VLazyTeleport` will not be rendered.

```javascript
// LazyTeleport
export default defineComponent({
  name: "LazyTeleport",
  // ...
  setup(props) {
    return {
      showTeleport: useFalseUntilTruthy(toRef(props, "show")),
      mergedTo: computed(() => {
        const { to } = props;
        return to ?? "body";
      }),
    };
  },
  render() {
    return this.showTeleport
      ? this.disabled
        ? getSlot("lazy-teleport", this.$slots)
        : h(
            Teleport,
            {
              disabled: this.disabled,
              to: this.mergedTo,
            },
            getSlot("lazy-teleport", this.$slots)
          )
      : // showTeleport depend on show, when it is false, the slot will not be rendered
        null;
  },
});
```

so when the setup function of `NModalBodyWrapper` is be executed, the parameter passed in `uselockhtmlscroll` is always `true` .

```javascript
// Modal/src/BodyWrapper.tsx
watch(toRef(props, "show"), (value) => {
  if (value) displayedRef.value = true;
});
// ignore blockScroll parameter here.
// displayedRef depend on props.show
useLockHtmlScroll(computed(() => props.blockScroll && displayedRef.value));
```

in `uselockhtmlscroll` , it seems that when we init to pass `false` , it will decrease `lockCount` by mistake:

```javascript
// _utils/composable/uselockhtmlscroll
onMounted(() => {
  watchStopHandle = watch(
    lockRef,
    (value) => {
      // 2. next time when value change to true, lockCount is -1 now.
      if (value) {
        // 3. so it will not execute the follow code. lock html failure. 
        if (!lockCount) {
          const scrollbarWidth = window.innerWidth - el.offsetWidth;
          if (scrollbarWidth > 0) {
            originalMarginRight = el.style.marginRight;
            el.style.marginRight = `${scrollbarWidth}px`;
            lockHtmlScrollRightCompensationRef.value = `${scrollbarWidth}px`;
          }
          originalOverflow = el.style.overflow;
          originalOverflowX = el.style.overflowX;
          originalOverflowY = el.style.overflowY;
          el.style.overflow = "hidden";
          el.style.overflowX = "hidden";
          el.style.overflowY = "hidden";
        }
        activated = true;
        lockCount++;
      } else {
        // 1.when init to false, lockCount minus one
        lockCount--;
        if (!lockCount) {
          unlock();
        }
        activated = false;
      }
    },
    {
      immediate: true,
    }
  );
});
```

i think maybe we should measure the old value in watch callback, it looks like:

```javascript
onMounted(() => {
  watchStopHandle = watch(
    lockRef,
    (value, oldValue) => {
      if (value) {
        if (!lockCount) {
          const scrollbarWidth = window.innerWidth - el.offsetWidth;
          if (scrollbarWidth > 0) {
            originalMarginRight = el.style.marginRight;
            el.style.marginRight = `${scrollbarWidth}px`;
            lockHtmlScrollRightCompensationRef.value = `${scrollbarWidth}px`;
          }
          originalOverflow = el.style.overflow;
          originalOverflowX = el.style.overflowX;
          originalOverflowY = el.style.overflowY;
          el.style.overflow = "hidden";
          el.style.overflowX = "hidden";
          el.style.overflowY = "hidden";
        }
        activated = true;
        lockCount++;
      }
      // make sure that increasing lockCount firstly then could decrease it secondly.
      else if (oldValue !== undefined) {
        lockCount--;
        if (!lockCount) {
          unlock();
        }
        activated = false;
      }
    },
    {
      immediate: true,
    }
  );
});
```