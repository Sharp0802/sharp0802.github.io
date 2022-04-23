---
layout: post
title: "Tilemap에서 C#을 이용해 타원그리기"
date: 2022-3-1
categories: [ "C#", "Math" ]
---

### 각도 t에 따른 타원 위의 점

위키피디아에 따르면, 각도 t에서 타원 위의 점 $(X(t), Y(t))$는 다음과 같다.
$$
X(t) = X_{c} + a * cos(t) * cos(\phi) - b * sin(t) * sin(\phi)
Y(t) = Y_{c} + a * cos(t) * sin(\phi) - b * sin(t) * cos(\phi)
$$
$(X_{c}, Y_{c})$는 타원의 중점, a는 장축의 절반, b는 단축의 절반의 길이값이다.

$\phi$는 x축을 기준으로 타원이 y축 방향으로 기울어진 방향을 의미한다.

필요한건, 기울어진 타원이 아니므로, 아래의 방정식으로 간단히 만든다.
$$
X(t) = X_{c} + a * cos(t)
Y(t) = Y_{c} + b * sin(t)
$$

### x에 따른 타원 위의 점

하지만, Tilemap에 타원을 그리기 위해선, 각도가 아니라, x에 따른 y값을 알아야 한다.

$(X_{c}, Y_{c}) = (0, 0)$이고, 각도 t에서 양의 x값을 $x_{p}$라고 가정하자.

이때, $x_{p} = a * cos(t)$이다.

xp와 a가 상수이므로, $x_{p} / a = cos(t)$다.

결과적으로, $acos(\cfrac{x_{p}}{a}) = t$다.

이를 Y(t)에 대입하면, $b * sin(acos(\cfrac{x_{p}}{a}))$다.

일반화 시키면, $y = b * sin(acos(\cfrac{x}{a}))$다.

$sin(acos(\cfrac{x}{a})) = sqrt(\cfrac{a^2 - x^2}{a^2})$이므로,

$y = b * sqrt(\cfrac{a^2 - x^2}{a^2})$와 같이 간단히 한다.

양수만 반환하므로, $\pm$를 붙여, $y = \pm b * \sqrt{\cfrac{a^2 - x^2}{a^2}}$로 쓴다.

### 구현

중점 (0,0)을 가지며 너비가 `float w`, 높이가 `float h`인 타원에 대하여,
x값이 `float x`일때, y값 두개를 Vector2로 반환한다.

```c#
public static Vector2 GetYOnEllipseByX(float x, float w, float h)
{
    var a = MathF.Max(w, h);
    var b = MathF.Min(w, h);

    var y = b * MathF.Sqrt((a * a - x * x) / (a * a));

    return new Vector2(y, -y);
}
```

보통 Tilemap은 int,int의 좌표를 갖는것이 일반적이므로

`MathF.Round(float) : float`와 형 변환 (e.g. `(int) MathF.Round(a)`) 혹은, 
`Mathf.RoundToInt(float) : int`를 고려하자.

Unity 엔진의 Tilemap을 이용한다면, 구현은 다음과 같을것이다
```c#
public static Vector2Int GetYOnEllipseByX(int x, float w, float h)
{
    var a = Mathf.Max(w, h);
    var b = Mathf.Min(w, h);

    var y = Mathf.RoundToInt(b * Mathf.Sqrt((a * a - x * x) / (a * a)));

    return new Vector2Int(y, -y);
}
```

중점이 (p,q)인 타원을 원한다면,

위 함수는 중점이 (0,0)이므로, 
x - p를 `float x`로 정하고, 
반환된 y에 각각 q를 더하면 될것이다

위 사항을 고려하는 구현은 다음과 같다.

```c#
public static Vector2Int GetYOnEllipseByXWithOffset(Vector2Int offset, int x, float w, float h)
{
    x -= offset.x;

    var a = Mathf.Max(w, h);
    var b = Mathf.Min(w, h);

    var y = Mathf.RoundToInt(b * Mathf.Sqrt((a * a - x * x) / (a * a)));

    return new Vector2Int(y, -y) + Vector2Int.one * offset.y;
}
```