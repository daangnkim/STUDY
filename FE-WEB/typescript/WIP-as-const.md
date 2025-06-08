type assertion과 const assertion은 다르다.

type assertion은 컴파일러에게 타입을 알려주는 것이고, const assertion은 객체가 상수임을 알려주는 것이다. 그래서 문자열/숫자가 리터럴 타입 및 readonly 타입으로 추론된다.

객체는 불변이다. 그러므로 프로퍼티의 값이 바뀔 수 있기 때문에 기본적으로 string 타입으로 추론된다. 여기에 만약 const assertion을 한다면, 리터럴 타입으로 추론된다.

매개변수의 타입을 readonly를 적용하면, const assertion한 변수도, 안한 변수도 모두 받을 수 있다. 하지만 readonly를 적용하지 않으면, const assertion한 변수는 받지 못한다.

[The power of const assertions | TkDodo's blog](https://tkdodo.eu/blog/the-power-of-const-assertions)