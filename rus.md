# Нативная связь данных в JavaScript 

Двусторонняя связь данных (*data-binding*) — это действительно важно, так как 
позволяет реализовать постоянную синхронизацию JS моделей с представлением,
избежать массового дублирования кода отвечающего за его обновления и улучшить 
опыт использования приложения. Мы рассмотрим два метода использования этой 
возможности на чистом JavaScript, без фреймворков — один из них основан на 
революционной технологии (`Object.observe`), а другой — на оригинальной 
концепции (расширения get/set). Забегая вперёд скажу, что второй метод лучше. 
См. tl;dr блок в конце статьи.

## 1: Object.observe && DOM.onChange

*Object.observe()* — [новичок на площадке][3]. Это встроенная возможность
JS, хотя, честно говоря, это *будущая* возможность, так как она предложена для
ES7, но уже (!) доступна в текущей [стабильной версии Chrome][4], допускает 
реактивные изменения объекта JS. Другими словами — обратный вызов срабатывает,
когда объект или его свойства изменяются.

Очевидный способ использования:

    log = console.log
    user = {}
    Object.observe(user, function(changes){    
        changes.forEach(function(change) {
            user.fullName = user.firstName + "" + user.lastName;         
        });
    });
     
    user.firstName = 'Билл';
    user.lastName = 'Клинтон';
    user.fullName // Билл Клинтон


Это уже само по себе довольно круто и допускает полноценное реактивное 
программирование в рамках JS - сохраняя все данные актуальными с помощью 
*push*.

Но давайте попробуем расширить этот способ:

    // <input id="foo">
    user = {};
    div = $("#foo");
    Object.observe(user, function(changes){    
        changes.forEach(function(change) {
            var fullName = (user.firstName || "") + "" + (user.lastName || "");         
            div.text(fullName);
        });
    });
     
    user.firstName = 'Билл';
    user.lastName = 'Клинтон';
     
    div.text() // Билл Клинтон
    

*[Пример на JSFiddle][5]*

Круто! Мы только что получили полноценную связь данных для отражения 
изменений модели в представлении (model-to-view). Давайте избавимся от
дублирования кода с помощью вспомогательной функции.

    // <input id="foo">
    function bindObjPropToDomElem(obj, property, domElem) { 
      Object.observe(obj, function(changes){    
        changes.forEach(function(change) {
          $(domElem).text(obj[property]);        
        });
      });  
    }
     
    user = {};
    bindObjPropToDomElem(user,'name',$("#foo"));
    user.name = 'Вильям'
    $("#foo").text() // Вильям
    

*[Пример на JSFiddle][6]*

Отлично! 

Попробуем другой способ — привяжем DOM-элемент к значению JS. Неплохим решением
будет использование плагина *.change* jQuery ([api.jquery.com][13]):

    // <input id="foo">
    $("#foo").val("");
    function bindDomElemToObjProp(domElem, obj, propertyName) {  
      $(domElem).change(function() {
        obj[propertyName] = $(domElem).val();
        alert("user.name теперь "+user.name);
      });
    }
     
    user = {}
    bindDomElemToObjProp($("#foo"), user, 'name');
    // Введите в поле ввода 'Обама'
    user.name // Обама. 


*[Пример на JSFiddle][7]*

Это уже кое-что. В завершение стоит сказать, что можно комбинировать оба
способа в одну функцию для создания двусторонней связи:

    function bindObjPropToDomElem(obj, property, domElem) { 
      Object.observe(obj, function(changes){    
        changes.forEach(function(change) {
          $(domElem).text(obj[property]);        
        });
      });  
    }
     
    function bindDomElemToObjProp(obj, propertyName, domElem) {  
      $(domElem).change(function() {
        obj[propertyName] = $(domElem).val();
        console.log("obj is", obj);
      });
    }
     
    function bindModelView(obj, property, domElem) {  
      bindObjPropToDomElem(obj, property, domElem)
      bindDomElemToObjProp(obj, propertyName, domElem)
    }
    

Обратите внимание на правильное взаимодействие с DOM в случае двусторонней
связи данных, так как разные DOM элементы (`input`, `div`, `textarea`, `select`) 
по-разному отвечают на разные вызовы (`text`, `val`). Также надо помнить, что 
двусторонняя связь не всегда необходима — элементы, которые отвечают за 
*отображение* редко требуют связи представление-модель, а элементы ввода редко
требуют связь модель-представление.

## 2: Копнём глубже: Изменение get и set

Можно сделать ещё лучше. Одна из проблем решения, приведённого выше,
заключается в том, что испоьзование `.change` не работает для изменений, которые
не вызывают событие `change` jQuery — например, программное изменение DOM.
Например, вот с этим кодом обратный вызов не сработает:

    $("#foo").val('Путин')
    user.name // Всё ещё Обама. Упс.     

Давайте обсудим более радикальный подход — переопределить геттеры и сеттеры.
Это выглядит не так *безопасно*, так как мы уже не просто наблюдаем, а 
будем менять базовую функциональность языка — чтение и запись переменных.
Тем не менее, немного метапрограммирования даёт огромные возможности, что мы
вскоре и увидим.

Так что, если *бы можно было* переопределить чтение и запись значений объектов?
В конце концов, это и есть суть связи данных. Оказывается, что с помощью
`Object.defineProperty()` [можно делать именно это][8].

Раньше использовался [нестандартизованный, устаревший способ][9], но сейчас
есть новый, крутой и, что самое главное, стандартизованный, с помощью 
`Object.defineProperty`:

    user = {}
    nameValue = 'Joe';
    Object.defineProperty(user, 'name', {
      get: function() { return nameValue }, 
      set: function(newValue) { nameValue = newValue; },
      configurable: true // Для того, чтобы можно было переопределить это позднее
    });
     
    user.name // Джо 
    user.name = 'Боб'
    user.name // Боб
    nameValue // Боб
    

