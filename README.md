<!-- TOC -->

- [1. 深入了解组件](#1-深入了解组件)
  - [1.1. 自定义事件](#11-自定义事件)
    - [1.1.1. 事件名](#111-事件名)
    - [1.1.2. 自定义组件的 v-model](#112-自定义组件的-v-model)
    - [1.1.3. 将原生事件绑定到组件](#113-将原生事件绑定到组件)
    - [1.1.4. sync 修饰符](#114-sync-修饰符)
- [2. 可复用性 & 组合](#2-可复用性--组合)
  - [2.1. 自定义指令](#21-自定义指令)
    - [2.1.1. 钩子函数](#211-钩子函数)
    - [2.1.2. 钩子函数参数](#212-钩子函数参数)
    - [2.1.3. 动态指令参数](#213-动态指令参数)
    - [2.1.4. 函数简写](#214-函数简写)

<!-- /TOC -->
# 1. 深入了解组件
## 1.1. 自定义事件
### 1.1.1. 事件名
不同于组件和 prop，事件名不存在任何自动化的大小写转换。触发的事件名需要完全匹配监听这个事件所用的名称。
`this.$emit('myEvent')`, 监听`@my-event` 没有效果 
> v-on 事件监听器在 DOM 模板中会被自动转换为全小写 (因为 HTML 是大小写不敏感的)，所以 v-on:myEvent 将会变成 v-on:myevent——导致 myEvent 不可能被监听到。所以要始终使用 kebab-case 的事件名。

### 1.1.2. 自定义组件的 v-model
一个组件上的 v-model 默认会利用名为 value 的 prop 和名为 input 的事件，但是像单选框、复选框等类型的输入控件可能会将 value 特性用于不同的目的。model 选项可以用来避免这样的冲突：
```javascript
  // MyCheckbox.vue
  export default {
    name: 'MyCheckbox',
    model: {
      prop: 'checked',
      event: 'change'
    },
    props: {
      checked: Boolean,
    },

    methods: {
      checkHandler(event) {
        this.$emit('change', event.target.checked)
      }
    },
  }
```
```html
  <my-checkbox v-model="ifChecked" >
```
[回到顶部](#1-深入了解组件)
### 1.1.3. 将原生事件绑定到组件
在组件的根元素上监听一个原生事件，可以使用 `.native` 修饰符

```html
  <base-input @focus.native="onFocus" />

  <!-- input 是根元素时，可以触发 onFocus
    BaseInput.vue -->
  <template>
    <input >
  </template>

  <!-- input 是非根元素时， onFocus 不会被触发 
    BaseInput.vue -->
  <template>
    <div>
      <input >
    </div>
  </template>
```
为了解决这个问题，Vue 提供了一个 $listeners 属性，它是一个对象，里面包含了作用在这个组件上的所有监听器。
```javascript
  export default {
    name: 'CustomEventInput',
    /* 默认情况下，父作用域的不被认作 props 的特性绑定 (attribute bindings) 
    将会“回退”且作为普通的 HTML 特性应用在子组件的根元素上; inheritAttrs 为
    false 时，关闭这些默认行为

    注意：这个选项不影响 class 和 style 绑定。 */ 
    inheritAttrs: false,
    props: {
      value: String,
      label: String,
    },

    computed: {
      inputListeners() {
        let vm = this;
        return Object.assign({},
          this.$listeners,
          {
            // 此处配合 v-model 工作，否则父组件收到的是input事件 [object InputEvent]
            input(event) {
              vm.$emit('input', event.target.value)
            },
          }
        )
      }
    },
  }
```
```html
  <custom-event-input 
    v-model="msg"
    label="greet：" 
    @focus="onFocus"
  />
```
[回到顶部](#1-深入了解组件)
### 1.1.4. sync 修饰符
在有些情况下，我们可能需要对一个 prop 进行“双向绑定”。不幸的是，真正的双向绑定会带来维护上的问题，因为子组件可以修改父组件，且在父组件和子组件都没有明显的改动来源.
推荐以 update:myPropName 的模式触发事件取而代之,`this.$emit('update:title', newTitle)`,父组件可以监听那个事件并根据需要更新一个本地的数据属性
```html
  <text-document
    v-bind:title="doc.title"
    v-on:update:title="doc.title = $event"
  ></text-document>

  <!-- 为了方便起见，我们为这种模式提供一个缩写，即 .sync 修饰符： -->

  <text-document v-bind:title.sync="doc.title"></text-document>

  <!-- 当我们用一个对象同时设置多个 prop 的时候，也可以将这个 .sync 修饰符和 v-bind 配合使用： -->

  <text-document v-bind.sync="doc"></text-document>
```
> 注意带有 .sync 修饰符的 v-bind 不能和表达式一起使用 (例如 `v-bind:title.sync=”doc.title + ‘!’”` 是无效的)。取而代之的是，你只能提供你想要绑定的属性名，类似 v-model。
> 将 v-bind.sync 用在一个字面量的对象上，例如 `v-bind.sync=”{ title: doc.title }”`，是无法正常工作的，因为在解析一个像这样的复杂表达式的时候，有很多边缘情况需要考虑。

[回到顶部](#1-深入了解组件)
# 2. 可复用性 & 组合  
## 2.1. 自定义指令
### 2.1.1. 钩子函数
一个指令定义对象可以提供如下几个钩子函数 (均为可选)：
+ `bind` ：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置。
+ `inserted` ：被绑定元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)。
+ `update` ：所在组件的 VNode 更新时调用，但是可能发生在其子 VNode 更新之前。指令的值可能发生了改变，也可能没有。但是你可以通过比较更新前后的值来忽略不必要的模板更新 (详细的钩子函数参数见下)。
+ `componentUpdated` ：指令所在组件的 VNode 及其子 VNode 全部更新后调用。
+ `unbind` ：只调用一次，指令与元素解绑时调用。
   
[回到顶部](#1-深入了解组件)
### 2.1.2. 钩子函数参数
指令钩子函数会被传入以下参数：
+ `el` : 指令所绑定的元素
+ `binding` : 一个对象，包含以下属性
  + `name` : 指令名，不包括‘v-’
  + `value` : 指令的绑定值
  + `oldValue` : 指令绑定的前一个值，仅在 update 和 componentUpdated 钩子中可用。无论值是否改变都可用。
  + `expression` : 字符串形式的指令表达式。例如 v-my-directive="1 + 1" 中，表达式为 "1 + 1"。
  + `arg` : 传给指令的参数，可选。例如 v-my-directive:foo 中，参数为 "foo"
  + `modifiers` : 一个包含修饰符的对象。例如：v-my-directive.foo.bar 中，修饰符对象为 { foo: true, bar: true }。
+ `vnode` : Vue 编译生成的虚拟节点
+ `oldVnode` : 上一个虚拟节点，仅在 update 和 componentUpdated 钩子中可用。

### 2.1.3. 动态指令参数
指令的参数可以是动态的。例如，在 `v-mydirective:[argument]="value"` 中，argument 参数可以根据组件实例数据进行更新！这使得自定义指令可以在应用中被灵活使用。
> `v-mydirective:[argument]="value"` 中的 `value` 应该是data中的变量，而不是字符串 'value'

### 2.1.4. 函数简写
在很多时候，你可能想在 `bind` 和 `update` 时触发相同行为，而不关心其它的钩子。比如这样写:
```javascript
  Vue.directive('color-swatch', function (el, binding) {
    el.style.backgroundColor = binding.value
  })
``` 
[回到顶部](#1-深入了解组件)
