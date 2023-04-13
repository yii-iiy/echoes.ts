# *echoes.ts*

~~~
🎃 make object likes namespace in TypeScript !
~~~

----

## 简述

这是一个高度抽象的依赖关联器，为结构体中的各个属性之间依赖关系的建立提供了一种途径。

### 定义

很简单，只有这些。🙃

~~~ tsx
namespace Echoes
{
    export 
    const echoes =
    <T = {[key: string]: any},> (waves: {[key: string]: (env: T) => any})
    : T =>
        Object.entries(waves).reduce
        ( (envs, [fn, f]) => ({... envs, [fn]: f(envs)}), {} as T ) ;
    
    export 
    const call = 
    <T extends Record<K, (...args: any) => any>, K extends keyof T>
    (obj: T, key: K): { [P in K]: ReturnType<T[P]>; }[K] =>
        echoes<{[P in K]: ReturnType<T[P]>}>(obj)[key] ;
    
} ;
~~~

核心的部分是 `echoes` 。它接受一个第一层属性在形式上都合乎某种规定的结构体，然后将这第一层属性转换为另一种形式，并在此过程中完成它们之间依赖关系的建立。

这是一个宽依赖的转换，因此使用了（万能的） `reduce` （在 Scala 里它叫 `fold` ）。

上述所有函数都是匿名函数 —— 因此可以像值一样传递。也都是 Pure 的，因此可以在任何时候在不论任何（版本足够的） TS 解释器中调用。

当使用改工具时，一些类型错误问题将不能在编译期间被发现，而是会在运行时造成 `ERR` 。

### 使用例

简单例：

~~~ tsx
const ffs =
{
    f1: (env: { [key: string]: Function }) => 
        
        (n: number): number => 1 + n ,
    
    f2: (env: { [key: string]: Function }) => 
        
        (x: number): number => env.f1(x * 2) ,
    
} ;

console.log( Echoes.echoes(ffs).f2(3) ); // 结果为 7
~~~

其中的 `(env: { [key: string]: Function }) =>` 是必要的部分。没有这个变量，就不能描述属性之间的依赖关系。

复杂例：

~~~ tsx
const xx =
{
    x0: (env: { [key: string]: any }) => 
        
        1 ,
    
    f: (env: { [key: string]: any }) => 
        
        (s: string)
        : number => s.length ,
    
    f2: (env: { [key: string]: any }) => 
        
        (s: string, n: number)
        : Promise<number> => Promise.resolve(env.f(s) + n - env.x0) ,
} ;

Echoes.echoes<{f2: ReturnType<typeof xx.f2>}>(xx).f2('aa',3).then(r => console.log(r)); // 结果为 4
Echoes.echoes(xx).f2('aaa',3).then( (r: number) => console.log(r) ); // 结果为 5

Echoes.call(xx,'f2')('a',3).then(r => console.log(r)); // 结果为 3
~~~

其中的 `Echoes.call` 是对 `Echoes.echoes` 指定泛型调用的封装。

在不指定泛型的情况下，（目前的）类型推导无法确定 `Promise` 的泛型，因而 `r` 必须写作 `(r: number)` 。


## 鸣谢

*在本项目的建立与错误排查中，使用了以下工具：*

