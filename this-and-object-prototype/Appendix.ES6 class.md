# Appendix A: ES6 class

## class

지금까지 이야기했던 것들은 모두 잊고, ES6 class에 대해서 이야기 해보도록 하겠습니다.

아래 코드는 앞에서 살펴보았던 Widget / Button 예제를 아래와 같이 class 사용으로 변경한 코드 입니다.

```js
class Widget {
    constructor(width,height) {
        this.width = width || 50;
        this.height = height || 50;
        this.$elem = null;
    }
    render($where){
        if (this.$elem) {
            this.$elem.css( {
                width: this.width + "px",
                height: this.height + "px"
            } ).appendTo( $where );
        }
    }
}

class Button extends Widget {
    constructor(width,height,label) {
        super( width, height );
        this.label = label || "Default";
        this.$elem = $( "<button>" ).text( this.label );
    }
    render($where) {
        super.render( $where );
        this.$elem.click( this.onClick.bind( this ) );
    }
    onClick(evt) {
        console.log( "Button '" + this.label + "' clicked!" );
    }
}
```

class를 사용하는 것은 아래와 같은 몇 가지 장점이 있습니다.  

- **.prototype** 을 사용하는 복잡한 코드 제거
- 상속을 위한 복잡한 코드 제거(Object.create , Object.setPrototypeOf)
- **super**를 이용한 다형성 코드 작성 가능(부모 클래스와 같은 이름의 메서드 사용이 편리해짐)
- **extends**를 사용하여 누구나(비숙련자)쉽게 상속을 구현 할 수 있음

## Something to watch out for

class 사용시, 아래와 같은 몇 가지 주의해야 할 점이 있습니다.

### 1. 부모 클래스의 변경 전파

이전 스터디에서 살펴본 것 처럼 상위 prototype에 변경이 발생하면, prototype chain에 의해 하위의 객체들도 영항을 받습니다.

이러한 현상은 아래와 같이 class를 사용한 코드에서도 동일하게 발생합니다.


```js
class C {
    constructor() {
        this.num = Math.random();
    }
    rand() {
        console.log( "Random: " + this.num );
    }
}

var c1 = new C();
c1.rand(); // "Random: 0.4324299..."

C.prototype.rand = function() {
    console.log( "Random: " + Math.round( this.num * 1000 ));
};

var c2 = new C();
c2.rand(); // "Random: 867"

c1.rand(); // "Random: 432" -- oops!!!
```

위 코드에서, C class의 prototype이 변경되기 전에 생성된 c1도 C class의 prototype의 변경에 영향을 받는 것을 알 수 있습니다.

### 2. class 변수 선언을 위한 prototype 사용

class는 class 변수를 정의 할 수 있는 방법을 제공하지 않습니다. class 변수를 사용해야 하는 경우에는

아래와 같이 prototype을 사용해야 합니다.

```js
class C {
    constructor() {
        // make sure to modify the shared state,
        // not set a shadowed property on the
        // instances!
        C.prototype.count++;

        // here, `this.count` works as expected
        // via delegation
        console.log( "Hello: " + this.count );
    }
}

// add a property for shared state directly to
// prototype object
C.prototype.count = 0;

var c1 = new C();
// Hello: 1

var c2 = new C();
// Hello: 2

c1.count === 2; // true
c1.count === c2.count; // true
```

### 3. 뜻하지 않은 오버라이딩??

class내의 같은 이름을 가진 변수나 함수를 사용하는 경우 예상하지 못한 오류가 발생 할 수 있습니다.

```js
class C {
    constructor(id) {
        // oops, gotcha, we're shadowing `id()` method
        // with a property value on the instance
        this.id = id;
    }
    id() {
        console.log( "Id: " + this.id );
    }
}

var c1 = new C( "c1" );
c1.id(); // TypeError -- `c1.id` is now the string "c1"
```

### 4. super에 대한 오해

이전 스터디에서 prototype chain에 대하여 완벽히(?) 이해 했기 때문에 super는 당연히 상위 prototype을 가르킬 것이라고 생각 할 수 있습니다.

하지만 super는 성능상의 이유로 동적 바인딩이 아닌 정적 바인딩을 사용하기 때문에 우리의 예상을 빗나간 동작을 수행하는 경우가 있습니다.

아래의 예제로 확인해 보도록 하겠습니다.

```js
class P {
    foo() { console.log( "P.foo" ); }
}

class C extends P {
    foo() {
        super.foo();
    }
}

var c1 = new C();
c1.foo(); // "P.foo"

var D = {
    foo: function() { console.log( "D.foo" ); }
};

var E = {
    foo: C.prototype.foo
};

// Link E to D for delegation
Object.setPrototypeOf( E, D );

E.foo(); // "P.foo"
```
위 코드는 E의 prototype을 D로 변경한 상태에서 D의 foo 함수를 호출 하는 코드 입니다.

D의 foo는 C prototype의 foo 함수이기 때문에 super.foo() 함수가 호출 되게 됩니다.

super가 동적 바인딩을 사용한다면 prototype chain에 의해 상위 객체 D의 foo()가 호출 되어야 하지만 아쉽게도 P.foo()가 호출 됩니다.

즉 class C의 super.foo()가 선언되는 시점에 상위 클래스인 P.foo를 정적으로 바인딩 했다고 할 수 있습니다.

우리의 의도대로 동작하는 코드는 아래와 같이 작성해야 합니다.

```js
var D = {
    foo: function() { console.log( "D.foo" ); }
};

// Link E to D for delegation
var E = Object.create( D );

// manually bind `foo`s `[[HomeObject]]` as
// `E`, and `E.[[Prototype]]` is `D`, so thus
// `super()` is `D.foo()`
E.foo = C.prototype.foo.toMethod( E, "foo" );

E.foo(); // "D.foo"
```