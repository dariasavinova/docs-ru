# Реактивность: Продвинутая {#reactivity-api-advanced}

## shallowRef() {#shallowref}

Неглубокая реализация [`ref()`](./reactivity-core#ref).

- **Тип**

  ```ts
  function shallowRef<T>(value: T): ShallowRef<T>

  interface ShallowRef<T> {
    value: T
  }
  ```

- **Подробности**

  В отличие от `ref()`, внутреннее значение неглубокого ref-объекта хранится и раскрывается как есть, и не становится глубоко реактивным. Реактивным является только изменение `.value`.

  `shallowRef()` обычно используется для оптимизации производительности больших структур данных или интеграции с внешними системами управления состоянием.

- **Пример**

  ```js
  const state = shallowRef({ count: 1 })

  // НЕ вызывает изменения
  state.value.count = 2

  // вызывает изменения
  state.value = { count: 2 }
  ```

- **См. также**
  - [Руководство — Уменьшение затрат на реактивность для больших неизменяемых структур](/guide/best-practices/performance#reduce-reactivity-overhead-for-large-immutable-structures)
  - [Руководство — Интеграция с внешними системами состояний](/guide/extras/reactivity-in-depth#integration-with-external-state-systems)

## triggerRef() {#triggerref}

Принудительный запуск эффектов, зависит от [неглубокого ref-объекта](#shallowref). Обычно это используется после глубоких изменений внутреннего значения неглубокого ref-объекта.

- **Тип**

  ```ts
  function triggerRef(ref: ShallowRef): void
  ```

- **Пример**

  ```js
  const shallow = shallowRef({
    greet: 'Привет, мир'
  })

  // Выведет в консоль "Привет, мир" один раз при первом проходе
  watchEffect(() => {
    console.log(shallow.value.greet)
  })

  // Это не вызовет эффекта, поскольку ref-объект неглубокий
  shallow.value.greet = 'Привет, вселенная'

  // Выведет "Привет, вселенная"
  triggerRef(shallow)
  ```

## customRef() {#customref}

Создаёт пользовательский ref-объект с возможностью явно контролировать отслеживание зависимостей и управлять вызовом обновлений.

- **Тип**

  ```ts
  function customRef<T>(factory: CustomRefFactory<T>): Ref<T>

  type CustomRefFactory<T> = (
    track: () => void,
    trigger: () => void
  ) => {
    get: () => T
    set: (value: T) => void
  }
  ```

- **Подробности**

  `customRef()` ожидает функцию-фабрику, которая получает в качестве аргументов функции `track` и `trigger` и должна возвращать объект с методами `get` и `set`.

  В общем случае `track()` следует вызывать внутри `get()`, а `trigger()` - внутри `set()`. Однако вы имеете полный контроль над тем, когда их следует вызывать и следует ли вызывать вообще.

- **Пример**

  Создание debounce ref-объекта, который обновляет значение только по истечении определенного времени после последнего вызова set:

  ```js
  import { customRef } from 'vue'

  export function useDebouncedRef(value, delay = 200) {
    let timeout
    return customRef((track, trigger) => {
      return {
        get() {
          track()
          return value
        },
        set(newValue) {
          clearTimeout(timeout)
          timeout = setTimeout(() => {
            value = newValue
            trigger()
          }, delay)
        }
      }
    })
  }
  ```

  Использование в компоненте:

  ```vue
  <script setup>
  import { useDebouncedRef } from './debouncedRef'
  const text = useDebouncedRef('hello')
  </script>

  <template>
    <input v-model="text" />
  </template>
  ```

  [Попробовать в песочнице](https://play.vuejs.org/#eNplUkFugzAQ/MqKC1SiIekxIpEq9QVV1BMXCguhBdsyaxqE/PcuGAhNfYGd3Z0ZDwzeq1K7zqB39OI205UiaJGMOieiapTUBAOYFt/wUxqRYf6OBVgotGzA30X5Bt59tX4iMilaAsIbwelxMfCvWNfSD+Gw3++fEhFHTpLFuCBsVJ0ScgUQjw6Az+VatY5PiroHo3IeaeHANlkrh7Qg1NBL43cILUmlMAfqVSXK40QUOSYmHAZHZO0KVkIZgu65kTnWp8Qb+4kHEXfjaDXkhd7DTTmuNZ7MsGyzDYbz5CgSgbdppOBFqqT4l0eX1gZDYOm057heOBQYRl81coZVg9LQWGr+IlrchYKAdJp9h0C6KkvUT3A6u8V1dq4ASqRgZnVnWg04/QWYNyYzC2rD5Y3/hkDgz8fY/cOT1ZjqizMZzGY3rDPC12KGZYyd3J26M8ny1KKx7c3X25q1c1wrZN3L9LCMWs/+AmeG6xI=)

  :::warning Используйте осторожно
  Когда мы используем `customRef`, нам следует быть осторожными с возвращаемым значением его геттера, особенно когда при каждом вызове геттера генерируется новый объектный тип данных. Это влияет на отношения между родительским и дочерним компонентами, где такой `customRef` был передан в качестве пропса.

  Рендер-функция родительского компонента может быть запущена из-за изменений в другом реактивном состоянии. Во время повторного рендеринга значение нашего `customRef` переоценивается, возвращая новый объектный тип данных в качестве пропса дочернему компоненту. Этот пропс сравнивается с его предыдущим значением в дочернем компоненте, и поскольку они разные, реактивные зависимости `customRef` запускаются в дочернем компоненте. В это время реактивные зависимости в родительском компоненте не запускаются, потому что геттер customRef не был вызван, и его зависимости не были запущены в результате.

  [Попробовать в песочнице](https://play.vuejs.org/#eNqFVEtP3DAQ/itTS9Vm1ZCt1J6WBZUiDvTQIsoNcwiOkzU4tmU7+9Aq/71jO1mCWuhlN/PyfPP45kAujCk2HSdLsnLMCuPBcd+Zc6pEa7T1cADWOa/bW17nYMPPtvRsDT3UVrcww+DZ0flStybpKSkWQQqPU0IVVUwr58FYvdvDWXgpu6ek1pqSHL0fS0vJw/z0xbN1jUPHY/Ys87Zkzzl4K5qG2zmcnUN2oAqg4T6bQ/wENKNXNk+CxWKsSlmLTSk7XlhedYxnWclYDiK+MkQCoK4wnVtnIiBJuuEJNA2qPof7hzkEoc8DXgg9yzYTBBFgNr4xyY4FbaK2p6qfI0iqFgtgulOe27HyQRy69Dk1JXY9C03JIeQ6wg4xWvJCqFpnlNytOcyC2wzYulQNr0Ao+Mhw0KnTTEttl/CIaIJiMz8NGBHFtYetVrPwa58/IL48Zag4N0ssquNYLYBoW16J0vOkC3VQtVqk7cG9QcHz1kj0QAlgVYkNMFk6d0bJ1pbGYKUkmtD42HmvFfi94WhOEiXwjUnBnlEz9OLTJwy5qCo44D4O7en71SIFjI/F9VuG4jEy/GHQKq5hQrJAKOc4uNVighBF5/cygS0GgOMoK+HQb7+EWvLdMM7weVIJy5kXWi0Rj+xaNRhLKRp1IvB9hxYegA6WJ1xkUe9PcF4e9a+suA3YwYiC5MQ79KlFUzw5rZCZEUtoRWuE5PaXCXmxtuWIkpJSSr39EXXHQcWYNWfP/9A/uV3QUXJjueN2E1ZhtPnSIqGS+er3T77D76Ox1VUn0fsd4y3HfewCxuT2vVMVwp74RbTX8WQI1dy5qx12xI1Fpa1K5AreeEHCCN8q/QXul+LrSC3s4nh93jltkVPDIYt5KJkcIKStCReo4rVQ/CZI6dyEzToCCJu7hAtry/1QH/qXncQB400KJwqPxZHxEyona0xS/E3rt1m9Ld1rZl+uhaxecRtP3EjtgddCyimtXyj9H/Ii3eId7uOGTkyk/wOEbQ9h)

  :::

## shallowReactive() {#shallowreactive}

Неглубокая реализация [`reactive()`](./reactivity-core#reactive).

- **Тип**

  ```ts
  function shallowReactive<T extends object>(target: T): T
  ```

- **Подробности**

  В отличие от `reactive()`, здесь нет глубокого преобразования: для неглубокого реактивного объекта реактивными являются только свойства корневого уровня. Значения свойств хранятся и раскрываются как есть — это также означает, что свойства с ref-значениями **не** будут автоматически разворачиваться.

  :::warning Используйте с осторожностью
  Неглубокие структуры данных следует использовать только для состояния корневого уровня компонента. Избегайте вложения их внутрь глубокого реактивного объекта, поскольку это создаёт дерево с непоследовательным поведением реактивности, которое трудно понять и отладить.
  :::

- **Пример**

  ```js
  const state = shallowReactive({
    foo: 1,
    nested: {
      bar: 2
    }
  })

  // изменение корневых свойств состояния является реактивным
  state.foo++

  // ...но не преобразует вложенные объекты
  isReactive(state.nested) // false

  // НЕ реактивно
  state.nested.bar++
  ```

## shallowReadonly() {#shallowreadonly}

Неглубокая реализация [`readonly()`](./reactivity-core#readonly).

- **Тип**

  ```ts
  function shallowReadonly<T extends object>(target: T): Readonly<T>
  ```

- **Подробности**

  В отличие от `readonly()`, здесь нет глубокого преобразования: только свойства корневого уровня становятся доступными для чтения. Значения свойств хранятся и раскрываются как есть - это также означает, что свойства с ref-значениями **не** будут автоматически разворачиваться.

  :::warning Используйте с осторожностью
  Неглубокие структуры данных следует использовать только для состояния корневого уровня компонента. Избегайте вложения их внутрь глубокого реактивного объекта, поскольку это создаёт дерево с непоследовательным поведением реактивности, которое трудно понять и отладить.
  :::

- **Пример**

  ```js
  const state = shallowReadonly({
    foo: 1,
    nested: {
      bar: 2
    }
  })

  // изменение корневых свойств состояния не удастся
  state.foo++

  // ...но работает с вложенными объектами
  isReadonly(state.nested) // false

  // работает
  state.nested.bar++
  ```

## toRaw() {#toraw}

Возвращает исходный объект из прокси, созданный во Vue.

- **Тип**

  ```ts
  function toRaw<T>(proxy: T): T
  ```

- **Подробности**

  `toRaw()` может возвращать исходный объект из прокси, созданных с помощью [`reactive()`](./reactivity-core#reactive), [`readonly()`](./reactivity-core#readonly), [`shallowReactive()`](#shallowreactive) или [`shallowReadonly()`](#shallowreadonly).

  Применяется в крайнем случае, когда требуется только чтение без доступа/отслеживания или запись без инициирования изменений. Не рекомендуется сохранять постоянную ссылку на оригинальный объект. Используйте с осторожностью.

- **Пример**

  ```js
  const foo = {}
  const reactiveFoo = reactive(foo)

  console.log(toRaw(reactiveFoo) === foo) // true
  ```

## markRaw() {#markraw}

Помечает объект таким образом, что он никогда не будет преобразован в прокси. Возвращает сам объект.

- **Тип**

  ```ts
  function markRaw<T extends object>(value: T): T
  ```

- **Пример**

  ```js
  const foo = markRaw({})
  console.log(isReactive(reactive(foo))) // false

  // также работает при вложении внутрь других реактивных объектов
  const bar = reactive({ foo })
  console.log(isReactive(bar.foo)) // false
  ```

  :::warning Используйте с осторожностью
  `markRaw()` и неглубокое API, такие как `shallowReactive()`, позволяют выборочно отказаться от стандартного глубокого преобразования reactive/readonly и внедрить в граф состояний необработанные, непроксированные объекты. Они могут использоваться по разным причинам:

  - Некоторые значение не должны быть реактивными. Например, сторонний комплексный экземпляр класса или объект компонента Vue.

  - Пропуск преобразования в прокси может улучшить производительность при отрисовке больших списков с иммутабельными (неизменяемыми) данными.

  Пропуск преобразования — продвинутая техника, потому что опциональное отключение доступно только на корневом уровне. Если установить вложенный неотмеченный исходный объект в реактивный объект и получить к нему доступ, то вернётся его проксированная версия. Это может привести к **опасности идентификации**, то есть к выполнению операции, которая основывается на идентификации объекта, но использует как исходную, так и проксированную версию одного и того же объекта:

  ```js
  const foo = markRaw({
    nested: {}
  })

  const bar = reactive({
    // хотя `foo` отмечен как raw, foo.nested не будет таким.
    nested: foo.nested
  })

  console.log(foo.nested === bar.nested) // false
  ```

  С опасностью идентификации сталкиваются редко. Однако, чтобы правильно использовать эти API, избегая опасности идентификации, необходимо хорошо понимать принцип работы системы реактивности.

  :::

## effectScope() {#effectscope}

Создаёт объект области действия эффекта, который может захватывать другие реактивные эффекты (например, вычисляемые свойства и наблюдатели), созданные внутри него, чтобы иметь возможность уничтожить все эти эффекты вместе. Подробные примеры использования этого API приведены в соответствующем [RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0041-reactivity-effect-scope.md).

- **Тип**

  ```ts
  function effectScope(detached?: boolean): EffectScope

  interface EffectScope {
    run<T>(fn: () => T): T | undefined // undefined если область действия неактивна
    stop(): void
  }
  ```

- **Пример**

  ```js
  const scope = effectScope()

  scope.run(() => {
    const doubled = computed(() => counter.value * 2)

    watch(doubled, () => console.log(doubled.value))

    watchEffect(() => console.log('Count: ', doubled.value))
  })

  // для уничтожения всех эффектов в области действия
  scope.stop()
  ```

## getCurrentScope() {#getcurrentscope}

Возвращает текущую активную[ область действия эффекта](#effectscope), если таковая есть.

- **Тип**

  ```ts
  function getCurrentScope(): EffectScope | undefined
  ```

## onScopeDispose() {#onscopedispose}

Регистрация коллбэка для текущей активной [области действия эффекта](#effectscope). Коллбэк будет вызван, когда связанная с ним область действия эффекта будет остановлена.

Этот метод можно использовать как не связанную с компонентами замену `onUnmounted` в переиспользуемых функциях композиций, поскольку функция `setup()` каждого компонента Vue также вызывается в области действия эффекта.

Если эта функция будет вызвана без активной области действия эффекта, будет выведено предупреждение. В версии 3.5+ это предупреждение можно отключить, передав `true` в качестве второго аргумента.

- **Тип**

  ```ts
  function onScopeDispose(fn: () => void, failSilently?: boolean): void
  ```
