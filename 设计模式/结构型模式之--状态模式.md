# 适用场景

- 如果代码中出现大量与对象状态有关的条件语句，并且这些语句会导致代码的可维护性和灵活性变差。不能方便的增加和删除状态。（主要适用场景）
- 如果对象的行为依赖于他的状态，状态的改变将导致行为的变化。（这一点跟策略模式很像）

**例如：搜狗输入法的输入状态切换 ， 自动售货机的工作状态切换**

> 状态模式和策略模式的UML类图和实现方式都很接近，但是这两个模式的差别就在于他们的“意图”。
>
> 举个例子，在狗血剧中，同样是KillPeople，有时候就是为了报仇，有时候就是为了解除队友的痛苦。
>
> 对于状态模式而言，context的行为随时可以委托到那些状态对象中的一个，当前状态在状态对象集合中游走改变，因此，context的行为也会跟着改变，但是注意，context的客户对于状态对象了解不多，甚至根本是浑然不觉，这是区别于策略模式的主要因素之一。
>
> 还不懂？那我们再直白一点，状态模式关注的是复杂的状态判断和状态之间的切换，用户并不会直接对状态进行切换。而策略模式是由用户设置不同的策略，而直接影响对象的行为。

# 定义

(源于Design Pattern)：当一个对象的内在状态改变时允许改变其行为，这个对象看起来像是改变了其类。

状态模式主要解决的是当控制一个对象状态的条件表达式过于复杂时的情况。把状态的判断逻辑转移到表示不同状态的一系列类中，可以把复杂的判断逻辑简化。

# 模式角色

- Context 上下文: 维护一个ConcreteState实例，这就是他当前的状态。
- State 状态接口: 每个状态都要实现它，它定义了所有可能发生的状态。
- ConcreteState具体状态类: 实现State接口，个性化需要被实现的方法。



# Demo

> 搜狗输入法在按下shift的时候可以在中文和英文之间切换，而按下Lock键可以在大小写之间切换，那么我们就用状态模式来实现这个场景。

- InputState  状态接口
- 中文输入法  状态实现类
- 英文输入法  状态实现类
- 大写输入法  状态实现类
- SouGou  Context上下文
- Client  用户

**State**

```java
/**
 *
 * 输入状态的接口
 * Created by YY on 2016/11/16.
 */

public interface InputState {

    void downShift();

    void downLock();

    void print();

}
```

**中文输入法**

```java
/**
 * 中文状态
 * Created by YY on 2016/11/16.
 */

public class ChinaInputState implements InputState{

    SouGou mSouGou;

    public ChinaInputState(SouGou sougou){
        mSouGou = sougou;
    }

    @Override
    public void downShift() {
        mSouGou.setState(mSouGou.getLetterInputState());
    }

    @Override
    public void downLock() {
        mSouGou.setState(mSouGou.getCapitalInputState());
    }

    @Override
    public void print() {
        System.out.println("中文状态");
    }

}
```

**英文输入法(小写)**

```java
/**
 * 英文状态
 * Created by YY on 2016/11/16.
 */

public class LetterInputState implements InputState{

    SouGou mSouGou;

    public LetterInputState(SouGou sougou){
        mSouGou = sougou;
    }

    @Override
    public void downShift() {
        mSouGou.setState(mSouGou.getChinaInputState());
    }

    @Override
    public void downLock() {
        mSouGou.setState(mSouGou.getCapitalInputState());
    }

    @Override
    public void print() {
        System.out.println("小写英文状态");
    }
}
```

**英文输入法(大写)**

```java
/**
 * 大写状态
 * Created by YY on 2016/11/16.
 */

public class CapitalInputState implements InputState{

    SouGou mSouGou;

    public CapitalInputState(SouGou sougou){
        mSouGou = sougou;
    }

    @Override
    public void downShift() {
        // 不做任何操作
    }

    @Override
    public void downLock() {
        mSouGou.setState(mSouGou.getLetterInputState());
    }

    @Override
    public void print() {
        System.out.println("大写英文状态");
    }
}
```

**搜狗输入法**

```java
/**
 * 搜狗输入法
 * Created by YY on 2016/11/16.
 */
public class SouGou{

    private InputState mState;

    private ChinaInputState chinaInputState;
    private LetterInputState letterInputState;
    private CapitalInputState capitalInputState;

    public SouGou(){
        chinaInputState = new ChinaInputState(this);
        letterInputState = new LetterInputState(this);
        capitalInputState = new CapitalInputState(this);
      	// 默认为中文输入状态
        mState = chinaInputState;
    }

    public void downShift(){
        mState.downShift();
    }

    public void downLock(){
        mState.downLock();
    }

    public void print(){
        mState.print();
    }

    public ChinaInputState getChinaInputState() {
        return chinaInputState;
    }

    public void setChinaInputState(ChinaInputState chinaInputState) {
        this.chinaInputState = chinaInputState;
    }

    public LetterInputState getLetterInputState() {
        return letterInputState;
    }

    public void setLetterInputState(LetterInputState letterInputState) {
        this.letterInputState = letterInputState;
    }

    public CapitalInputState getCapitalInputState() {
        return capitalInputState;
    }

    public void setCapitalInputState(CapitalInputState capitalInputState) {
        this.capitalInputState = capitalInputState;
    }

    public void setState(InputState state){
        mState = state;
    }

    public InputState getState(){
        return mState;
    }
}
```

**用户**

```java
/**
 * 模拟用户按下按键时候的操作
 * Created by L on 2016/11/17.
 */
public class Client {

    public void main(){

        SouGou souGou = new SouGou();

        souGou.downShift();
        souGou.print();

        souGou.downShift();
        souGou.print();

        souGou.downShift();
        souGou.print();

        souGou.downLock();
        souGou.print();

        souGou.downLock();
        souGou.print();
    }

}
```

