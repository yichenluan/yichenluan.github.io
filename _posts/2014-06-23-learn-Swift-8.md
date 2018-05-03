---
title:  Swift 学习笔记 「枚举」
layout: post
tags: [Program]
---

###枚举语法

例：

```
enum CompassPoint {
  case North
  case South
  case East
  case West
}
```

case 关键字表明将定义新的一行成员值，与 C 语言中枚举不一样的是，Swift 枚举成员值不会被赋予一个默认的整数值。

```
var directionToHead = CompassPoint.West
```

###匹配枚举值和 switch 语句

```
directionToHead = .South
switch directionToHead {
case .North:
    println("Lots of planets have a north")
case .South:
    println("Watch out for penguins")
case .East:
    println("Where the sun rises")
case .West:
    println("Where the skies are blue")
}
```

###相关值

```
enum Barcode {
  case UPCA(Int, Int, Int)
  case QRCode(String)
}

var productBarcode = Barcode.UPCA(8, 85909_51226, 3)
productBarcode = .QRCode("ABCDEFGHIJKLMNOP")
```

###原始值

```
enum ASCIIControlCharacter: Character {
    case Tab = "\t"
    case LineFeed = "\n"
    case CarriageReturn = "\r"
}
```

```
enum Planet: Int {
    case Mercury = 1, Venus, Earth, Mars, Jupiter, Saturn, Uranus, Neptune
}
```

原始值的递增，使得 Venus 等于 2

```
let earthsOrder = Planet.Earth.toRaw()
// earthsOrder is 3

let possiblePlanet = Planet.fromRaw(7)
// possiblePlanet is of type Planet? and equals Planet.Uranus
```
获取原始值，通过原始值找到元素

---
END