Хорошо, теперь `user.name` является алиасом свойства `nameValue`. Но мы можем 
больше, чем переадресовывать переменную — мы можем создать связь между моделью
и представлением. Смотрите:

    //<input id="foo">
    Object.defineProperty(user, 'name', {
      get: function() { return document.getElementById("foo").value }, 
      set: function(newValue) { document.getElementById("foo").value = newValue; },
      configurable: true // Для того, чтобы можно было переопределить это позднее
    });
    

`user.name` теперь привязано к значению поля `#foo`. Это очень краткий пример
*связывания* (биндинга) на уровне языка — определяя (или расширяя) нативный
`get`/`set`. Эта реализация очень примитивная, её можно легко расширить 
или изменить для специфической ситуации — связывая чтение/запись или расширяя
только один из методов, например, для связывания других типов данных.

Как обычно, избегайте повторений кода. Приведём к такой форме:

    function bindModelInput(obj, property, domElem) {
      Object.defineProperty(obj, property, {
        get: function() { return domElem.value; }, 
        set: function(newValue) { domElem.value = newValue; },
        configurable: true
      });
    }
    

Использование:

    user = {};
    inputElem = document.getElementById("foo");
    bindModelInput(user,'name',inputElem);
     
    user.name = "Джо";
    alert("Значение поля теперь "+inputElem.value) // Значение поля теперь 'Джо';
     
    inputElem.value = "Боб";
    alert("Значени user.name теперь "+user.name) // Значение модели теперь 'Боб';
    

*[JSFiddle][10]*

Обратите внимание, что приведённый выше код до сих пор использует
`domElem.value` и так и будет при работе с элементами `<input>`. Это может
быть расширено внутри `bindModelInput` для определения типа элемента и
правильного метода для изменения его значения.

Обсуждение:

*   `DefineProperty` доступно [практически в любом браузере][11].
*   Стоит упомянуть, что в приведённой выше реализации представление (*view*) 
является «последней инстанцией» (по крайней мере, в некоторой перспективе). 
В целом, это ничем не примечательно (так как смысл двустороннего дата-биндинга 
в эквивалентности данных). Тем не менее на принципиальном уровне это может быть 
неудобно, и в некоторых случаях может иметь неприятные последствия — например, 
при удалении DOM-элемента, станет ли наша модель бесполезной? Ответ — нет, не 
станет. Функция `bindModelInput` создаёт замыкание вокруг `domElem`, оставляя
его в памяти и сохраняя его поведение, как будто связь с моделью, даже, если
DOM-элемент был удалён. Таким образом модель продолжает существовать. Обратное
также верно — если модель удалена, то с представлением ничего не случится. 
Понимание того, как это работает изнутри, может оказаться полезным в
крайних случаях при обновлении и модели, и представления.

Используя такой простой подход предоставляет некоторые преимущества перед
использованием фреймворка вроде Knockout или Angular для дата-биндинга,
например:

* Понимание: раз исходный код связи данных написан собственноручно, его 
гораздо проще изменять и поддерживать.
* Проиводительность: не надо связывать «мух с котлетами», а только необходимое,
для того, чтобы не создавать проблем со скоростью работы при большом числе
наблюдаемых объектов.
* Избегание зависимостей: умение связывать данные даёт огромную свободу, если
вы не ограничены уже используемым фреймворком.

Одна из слабостей состоит в том, что так как это не "настоящее" связывание
(отсутствует проверка на "грязные" свойства объекта), в некоторых случаях 
изменение представления не вызовет никаких изменений в модели, например,
не получится синхронизировать два DOM-элемента *с помощью представления*.
То есть, при привязывания двух элементов к модели они обновятся только после
того, как модель будет *тронута* (touch). Это можно сделать с помощью
специальной функции:

    // <input id='input1'>
    // <input id='input2'>
    input1 = document.getElementById('input1')
    input2 = document.getElementById('input2')
    user = {}
    Object.defineProperty(user, 'name', {
      get: function() { return input1.value; }, 
      set: function(newValue) { input1.value = newValue; input2.value = newValue; },
      configurable: true
    });
    input1.onchange = function() { user.name = user.name } // Поля синхронизированы
    

## TL;DR:

Простой способ создать двустороннюю связь данных между моделью и
представлением с помощью нативного Javascript:

    function bindModelInput(obj, property, domElem) {
      Object.defineProperty(obj, property, {
        get: function() { return domElem.value; }, 
        set: function(newValue) { domElem.value = newValue; },
        configurable: true
      });
    }
     
    // <input id="foo">
    user = {}
    bindModelInput(user,'name',document.getElementById('foo')); // Вуаля, получаем двусторонний дата-биндинг
    

Спасибо за чтение. Обсуждение [на reddit][12] или [sella.rafaeli@gmail.com][14].

 [1]: http://www.sellarafaeli.com/
 [2]: http://www.sellarafaeli.com/blog
 [3]: http://www.html5rocks.com/en/tutorials/es7/observe/
 [4]: http://kangax.github.io/compat-table/es7/#Object.observe
 [5]: http://jsfiddle.net/v2bw6658/
 [6]: http://jsfiddle.net/v2bw6658/2/
 [7]: http://jsfiddle.net/v2bw6658/4/

 [8]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

 [9]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__
 [10]: http://jsfiddle.net/v2bw6658/6/
 [11]: http://kangax.github.io/compat-table/es5/#Object.defineProperty
 [12]: http://redd.it/2manfb
 [13]: http://api.jquery.com/change/
 [14]: mailto:sella.rafaeli@gmail.com