**输出**

```
 小写英文状态
 中文状态
 小写英文状态
 大写英文状态
 小写英文状态
```



到这里，我的20W广告费已经到账了，不……是我们已经完成整改搜狗输入法的状态切换了，他能够轻松的再任何状态下都做出正确的动作，可是别高兴的太早。



> 搜狗的产品经理在某天晚上失恋了，然后他在极度郁闷的情况下召开了会议，会议中他将需求改成了如下方案：
>
> 按Lock键切换到大写字母，再次按下Lock键依旧要回到之前的状态。同时在大写状态的时候按shift键也可以切换到小写字母。
>
> 例如当前是中文状态，那么按下两次Lock键依旧要回到中文状态。
>
> 他以为可以让一群人跟着他伤心，但他不知道我们已经用了状态模式来写这个模块，只要稍加改动就能完成它的需求改动。

我们只需要在SouGou这个上下文中增加一个State变量，在除了大写状态之外的状态实现类中，来记录按下Lock键之前的状态。下面我把改动后的代码放上来，除了类名上面的段落注释，代码内的段落注释就是我们改动的地方。

**英文状态（大写）**

```java
/**
 * 大写状态
 * Created by YY on 2016/11/16.
 */

public class CapitalInputState implements InputState{

    SouGou mSouGou;

    public CapitalInputState(SouGou sougou){
        mSouGou = sougou;
    }

    @Override
    public void downShift() {
        /**
         *  之前是不做任何操作
         */
        mSouGou.setState(mSouGou.getLetterInputState());
    }

    @Override
    public void downLock() {
        //mSouGou.setState(mSouGou.getLetterInputState());
        /**
         * 这里注释掉之前的代码 改为下面这行
         */
        mSouGou.setState(mSouGou.getLastState());
    }

    @Override
    public void print() {
        System.out.println("大写英文状态");
    }
}
```

**英文状态（小写）**

```java
/**
 * 英文状态
 * Created by YY on 2016/11/16.
 */

public class LetterInputState implements InputState{

    SouGou mSouGou;

    public LetterInputState(SouGou sougou){
        mSouGou = sougou;
    }

    @Override
    public void downShift() {
        mSouGou.setState(mSouGou.getChinaInputState());
    }

    @Override
    public void downLock() {
        mSouGou.setState(mSouGou.getCapitalInputState());
        /**
         * 在每次按Lock键的时候记录当前的状态
         */
        mSouGou.setLastState(this);
    }

    @Override
    public void print() {
        System.out.println("小写英文状态");
    }
}
```

**中文状态**

```java
/**
 * 中文状态
 * Created by YY on 2016/11/16.
 */

public class ChinaInputState implements InputState{

    SouGou mSouGou;

    public ChinaInputState(SouGou sougou){
        mSouGou = sougou;
    }

    @Override
    public void downShift() {
        mSouGou.setState(mSouGou.getLetterInputState());
    }

    @Override
    public void downLock() {
        mSouGou.setState(mSouGou.getCapitalInputState());
        /**
         * 在每次按Lock键的时候记录当前的状态
         */
        mSouGou.setLastState(this);
    }

    @Override
    public void print() {
        System.out.println("中文状态");
    }

}
```

**搜狗输入法**

```java
/**
 * 搜狗输入法
 * Created by YY on 2016/11/16.
 */
public class SouGou{

    private InputState mState;

    private ChinaInputState chinaInputState;
    private LetterInputState letterInputState;
    private CapitalInputState capitalInputState;

    /**
     *   这个状态用于存放之前的状态
     */
    InputState mLastState;

    public SouGou(){
        chinaInputState = new ChinaInputState(this);
        letterInputState = new LetterInputState(this);
        capitalInputState = new CapitalInputState(this);
        mState = chinaInputState;
    }

    public void downShift(){
        mState.downShift();
    }

    public void downLock(){
        mState.downLock();
    }

    public void print(){
        mState.print();
    }

    /**
     * 为了阅读方便，我们把之前的get set方法隐藏于此
     */

    /**
     * 这里提供了get set方法
     */

    public InputState getLastState() {
        return mLastState;
    }

    public void setLastState(InputState mLastState) {
        this.mLastState = mLastState;
    }
}
```

甚至比想象的还要简单，接着我们来测试一下：

```java
/**
 * 模拟用户按下按键时候的操作
 * Created by L on 2016/11/17.
 */
public class Client {


    public void main(){

        SouGou souGou = new SouGou();
        souGou.print();

        souGou.downLock();
        souGou.print();

        souGou.downLock();
        souGou.print();

        souGou.downShift();
        souGou.print();

        souGou.downLock();
        souGou.print();

        souGou.downLock();
        souGou.print();

        souGou.downLock();
        souGou.print();

        souGou.downShift();
        souGou.print();
    }

}
```

我的按键顺序是这样的(默认起始状态是中文)：

- Lock 大写
- Lock 中文
- Shift 英文
- Lock 大写
- Lock 英文
- Lock 大写
- Shift 英文

这是我们的期望结果，让我们看看是不是这样：

```
大写英文状态
中文状态
小写英文状态
大写英文状态
小写英文状态
大写英文状态
小写英文状态
```

如果你安装了搜狗输入法，那么可以试着按一下。

# 总结

状态模式是为了解决复杂的状态结构，其优点是将复杂的状态判断条件拆分成了对象，各个状态各扫门前雪，似的扩展更加方便。

但是如果你细心的话会发现，他并没有很好的实现开闭原则，我们改变需求的同时几乎要对每个文件的代码都进行改动，并且如果状态更多会产生更多的文件，这也是该模式的缺点。

但还是要记住，只要不滥用，每个模式都是好模式。