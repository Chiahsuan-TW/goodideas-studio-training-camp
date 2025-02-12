---
title: Dynamic component - :is  用 Render function
lang: zh-TW
description: vue 的 :is 搭配 Render function 處理一些特殊情況
date: 2021-01-12
---

# vue 的 :is 搭配 Render function 處理一些特殊情況

一般在 SFC 裡大概長這樣：

```typescript
<templete>
  <div>
    <div class="flex gap-2">
      <button v-for="tab in ['componentA', 'componentB']" :key="tab" @click="() => handleClick(tab)">
        {{ tab }}
      </button>
    </div>
    <component :is="current" class="tab" />
  </div>
</templete>
<script>
import componentA from './componentA.vue'
import componentB from './componentB.vue'

export default {
  data() {
    return {
      current: 'componentA',
    }
  }
  components: {
    componentA,
    componentB,
  },
  methods: {
    handleClick(componentName) {
      this.current = componentName
    }
  }
}
</script>
```

current 在 vue 2 是放 component key name，
到 vue 3 `<script setup />` 裡可以直接放 `component` (請見官網)

### 想解決的狀況

- 想直接組裝出一個新的小元件，又不想額外多切一個 SFC 出來
- `<component>` 放的 component 有各自不同的 slot，
  全都寫在 templete 進行判斷會錯綜複雜?
- dynamic component 是放在其他 component 的 slot 裡，但必須依照不同 component 放置在不同 namedSlot 裡? (是想多複雜?)

該怎麼做?
目前看起來 SFC 裡面要使用 dynamic component 就得把 component 都切好、額外放成 SFC，
即使他的變動很小也一樣，因為 `:is` 要的是 component key name?

---

仔細看 api documents

`is` 除了 String 另外還可以接受 **Object (component’s options object)** 當作參數
但這個 Object 長什麼樣子卻好像沒怎麼說明?
其實就是 SFC export 出去的那個 Object....
就是 vuejs 最原本的那種寫法，只是好像大家太常處在 vue cli + SFC 的世界裡，
真的會不太清楚到底怎麼寫...?

### 用 render function

在 `this`(vm) 裡面有一個 $createElement，
他通常只出現在 `render` 接收到的參數，
文件說是 `模板字串` 的替代方法。
它 return 的東西是 virtual DOM 的 node(節點)，
文件稱它 `virtual node`，簡稱 `VNode`，
有沒有覺得 VNode 很常出現在 vue api 文件裡，但好像沒什麼地方有說明它到底是什麼?
他被說明在[這裡](https://vuejs.org/v2/guide/render-function.html#The-Virtual-DOM)
請順便往下看 createElement 的用法。

所以 vue 2 裡面可以這樣寫

```typescript
<template>
  <div>
    <div>
      <button
        v-for="tab in ['componentA', 'componentB', 'newComponent']"
        :key="tab"
        @click="() => handleClick(tab)"
      >
        {{ tab }}
      </button>
    </div>
    <component :is="current" class="tab" />
  </div>
</template>
<script>
import componentA from './componentA.vue'
import componentB from './componentB.vue'
export default {
  components: {
    componentA,
    componentB,
  },
  data() {
    const newComponent = {
      render: (createElement) =>
        createElement(
          componentA,
          {
            attrs: {
              id: 'componentC',
              class: 'component_C_class',
              style: 'background-color: lightblue',
            },
            on: {
              click: () => alert('componentC clicked!'),
            },
            scopedSlots: {
              default: (props) => createElement(componentB),
            },
          },
          [
            'This is text node inside the ComponetC',
            // 也可以直接用 this 底下的 $createElement，都是一樣的東西
            this.$createElement('p', {}, 'This is child <p> tag'),
          ],
        ),
      props: {
        test: String,
      },
      created() {
        console.log('created')
      }
    }
    return {
      current: null,
      newComponent: newComponent,
    }
  },
  methods: {
    handleClick(componentName) {
      if (componentName === 'newComponent') {
        console.log(componentName)
        this.current = this.newComponent
      } else {
        this.current = componentName
      }
    },
  },
}
</script>

```

vue 3 一樣。
只是

- 原本 render function 收到的參數 - `createElement`，在文件裡的名稱從 `createElement` 變成 `h`
- 並且 `h` 被拉升到 global，需要另外 Import 近來使用，不會在 render function 裡收到了。
- VNode props 的書寫格式平版化，像是 `id`, `style`, `class`, 不必再包在 `attrs` 裡，click 拉升變成 onClick。

最後附上用 vue 3 `<script setup>` 寫的 demo

<vue-DynamicComponentDemo />

```typescript
<template>
  <div class="w-96 h-40 border-2">
    <div class="flex gap-2">
      <button
        v-for="tab in ['componentA', 'componentB', 'componentC']"
        :key="tab"
        @click="() => handleClick(tab)"
        class="border border-black bg-gray-200 p-2"
      >
        {{ tab }}
      </button>
    </div>
    <component :is="current" class="m-2" />
  </div>
</template>

<script setup>
import { ref, h } from 'vue'
import componentA from './A.vue'
import componentB from './B.vue'

const current = ref(componentA)
const handleClick = tab => {
  console.log(tab)
  switch (tab) {
    case 'componentA':
      current.value = componentA
      break
    case 'componentB':
      current.value = componentB
      break
    case 'componentC':
      current.value = {
        render: () =>
          h(
            componentA,
            {
              id: 'componentC',
              class: 'p-2',
              style: 'background-color: lightblue',
              onClick: () => alert('clicked!'),
            },
            {
              default: props => h('p', 'This is <p>, placed at default slot'),
            }
          ),
        props: {
          test: String,
        },
        created() {
          console.log('created')
        },
      }
      break
    default:
      return
  }
}
</script>
```
