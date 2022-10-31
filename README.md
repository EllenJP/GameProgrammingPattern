# 目次
* [単一責任の原則](#単一責任の原則)
* [オープンクローズドの原則](#オープンクローズドの原則)
* [リスコフの置換原則](#リスコフの置換原則)
* [インターフェースの原則](#インターフェースの原則)
* [依存性逆転の原則](#依存性逆転の原則)
* [Factory](#Factory)
* [ObjectPool](#ObjectPool)
* [Singleton](#Singleton)
* [Command](#Command)
* [State](#State)
* [Observer](#Observer)
* [MVP](#MVP)

<br><br><br>

# 概要
ここでは、ゲームプログラミングパターンについて紹介します。

ある程度の規模のゲームを作ったことがある人なら分かると思いますが、はじめから仕様が全て決まっていることは、ほとんどありません。

むしろ長く遊んでもらうためには、新機能の追加が必要になります。

何度も修正が必要になるにも関わらず、何も考えずにコーディングをしていれば「この機能ってどこに処理が書かれてたっけ...」、「このメソッドって勝手に変更していいのかな？」と...毎回修正箇所を探したり、修正した影響でバグが発生していないか確認したりと、時間がかかります。

コードを読みなれている人なら、ある程度短縮できると思いますが、あまり読みなれていない方にとっては、時間もかかり、また新たなバグが発生しないかどうか、恐る恐る修正しなければなりません...

当然、毎回こんなことをしていれば、ストレスでモチベーションが下がります。それを防ぐために、修正がしやすくバグが発生しにくい設計の仕方を学ぶ必要があります。

前置きが長くなりましたが、本記事では、ゲームプログラミングで利用できそうな設計パターンや原則を簡単なコードを交えながら説明していきます。
参考コードはUnityさんが公開している[リポジトリ](https://github.com/Habrador/Unity-Programming-Patterns)を加筆修正等を行い利用させていただいています。
その為、実行環境はUnity(C#)です。

**※これが必ず正解というわけではないので、あくまで参考程度に考えてください。**

<br><br><br>

# 単一責任の原則<a id="単一責任の原則"></a>
***1つのクラスに1つの機能を持たせる原則***のことです。

これだけ聞くと、「たくさん機能を持ったクラスを作らずに、そのクラスの役割は一つにすることね、完全に理解した」と、私のように単一責任の原則を終えてしまう方もいるかもしれません。

ですが、「なぜ単一責任の原則を使うのか？そのメリットは？」、「1つのクラスに1つの役割とは具体的にどうやって判断するのか？」など実際にコードに組み込もうとすると理解が曖昧なところが出てきます。

## 原則違反の例
ここではUnrefactoredPlayerクラスを例とします。

以下のUnrefactoredPlayerクラスは次の仕様になっています。
* プレイヤーは← →キーで入力を行い移動することができる。(移動範囲あり)
* プレイヤーがオブジェクトに接触したら接触音がなる

```cs
public class UnrefactoredPlayer : MonoBehaviour
{
    private string _inputAxisName = "Horizontal";
    private float _xPosition;
    private float _speed = 4f;
    private AudioSource _bounceSfx;

    private void Start()
    {
        _bounceSfx = GetComponent<AudioSource>();
    }

    private void Update()
    {
        float deltaAxisValue = Input.GetAxis(_inputAxisName) * Time.deltaTime;

        _xPosition = Mathf.Clamp(_xPosition + deltaAxisValue * _speed, -5, 5);

        transform.position = new Vector3(_xPosition, transform.position.y, transform.position.z);
    }

    private void OnTriggerEnter(Collider other)
    {
        _bounceSfx.Play();
    }
}

```
UnrefactoredPlayerクラスには入力、移動、オーディオの3つに関するパラメータやロジックが含まれています。

まず、UnrefactoredPlayerクラスが持っている責任を洗い出します。
* 入力に関する責任
* 移動に関する責任
* オーディオに関する責任
* 上3つをまとめる責任

パッと分けましたが、どう責任範囲を決めればよいかは難しいところです。

一見、機能(メソッド)＝責任と考えてもうまくいきそうですが、その後仕様追加される内容を考えてみると責任範囲が決めやすいと思います。

例えば、もし入力と移動を一つの責任として考えていた場合、「矢印キーでなくてゲームパッドでも操作できるようにしたい」と仕様が追加されたら、入力方法は変わりますが、移動ロジックには手を加える必要がありません。そのため、入力と移動は違う役割として別々にクラスを作ろうと考えることができます。

以上をもとに、それぞれの責任でクラスを作成します。
* 入力に関する責任: PlayerInput
* 移動に関する責任: PlayerMovement 
* オーディオに関する責任: PlayerAudio
* 上3つをまとめる責任: Player(Playerクラスは他3つをまとめるだけで、各処理は他のクラスに任せています。)

```cs
public class PlayerInput : MonoBehaviour
{
    private string inputAxisName = "Horizontal";

    public float InputValue()
    {
        return Input.GetAxis(inputAxisName) * Time.deltaTime;
    }
}
```

```cs
public class PlayerMovement : MonoBehaviour
{
    private float _xPosition;
    private float _speed = 4f;

    public void Move(float deltaAxisValue)
    {
        _xPosition = Mathf.Clamp(_xPosition + deltaAxisValue * _speed, -5, 5);
        transform.position = new Vector3(_xPosition, transform.position.y, transform.position.z);
    }
}
```

```cs
public class PlayerAudio : MonoBehaviour
{
    private AudioSource bounceSfx;

    private void Start()
    {
        bounceSfx = GetComponent<AudioSource>();
        Debug.Log(bounceSfx);
    }

    public void PlaySound()
    {
        bounceSfx.Play();
    }
}
```


```cs
public class Player : MonoBehaviour
{
    [SerializeField] private PlayerAudio _playerAudio;
    [SerializeField] private PlayerInput _playerInput;
    [SerializeField] private PlayerMovement _playerMovement;

    private void Start()
    {
        _playerAudio = GetComponent<PlayerAudio>();
        _playerInput = GetComponent<PlayerInput>();
        _playerMovement = GetComponent<PlayerMovement>();
    }

    private void Update()
    {
        var deltaValue = _playerInput.InputValue();
        _playerMovement.Move(deltaValue);
    }

    private void OnTriggerEnter(Collider other)
    {
        Debug.Log("hello");
        _playerAudio.PlaySound();
    }
}
```

各責任ごとにクラスを分けたことで、もし入力ロジックを変更したい場合でも、PlayerInputクラスを見ればすぐに原因箇所がわかります。また、関係のないオーディオや移動ロジックをクライアント側から隠せるので、間違えて修正するリスクが減ります。

<br><br>

## 考察
### ***とりあえずGameManagerを作ればいいですか？***
普段何気なく作っているGameManagerクラスはゲーム全体を管理する責任を持っています。そのため、画面遷移、Player、Enemy、バトルロジック等全てを書こうと思えば書けてしまいます。その結果コードが肥大化します。

なので、仕様変更時にどの範囲まで責任があるのか考えることで、他の責任と切り離してクラスにまとめることで、すっきりするのかなと。

例えば画面遷移の例を挙げると、ホーム画面、ガチャ画面、クエスト画面などの各画面ごとにManagerクラスを用意することで、修正範囲をその画面のみにすることができます。それらのManagerクラスをまとめるクラスを用意してあげると画面遷移のみを管理することができるので、新しい画面を追加する時も、新規画面のManagerクラスと画面遷移を管理しているクラスに追記することで追加がしやすいと思います。


<br><br>

## 参考資料
* https://qiita.com/MinoDriven/items/76307b1b066467cbfd6a
    * 単一責任の原則については、こちらの記事が大変わかりやすかったです。記事で紹介されていたObjectValueパターンについては後日まとめます。

<br><br><br>

# オープンクローズドの原則<a id="オープンクローズドの原則"></a>
***機能拡張がしやすく、既にある機能には手を加えないようにする原則*** です。

つまり、簡単に後で新しい機能を追加することができて、既存のメソッドを修正する必要がありません。

もし、既存のメソッドを修正してしまうと、メソッド内の他の処理にも影響がないかどうか、テストしなければならず余計な時間がかかってしまいます。。

まずは原則が守られているプログラムで説明します。
以下のプログラムは、四角形、円のそれぞれの面積を求めるプログラムです。
```cs
public abstract class Shape
{
    public abstract float CalculateArea();
}
```
```cs
public class Rectangle : Shape
{
    public float Width { get; set; }
    public float Height { get; set; }

    public override float CalculateArea()
    {
        return Width * Height;
    }
}
```
```cs
public class Circle : Shape
{
    public float Radius { get; set; }

    public override float CalculateArea()
    {
        return Radius * Radius * Mathf.PI;
    }
}
```
```cs
public class AreaCalculator 
{
    public float GetArea(Shape shape)
    {
        return shape.CalculateArea();
    }
}
```
まず抽象クラスであるShapeクラスをRectangle, Circleが継承することで、下記のように複数の図形を処理する場合でも抽象クラスのメソッドで処理を行うことができます。

これにより、新たに三角形の面積を求める処理を追加しても、同じメソッド(GetArea)で呼び出すことができます。

また、呼び出し側に各図形の面積を求めるロジックを見せないため、不必要にロジックが変更されるリスクを減らせます。
```cs
public class OpenClosedSample : MonoBehaviour
{
    void Start()
    {
        List<Shape> shapeList = new List<Shape>() { new Rectangle(), new Circle() };
        AreaCalculator areaCalculator = new AreaCalculator();
        foreach (var item in shapeList)
        {
            // 異なる図形も同一に扱える
            Debug.Log(areaCalculator.GetArea(item));
        }
    }
}
```

AreaCalculatorクラスに少し触れると、このクラスはクライアント側(呼び出し側)に各図形の面積を返す責任を持っています。
```cs
public class OpenClosedSample : MonoBehaviour
{
    void Start()
    {
        List<Shape> shapeList = new List<Shape>() { new Rectangle(), new Circle() };
        foreach (var item in shapeList)
        {
            Debug.Log(item.CalculateArea(item));
        }
    }
}
```
今回はAreasCalculatorクラスから呼び出すメリットがあまり感じられません。直接各図形クラスの面積メソッドを呼び出すこともできるので...
ただ各図形クラスのCalculateArea()をラップしただけでは、クライアント側がどちらを使えばよいか迷ってしまいます。おそらく、**全ての図形クラスで共通の処理を行いたい場合** に効果を発揮すると思います。

例えば、「全ての図形の面積を2倍にする」といった場合です。
```cs
public class AreaCalculator 
{
    public float GetArea(Shape shape)
    {
        return shape.CalculateArea() * 2f;
    }
}
```

もちろん、クライアント側(呼び出し側)で返ってきた値に対して2倍することもできますが、倍率を変更したい場合に呼び出し側が複数あると変更箇所が増えてしまいます。

また、以下のように各図形クラスで書かれていた面積を求めるロジックを、AreaCalculatorクラスにもってくることもできます。

こんな感じで
```cs
public class AreaCalculator 
{
    public float GetArea(Shape shape)
    {
        float area = 0f;
        if (shape.GetType() == typeof(Rectangle))
        {
            // アップキャスト
            Rectangle rectangle = shape as Rectangle;
            area = rectangle.Width * rectangle.Height;
        }
        else if (shape.GetType() == typeof(Circle))
        {
            Circle circle = shape as Circle;
            area = circle.Radius * circle.Radius * Mathf.PI;
        }
        return area;
    }
}
```

簡単に説明すると、サブクラスの型で条件分岐して、アップキャストを用いて各パラメータを元に面積を求めます。

見た目も複雑ですが、もし三角形の面積を求めるロジックを追加する場合、三角形クラスを作成し、AreaCalculatorクラスのGetArea()に新たに条件分岐を追加する必要があります。

ここでオープンクローズドの原則 **機能拡張がしやすく、既にある機能には手を加えないようにする**を思い出してください。

機能拡張は今回だと、まだ簡単にできますが、既にある機能(GetArea())に対しては手を加えてしまっています。もし複雑な処理の場合追加機能だけでなく既存の機能についてもテストを行わなければなりません。

これを防ぐために各図形の面積ロジックは各自で持つようにしています。

<br><br>

## 考察
### ***抽象クラス(メソッド)、インターフェース、仮想メソッドどれを使う？***
今回の例では抽象クラスShapeを継承して各図形クラスを作成しました。同様の処理をインターフェース、仮想メソッドを用いて書くこともできます。

まずはインターフェースでの実装です。

```cs
public class OpenClosedSample : MonoBehaviour
{
    void Start()
    {
        var circle = new Circle();
        var calculator = new AreaCalculator();
        // クライアント側からの呼び出し
        Debug.Log(calculator.GetArea(circle));
    }
}

public interface IShape
{
    float CalculateArea();
}

public class Circle : IShape
{
    public float Radius { get; set; } = 2f;
    // インターフェース実装
    public float CalculateArea()
    {
       return Radius * Radius * Mathf.PI;
    }
}

public class AreaCalculator
{
    // 引数の型がIShpaeになっていることに注意
    public float GetArea(IShape shape)
    {
        return shape.CalculateArea();
    }
}
```

インターフェースで実装することで、メソッドのオーバーライドを強制します。またインターフェースは複数実装することができることもポイントです。

続いて抽象クラスです。クライアント側（呼び出し側）のOpenClosedSampleクラスは変更がないため省略します。

```cs
public abstract class Shape
{
    public abstract float CalculateArea();
}

public class Circle : Shape
{
    public float Radius { get; set; } = 2f;
    // 抽象メソッドをオーバーライド
    public override float CalculateArea()
    {
        return Radius * Radius * Mathf.PI;
    }
}

public class AreaCalculator
{
    public float GetArea(Shape shape)
    {
        return shape.CalculateArea();
    }
}
```
先ほどのインターフェースと比べてメソッドのオーバーライドを強制することは同じですが、多重継承することができません。これだけだとインターフェースの方が使い勝手が良さそうです。

抽象クラスは全体に前処理や後処理を行いたい時に使うのが良いかと思います。「全ての図形で面積を2倍にする」みたいな処理を書きたいときに使うのが良いかと。
こんな感じです。
```cs
public abstract class Shape
{
    // 公開
    public float CalculateArea()
    {
        return SubCalculateArea() * 2f;
    }
    // 抽象メソッド
    private abstract float SubCalculateArea();
}
```

ただ、全体処理を行う場合はAreaCalculatorクラスのような計算用のクラスを作る方が良いかなと思っています。抽象クラス内に他の抽象メソッドもあるので、複雑なロジックを書くと抽象クラス自体が肥大化してしまうからです。

最後に仮想メソッドです。
```cs
// 具象クラスでもエラーは発生しない
public abstract class Shape
{
    // 仮想メソッドは実装が必要。
    public virtual float CalculateArea()
    {
        return 1f;
    }
}

public class Circle : Shape
{
    public float Radius { get; set; } = 2f;
    // オーバーライドが必要
    public override float CalculateArea()
    {
        // 親クラスのメソッドを呼びたいときは
        // return base.CalculateArea();
        return Radius * Radius * Mathf.PI;
    }
}

public class AreaCalculator
{
    public float GetArea(Shape shape)
    {
        return shape.CalculateArea();
    }
}
```
仮想メソッドは正直使うメリットがわかりませんでした。
抽象クラスと同じようにサブクラスでオーバーライドすることができますが、強制ではありません。そのため、実装忘れをしてしまう恐れがあります。

また、仮想メソッドは抽象クラス、具象クラスどちらでも定義することができます。少し話が逸れますが、抽象クラス、具象クラス両方で書ける場合は、抽象クラスにするのが良いみたいです。

以前ロジックのみのクラスを具象クラスとして継承したことがありましたが、親クラスとサブクラスに処理が分かれてしまい、可読性が下がってしまった経験があります。具象クラスを継承する例では、フッターやヘッダーなどの画面レイアウトを継承する場合に使うのが良いらしいです。

まとめると、基本はインターフェースで実装をしてみて、全体処理が必要な場合は適宜抽象クラスを用いる感じがよさそうです。


<br><br>

# リスコフの置換原則<a id="リスコフの置換原則"></a>




# インターフェースの原則<a id="インターフェースの原則"></a>
# 依存性逆転の原則 <a id="依存性逆転の原則"></a>
# Factory<a id="Factory"></a>
# Object Pool<a id="Object Pool"></a>
# Singleton<a id="Singleton"></a>
# Command<a id="Command"></a>
# State<a id="State"></a>
# Observer<a id="Observer"></a>
# MVP<a id="MVP"></a>
