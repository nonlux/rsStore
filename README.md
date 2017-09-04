# rsStore
Цель проекта: Оставить простоту работы с состоянием как redux.  
Но  при этом:  
 -  исключить "шаблоный код" редюсеров.  
 -  убрать размазывание бизнес-логики по файлам ( reselect  селекторы, саги)
 -  поддержка вычисляемых значений.
 
 
## 1. Определяем карту сотояния
### Простые значения
  ```
  const stateMap = {
     score: 0,
     clickCount: 0,       
  }
  
  const store = createStreamedStore(stateMap);
 ```

Простые значения в  store  преобразуются в rxjs/Subject. 
Т.е  на их изменения можно осуществить подписку.
Но тригер на изменения будет сробатывать только при реальном изменении значения. 

```
 store.score.subscribe(console.log) 
 // log -> 0
 store.score.next(1) 
 // log -> 1
  store.score.next(1) 
 // ничего не выведет т.к предыдущее значение  было 1
 ```
 ### Вычисляемые значения
 ```
 const stateMap = {
     score: 0,
     isWin({score})  return score.map(value > 100)  
  }
  
 const store = createStreamedStore(stateMap);
 store.score.subscribe( value => console.log('score:', value)) 
 store.isWin.subscribe( value => console.log('isWin:', value)) 
 // log -> score: 0
 // log -> isWin: false
 
 store.score.next(1) 
 // log -> score: 1
  
   store.score.next(101) 
 // log -> score: 101
 // log -> isWin: true
  ```
  Вычисляемое значение  задаются как функция на входе которой store (объект содержащий потоки простых значений) а выходе новый поток.
  
  ### Actions (Aka redux reducer)
  Actions - работаю так же как и редюсеры в редукс. Но с несколькими отличиями
   - Достаточно вернуть только измененные значения
   - Могут быть:
       * синхронными - возващать новое состояние
       * асинхномыми - возвращить промис, который вернет состояние
       * реактивным потоком новых состояний
   - Вызываются на прямую без dispatch(something)
   
 ```
 actions: (state) => {
    return {
      // синхронный экщен
      increment() {
        const { value } = state;
        return {
          value: value*2+1,
        };
      },
      // асинхронный экщен
      asyncIncrement() {
        const { value } = state;
        return new Promise(resolve => setTimeout(() => resolve({ value: value+ 10}), 1000 ));
      },
      // поток
      streamIncrement() {
        const stream$ = Observable.range(1,20).zip(Observable.interval(1000), (timer)=>{
          const { value } = state;
          return {
            value: value*timer+1
          }
        });
        return stream$;
      }
    };
```
     
 ## 2. Коннект к react компоненту
 ``rsConnect( mapStateToProps, mapActionsToProp)(ReactComponent)``
  
  ``mapStateToProps ``- функция которая преобразует state в react props
  
  ``mapActionsToProps ``  - функция которая  преобразует actions в  react props
  
  Пример:
  ```
  function Foo({value, onClick}) {
   return (<button onClick={onClick}>{value}</button>)
   }
   
   const App = rsConnect(
     ({score}) => ({ value: score}),
     ({increment}) => ({ onClick: increment }),
     )(Foo)
     
  ReactDOM.render(<App store={store} /> , document.getElementById('root');
  ```
  
  ## Полный пример
  
  store.js
  ```
 import { createStreamedStore } from 'rsStore';
   
 const stateMap = {
  value: 0,
  trigger({ value }) {
    return  value.map(val => val > 100)
  },
  actions: (state) => {
    return {
      increment() {
        const { value } = state;
        return {
          value: value*2+1,
        };
      },
    };
  },
};

export default createStreamedStore(stateMap);
```

index.js 
```
import React from 'react';
import ReactDOM from 'react-dom';
import store from './store';

function Foo({win, value, onClick}) {
   return (<div>
   <button onClick={onClick}>{value}</button>
   {win && 'bingo' }
    </div)
   }
   
const App = rsConnect(
     ({score, trigger}) => ({
     value: score,
     win: trigger,
     }),
     ({increment}) => ({ onClick: increment }),
     )(Foo)
     
  ReactDOM.render(<App store={store} /> , document.getElementById('root');
```

     
  
 
 
