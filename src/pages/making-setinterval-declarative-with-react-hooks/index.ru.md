---
title: Создание декларативного setInterval с помощью React хуков
date: '2019-02-04'
spoiler: Как я научился не волноваться и любить рефы.
---

Если вы пользовались [React Хуками](https://ru.reactjs.org/docs/hooks-intro.html) хотя бы нескольких часов, вы, вероятно, столкнулись с интересной проблемой: `setInterval` [не работает](https://stackoverflow.com/questions/53024496/state-not-updating-when-using-react-state-hook-within-setinterval) так, как хотелось бы.

По [словам](https://mobile.twitter.com/ryanflorence/status/1088606583637061634) Райана Флоренс (Ryan Florence):

>Мне многие указывали на то, что setInterval с хуками, это как какое-то бельмо на глазу React’а

Хотя, я думаю, что это точка зрения этих людей. Эта проблема только на первый взгляд является запутанной

Я также пришел к пониманию, что это не недостаток хуков, а несоответствия между [моделью программирования React](/react-as-a-ui-runtime/) и `setInterval`. Хуки более ближе к модели программирования React чем классы, что делает это несоответствие более заметным.

**Есть способ заставить их работать вместе и при этом очень хорошо, хотя этот способ является не совсем интуитивным.**

В этом посте мы рассмотрим, _как_ сделать так, чтобы интервалы и хуки хорошо иполнялись вместе, _почему_ это решение имеет смысл, и какие *новые* возможности это может дать вам.

-----

**Предупреждение: этот пост фокусируется на _патологическом случае_. Даже если API упрощает сотни вариантов использования, обсуждение всегда будет сосредоточено на том, что стало сложнее.**

Если вы новичок в хуках и не понимаете, о чем идет речь, прочитайте [это введение](https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889) и [документацию](https://ru.reactjs.org/docs/hooks-intro.html). Этот пост предполагает, что вы работали с хуками более чем час.

---

## Просто покажи мне код

Без лишних слов покажем счетчик, который увеличивается каждую секунду:

```jsx{6-9}
import React, { useState, useEffect, useRef } from 'react';

function Counter() {
  let [count, setCount] = useState(0);

  useInterval(() => {
    // Ваша логика здесь
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}
```

*([Демо на CodeSandbox](https://codesandbox.io/s/105x531vkq).)*

Этот `useInterval` не является встроенным хуком React; это [пользовательский хук](https://ru.reactjs.org/docs/hooks-custom.html), который написал я:

```jsx
import React, { useState, useEffect, useRef } from 'react';

function useInterval(callback, delay) {
  const savedCallback = useRef();

  // Запоминает последний колбэк.
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  // Настраивает интервал.
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }
    if (delay !== null) {
      let id = setInterval(tick, delay);
      return () => clearInterval(id);
    }
  }, [delay]);
}
```

*([Демо на CodeSandbox](https://codesandbox.io/s/105x531vkq) на случай, если вы пропустили его раньше.)*

**Мой хук `useInterval` устанавливает интервал и очищает его после размонтирования.** Это сочетание `setInterval` и `clearInterval` связано с жизненным циклом компонентов.

Не стесняйтесь копировать и вставлять код в свой проект или поместить его на npm.

**Если вам все равно, как все это работает, вы можете прекратить чтение! Остальная часть поста в блоге предназначена для людей, которые готовы глубоко погрузиться в React Хуки.**

---

## Подожди, что это?! 🤔

Я знаю, о чем ты думаешь:

>Дэн, этот код не имеет никакого смысла. Что случилось с "Простым JavaScript"? Допустим, что React прыгнул на акулу с хуками!

**Я тоже так думал, но передумал, и я собираюсь изменить твое мнение.** Прежде чем объяснить, почему этот код имеет смысл, я хочу показать, что он может сделать.

---

## Почему `useInterval()` является более лучшим API

Напомню, мой хук `useInterval` принимает функцию и задержку:

```jsx
  useInterval(() => {
    // ...
  }, 1000);
```

Это очень похоже на `setInterval`:

```jsx
  setInterval(() => {
    // ...
  }, 1000);
```

**Так почему бы просто не использовать `setInterval` напрямую?**

Поначалу это может быть не очевидным, но разница между “setInterval”, который вы знаете, и моим хуком "use Interval" заключается в том, что **его аргументы являются "динамическими"**.

Я проиллюстрирую этот момент на конкретном примере.

---

Допустим, мы хотим, чтобы задержка интервала была регулируема:

![Счетчик с полем ввода, которое регулирует задержку интервала](./ counter_delay.gif)

Хотя вы не обязательно контролируете задержку с помощью *input*, динамическая настройка может быть полезна — например, для опроса некоторых обновлений AJAX, реже, когда пользователь переключился на другую вкладку.

Как бы вы сделали это используя `setInterval` в классе? У меня получилось вот что:

```jsx{7-26}
class Counter extends React.Component {
  state = {
    count: 0,
    delay: 1000,
  };

  componentDidMount() {
    this.interval = setInterval(this.tick, this.state.delay);
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.delay !== this.state.delay) {
      clearInterval(this.interval);
      this.interval = setInterval(this.tick, this.state.delay);
    }
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  tick = () => {
    this.setState({
      count: this.state.count + 1
    });
  }

  handleDelayChange = (e) => {
    this.setState({ delay: Number(e.target.value) });
  }

  render() {
    return (
      <>
        <h1>{this.state.count}</h1>
        <input value={this.state.delay} onChange={this.handleDelayChange} />
      </>
    );
  }
}
```

*([Демо на CodeSandbox](https://codesandbox.io/s/mz20m600mp).)*

Не так уж и плохо!

Как же выглядит версия с Хуком?

<font size="50">🥁🥁🥁</font>

```jsx{5-8}
function Counter() {
  let [count, setCount] = useState(0);
  let [delay, setDelay] = useState(1000);

  useInterval(() => {
    // Your custom logic here
    setCount(count + 1);
  }, delay);

  function handleDelayChange(e) {
    setDelay(Number(e.target.value));
  }

  return (
    <>
      <h1>{count}</h1>
      <input value={delay} onChange={handleDelayChange} />
    </>
  );
}
```

*([Демо на CodeSandbox](https://codesandbox.io/s/329jy81rlm).)*

Да, *это то, что нужно*.

В отличие от классовой версии, чтобы иметь динамически настроенную задержку, здесь нет никакого сложного промежутка для "обновления" хука `useInterval`:

```jsx{4,9}
  // Constant delay
  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  // Adjustable delay
  useInterval(() => {
    setCount(count + 1);
  }, delay);
```

Когда хук `useInterval` видит другую задержку, он снова настраивает интервал.

** Вместо того, чтобы писать код для *set* и *clear* интервала, я могу *объявить* интервал с определенной задержкой — и наш хук `useInterval` делает это.**

Что, если я хочу временно *приостановить* свой интервал? Я могу тоже самое сделать с состоянием:

```jsx{6}
  const [delay, setDelay] = useState(1000);
  const [isRunning, setIsRunning] = useState(true);

  useInterval(() => {
    setCount(count + 1);
  }, isRunning ? delay : null);
```

*([Демо](https://codesandbox.io/s/l240mp2pm7)!)*

Это то, что заставляет меня волноваться о Хуках и React снова и снова. Мы можем обернуть существующие императивные API и создать декларативные API, более точно выражающие наши намерения. Так же, как и при рендеринге, мы можем **описывать процесс во все моменты времени одновременно** вместо того, чтобы тщательно выдавать команды для управления им.

---

Надеюсь, вы это реализовали на хуке `useInterval()`, который является более приятным API — по крайней мере, когда мы делаем это из компонента.

**Но почему же использование `setInterval()` и `clearInterval()` с хуками раздражает?** Давайте вернемся к нашему примеру со счетчиком и попробуем реализовать его вручную.

---

## Первая Попытка

Я начну с простого примера, который просто отображает начальное состояние:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <h1>{count}</h1>;
}
```

Теперь мне нужен интервал, который увеличивает это состояние каждую секунду. Это [побочный эффект, которому нужен сброс](https://ru.reactjs.org/docs/hooks-effect.html#effects-with-cleanup) поэтому я собираюсь в `useEffect()` вернуть функцию сброса:

```jsx{4-9}
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  });

  return <h1>{count}</h1>;
}
```

*([Демо на CodeSandbox](https://codesandbox.io/s/7wlxk1k87j).)*

Довольно просто?

**Однако, этот код имеет странное поведение.**

React по умолчанию повторно применяет эффекты после каждого рендеринга. Это делается намеренно и помогает избежать [целого класса ошибок](https://ru.reactjs.org/docs/hooks-effect.html#explanation-why-effects-run-on-each-update), которые присутствуют в классовых компонентах React.

Это обычно хорошо, потому что многие API подписок могут с радостью удалить старый и добавить новый прослушиватель в любое время. Однако `setInterval` не является одним из них. Когда мы запускаем `clearInterval` и `setInterval` их синхронизация сдвигается. Если мы повторно рендерим и слишком часто повторно применяем эффекты, интервал никогда не получит шанс выстрелить!

Мы можем увидеть ошибку, если сделаем повторный рендер нашего компонента с *меньшим* интервалом:

```jsx
setInterval(() => {
  // Re-renders and re-applies Counter's effects
  // which in turn causes it to clearInterval()
  // and setInterval() before that interval fires.
  ReactDOM.render(<Counter />, rootElement);
}, 100);
```

*([Демо](https://codesandbox.io/s/9j86r218y4) этого бага.)*

---

## Вторая попытка

Возможно, вы знаете, что `useEffect()` позволяет нам [*отказаться*](https://ru.reactjs.org/docs/hooks-effect.html#tip-optimizing-performance-by-skipping-effects) от повторного применения эффектов. Вы можете указать массив зависимостей в качестве второго аргумента, и React будет повторно запускать эффект только в том случае, если что-то в этом массиве изменится:

```jsx{3}
useEffect(() => {
  document.title = `You clicked ${count} times`;
}, [count]);
```

Когда мы хотим *только* запустить эффект при монтировании и сбросить его при размонтировании, мы можем передать пустой массив зависимостей `[]`.

Однако это распространенный источник ошибок, если вы не очень знакомы с JavaScript замыканиями. Мы собираемся сделать эту ошибку прямо сейчас! (Мы также создадим [линтер правило](https://github.com/facebook/react/pull/14636) чтобы выявить эти ошибки зараннее, но он еще не готов.)

В первой попытке наша проблема была в том, что повторный запуск эффектов привел к слишком ранней очистке таймера. Мы можем попытаться исправить это, никогда не перезапуская их:

```jsx{9}
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

Однако теперь наш счетчик обновляется до 1 и остается в таком состоянии. ([Смотрите ошибку в действии](https://codesandbox.io/s/jj0mk6y683).)

Что случилось?!

**Проблема в том, что `useEffect` захватывает `count` с первого рендеринга.** Он равен `0`. Мы никогда ни применим эффект повторно, поэтому замыкание в `setInterval` всегда ссылается на `count` с первого рендеринга, а `count + 1` всегда является `1`. Упс!

**Я слышу, как ты скрежещешь зубами. Хуки так раздражают, правда?**

[Один из способов](https://codesandbox.io/s/j379jxrzjy) исправить это - заменить `setCount(count + 1)` на "обновляемую" форму  `setCount(c => c + 1)`. Оно всегда может прочитать новое состояние для этой переменной. Но, например, это не поможет вам прочитать свежий пропс.

[[Другой способ исправления](https://code sandbox.io/s/00o9o95jyv) - это [`useReducer()`](https://ru.reactjs.org/docs/hooks-reference.html#usereducer). Этот подход дает вам больше гибкости. Внутри редюсера у вас есть доступ как к текущему состоянию, так и к новым пропсам. Функция `dispatch` сама никогда не меняется, поэтому вы можете заслать данные в нее из любого замыкания. Одним из ограничений `useReducer()` является то, что вы не можете генерировать в нем побочные эффекты. (Однако вы можете вернуть новое состояние, вызвав некоторый эффект.)

**Но почему это становится таким запутанным?**

---

## The Impedance Mismatch

This term is sometimes thrown around, and [Phil Haack](https://haacked.com/archive/2004/06/15/impedance-mismatch.aspx/) explains it like this:

>One might say Databases are from Mars and Objects are from Venus. Databases do not map naturally to object models. It’s a lot like trying to push the north poles of two magnets together.

Our “impedance mismatch” is not between Databases and Objects. It is between the React programming model and the imperative `setInterval` API.

**A React component may be mounted for a while and go through many different states, but its render result describes *all of them at once.***

```jsx
  // Describes every render
  return <h1>{count}</h1>
```

Hooks let us apply the same declarative approach to effects:

```jsx{4}
  // Describes every interval state
  useInterval(() => {
    setCount(count + 1);
  }, isRunning ? delay : null);
```

We don’t *set* the interval, but specify *whether* it is set and with what delay. Our Hook makes it happen. A continuous process is described in discrete terms.

**By contrast, `setInterval` does not describe a process in time — once you set the interval, you can’t change anything about it except clearing it.**

That’s the mismatch between the React model and the `setInterval` API.

---

Props and state of React components can change. React will re-render them and “forget” everything about the previous render result. It becomes irrelevant.

The `useEffect()` Hook “forgets” the previous render too. It cleans up the last effect and sets up the next effect. The next effect closes over fresh props and state. This is why our [first attempt](https://codesandbox.io/s/7wlxk1k87j) worked for simple cases.

**But `setInterval()` does not “forget”.** It will forever reference the old props and state until you replace it — which you can’t do without resetting the time.

Or wait, can you?

---

## Refs to the Rescue!

The problem boils down to this:

* We do `setInterval(callback1, delay)` with `callback1` from first render.
* We have `callback2` from next render that closes over fresh props and state.
* But we can’t replace an already existing interval without resetting the time!

**So what if we didn’t replace the interval at all, and instead introduced a mutable `savedCallback` variable pointing to the *latest* interval callback?**

Now we can see the solution:

* We `setInterval(fn, delay)` where `fn` calls `savedCallback`.
* Set `savedCallback` to `callback1` after the first render.
* Set `savedCallback` to `callback2` after the next render.
* ???
* PROFIT

This mutable `savedCallback` needs to “persist” across the re-renders. So it can’t be a regular variable. We want something more like an instance field.

[As we can learn from the Hooks FAQ,](https://reactjs.org/docs/hooks-faq.html#is-there-something-like-instance-variables) `useRef()` gives us exactly that:

```jsx
  const savedCallback = useRef();
  // { current: null }
```

*(You might be familiar with [DOM refs](https://reactjs.org/docs/refs-and-the-dom.html) in React. Hooks use the same concept for holding any mutable values. A ref is like a “box” into which you can put anything.)*

`useRef()` returns a plain object with a mutable `current` property that’s shared between renders. We can save the *latest* interval callback into it:

```jsx{8}
  function callback() {
    // Can read fresh props, state, etc.
    setCount(count + 1);
  }

  // After every render, save the latest callback into our ref.
  useEffect(() => {
    savedCallback.current = callback;
  });
```

And then we can read and call it from inside our interval:

```jsx{3,8}
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);
```

Thanks to `[]`, our effect never re-executes, and the interval doesn’t get reset. However, thanks to the `savedCallback` ref, we can always read the callback that we set after the last render, and call it from the interval tick.

Here’s a complete working solution:

```js{10,15}
function Counter() {
  const [count, setCount] = useState(0);
  const savedCallback = useRef();

  function callback() {
    setCount(count + 1);
  }

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/3499qqr565).)*

---

## Extracting a Hook

Admittedly, the above code can be disorienting. It’s mind-bending to mix the opposite paradigms. There’s also a potential to make a mess with mutable refs.

**I think Hooks provide lower-level primitives than classes — but their beauty is that they enable us to compose and create better declarative abstractions.**

Ideally, I just want to write this:

```js{4-6}
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}
```

I’ll copy and paste the body of my ref mechanism into a custom Hook:

```jsx
function useInterval(callback) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);
}
```

Currently, the `1000` delay is hardcoded. I want to make it an argument:

```jsx
function useInterval(callback, delay) {
```

I will use it when I set up the interval:

```jsx
    let id = setInterval(tick, delay);
```

 Now that the `delay` can change between renders, I need to declare it in the dependencies of my interval effect:

```js{8}
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]);
```

Wait, didn’t we want to avoid resetting the interval effect, and specifically passed `[]` to avoid it? Not quite. We only wanted to avoid resetting it when the *callback* changes. But when the `delay` changes, we *want* to restart the timer!

Let’s check if our code works:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}

function useInterval(callback, delay) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

*(Try it on [CodeSandbox](https://codesandbox.io/s/xvyl15375w).)*

It does! We can now `useInterval()` in any component and not think too much about its implementation details.

## Bonus: Pausing the Interval

Say we want to be able to pause our interval by passing `null` as the `delay`:

```jsx{6}
  const [delay, setDelay] = useState(1000);
  const [isRunning, setIsRunning] = useState(true);

  useInterval(() => {
    setCount(count + 1);
  }, isRunning ? delay : null);
```

How do we implement this? The answer is: by not setting up an interval.

```js{6}
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    if (delay !== null) {
      let id = setInterval(tick, delay);
      return () => clearInterval(id);
    }
  }, [delay]);
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/l240mp2pm7).)*

That’s it. This code handles all possible transitions: a change of a delay, pausing, or resuming an interval. The `useEffect()` API asks us to spend more upfront effort to describe the setup and cleanup — but adding new cases is easy.

## Bonus: Fun Demo

This `useInterval()` Hook is really fun to play with. When the side effects are declarative, it’s much easier to orchestrate complex behaviors together.

**For example, we can have a `delay` of one interval be controlled by another:**

![Counter that automatically speeds up](./counter_inception.gif)

```js{10-15}
function Counter() {
  const [delay, setDelay] = useState(1000);
  const [count, setCount] = useState(0);

  // Increment the counter.
  useInterval(() => {
    setCount(count + 1);
  }, delay);

  // Make it faster every second!
  useInterval(() => {
    if (delay > 10) {
      setDelay(delay / 2);
    }
  }, 1000);

  function handleReset() {
    setDelay(1000);
  }

  return (
    <>
      <h1>Counter: {count}</h1>
      <h4>Delay: {delay}</h4>
      <button onClick={handleReset}>
        Reset delay
      </button>
    </>
  );
}
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/znr418qp13)!)*

## Closing Thoughts

Hooks take some getting used to — and *especially* at the boundary of imperative and declarative code. You can create powerful declarative abstractions with them like [React Spring](https://www.react-spring.io/docs/hooks/basics) but they can definitely get on your nerves sometimes.

This is an early time for Hooks, and there are definitely still patterns we need to work out and compare. Don’t rush to adopt Hooks if you’re used to following well-known “best practices”. There’s still a lot to try and discover.

I hope this post helps you understand the common pitfalls related to using APIs like `setInterval()` with Hooks, the patterns that can help you overcome them, and the sweet fruit of creating more expressive declarative APIs on top of them.