- *基于 [GPT-4 AI](https://openai.com/research/gpt-4) 的 [Bing Chat](https://bing.com/chat) ：被用于灵感的提供以及对错误的可能原因的排查。*
- *以 Web Browser 为界面的 [TypeScript Playground](https://www.typescriptlang.org//play) ：被用于代码执行、测试、验证、提示，以及其它所有的 IDE 功能。*

*项目名称的灵感来自[《JOJO の奇妙な冒険》](https://moegirl.org/JOJO%E7%9A%84%E5%A5%87%E5%A6%99%E5%86%92%E9%99%A9)中的 [替身 Echoes](https://jojowiki.com/Echoes) 。*

*在这部漫画作品里，替身 Echoes 的 ACT1 和 ACT2 形态具有将字生成为相应的事实的能力。对应在本项目这里，它能将数据化的程序变得能够执行。*

*当然，还有另一层含义。 Echoes 在英语里就是回声的意思，而该项目原本被设计的作用就是：我能对它传入一些互相有依赖关系的函数，但是让依赖关系传入之后再建立。这仿佛就是不同波长和振幅的声波彼此干涉，最终形成了一个包含所有波特征的复杂波一样，而混合完成前后的边界就是这个 Echoes 了。*


## 故事

我早先在 SHell 里，是用类似于 `NS0 () ( NS1 () ( fun () { ... ; } && "$@" ) && "$@" )` 的形式来组织代码的，它会提供类似于子命令一样的效果。

我想在 TS 里也能够如此，此时我还不知道 `namespace` 这个概念，但我看上了 `structure` 字面量。因为 JSON 就是 JS 的字面量，而这玩意显然已经不仅仅被用于 JS 了，所以把它玩儿转了一定很酷。（后来通过问 Scala 里的 `object` 在 TS 里有没有等价的东西才知道了 `namespace` 。）

但是， `structure` 字面量的属性之间是无法互相依赖的。如果想要有这样的依赖，必须先构建一个字面量，再构建一个多个属性都用到了它的某处的下一个字面量。我一开始觉得这样也挺酷的，直到这并不能满足我的代码抽象体验 …… 我想要让一个结构体里有两个函数，但其中一个又用到了另一个。

然后就有了下面的故事了。

……

最初要实现的需求就是这三条：

1. 有一个工具，这个工具的参数要接受一系列匿名函数且这些匿名函数之间存在依赖关系。
2. 但是，依赖关系必须在本工具内部才建立，不能在传进来之前就建立，那样就没有意义了。
3. 要返回一个新的结构体，可以根据属性名取得结构体中的匿名函数，并且此时这些函数应当已经建立依赖关系并且能够使用。

我用具体的简洁代码向 New Bing 描述了我的需求（怕只用文字它看不懂）。它给了我一些实现，但都是要么不满足这个条件要么不满足那个，而且在某次对话里还特别喜欢给我命令式的代码、或者是直接把匿名函数定义在我的这个工具内部。

我最先问明白的是第三条。然后我专门只针对前两条来问。

最初我还想是否有必要使用元编程，好在 TS 里不能像 Rust 或者 Elixir 里那样搞宏。转机在于柯里化。这个点子是也是我自然而然就想到的 —— 不能用宏就只能这样了，我只是这样觉得。但我所想的是增加一个外部传入的函数而不是环境 `env` 。在我重复了几次要求的情况下（就是说不要这样要怎么怎么样还能做到吗之类的），意外地在某一次， New Bing 把它换成了一个 `obj` ，然后我就注意到了那就是一个存储了环境的结构体，从而明白了其实没必要显式地生成出 `const f1 = ...` 这样的代码。

接下来，虽然它给我的还是代码，但已经没关系了。我告诉它 `reduce` 应该能达到等价的效果，要求它不要写任何循环的代码。然后它为我完成了这个转换。看来等价代码之间的转换以后就不必依靠人脑了 —— 当然了这样的代码仍然需要验证，毕竟这并不是一个明确了两种代码转换逻辑的工具而是一个训练出来的神经网络。明确的逻辑我认为也是可能的，但现在的情况就是有个神经网络也比没有它要好上很多了，相当于是一个外置的帮你思考的脑子 —— 毕竟它已经被训练得能够对一些较为复杂的形式逻辑的思考活动予以模拟了。

然后这个工具的雏形也就有了。之后就是一些类型方面的问题。我原本是自己加的泛型，但很显然加错了。我也把我自己给绕晕了，类型方面。对这一点的询问最初也没有得到确切的回应，我改回我泛型化之前的版本并询问为什么这样就没有问题，它便给出了正确的解释。我又询问了为何 TS 的类型推导不能得出 `Promise` 的泛型以至于 `r` 必须写作 `(r: number)` 、以及添加怎样一些代码可以不用给 `r` 加这个标记，然后我就从它那里得到了正确的使用和标记泛型的方式（实际上是它先给了我带泛型的调用我又回馈了定义上增加泛型的必要性然后它才给了我正确的有泛型的定义）。我看调用写起来有些繁琐，问能不能搞个抽象，我只需要给出结构体的名字和里面函数的名字就能取出这个函数，然后它就给了我 `call` ，我又问能不能改箭头函数，它给了。

然后 …… 我又改了改变量名，再用 `namespace` 把它装起来，就是你们现在看到的这个项目了。

它只花了两天，然后就能够满足我最开始的需要了，没有再报错。

这放在 GPT-4 以前不知道需要多久 …… 就是 GPT-3.5 也都很可能会乱写一通的，而且 New Bing 还平添了许多限制，但仍然可以通过 *苏格拉底式问话* 引导它帮你把答案得出。而且只花了两个白天和一个晚上，且白天也并非全在这件事上投入的。虽然整个过程磕磕绊绊不算愉快，有由于给的代码太多直接拒答的情况（这其实应该是由于 Bing 会拦截不合乎要求的问话从而导致问话根本没有被交给 GPT 模型）、也有由于模型优先进行更加懒惰的思考从而会先给出一个能够带跑题的答案的情况，但加上我自己梳理思路并简化对疑问的描述，最终还是用它快速地完成了工作。
